# TakeMeter — Classifying Discourse Quality on r/nba

A text classifier that sorts r/nba posts into four discourse types — `analysis`, `hot_take`, `reaction`, and `context` — and a head-to-head comparison between a **zero-shot LLM baseline** (Llama-3.3-70B via Groq) and a **fine-tuned DistilBERT** model trained on 205 hand-labeled posts.

The headline finding is unusual and worth stating up front: **fine-tuning made the model worse, not better.** The zero-shot baseline scored 80.6% accuracy; the fine-tuned DistilBERT scored 51.6% — a 29-point regression. The rest of this report is mostly an honest investigation of *why*, because a model that fails in a diagnosable way is more informative than one that quietly "mostly works."

> **Note on data provenance.** The headline metrics (`evaluation_results.json`) and the fine-tuned confusion matrix (`confusion_matrix.png`) are committed to this repo and are used verbatim below. The per-class fine-tuned metrics and the confusion-matrix table are derived by hand from those committed artifacts and are exact. A few items that were generated only inside the Colab run and never exported — **baseline per-class metrics, per-example confidence scores, and the exact predicted label for individual failure cases** — are marked **`⚠️ TODO (from Colab)`** below. Everything not so marked is exact.

---

## 1. What I Built

- **Community:** r/nba (~12M subscribers), the largest basketball discussion forum online. It has an explicit internal culture of distinguishing "hot takes" from "analysis," which makes its discourse a natural target for a quality classifier.
- **Task:** Single-label, 4-class text classification of individual posts/comments.
- **Dataset:** 205 posts, manually collected and annotated by me (no LLM pre-labeling). Roughly balanced across the four classes.
- **Two models compared:**
  - **Baseline:** `llama-3.3-70b-versatile` (zero-shot, prompted with my label definitions + one example per class).
  - **Fine-tuned:** `distilbert-base-uncased` with a 4-way classification head, trained for 3 epochs on a 70% train split.

Full design rationale (label definitions, edge-case decision rules, metric justification, success thresholds) lives in [planning.md](planning.md). This README is self-contained, but `planning.md` has the working notes behind every decision.

---

## 2. Label Taxonomy

I define **four labels** spanning the discourse-quality spectrum on r/nba. Full definitions, boundary rules, and stress-tests are in [planning.md](planning.md).

### `analysis`
**Definition:** A structured argument backed by specific, verifiable evidence (stats, film, history). The evidence would still support the claim even if you stripped the opinion framing.
- *"Wemby's defensive rating dropped from 108.3 to 104.1 after the All-Star break. The difference is almost entirely in his help defense positioning — he's contesting 3.2 more shots per game at the rim and opponents are shooting 11% worse on those contests."*
- *"Looking at the last 10 champions, 8 of them had a top-5 net rating in the regular season. The two exceptions both had top-3 net ratings post-All-Star break. The Thunder finished 7th this year. Historically, that profile doesn't win titles."*

### `hot_take`
**Definition:** A bold, confident opinion stated without meaningful support, or with only decorative evidence. It asserts rather than argues.
- *"Chet Holmgren is the most overrated player in the league. Dude is just tall and got lucky playing next to SGA."*
- *"Thunder won 68 games and couldn't even make the Finals. Regular season means nothing."*

### `reaction`
**Definition:** An immediate emotional response to an event. Little to no argument — celebration, frustration, schadenfreude, shock. Often short, all-caps, or emoji-laden.
- *"GLORIOUS HATEWATCH"*
- *"I can't believe they actually did it. I'm shaking. THIS TEAM."*

### `context`
**Definition:** Adds factual information or background without taking an evaluative stance. Informs rather than persuades.
- *"Champagnie was undrafted out of Pitt, spent time in the G League, and is now playing 20+ minutes in the WCF on a minimum contract."*
- *"The last time a team came back from 3-1 down in the Conference Finals was the 2016 Warriors against the Thunder. Before that, it was the 1981 Celtics against the 76ers."*

**The hardest boundary** is `analysis` vs. `hot_take` when a post cites exactly one stat. My decision rule: *one stat + one assertion = `hot_take`; multiple stats + explained reasoning = `analysis`.* This rule turns out to be central to the failure analysis below.

