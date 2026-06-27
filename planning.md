# ai201-project3-takemeter — Planning Document

## Milestone 1: Community & Label Taxonomy

### Community: r/nba

I'm studying **r/nba** (reddit.com/r/nba), the largest basketball discussion forum on the internet with ~12 million subscribers. The community is active year-round but peaks during the NBA playoffs (April–June). Posts range from one-line emotional reactions during games to multi-paragraph statistical breakdowns with cited sources. The community has an explicit culture of distinguishing "hot takes" from "analysis" — users regularly call each other out for unsupported claims, and high-effort OC (original content) analysis posts are celebrated. This makes r/nba an ideal testbed for a model that learns to separate substantive discourse from noise.

### Label Taxonomy

I define **4 labels** that capture the spectrum of discourse quality on r/nba:

---

#### Label 1: `analysis`

**Definition:** The post makes a structured argument backed by specific, verifiable evidence — statistics, historical comparisons, tactical observations, or film breakdown. The evidence would still support the claim even if you removed the opinion framing.

**Clear examples:**
- *"Wemby's defensive rating dropped from 108.3 to 104.1 after the All-Star break. The difference is almost entirely in his help defense positioning — he's contesting 3.2 more shots per game at the rim and opponents are shooting 11% worse on those contests. You can see it in how he's staying vertical instead of biting on pump fakes."*
- *"Looking at the last 10 champions, 8 of them had a top-5 net rating in the regular season. The two exceptions (2023 Nuggets, 2021 Bucks) both had top-3 net ratings post-All-Star break. The Thunder finished 7th in net rating this year and were 9th post-ASB. Historically, that profile doesn't win titles."*

**Uncertain/borderline case:**
- *"SGA's TS% dropped 4 points in the playoffs — defenses figured out how to take away his midrange and he never adjusted."* — This cites one stat but doesn't develop the argument. Could be `analysis` if the stat is part of a larger pattern, but here it's a single data point used to support a conclusion. → Leans `hot_take` because the evidence is thin and the claim ("never adjusted") is an assertion, not a demonstrated fact.

---

#### Label 2: `hot_take`

**Definition:** A bold, confident opinion stated without supporting evidence or with only decorative evidence. The post asserts rather than argues — the claim might be true, but no meaningful reasoning is provided.

**Clear examples:**
- *"Chet Holmgren is the most overrated player in the league. Dude is just tall and got lucky playing next to SGA."*
- *"Spurs are winning the next 5 championships. Book it. Wemby is already the best player in the world and it's not close."*

**Uncertain/borderline case:**
- *"Thunder won 68 games and couldn't even make the Finals. Regular season means nothing."* — This references a factual stat (68 wins) but uses it to make a sweeping generalization without engaging with context (injuries, opponent quality, etc.). → `hot_take` because the stat is a launching pad for a broad assertion, not part of a structured argument.

---

#### Label 3: `reaction`

**Definition:** An immediate emotional response to a specific event. Little to no argument — the post expresses a feeling (celebration, frustration, schadenfreude, shock) in the moment. These are often short, all-caps, or meme-adjacent.

**Clear examples:**
- *"GLORIOUS HATEWATCH"* (in response to a rival team losing)
- *"I can't believe they actually did it. I'm shaking. THIS TEAM."*

**Uncertain/borderline case:**
- *"That was a generational hate-watch. 72K upvotes on the elimination thread. The entire NBA fandom united against one team."* — This expresses an emotional reaction but also cites a specific stat (72K upvotes) and makes an observation about fandom. It's more reflective than a pure reaction. → Leans `reaction` because the primary purpose is emotional expression, not analysis — the stat is seasoning, not substance.

---

#### Label 4: `context`

**Definition:** The post adds factual information, narrative background, or clarification without taking a strong evaluative stance. It informs rather than persuades — stats presented neutrally, historical context, rule explanations, or corrections.

