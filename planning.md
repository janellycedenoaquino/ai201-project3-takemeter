# TakeMeter — Planning Document

## Community

**Chosen community:** r/worldcup and r/soccer during the 2026 FIFA World Cup.

These subreddits generate high volumes of text-based discourse ranging from tactical breakdowns to pure emotional reactions, making them ideal for measuring argument quality. The distinction between a well-reasoned take and empty hyperbole is one that regular participants in these communities actively care about and debate.

**Why this community works for a classification task:**

The discourse in these subreddits spans the full quality spectrum within a single thread — a post-match discussion will contain tactical breakdowns with specific observations, bold unsupported claims about players being "finished," and pure emotional reactions all in the same comment section. That variance is what makes it a good classification task: the signal is real and the classes are genuinely distinct, not artificially constructed. A classifier that learns to separate these isn't just pattern-matching on surface features — it's learning something meaningful about how people argue.

This community also has strong implicit norms about discourse quality. Regular participants actively distinguish between "actually watching the game" analysis and "casual fan hot takes" — the labels we're using reflect categories the community itself recognizes, which means the signal is grounded in real social meaning rather than arbitrary distinctions.

---

## Label Taxonomy

### `insightful`
**Definition:** Post makes a specific claim and supports it with at least one reason, observation, or stat. You could steelman the argument even if you disagree with the conclusion.

**Clear examples:**
- *"France's high press only works when Tchouaméni sits deep enough to cover the channels. Against Morocco they left those spaces wide open and got punished on the counter twice."*
- *"The reason Japan beats European teams is European managers underestimate their pressing intensity. Japan's defensive line forces teams to play long balls which their center backs are excellent against in the air."*

**Uncertain example:**
- *"African football will never develop until FIFA fixes the calendar conflict with club football."*
- This could be `hot_take` (bold claim, no evidence) or `insightful` (identifies a specific structural mechanism). Decision rule: if the post names a specific cause or mechanism even without data, lean `insightful`. The reasoning is traceable even if unproven.

**What it is NOT:**
- A strong opinion stated confidently without reasoning
- A stat cited in isolation with no argument built around it

---

### `hot_take`
**Definition:** Post makes a bold or confident claim with no supporting reasoning. Asserts rather than argues. The claim might be true, but the post gives you no reason to believe it.

**Clear examples:**
- *"Mbappe is literally the worst player at this World Cup I can't believe people still hype him up"*
- *"England will never win another World Cup in our lifetime. Just accept it."*

**Uncertain example:**
- *"Mbappe is overrated — his goals per game at this tournament is 0.3."*
- This could be `insightful` (cites a stat) or `hot_take` (stat is decorative, framing is accusatory). Decision rule: if the stat is connected to a broader argument or comparison, label `insightful`. If it's dropped in to sound credible but the post doesn't reason from it, label `hot_take`. This one-stat post with accusatory framing → `hot_take`.

**What it is NOT:**
- An emotional reaction with no claim (that's `low_effort`)
- A strong opinion that's actually backed by a reason (that's `insightful`)

---

### `low_effort`
**Definition:** Post makes no argument at all. Pure reactions, one-liners, emoji responses, memes, or crowd commentary. There is no falsifiable claim being made.

**Clear examples:**
- *"GOLAZOOOOO 😭😭😭😭 I can't breathe"*
- *"that crowd is LOUD"*