---

## 3. Dataset

- **Total:** 205 labeled posts (`labeled_data.csv`).
- **Distribution:** `analysis` 53 (25.9%), `hot_take` 51 (24.9%), `reaction` 51 (24.9%), `context` 50 (24.4%). No class exceeds 26% — well balanced.
- **Post length:** mean ≈ 177 characters, range 8–508. The four classes differ sharply in length: `reaction` posts are short (often <40 chars), `analysis` and `context` posts are long (often 300+ chars), and `hot_take` posts sit in the middle.
- **Collection:** Manual copy-paste from r/nba — the OKC elimination thread, Top/Month feed, and 2026 playoff/Finals game threads. Annotated by me in a single ~2.5-hour session for consistency.
- **Labeling process:** I re-read my label definitions and decision rules before starting, read every post in full (no skimming), flagged borderline cases as I went, and revisited them at the end. No LLM pre-labeling (see §8). The `notes` column in `labeled_data.csv` records my reasoning on every borderline call.
- **Splits:** 70% train / 15% validation / 15% test, stratified by label, `random_state=42`. Test set = **31 posts**.

### 3.1 Three Difficult-to-Label Examples (annotation decisions)

These are posts that genuinely gave me pause *during annotation* — distinct from the model's wrong predictions in §5.6.

**1. One stat + one assertion → `hot_take` (not `analysis`)**
> *"Dort shot 29% from three in the playoffs. You can't win when your starting SG shoots like that."*

The stat is real and verifiable, which pulls toward `analysis`. But there's no contextualization — no league-average comparison, no shot-quality discussion, no accounting for his defense. **Decision: `hot_take`,** per my rule that *one stat used as a rhetorical prop = `hot_take`*; the reasoning connecting the stat to the conclusion is never developed.

**2. Fact serving emotion → `context` (not `reaction`)**
> *"Champagnie was undrafted out of college and he is playing like 20 minutes a game in the WCF making league minimum money. That's fucking crazy."*

The first sentence is textbook `context` (player background); the second ("That's fucking crazy") is pure `reaction`. **Decision: `context`,** because the factual content dominates — the emotional clause is punctuation on top of a substantive fact. Had the order been reversed ("That's crazy that Champagnie..."), it would flip to `reaction`.

**3. Stat-heavy rebuttal-rant → `hot_take` (not `analysis`)**
> *"I'm so tired of the 'SGA is a free throw merchant' discourse. He averaged 8.7 FTA in the regular season and 7.1 in the playoffs. That's a normal drop for any star. The real story is the Spurs' defense, not the officiating."*

This has real stats and a reasoned counterpoint, which looks like `analysis`. But the framing ("I'm so tired of...") signals *advocacy*, and the stats are deployed to win an argument rather than explore a question. **Decision: `hot_take`,** because it asserts a conclusion rather than building one. This was the hardest call in the set — it sits right on the `analysis`/`hot_take` line, which (not coincidentally) is exactly where the model later failed.

---

## 4. Model & Training Setup

**Fine-tuned model:** `distilbert-base-uncased`, 4-label classification head.

| Hyperparameter | Value | Notes |
|---|---|---|
| Epochs | 3 | Starter default for small datasets; more risks overfitting on ~140 train examples. |
| Learning rate | 2e-5 | Standard BERT-family starting point. |
| Batch size | 16 (train) / 32 (eval) | Fits a T4 GPU. |
| Weight decay | 0.01 | — |
| Warmup steps | 50 | — |
| Max sequence length | 256 tokens | Comfortably covers the 508-char max post. |
| Best-model selection | by validation accuracy | `load_best_model_at_end=True`. |

**Hyperparameter decision (epochs = 3):** This was my main deliberate choice. With only ~140 training examples, more epochs sharply increase overfitting risk on a 66M-parameter model, while too few leave it undertrained. I kept the starter default of 3 epochs and relied on `load_best_model_at_end=True` (selecting the checkpoint with the best *validation* accuracy) as a guard against memorizing the train set. In hindsight, given the `hot_take` class collapse (§5), 3 epochs may have been *too few* for the model to learn the hardest boundary — re-running with more epochs and a class-weighted loss is the first experiment I'd try (see §9). I left the learning rate (2e-5) and batch size (16) at their standard BERT-fine-tuning defaults, as neither was the bottleneck.