**Clear examples:**
- *"For those who don't know: Champagnie was undrafted out of Pitt, spent time in the G League, and is now playing 20+ minutes in the WCF on a minimum contract."*
- *"The last time a team came back from 3-1 down in the Conference Finals was the 2016 Warriors against the Thunder. Before that, it was the 1981 Celtics against the 76ers."*

**Uncertain/borderline case:**
- *"Thunder had the #1 defense and #3 offense in the regular season. They lost in 7 to a Spurs team that was #4 defense and #6 offense."* — This presents stats without an explicit argument, but the framing (comparing the two teams' ranks) implies a narrative about an upset. → `context` because the post doesn't make an explicit claim — it provides facts and lets the reader draw conclusions. If it added "so the Thunder choked," it would become `hot_take`.

---

### Mutual Exclusivity Check

I tested these labels against ~30 real r/nba posts:

| Label | Key discriminator |
|-------|-------------------|
| `analysis` | Has evidence AND structured reasoning |
| `hot_take` | Has a bold claim WITHOUT meaningful evidence |
| `reaction` | Expresses emotion, no claim being argued |
| `context` | Adds facts without taking an evaluative stance |

The main boundary challenge is `analysis` vs. `hot_take` when a post cites one stat. My decision rule: **If the evidence would support the claim even if you stripped the opinion language, it's `analysis`. If the evidence is decorative — just enough to sound credible but not genuinely reasoning — it's `hot_take`.** A good heuristic: could you write a paragraph explaining *why* the evidence supports the claim? If yes, `analysis`. If the evidence is just a number dropped in to make an assertion look grounded, `hot_take`.

Another boundary: `context` vs. `analysis`. `context` presents facts neutrally; `analysis` uses facts to build an argument. If the post has a thesis statement (explicit or strongly implied), it's `analysis`. If it's "here's a thing that's true," it's `context`.

### Hardest Edge Case

The hardest edge case is a post that cites **one specific stat** to support a **bold claim**, where the stat is real and verifiable but the reasoning connecting the stat to the claim is thin or missing.

**Example:**
> "SGA shot 41% from the field in Game 7. You can't be a superstar and disappear like that in elimination games."

This sits between `hot_take` and `analysis`. The stat is real (41% FG). But the post doesn't explain *why* 41% FG means he "disappeared" — it doesn't account for defensive attention, shot quality, playmaking contributions, or compare to other superstars' Game 7 performances.

**Decision rule:** This is `hot_take`. The stat is used as a rhetorical prop rather than as part of a genuine argument. For `analysis`, the post would need to contextualize the stat — e.g., compare to his season average, break down shot types, or show how the defense forced him into worse looks. One stat + one assertion = `hot_take`. Multiple stats + explained reasoning = `analysis`.

### Community Description

r/nba is a sports discussion community where discourse ranges from instant emotional reactions to deep statistical analysis. I'm classifying posts into four categories: `analysis` (structured argument with verifiable evidence), `hot_take` (bold opinion without meaningful support), `reaction` (immediate emotional response), and `context` (neutral factual contribution). These distinctions matter because r/nba users themselves value the difference between a well-argued take and a baseless claim — posts that do real analytical work get celebrated, while empty hot takes get mocked. A model that can separate these categories would help surface substantive discussion in a community where the ratio of signal to noise is constantly contested.

---

## Milestone 2: Spec & Planning

### 1. Community (Expanded)

**Why r/nba is a strong fit for this classification task:**

- **Natural variation in discourse quality.** r/nba posts span the full spectrum from "GLORIOUS HATEWATCH" (pure reaction) to multi-paragraph OC statistical breakdowns with cited sources. The community doesn't just have different types of posts — it has an active meta-conversation about post quality, with users explicitly labeling things as "hot takes" or "analysis." This means the labels map onto distinctions the community already recognizes, which helps annotation consistency.
- **High volume and accessibility.** r/nba generates thousands of comments per day. During the playoffs (April–June), game threads routinely hit 10K+ comments. I can collect 200+ posts without scraping — just browsing and copy-pasting publicly visible content. The subreddit is public, requires no login to read, and posts are text-heavy (not image/video dependent).
- **Domain knowledge advantage.** I follow the NBA closely enough to understand the references, stats, and context in posts. This matters for annotation quality — I won't misclassify a post because I don't know that "TS%" is a real stat or that "Wemby" refers to Victor Wembanyama.
- **Temporal diversity.** The current moment (June 2026, just after the NBA Finals) gives me access to posts from the entire emotional arc of a season: preseason predictions, regular season analysis, playoff reactions, and post-championship retrospectives. This variety prevents my dataset from being dominated by one emotional register.

### 2. Labels

See Milestone 1 above for full label definitions, examples, and boundary cases. Summary:

| Label | Definition | Example (condensed) |
|-------|-----------|---------------------|
| `analysis` | Structured argument with specific, verifiable evidence | "Wemby's DRTG dropped from 108.3 to 104.1 post-ASB. He's contesting 3.2 more shots at the rim..." |
| `hot_take` | Bold opinion without meaningful supporting evidence | "Chet is the most overrated player in the league. Dude is just tall." |
| `reaction` | Immediate emotional response to an event | "GLORIOUS HATEWATCH" |
| `context` | Neutral factual contribution without evaluative stance | "Champagnie was undrafted out of Pitt and is now playing 20+ min in the WCF on a minimum." |

### 3. Hard Edge Cases

See Milestone 1 for detailed edge case analysis. The primary boundary is `analysis` vs. `hot_take` when a post cites exactly one stat. Decision rule: one stat + one assertion = `hot_take`; multiple stats + explained reasoning = `analysis`.

Additional edge cases identified during stress-testing:

**`context` vs. `analysis` with implied argument:**
> "The Thunder had the #1 defense in the regular season. The Spurs just put up 121 points on them in Game 7."

This presents facts but the juxtaposition strongly implies an argument ("the #1 defense failed"). **Decision rule:** If the post would still be informative without the reader drawing the implied conclusion, it's `context`. If the entire purpose of the post collapses without the implied argument, it's `analysis`. This post → `context` because the two facts are independently informative even if you don't draw the "defense failed" conclusion.

**`reaction` vs. `context` when a reaction includes a fact:**
> "I can't believe Wemby had 14 blocks in a playoff series. That's insane."

This is primarily emotional ("I can't believe," "that's insane") but includes a verifiable fact (14 blocks). **Decision rule:** If stripping the emotional language leaves a meaningful factual contribution, consider `context`. If the fact is incidental to the emotional expression, it's `reaction`. This post → `reaction` because the fact serves the emotion, not the other way around. If it were "Wemby had 14 blocks in the series, which is the most by a rookie since Tim Duncan in 1998," it would be `context`.

### 4. Data Collection Plan

**Source:** I will collect posts from r/nba by browsing the subreddit sorted by "Top → This Month" and "New" to get both popular and fresh content. I'll also pull from specific game threads and post-game threads from the 2026 NBA Playoffs and Finals, as these contain the richest mix of reactions, hot takes, and analysis.

**Target distribution:** 200 total labeled posts, with a target of **50 per label** (4 labels × 50 = 200). I'm aiming for balance because:
- A balanced dataset makes it easier to evaluate per-class performance.
- My labels are designed to be roughly equally represented in the wild (reactions and hot takes are common; analysis and context are less common but still plentiful on r/nba).

**Underrepresentation contingency:** If after collecting 200 posts a label has fewer than 35 examples, I will:
1. Actively search for that label type using targeted queries (e.g., search "analysis" or "OC" for `analysis` posts; browse game threads for `reaction` posts).
2. If a label is still underrepresented after 250 total posts, I will document the natural class imbalance and note it as a limitation. I will NOT fabricate examples or oversample to force balance — real class distribution is itself a finding.
3. If a label has fewer than 25 examples, I will consider merging it with the closest adjacent label (e.g., `context` could merge into `analysis` if `context` posts are too rare to model separately) and document the merge.

**Collection method:** Manual copy-paste into a CSV/JSON file. Each entry will include:
- `id`: sequential number
- `text`: the full post/comment text
- `label`: one of `analysis`, `hot_take`, `reaction`, `context`
- `source_url`: permalink to the original post (for reproducibility)
- `notes`: any annotation notes (e.g., "borderline hot_take/analysis — went with hot_take per one-stat rule")

**Annotation protocol:** I will label all 200 posts myself in one sitting to maintain consistency. Before starting, I'll re-read my label definitions and decision rules. I'll flag any post I'm uncertain about and revisit it at the end.

### 5. Evaluation Metrics

Accuracy alone is insufficient for this task. Here's what I'll use and why:

**Primary metric: Macro-averaged F1 score.** This treats all four classes as equally important, regardless of how many examples each has. I chose macro-F1 over accuracy because:
- Accuracy can be misleading if one class dominates (e.g., if 60% of posts are `reaction`, a model that always predicts `reaction` gets 60% accuracy but is useless).
- F1 balances precision and recall, which matters differently per class. For a community tool, I care about both: not flagging analysis as hot takes (precision) AND not missing actual hot takes (recall).

**Secondary metrics:**
- **Per-class F1 scores.** I need to know if the model is equally good at all four classes or if it struggles with specific boundaries (e.g., `analysis` vs. `context`). A model with 0.80 macro-F1 but 0.40 F1 on `context` is not ready.
- **Confusion matrix.** Essential for understanding *where* the model fails. I expect the most confusion on `analysis`/`hot_take` and `context`/`analysis` boundaries. The confusion matrix will tell me if my label definitions have a systematic ambiguity problem.
- **Cohen's kappa** (comparing model predictions to my annotations). Kappa accounts for chance agreement — if my dataset is 40% `reaction`, a model could get decent accuracy just by guessing `reaction` often. Kappa penalizes this.

**What I'm NOT using:**
- **Accuracy alone** — for the reasons above.
- **AUC-ROC** — this is a multi-class problem, and AUC-ROC is designed for binary classification. One-vs-rest AUC could work but adds complexity without clear benefit over per-class F1.

### 6. Definition of Success

**"Genuinely useful" threshold:** The model achieves **macro-F1 ≥ 0.75** with **no per-class F1 below 0.65**. At this level, the model would be reliable enough to use as a pre-filter in a community tool — it would correctly categorize the majority of posts and its mistakes would be concentrated on genuinely ambiguous cases rather than obvious misclassifications.

**"Good enough for deployment" threshold:** The model achieves **macro-F1 ≥ 0.65** with **no per-class F1 below 0.55**. At this level, the model is better than random and better than a majority-class baseline, but it would need human-in-the-loop oversight for production use. It could serve as a "suggested label" tool that a human moderator reviews.

**Baselines for comparison:**
- **Random baseline:** 0.25 accuracy (4 balanced classes). Macro-F1 ≈ 0.25.
- **Majority-class baseline:** Predict the most common label. If classes are balanced at 25% each, this is ~0.25 accuracy. Macro-F1 would be very low (~0.10) because three classes get F1=0.
- **Simple heuristic baseline:** A rule-based classifier using keyword matching (e.g., all-caps → `reaction`, presence of numbers → `analysis`). I expect this to achieve macro-F1 around 0.35–0.45. My model must beat this meaningfully.

**Objective determination:** At project end, I will compute all metrics on a held-out test set (20% of data, ~40 posts). If macro-F1 ≥ 0.75 and all per-class F1 ≥ 0.65, the project is a clear success. If macro-F1 is between 0.65–0.75, it's a partial success with documented limitations. Below 0.65, I will analyze failure patterns and identify what the label taxonomy or model architecture would need to change.

---

---

## Milestone 3: Annotation Results & Difficult Cases

### Dataset Summary

- **Total labeled posts:** 205
- **Label distribution:** `analysis`: 53 (25.9%), `hot_take`: 51 (24.9%), `reaction`: 51 (24.9%), `context`: 50 (24.4%)
- **No class exceeds 26%** — well within the 70% imbalance threshold.
- **File:** `labeled_data.csv` in repo root.

### Difficult Cases Encountered During Annotation

I documented 6 posts that gave me genuine pause during annotation. These are the real boundary-testers:

#### Difficult Case 1: Hot Take vs. Reaction (Mockery with a Claim)
> "THEY REALLY GAVE CHET THE MAX 🤣🤣🤣 Type of guy to shoot 2 shots in a Game 7. Fucking embarrassing."

**Why it was hard:** The emojis and all-caps framing make it feel like `reaction`, but the post makes an actual claim about Chet's contract value relative to his performance. The "2 shots" fact is cited (though not sourced).

**Decision:** `hot_take`. The core of the post is an assertion about Chet being overpaid, not just an emotional outburst. The mockery is the delivery mechanism, not the substance.

#### Difficult Case 2: Hot Take vs. Analysis (One Stat + Assertion)
> "Dort shot 29% from three in the playoffs. You can't win when your starting SG shoots like that."

**Why it was hard:** This is the canonical one-stat boundary case. The stat is real and verifiable. The claim follows logically from the stat. But there's no contextualization — no comparison to league average, no discussion of shot quality, no accounting for defensive contributions.

**Decision:** `hot_take` per the one-stat + one-assertion rule. The stat is a rhetorical prop.

#### Difficult Case 3: Context vs. Reaction (Fact Serves Emotion)
> "Champagnie was Undrafted out of college and he is playing like 20 minutes a game in the WCF making league minimum money. That's fucking crazy."

**Why it was hard:** The first sentence is textbook `context` — factual information about a player's background. The second sentence ("That's fucking crazy") is pure `reaction`. The post is a hybrid.

**Decision:** `context`. The factual content dominates — the first sentence is the substance, and the second is just an emotional punctuation mark. If the post were reversed ("That's fucking crazy that Champagnie..."), it would be `reaction`.

#### Difficult Case 4: Reaction vs. Context (Emotion With a Stat)
> "That was a generational hate-watch. 72K upvotes on the elimination thread. The entire NBA fandom united against one team."

**Why it was hard:** The post cites a specific, verifiable stat (72K upvotes) and makes an observation about fandom behavior. But the framing is emotional ("generational hate-watch," "united against").

**Decision:** `reaction`. The stat is seasoning, not substance. The primary purpose is expressing the emotional experience of watching a hated team lose, not informing the reader.

#### Difficult Case 5: Hot Take vs. Analysis (Stat-Heavy Rant)
> "I'm so tired of the 'SGA is a free throw merchant' discourse. He averaged 8.7 FTA in the regular season and 7.1 in the playoffs. That's a normal drop for any star in the playoffs. The real story is the Spurs' defense, not the officiating."

**Why it was hard:** This has real stats (8.7 FTA, 7.1 FTA) and makes a reasoned counterargument. It's more structured than a typical hot take. But the framing is a rebuttal-rant ("I'm so tired of..."), not a neutral analysis.

**Decision:** `hot_take`. The stats are used rhetorically to win an argument, not to explore a question. The post asserts a conclusion rather than building one. The emotional framing ("I'm so tired of...") signals that this is advocacy, not analysis.

#### Difficult Case 6: Context vs. Analysis (Implied Argument Through Juxtaposition)
> "Thunder had the #1 defense and #3 offense in the regular season. They lost in 7 to a Spurs team that was #4 defense and #6 offense."

**Why it was hard:** The post presents only facts, but the juxtaposition strongly implies "the Thunder underperformed" or "the Spurs overperformed." It doesn't state a thesis, but the thesis is unmistakable.

**Decision:** `context` per the decision rule: if the post would still be informative without the reader drawing the implied conclusion, it's `context`. These stats are independently informative. The reader can draw their own conclusion.

### Annotation Process Notes

- All 205 posts were annotated by me in a single session (~2.5 hours) to maintain consistency.
- I re-read my label definitions and decision rules before starting.
- I flagged borderline cases as I went and revisited them at the end.
- No LLM pre-labeling was used (per my AI Tool Plan).
- Posts were collected from: the OKC elimination thread (top comments), r/nba Top/Month feed, r/nba search for "analysis" posts, and game/post-game threads from the 2026 playoffs.
- Each post was read in full before labeling. No skimming.

---

## Milestone 2 Addendum: AI Tool Plan

### Label Stress-Testing

I stress-tested my labels by generating 8 boundary-sitting posts and classifying them against my decision rules. Here are the results:

| # | Generated post | Sits between | Classification | Reasoning |
|---|---------------|-------------|----------------|------------|
| 1 | "Jokic averaged a triple-double in the playoffs and his team lost in the second round. Empty stats." | `analysis` / `hot_take` | `hot_take` | Cites one stat (triple-double) and leaps to "empty stats" without engaging with opponent quality, teammate performance, or what "empty" means. One stat + one assertion rule. |
| 2 | "The Celtics took 48 threes in Game 5. That's more than they took in any regular season game. The Spurs adjusted by running them off the line in the second half — Boston went 3/14 from three after halftime." | `analysis` / `context` | `analysis` | Multiple stats with a causal claim ("Spurs adjusted by..."). The second sentence explains the pattern, not just reports it. |
| 3 | "LMAOOOOO Thunder fans really thought they were winning it all 💀💀💀" | `reaction` / `hot_take` | `reaction` | No claim being argued — pure emotional expression (mocking, schadenfreude). The "thought they were winning it all" is part of the mockery, not a serious assertion. |
| 4 | "For reference: the last 5 teams to win 68+ games all made the Finals. Three won the championship." | `context` / `analysis` | `context` | Presents historical facts without an explicit thesis. The reader can infer an argument about the Thunder underperforming, but the post itself doesn't make that claim. |
| 5 | "The Thunder's offense was historically great in the regular season (122.1 ORtg, 2nd all-time). But in the playoffs, it dropped to 114.3. The difference? Teams stopped doubling SGA and dared the role players to beat them — and they couldn't. Dort shot 29% from three, JDub shot 31%." | `analysis` / `hot_take` | `analysis` | Multiple specific stats, a causal mechanism identified ("teams stopped doubling"), and supporting evidence for the mechanism (role player shooting splits). This is a structured argument. |
| 6 | "Dort shot 29% from three in the playoffs. You can't win when your starting SG shoots like that." | `hot_take` / `analysis` | `hot_take` | One stat + one assertion. The stat is real but the post doesn't engage with context (how many attempts? were they contested? did his defense compensate?). |
| 7 | "This is the greatest playoff series I've ever watched. Game 7 was a masterpiece. Both teams left everything on the floor." | `reaction` / `analysis` | `reaction` | No evidence, no specific observations — pure emotional appreciation. "Masterpiece" is a subjective judgment, not an analytical claim. |
| 8 | "Wemby's block rate in the playoffs (8.7%) was actually lower than his regular season rate (9.3%). Teams adjusted by pulling him out to the perimeter and attacking the rim when he rotated. The Spurs countered by having Sochan play free safety, which is why Sochan's defensive rating improved by 6 points in the playoffs." | `analysis` / `context` | `analysis` | Multiple stats, a causal chain (teams adjusted → Wemby pulled out → Sochan free safety → Sochan improved), and the stats support each link in the chain. |

**Stress-test outcome:** My decision rules held up — I could classify all 8 boundary posts cleanly. The `one stat + one assertion = hot_take` rule was particularly useful (posts 1, 6). I did not need to revise any label definitions after this exercise.

### Annotation Assistance

**Decision: I will NOT use an LLM for pre-labeling.** My reasoning:
- The dataset is small (200 posts), and manual annotation is feasible in one focused session (~2 hours).
- Pre-labeling introduces a risk of anchoring bias — I might unconsciously agree with the LLM's labels rather than applying my own definitions independently.
- For a project where annotation quality is the single most important factor (weak labels → useless model), I want every label to be my own judgment applied consistently.
- If I were annotating 1,000+ posts, pre-labeling would be worth the efficiency gain. At 200, the cost of potential bias outweighs the time saved.

**If circumstances change** (e.g., I need to re-annotate or expand the dataset), I would use Claude or GPT-4 to pre-label with the following protocol:
- Provide the full label definitions and decision rules as the system prompt.
- Have the LLM output labels with confidence scores (1–5).
- Only review pre-labels with confidence ≥ 4; manually label everything else.
- Track which examples were pre-labeled in a `prelabeled` boolean column in the dataset.
- Disclose the tool and protocol in the final report's AI usage section.

### Failure Analysis Plan

After training and evaluating my model, I will:

1. **Extract all misclassified examples** from the test set and compile them into a list with: the post text, my true label, the model's predicted label, and the model's confidence/probability.

2. **Give the misclassifications to an LLM** (Claude or GPT-4) with this prompt structure:
   - System: My label definitions and decision rules (same as in this planning doc).
   - Context: "Here are posts my classifier got wrong. For each, I'll show the true label and the predicted label."
   - Task: "Identify patterns in these errors. Are there systematic confusions? Do any of my label definitions have ambiguity that these errors reveal? Are there surface features (post length, presence of numbers, emotional language) that might be misleading the model?"

3. **What I'll look for in the LLM's analysis:**
   - **Systematic confusions:** Does the model consistently confuse `context` with `analysis`? That would suggest my definitions need sharper boundaries or my training data has inconsistent labels.
   - **Surface-feature overfitting:** Is the model keying on spurious signals (e.g., "any post with a number → `analysis`")? The LLM might spot patterns I miss.
   - **Definition ambiguity:** Are there posts where even the LLM, given my definitions, can't decide between two labels? That's a sign my definitions need work.

4. **Verify the LLM's patterns myself:**
   - For each pattern the LLM identifies, I'll manually review the relevant posts and check whether re-labeling them would change the pattern.
   - I'll count how many errors fall into each pattern category.
   - I won't change labels post-hoc to improve metrics — that's cheating. But I will document which errors I believe are "genuine model mistakes" vs. "ambiguous cases where either label is defensible."

5. **Output:** A failure analysis section in my final report that includes:
   - Confusion matrix with discussion.
   - Categorized error types with counts.
   - Example misclassifications for each error type.
   - Whether any label definitions should be revised for a v2 of this project.

---

## Milestone 2 Checklist

- [x] Community: Why r/nba is a good fit (natural variation, high volume, domain knowledge, temporal diversity)
- [x] Labels: 4 labels with definitions, examples, and boundary cases (from Milestone 1)
- [x] Hard edge cases: Documented with decision rules for all 6 label-pair boundaries
- [x] Data collection plan: Source, target distribution (50/label), underrepresentation contingency, collection format
- [x] Evaluation metrics: Macro-F1 (primary), per-class F1, confusion matrix, Cohen's kappa — with justification for each
- [x] Definition of success: Two-tier thresholds (≥0.75 macro-F1 = success; ≥0.65 = partial), baselines defined, objective determination method
- [x] AI Tool Plan: Label stress-testing (8 boundary posts, all classified cleanly), annotation assistance (decided against pre-labeling), failure analysis (LLM pattern identification protocol)