**Uncertain example:**
- *"This referee should never work again."*
- This could be `low_effort` (pure venting) or `hot_take` (implies a claim about the referee's competence). Decision rule: if the post implies a falsifiable assertion about someone's quality or competence, label `hot_take`. If it's pure emotion with no embedded claim, label `low_effort`. This post implies "this referee is incompetent" → `hot_take`.

**What it is NOT:**
- A post expressing frustration that still implies a claim about quality or competence (that's `hot_take`)

---

## Edge Cases and Decision Rules

### Hard case 1: Referee complaints
*"This referee should never work again"*

**Could be:** `hot_take` (making a claim about the referee's competence) or `low_effort` (pure venting, no real argument)

**Decision rule:** If the post implies a falsifiable claim about someone's competence or quality — even without evidence — label it `hot_take`. If it's pure venting with no embedded claim (*"I can't watch"*, *"NOOOOO"*), label it `low_effort`. The test: is there an assertion buried in the post that could in principle be argued against?

→ *"This referee should never work again"* = `hot_take` (implies: this referee is incompetent)
→ *"THIS REFEREE 😭😭😭"* = `low_effort` (no claim, just emotion)

---

### Hard case 2: One-stat posts
*"Mbappe is overrated — his goals per game at this tournament is 0.3"*

**Could be:** `insightful` (cites a stat) or `hot_take` (stat is cherry-picked, framing is accusatory)

**Decision rule:** If the stat is used to build an argument — connected to a cause, comparison, or broader point — label it `insightful`. If the stat is decorative — dropped in to sound credible but not genuinely reasoning — label it `hot_take`.

---

### Hard case 3: Broad structural claims
*"African football will never develop until FIFA fixes the calendar conflict with club football"*

**Could be:** `insightful` (identifies a structural cause) or `hot_take` (bold claim, no evidence)

**Decision rule:** If the post identifies a specific mechanism or cause, even without citing data, lean `insightful`. The claim is falsifiable and the reasoning is traceable. If it's a sweeping generalization with no mechanism (*"African teams always underperform"*), label it `hot_take`.

---

## Label Distribution Target

| Label | Target count | Minimum |
|-------|-------------|---------|
| `insightful` | 67 | 40 |
| `hot_take` | 67 | 40 |
| `low_effort` | 67 | 40 |
| **Total** | **200** | **120** |

Because hot takes and low effort posts dominate real World Cup discourse naturally, insightful posts will need to be actively hunted — they appear most often in post-match analysis threads rather than live match threads.

---

## Data Collection Plan

**Sources:**
- r/worldcup — match threads and post-match discussion
- r/soccer — World Cup megathreads and tactical analysis posts

**Collection method:** Manual collection during live matches and post-match windows, supplemented by generated examples for balance.

**Labeling process:** Single annotator (author). Each post read in full and assigned one label based on the definitions and decision rules above. Ambiguous cases noted and resolved using decision rules. At least 3 genuinely difficult cases documented in README.

---

## Train / Validation / Test Split

| Split | Size | Purpose |
|-------|------|---------|
| Train | 140 (70%) | Fine-tune DistilBERT |
| Validation | 30 (15%) | Hyperparameter tuning, early stopping |
| Test | 30 (15%) | Final evaluation, baseline comparison |

Splits will be stratified by label to maintain class balance across all three sets.

---

## Model Plan

**Base model:** `distilbert-base-uncased`
- Fast to fine-tune on CPU/small GPU
- Strong baseline for short text classification
- Well documented with HuggingFace Trainer API

**Baseline:** Zero-shot classification using Groq's `llama-3.3-70b-versatile` with a prompt describing the three labels and asking for a classification. No task-specific training.

**Key hyperparameter decisions to document:**
- Learning rate (likely 2e-5)
- Number of epochs (likely 3-5)
- Batch size (likely 16)
- Whether to use early stopping based on validation loss

---

## Evaluation Metrics

**Metrics used and why:**

**Overall accuracy** is reported for both models as a baseline comparison number. However accuracy alone is misleading here — if the model learned to predict `hot_take` most of the time it could get decent accuracy just from class frequency. Accuracy tells you nothing about whether it's actually learning each class.

**Per-class F1 score** is the primary metric. F1 is the harmonic mean of precision and recall, and it matters here because the cost of errors is not symmetric:
- A false `insightful` (labeling a hot take as insightful) is worse than a false `hot_take` — it would actively mislead someone using this tool to find quality discourse
- A missed `low_effort` (labeling noise as something else) wastes a reader's time
- Per-class F1 surfaces these asymmetries that overall accuracy hides

**Confusion matrix** shows which labels the model confuses most often. The most expected confusion is `hot_take` vs `low_effort` — both lack reasoning, and the model may conflate them. Seeing this in the matrix helps explain *why* the model fails, not just *that* it fails.

**Why not just accuracy:** With 3 balanced classes a random classifier gets ~33% accuracy. A model that learns one surface feature (like post length) might get 50-60% accuracy while being completely wrong in ways that matter. F1 per class catches this.

---

## Definition of Success

A classifier is genuinely useful for a real community tool if it can reliably distinguish `insightful` from the other two classes — that's the hardest and most valuable distinction.

**Specific thresholds:**

| Metric | Minimum acceptable | Target |
|--------|-------------------|--------|
| Overall accuracy | 65% | 75% |
| F1 — insightful | 0.65 | 0.75 |
| F1 — hot_take | 0.60 | 0.70 |
| F1 — low_effort | 0.65 | 0.75 |
| Improvement over zero-shot baseline | +5% accuracy | +10% accuracy |

**Why these numbers:** A random baseline on 3 balanced classes gives ~33% accuracy and ~0.33 F1. A zero-shot LLM baseline will likely hit 50-60% without any fine-tuning. For the fine-tuned model to be worth deploying it needs to meaningfully beat both baselines, especially on `insightful` which is the class users would actually care about filtering for.

**Definition of failure:** If the fine-tuned DistilBERT does not outperform the zero-shot Llama baseline by at least 5% accuracy on the test set, the fine-tuning approach should be questioned — it may mean 200 examples is insufficient or the label boundaries are too inconsistent to learn from.

---

## AI Tool Plan

### Label stress-testing
Before annotating 200 examples, use Claude to generate 10 boundary posts — posts that deliberately sit between two labels — to verify definitions are tight enough. Specifically:
- 4 posts on the `hot_take` / `insightful` boundary (strong opinion + one stat)
- 4 posts on the `hot_take` / `low_effort` boundary (referee/player complaints)
- 2 posts on the `insightful` / `low_effort` boundary (short but reasoned)

If any generated post cannot be cleanly classified using the decision rules in this document, the definition for that label needs tightening before annotation begins.

**Status:** Partially completed during planning. The referee complaint edge case was identified and a decision rule written. One-stat posts and broad structural claims were also identified and resolved. No further tightening required before annotation.

### Annotation assistance
AI pre-labeling will **not** be used for the primary dataset. Reasons:
- The dataset is small enough (200 examples) that manual labeling is feasible in 2-3 hours
- Pre-labeling introduces model bias into training data — the fine-tuned model may just learn to replicate LLM labeling patterns rather than the true signal
- Manual labeling forces the annotator to confront edge cases directly, which produces better decision rules

All labels in the dataset are assigned by the author. Any example where the author was uncertain is flagged in the README as a hard case.

### Failure analysis
After evaluation, the list of wrong predictions from the test set will be passed to Claude with the following prompt structure:
- Provide all misclassified examples with true label and predicted label
- Ask Claude to identify surface patterns: do wrong predictions share length, punctuation, capitalization, or topic patterns?
- Ask Claude to identify whether errors cluster in specific label pairs (e.g. all `hot_take` → `low_effort` confusions)
- Verify any identified patterns manually by reading the examples — do not take the pattern analysis at face value

The goal is to distinguish between two failure modes: (1) the model learned the wrong signal entirely, or (2) the model learned the right signal but the label boundary was genuinely ambiguous in those specific cases.

---

## Label Stress-Test Results

Before annotation, Claude was asked to generate 10 boundary posts — posts designed to sit between two labels — to verify the definitions hold under pressure.

### Boundary: `hot_take` / `insightful` (strong opinion + one stat)

1. *"Mbappe is clearly not a big game player — he's only scored 2 goals in 8 knockout games at this World Cup."*
   → **hot_take.** Stat is cherry-picked and framing is accusatory. No broader argument built from it.

2. *"Spain's possession stats are misleading — they average 68% but only 12% of their passes enter the final third."*
   → **insightful.** Two linked stats that together make a coherent argument about the nature of Spain's possession.

3. *"Argentina has been lucky — they've only outshot their opponents in 2 of 5 games."*
   → **hot_take.** "Lucky" is the claim, the stat is decorative. Doesn't explain mechanism.

4. *"Brazil's xG per game is 2.1 but they're converting at 40% of that — finishing is the real problem."*
   → **insightful.** Identifies a specific mechanism (finishing rate) from the stat. Reasoning is traceable.

### Boundary: `hot_take` / `low_effort` (referee/player complaints)

5. *"That linesman has been wrong every single time tonight."*
   → **hot_take.** Makes a falsifiable claim about the linesman's accuracy.

6. *"REF 😭😭😭😭😭"*
   → **low_effort.** No claim. Pure emotional reaction directed at the referee.

7. *"The referee is clearly biased against us. Every 50/50 has gone against us."*
   → **hot_take.** Makes a claim about bias + implies pattern. Arguable even if unsubstantiated.

8. *"I cannot believe that referee omg"*
   → **low_effort.** No assertion. Could refer to a good or bad call equally.

### Boundary: `insightful` / `low_effort` (short but reasoned)

9. *"Their press only works when they're fresh."*
   → **insightful.** Short but makes a specific conditional claim about a tactical system.

10. *"That's just how they play."*
    → **low_effort.** No claim. Descriptive non-statement.

**Conclusion:** All 10 posts classified cleanly using existing decision rules. No definition changes required before annotation.

---

## Underrepresentation Contingency Plan

If after collecting 200 examples a label falls below 40 examples:

**If `insightful` is underrepresented** (most likely scenario):
- Shift collection to post-match analysis threads rather than live match threads
- Search r/soccer for threads titled "tactical breakdown" or "match analysis"
- Post-match threads 24-48 hours after a game tend to have higher density of insightful posts than live threads

**If `hot_take` is underrepresented** (unlikely — these are abundant):
- Pull from live match threads during tense moments (red cards, late goals, penalty decisions)
- Twitter/X crossposted content in match threads tends to be more hyperbolic

**If `low_effort` is underrepresented** (very unlikely):
- Any live match thread will contain dozens of these — sample the top comments during goal moments

**Hard floor:** If any class falls below 40 examples after exhausting the above strategies, reduce the total dataset to maintain balance rather than proceeding with a severely imbalanced set. A balanced 120-example dataset trains better than an imbalanced 200-example one.

---

## Baseline Reflection (Milestone 4)

**Model:** llama-3.3-70b-versatile (zero-shot, no fine-tuning)
**Test set:** 30 examples (10 per class)

**Results:**

| Label | Precision | Recall | F1 |
|-------|-----------|--------|----|
| insightful | 1.00 | 1.00 | 1.00 |
| hot_take | 1.00 | 0.80 | 0.89 |
| low_effort | 0.83 | 1.00 | 0.91 |
| **overall accuracy** | | | **0.933** |

**Where the baseline struggled:**

`hot_take` had the lowest recall (80%) — 2 out of 10 hot takes were misclassified. `low_effort` had the lowest precision (83%) — meaning 2 posts were predicted as `low_effort` that were actually `hot_take`. This confirms the expected confusion boundary: both classes lack reasoning, so even a 70B model occasionally conflates them.

`insightful` was perfect (100% F1) — the presence of structured reasoning is a strong enough signal that a large general model detects it reliably from the label description alone.

**Hypothesis going into fine-tuning:**

DistilBERT fine-tuned on 140 examples will likely struggle more on the `hot_take` / `low_effort` boundary than Llama did. The fine-tuned model has less world knowledge to draw on and will need to learn the distinction from surface features in the training data. If the training examples don't consistently differentiate the two classes at a textual level, the fine-tuned model may underperform the zero-shot baseline on these two labels specifically — even if it matches or beats it on `insightful`.

**Success bar:** Fine-tuned DistilBERT needs to hit at least 88% overall accuracy to come within 5% of this baseline, which was defined in planning.md as the minimum acceptable improvement threshold.