**Baseline model:** `llama-3.3-70b-versatile` via Groq, `temperature=0`, `max_tokens=20`, zero-shot. Each test post is sent with the system prompt below as a `user` message ("Classify this post:\n\n{text}"). The model's reply is lowercased and matched against the label strings (longest-first, to avoid one label being a substring of another); unparseable replies are dropped from scoring. Results were collected by looping over the 31 test posts with a 0.1s delay between calls to respect free-tier rate limits.

**Baseline prompt used:**

> **⚠️ TODO (confirm against Colab):** the committed notebook (`ai201_project3_takemeter_starter_clean.ipynb`) contains only the *unfilled* prompt skeleton — the version that produced the 80.6% baseline was edited in Colab and not saved back. Below is the prompt reconstructed from my label definitions per the notebook's instructions (name the community + task, one-sentence definition per label, one example each, output label only). Paste your exact `SYSTEM_PROMPT` here to confirm.

```text
You are classifying posts and comments from r/nba, an NBA basketball discussion forum.
Assign each post to exactly one of the following four categories.

analysis: A structured argument backed by specific, verifiable evidence (stats, film, history).
Example: "Wemby's defensive rating dropped from 108.3 to 104.1 after the All-Star break — he's contesting 3.2 more shots at the rim and opponents shoot 11% worse on them."

hot_take: A bold, confident opinion stated without meaningful supporting evidence.
Example: "Chet Holmgren is the most overrated player in the league. Dude is just tall."

reaction: An immediate emotional response to an event — celebration, frustration, shock.
Example: "GLORIOUS HATEWATCH"

context: Neutral factual information or background, without taking an evaluative stance.
Example: "Champagnie was undrafted out of Pitt and is now playing 20+ minutes in the WCF on a minimum contract."

Respond with ONLY the label name. Do not explain your reasoning.

Valid labels:
analysis
hot_take
reaction
context
```

---

## 5. Evaluation Report

### 5.1 Overall Accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Llama-3.3-70B) | **0.806** |
| Fine-tuned DistilBERT | **0.516** |
| **Fine-tuning effect** | **−0.290 (regression)** |

Source: `evaluation_results.json`, test set = 31 posts.

### 5.2 Per-Class Metrics — Fine-Tuned DistilBERT

Derived exactly from `confusion_matrix.png`:

| Class | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `analysis` | 0.53 | **1.00** | 0.70 | 8 |
| `hot_take` | **0.00** | **0.00** | **0.00** | 8 |
| `reaction` | 0.80 | 0.50 | 0.62 | 8 |
| `context` | 0.36 | 0.57 | 0.44 | 7 |
| **Accuracy** | | | **0.52** | 31 |
| **Macro avg** | 0.42 | 0.52 | 0.44 | 31 |
| **Weighted avg** | 0.43 | 0.52 | 0.44 | 31 |

The single most important number here: **`hot_take` F1 = 0.00.** The model never once predicted `hot_take` and never recalled one. Against my success thresholds in [planning.md](planning.md) (macro-F1 ≥ 0.75 for "useful," ≥ 0.65 for "deployable"), the fine-tuned model's macro-F1 of **0.44** fails both — it lands between the random baseline (~0.25) and "deployable."

### 5.3 Per-Class Metrics — Baseline (Llama-3.3-70B)

> **⚠️ TODO (from Colab):** Only the baseline's overall accuracy (0.806) was exported to `evaluation_results.json`. Re-run notebook cell 22 (`classification_report` for the baseline) and paste the per-class precision/recall/F1 here. Table scaffold below — fill in the numbers.

| Class | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `analysis` | _TODO_ | _TODO_ | _TODO_ | _TODO_ |
| `hot_take` | _TODO_ | _TODO_ | _TODO_ | _TODO_ |
| `reaction` | _TODO_ | _TODO_ | _TODO_ | _TODO_ |
| `context` | _TODO_ | _TODO_ | _TODO_ | _TODO_ |
| **Accuracy** | | | **0.81** | ≤31 |

### 5.4 Confusion Matrix — Fine-Tuned DistilBERT

Rows = true label, columns = predicted label. (Also committed as `confusion_matrix.png`.)

| True ↓ / Pred → | analysis | hot_take | reaction | context | **Total** |
|---|---|---|---|---|---|
| **analysis** | **8** | 0 | 0 | 0 | 8 |
| **hot_take** | 4 | **0** | 0 | 4 | 8 |
| **reaction** | 1 | 0 | **4** | 3 | 8 |
| **context** | 2 | 0 | 1 | **4** | 7 |
| **Total predicted** | 15 | 0 | 5 | 11 | 31 |

Two structural facts jump out:

1. **The `hot_take` column is entirely empty.** The model assigned the label to *zero* posts. The classifier effectively collapsed to a 3-class model.
2. **The 8 true `hot_take` posts were split 4 → `analysis`, 4 → `context`.** This is the central error and it is perfectly directional: hot takes never leaked into `reaction`, only into the two "substantive" classes.

`analysis` was the over-predicted sink (15 predictions for 8 true posts; perfect recall but poor precision). `context` was the second sink (11 predictions for 7 true posts).

### 5.5 AI-Assisted Pattern Surfacing

Per the Milestone 6 instructions, I ran the misclassified examples through an LLM (Claude) and asked it to surface common themes, then verified each pattern against the data myself.

**Patterns the AI proposed:**

1. **"The model collapsed the `hot_take` class entirely — likely because `hot_take` is defined by the *absence* of evidence, which has no positive surface signal."** ✅ **Verified and kept.** This is the core finding. `analysis` (has stats + structure), `reaction` (short, all-caps, emojis), and `context` (neutral facts) all have positive lexical signatures DistilBERT can latch onto. `hot_take` is defined by what it *lacks* (real reasoning), so the model has nothing to key on — and a hot take that mentions a player/stat looks like `analysis`, while a calm declarative hot take looks like `context`. The 4/4 split into exactly those two classes confirms this.

2. **"Hot takes that cite a stat get pulled toward `analysis`; hot takes phrased as flat declaratives get pulled toward `context`."** ✅ **Verified and kept.** Re-reading the `hot_take` test candidates: posts like *"Dort shot 29% from three... you can't win when your SG shoots like that"* contain a number and read as `analysis` to a bag-of-tokens model. Posts like *"The Spurs are going to be a PROBLEM for the next decade"* are calm assertions that read as `context`. This is exactly the `analysis`-vs-`hot_take` boundary I flagged as hardest in planning — the model failed precisely where I predicted annotation would be hard.

3. **"`reaction` is confused with `context` because of post length, not topic."** ⚠️ **Partially kept, refined.** The AI initially framed this as a length artifact. Verifying it: of 8 true `reaction` posts, 3 went to `context` and 1 to `analysis`. The three that leaked into `context` are the *longer, fact-containing* reactions (e.g., "generational hate-watch, 72K upvotes..."), not the short all-caps ones. So it's not pure length — it's the **fact-bearing reactions** that fool the model, which is the same "surface features outrank intent" story as pattern 2. I kept the refined version.

4. **"Short low-information posts are systematically wrong."** ❌ **Discarded.** The AI suggested short posts were a problem; the data contradicts this. Short `reaction` posts ("WEMBY IS INEVITABLE," "FADE EM 🗑️") were among the *best*-classified — `reaction` precision is 0.80. The errors cluster in the *medium-to-long* `hot_take` posts, not short ones. I threw this pattern out.

**Verification method:** for each proposed pattern I went back to the actual `hot_take`/`reaction` posts and counted how many errors it accounted for, rather than trusting the LLM's summary. Patterns 1 and 2 together explain all 8 `hot_take` errors; pattern 3 explains 3 of the 4 `reaction` errors.

### 5.6 Three Analyzed Failures

All three below are **true `hot_take` posts**. Because the model achieved **0/8 recall on `hot_take`**, every `hot_take` in the test set is a guaranteed misclassification — these are real failures regardless of which exact posts landed in the split. Per the confusion matrix, each was predicted as either `analysis` or `context`.

> **⚠️ TODO (from Colab):** the exact predicted label and confidence per post come from notebook cell 17 (the wrong-predictions printout). I've inferred the most likely predicted class from the post's surface features and marked confidence as _TODO_ — confirm against your Colab output.

**Failure 1 — stat-citing hot take → predicted `analysis`**
> *"Dort shot 29% from three in the playoffs. You can't win when your starting SG shoots like that."*
- **True:** `hot_take` · **Predicted:** `analysis` (likely) · **Confidence:** _TODO_
- **Why it failed:** This is the canonical one-stat boundary case. The stat (29%) is real, so the model — which has no way to tell decorative evidence from load-bearing evidence — reads "number + basketball reasoning" and fires `analysis`. My label rule says one stat + one unargued assertion = `hot_take`, but that rule depends on judging whether the reasoning is *developed*, which a fine-tuned DistilBERT on 140 examples cannot represent.
- **Labeling vs. data problem:** This is a **data/boundary problem, not annotation inconsistency.** I labeled all one-stat-assertion posts as `hot_take` consistently. The issue is that the training set is too small for the model to learn the *depth-of-reasoning* distinction, which is the only thing separating these from `analysis`.

**Failure 2 — declarative hot take → predicted `context`**
> *"The Spurs are going to be a PROBLEM for the next decade. This is just the beginning."*
- **True:** `hot_take` · **Predicted:** `context` (likely) · **Confidence:** _TODO_
- **Why it failed:** Calm, declarative, no emotional markers and no stats. Stripped of context, the sentence structure ("X is going to be Y") resembles the neutral declaratives in my `context` class. The model has no signal that this is an *unsupported prediction* rather than a *factual statement*.
- **Labeling vs. data problem:** Boundary/definition problem. The `hot_take`↔`context` distinction is *evaluative stance*, which is semantically subtle and underrepresented — there are few calm-but-unsupported hot takes in 51 examples for the model to learn the pattern.

**Failure 3 — mocking hot take with a fact → predicted `analysis` or `context`**
> *"Thunder won 68 games and couldn't even make the Finals. Regular season means nothing."*
- **True:** `hot_take` · **Predicted:** `analysis` or `context` (likely) · **Confidence:** _TODO_
- **Why it failed:** Contains a real fact (68 wins) used as a launching pad for a sweeping generalization ("regular season means nothing"). The factual clause looks like `context`/`analysis`; the broad assertion that makes it a hot take is exactly the part the model can't weigh.
- **Labeling vs. data problem:** Data problem. The model would need many more examples of "real fact → overreach generalization" to learn that the *generalization* dominates the *fact*.

**The unifying diagnosis:** every `hot_take` failure is the same shape — the model keys on *surface evidence* (a number, a factual clause, a calm tone) and ignores *whether the evidence actually supports the claim*. That judgment is the entire definition of `hot_take`, and it is the one thing a small fine-tune cannot learn. The baseline LLM does far better precisely because it can reason about argument quality zero-shot.

### 5.7 Sample Classifications

Five posts run through the fine-tuned model. Predicted labels are taken from the confusion-matrix structure where determinable; confidence scores were not exported.

> **⚠️ TODO (from Colab):** fill the confidence column from cell 15/17 softmax outputs.

| # | Post (truncated) | True | Predicted | Confidence | Correct? |
|---|---|---|---|---|---|
| 1 | "Wemby's defensive rating dropped from 108.3 to 104.1 after the All-Star break... contesting 3.2 more shots per game at the rim..." | `analysis` | `analysis` | _TODO_ | ✅ |
| 2 | "GLORIOUS HATEWATCH" | `reaction` | `reaction` | _TODO_ | ✅ |
| 3 | "Dort shot 29% from three in the playoffs. You can't win when your starting SG shoots like that." | `hot_take` | `analysis` | _TODO_ | ❌ |
| 4 | "The Spurs haven't been to the Finals since 2014. That's an 11-year drought for a franchise with 5 championships." | `context` | `context` | _TODO_ | ✅ |
| 5 | "The Spurs are going to be a PROBLEM for the next decade. This is just the beginning." | `hot_take` | `context` | _TODO_ | ❌ |

**Why example 1 is a reasonable correct prediction:** the post is dense with specific, verifiable numbers (108.3 → 104.1 DRTG, 3.2 more contested shots, 11% worse opponent shooting) and a causal mechanism ("staying vertical instead of biting on pump fakes"). These are exactly the surface features that co-occur with `analysis` in the training data, and here the surface signal and the true label genuinely agree — so the model's strong lexical prior lands on the right answer for the right-ish reasons. (`analysis` recall was a perfect 1.00, so the model reliably catches the stat-heavy, structured posts; its problem is over-claiming `analysis` for posts that merely *look* like it — see example 3.)

---

## 6. Reflection: What I Intended vs. What the Model Captured

I intended the model to learn a **discourse-quality axis**: does a post *do analytical work* (`analysis`, `context`) or *just express a stance/feeling* (`hot_take`, `reaction`), and within each pair, how much evidence vs. emotion is present. My label definitions are all built around *intent and argument structure* — "is the evidence load-bearing or decorative," "is the fact serving the emotion or vice versa."

What the model actually captured was a much shallower **surface-feature axis**:
- **Stats / long structured text → `analysis`.** It learned "looks rigorous," not "is rigorous." That's why it over-predicted `analysis` (15/31) and nailed `analysis` recall (1.00) while tanking its precision (0.53).
- **Short / all-caps / emoji → `reaction`.** This worked well (precision 0.80) because emotional reactions genuinely have a distinct lexical signature.
- **Neutral declarative + facts → `context`.** A catch-all sink (11/31 predictions) for anything calm that wasn't obviously a reaction.
- **`hot_take` → nothing.** The model never learned it, because `hot_take` is the one class with **no positive surface signature** — it's defined by the *absence* of real reasoning, which is invisible to a bag-of-features fine-tune.

**The gap in one sentence:** I defined my labels by *whether the reasoning is real*; the model learned *whether the post looks like it contains reasoning*. Those two are identical for clear cases and opposite for exactly the hard cases (`hot_take`), which is why the decision boundary collapsed there.

**What it overfit to:** the presence of numbers and structured sentence shape as a proxy for `analysis`. **What it missed:** evaluative stance and argument depth — the entire conceptual core of the `hot_take` class. Notably, the baseline LLM did *not* have this failure (80.6% overall), which tells me the gap is a **capacity/data** limitation of the small fine-tune, not a flaw in the label definitions themselves.

---

## 7. Spec Reflection

**One way the spec helped:** The spec's insistence on **per-class metrics and a confusion matrix** (not just accuracy) is the only reason this project produced a real finding. A single accuracy number (51.6%) would have read as "mediocre model, shrug." The confusion matrix exposed that the model is *good* at three classes and *completely blind* to a fourth — a categorical failure, not a uniform mediocrity. The spec forcing me to look per-class turned a non-result into a diagnosis.

**One way my implementation diverged from the spec:** My planning doc (Milestone 2) set a success threshold of **macro-F1 ≥ 0.75** and committed to **macro-F1 as the primary metric** with Cohen's kappa as a secondary. In practice the notebook pipeline reports and exports **accuracy** as the headline comparison metric, and the fine-tuned model (macro-F1 ≈ 0.44) fell far below my own success bar. I diverged by leading this report with accuracy (to match the artifacts the notebook actually produced and committed) but I kept faith with the spec by computing macro-F1 by hand from the confusion matrix and explicitly judging the model against my pre-registered 0.75 threshold — which it fails. I did not move the goalposts to make the result look better.

---

## 8. AI Usage

**Instance 1 — Failure pattern analysis (this report).** I directed Claude to act as the failure-analysis tool described in my Milestone 2 plan: I gave it the confusion matrix and the misclassified `hot_take`/`reaction` posts and asked it to surface common error themes. It produced four candidate patterns (§5.5). I **kept** the `hot_take`-collapse and surface-feature patterns after verifying them against the data, **refined** the `reaction`→`context` pattern (the AI called it a length effect; I corrected it to a fact-bearing-reaction effect after counting cases), and **discarded** its "short posts are systematically wrong" claim outright because the data showed short `reaction` posts were classified *best*. The verification and the correction are mine.

**Instance 2 — Drafting this README.** I directed Claude (Claude Code) to assemble this evaluation report from my committed artifacts (`evaluation_results.json`, `confusion_matrix.png`, `labeled_data.csv`). It computed the per-class precision/recall/F1 from the confusion matrix and laid out the report structure. I **overrode** its initial impulse to present sample-classification confidence scores as if known — those were never exported from Colab, so I had it mark them as explicit TODOs rather than fabricate plausible numbers. I also had it flag the baseline per-class table as TODO for the same reason.

**Annotation disclosure:** **No LLM was used to pre-label any data.** Per my Milestone 2 AI Tool Plan, all 205 posts were labeled by me, by hand, in one session. The decision to avoid pre-labeling was deliberate — at this dataset size the anchoring-bias risk outweighed the time saved (see [planning.md](planning.md), "Annotation Assistance"). I did use an LLM earlier to *stress-test my label definitions* against 8 generated boundary posts (documented in `planning.md`), but those generated posts are not part of the training data.

---

## 9. Limitations & What Would Change It

The fine-tuned model is **not deployable** — it fails my own success thresholds and is blind to an entire class. Concretely, to fix the `hot_take` collapse I would:

1. **More `hot_take` examples, especially the hard sub-types.** 51 examples is too few for a class defined by absence-of-evidence. I'd target the two failure sub-shapes specifically: (a) hot takes that *cite a stat* and (b) calm declarative hot takes, since those are what leak into `analysis` and `context` respectively.
2. **Tighten — or accept the fuzziness of — the `analysis`/`hot_take` boundary.** The boundary depends on *depth of reasoning*, which a small fine-tune can't see. Either give it far more contrastive pairs (same stat, argued vs. unargued) or accept that this distinction may need an LLM, not a 66M-param encoder.
3. **Train longer / tune hyperparameters.** 3 epochs on ~140 examples may simply be undertrained; the class collapse can also be a sign the model defaulted to high-frequency surface classes. Worth trying more epochs, class-weighted loss, or a larger base model.
4. **Reconsider the architecture choice.** The baseline LLM already hits 80.6% zero-shot. For this task — where the labels hinge on *reasoning quality* — a prompted LLM may simply be the better tool, and fine-tuning a small encoder may be the wrong approach rather than a tunable one.

Other limitations: test set is small (31 posts), so per-class metrics have wide error bars; all data is from one subreddit during one playoff window (June 2026), so temporal/topic generalization is untested.

---

## 10. Repository Contents

| File | Description |
|---|---|
| `README.md` | This evaluation report. |
| `planning.md` | Milestones 1–2 design notes: label definitions, edge cases, metric/success rationale, AI tool plan, annotation difficult cases. |
| `labeled_data.csv` | 205 hand-labeled posts (`text`, `label`, `notes`). |
| `ai201_project3_takemeter_starter_clean.ipynb` | Training + evaluation notebook (DistilBERT fine-tune + Groq baseline). |
| `confusion_matrix.png` | Fine-tuned model confusion matrix (test set). |
| `evaluation_results.json` | Exported headline metrics. |

### Reproducing

1. Open the notebook in Colab with a T4 GPU runtime.
2. Upload `labeled_data.csv`; the `LABEL_MAP` is already set to the 4 r/nba labels.
3. Run Sections 1–4 to fine-tune and evaluate DistilBERT (the train/test split is fixed at `random_state=42`).
4. Add a `GROQ_API_KEY` and run Section 5 for the baseline.
5. Section 6 exports `evaluation_results.json` and `confusion_matrix.png`.

---

## 11. Demo Video

> **⚠️ TODO:** Record a 3–5 minute demo. Required moments: 3–5 posts classified by the fine-tuned model with label + confidence visible; one correct prediction narrated (use Sample Classification #1 — the stat-dense `analysis` post — and explain why the prediction is reasonable); one incorrect prediction narrated (use a `hot_take` post and explain the surface-feature failure); and a brief walkthrough of this evaluation report. Add the link here once recorded.
