# TakeMeter

A fine-tuned text classifier that evaluates discourse quality in World Cup soccer communities on Reddit. Given a post or comment from r/worldcup or r/soccer, TakeMeter predicts whether it is **insightful**, a **hot take**, or **low effort**.

---

## Community and Task

**Community:** r/worldcup and r/soccer during the 2026 FIFA World Cup.

These subreddits generate high volumes of text-based discourse ranging from tactical breakdowns to pure emotional reactions — often within the same comment thread. The distinction between a well-reasoned take and empty hyperbole is one that regular participants in these communities actively care about. A post-match thread will contain everything from specific tactical observations to "this team is FINISHED" to "YESSSSS 😭😭😭" — all in the same scroll. That variance makes it a strong classification task: the signal is real, the classes are distinct, and the labels reflect categories the community itself recognizes.

---

## Label Taxonomy

| Label | Definition |
|-------|-----------|
| `insightful` | Makes a specific claim supported by at least one reason, observation, or stat. You could steelman the argument even if you disagree with the conclusion. |
| `hot_take` | Makes a bold or confident claim with no supporting reasoning. Asserts rather than argues. The claim might be true but the post gives you no reason to believe it. |
| `low_effort` | Makes no argument at all. Pure reactions, one-liners, emoji responses, memes, or crowd commentary. No falsifiable claim being made. |

**Key decision boundary:** The hardest distinction is `hot_take` vs `low_effort`. Both lack reasoning, but a hot take makes a falsifiable claim while low effort makes no claim at all. The test: is there an assertion buried in the post that could in principle be argued against? "This referee should never work again" = `hot_take` (implies incompetence). "THIS REFEREE 😭😭😭" = `low_effort` (no claim, just emotion).

---

## Dataset

**Size:** 200 labeled examples  
**Source:** Generated examples modeled on real r/worldcup and r/soccer discourse during the 2026 World Cup, supplemented by manual collection from live match threads.  
**Format:** `takemeter_dataset_final.csv` with columns: `id`, `text`, `label`, `notes`

**Label distribution:**

| Label | Count |
|-------|-------|
| insightful | 68 |
| hot_take | 66 |
| low_effort | 66 |
| **Total** | **200** |

**Train / Validation / Test split:** 140 / 30 / 30 (70% / 15% / 15%), stratified by label. Test set is perfectly balanced at 10 examples per class.

### Hard annotation cases

**Case 1 — ID 19:** *"THIS REFEREE SHOULD NEVER WORK AGAIN"*
Could be `hot_take` (implies competence claim) or `low_effort` (pure venting). Decision: `low_effort` — all caps with no specific claim, pure emotional venting. Contrast with ID 59 (*"The worst refereeing display I have ever seen at any level of football"*) which is `hot_take` because it makes a specific quality assertion.

**Case 2 — ID 65:** *"Mbappe can't perform in big games. Stats don't lie."*
Could be `insightful` (implies stats exist) or `hot_take` (no stat is cited). Decision: `hot_take` — "Stats don't lie" without citing any stat is a rhetorical move, not an argument.

**Case 3 — ID 37:** *"African teams always underperform at World Cups because of poor preparation and federation politics. The talent is there — the infrastructure isn't."*
Could be `insightful` (names structural causes) or `hot_take` (broad generalization, no evidence). Decision: `insightful` — names two specific mechanisms and distinguishes between talent and infrastructure. Underdeveloped but the reasoning is traceable.

---

## Model

**Base model:** `distilbert-base-uncased`  
**Task:** Sequence classification (3 labels)  
**Framework:** HuggingFace Transformers + Trainer API  
**Training hardware:** Google Colab T4 GPU

**Hyperparameters:**

| Parameter | Value | Reasoning |
|-----------|-------|-----------|
| `num_train_epochs` | 3 | Good default for datasets of 100–500 examples; more epochs risk overfitting |
| `learning_rate` | 2e-5 | Standard starting point for fine-tuning BERT-family models |
| `per_device_train_batch_size` | 16 | Fits T4 GPU comfortably |
| `weight_decay` | 0.01 | Light regularization |
| `warmup_steps` | 50 | Stabilizes early training |
| `load_best_model_at_end` | True | Saves checkpoint with best validation accuracy |

**Training curve:**

| Epoch | Training Loss | Validation Loss | Validation Accuracy |
|-------|--------------|-----------------|---------------------|
| 1 | — | 1.083 | 0.567 |
| 2 | 1.088 | 1.050 | 0.700 |
| 3 | 1.056 | 0.981 | 0.767 |

Accuracy improved each epoch. The validation accuracy was still climbing at epoch 3 which suggests the model may benefit from additional epochs — a direction worth exploring if retraining.

---

## Baseline

**Model:** `llama-3.3-70b-versatile` via Groq API  
**Approach:** Zero-shot classification — the model received the label definitions and one example per label, with no task-specific training.  
**Prompt:** Described the community, provided one-sentence definitions and one example per label, and instructed the model to output only the label name.

---

## Evaluation Results

### Overall Accuracy

| Model | Accuracy |
|-------|----------|
| Zero-shot Groq baseline | **0.933** |
| Fine-tuned DistilBERT | **0.667** |
| Delta | -0.267 (regression) |

### Per-Class Metrics — Zero-Shot Baseline (Groq)

| Label | Precision | Recall | F1 | Support |
|-------|-----------|--------|----|---------|
| insightful | 1.00 | 1.00 | 1.00 | 10 |
| hot_take | 1.00 | 0.80 | 0.89 | 10 |
| low_effort | 0.83 | 1.00 | 0.91 | 10 |
| **macro avg** | **0.94** | **0.93** | **0.93** | 30 |

### Per-Class Metrics — Fine-Tuned DistilBERT

| Label | Precision | Recall | F1 | Support |
|-------|-----------|--------|----|---------|
| insightful | 0.53 | 1.00 | 0.69 | 10 |
| hot_take | 0.75 | 0.30 | 0.43 | 10 |
| low_effort | 1.00 | 0.70 | 0.82 | 10 |
| **macro avg** | **0.76** | **0.67** | **0.65** | 30 |

### Confusion Matrix — Fine-Tuned DistilBERT

Rows = true label, Columns = predicted label.

| | insightful | hot_take | low_effort |
|--|-----------|----------|------------|
| **insightful** | 10 | 0 | 0 |
| **hot_take** | 7 | 3 | 0 |
| **low_effort** | 2 | 1 | 7 |

See `confusion_matrix.png` for the visual version.

---

## Failure Analysis

**10 out of 30 test examples were misclassified** by the fine-tuned model. The dominant pattern is clear from the confusion matrix: the model predicted `insightful` for 9 of its 10 errors. It is using `insightful` as a catch-all for any post it is uncertain about.

Before writing this analysis, the wrong predictions were passed to Claude and asked to identify common surface patterns. Claude identified three patterns: (1) posts with formal or complete sentence structure being predicted as insightful regardless of content, (2) posts containing superlatives or comparative language ("worst ever," "at any level") being treated as analytical claims, and (3) posts with appeal-to-evidence phrases ("history proves it," "stats don't lie") triggering insightful predictions even when no evidence was cited. All three patterns held up on manual review.

### Wrong prediction 1
**Text:** *"That was the worst half of football I've ever watched"*  
**True:** `hot_take` | **Predicted:** `insightful` | **Confidence:** 0.34

**Analysis:** This is a complete, grammatically formal sentence — no all-caps, no emoji, no slang. The model appears to have learned that sentence formality correlates with `insightful`. But formality is a surface feature, not an argument. The post makes a strong claim ("worst half I've ever watched") with zero reasoning. This is a data problem: most `insightful` posts in the training set happen to be longer and more formally written, so the model learned the wrong signal.

### Wrong prediction 2
**Text:** *"The worst refereeing display I have ever seen at any level of football"*  
**True:** `hot_take` | **Predicted:** `insightful` | **Confidence:** 0.37

**Analysis:** The phrase "at any level of football" is a comparative claim — it implies the speaker has a frame of reference across levels. The model may have learned that comparative or hierarchical language signals `insightful`. But a comparison without evidence is still a hot take. This reveals a specific gap in the training data: there are not enough examples of comparative hot takes to teach the model that comparison alone doesn't make something insightful.

### Wrong prediction 3
**Text:** *"Every team from that continent is always going to underperform. History proves it."*  
**True:** `hot_take` | **Predicted:** `insightful` | **Confidence:** 0.38

**Analysis:** "History proves it" is an appeal to evidence — it sounds like the speaker is invoking data. The model treats evidence-adjacent language as a signal for `insightful` even when no actual evidence is provided. This is the most important failure mode: the model learned to detect the *form* of evidence-based argument rather than the *substance* of one. A fix would require more training examples that explicitly demonstrate this pattern — bold claim + appeal to vague evidence = `hot_take`, not `insightful`.

### Root cause

The fine-tuned model learned to use `insightful` as its default prediction for any post that isn't obviously low effort (emoji, all-caps, one word). It never learned to distinguish between posts that *sound* analytical and posts that *are* analytical. This is a training data distribution problem: 140 examples is too few to learn a boundary this subtle, especially when the boundary depends on semantic content (is reasoning present?) rather than surface features (does the post look like reasoning?).

---

## Sample Classifications

The following examples were run through the fine-tuned model:

| Text (truncated) | True Label | Predicted | Confidence |
|-----------------|-----------|-----------|------------|
| "France's high press only works when Tchouaméni sits deep..." | insightful | insightful | 0.71 |
| "GOLAZOOOOO 😭😭😭😭 I can't breathe" | low_effort | low_effort | 0.89 |
| "Mbappe is literally the worst player at this World Cup" | hot_take | insightful | 0.34 |
| "Spain is boring to watch I'm sorry" | hot_take | hot_take | 0.61 |
| "lol that keeper is cooked" | low_effort | low_effort | 0.78 |

**Why the first prediction is reasonable:** The France/Tchouaméni post names a specific player, a specific tactical role (sitting deep), and a specific consequence (exposed on the counter). The model correctly identifies that this post builds an argument rather than just asserting one. This is the clearest example of what `insightful` is supposed to look like, and the model gets it right with reasonable confidence.

**Note on confidence:** The model's confidence on wrong predictions is consistently low (0.34–0.38), which is actually a useful signal. In a production tool, predictions below a confidence threshold could be flagged for human review rather than shown to users.

---

## Reflection: What the Model Captured vs. What I Intended

I intended the model to learn the presence or absence of reasoning — to detect whether a post is making an argument or just asserting. What it actually learned is closer to: does this post look like it could be analytical?

The decision boundary the model found is approximately: formal sentence structure + no emoji + more than 10 words = `insightful`. That captures most actual insightful posts (which tend to be longer and more formal) but it also captures hot takes that happen to be written in complete sentences.

The model never learned to distinguish between "reasoning is present" and "reasoning-adjacent language is present." The phrase "history proves it" is not reasoning — but it activates the same surface pattern as actual reasoning. This is the gap between intended and learned behavior.

To fix this, I would need either significantly more training data (500+ examples) or a deliberate effort to include more examples of formal hot takes — posts that are well-written but make unsupported claims — so the model sees that formality and argument quality are independent features.

---

## Spec Reflection

**Where the spec helped:** The requirement to define a specific success threshold before training forced me to commit to a number (88% accuracy minimum) before seeing results. This made the regression result meaningful rather than just disappointing — I had a concrete target to compare against, which made the failure analysis sharper and more honest.

**Where implementation diverged:** The spec assumed fine-tuned DistilBERT would outperform or closely match the zero-shot baseline. In practice the fine-tuned model significantly underperformed (66.7% vs 93.3%). This divergence revealed something the spec didn't anticipate: for a task where the signal is subtle and semantic, a large zero-shot LLM with strong world knowledge may simply be a better fit than a small fine-tuned model trained on 140 examples. The spec's framing treated fine-tuning as an improvement step — this project shows it's a tradeoff that depends heavily on dataset size and task complexity.

---

## AI Usage

**1. Label taxonomy stress-testing**
Claude was used to generate 10 boundary posts — posts designed to sit between two labels — to verify label definitions before annotation began. All 10 were classifiable using the existing decision rules, confirming no definition changes were needed. The stress-test results are documented in planning.md.

**3. Failure pattern analysis**
After evaluation, the 10 wrong predictions were passed to Claude and asked to identify common surface patterns. Claude identified three patterns (sentence formality, comparative language, appeal-to-evidence phrases). All three were verified manually by re-reading the examples. One pattern Claude suggested — that short posts were more likely to be misclassified — did not hold up on review and was discarded.

---

## Repository Structure

```
takemeter/
├── README.md                      ← this file
├── planning.md                    ← design notes, label definitions, edge cases
├── takemeter_dataset_final.csv    ← 200 labeled examples
├── baseline_groq.py               ← zero-shot baseline script
├── baseline_results.json          ← Groq baseline output
├── evaluation_results.json        ← fine-tuned model results
└── confusion_matrix.png           ← confusion matrix visualization
```

---

## How to Reproduce

**Baseline:**
```bash
pip install groq scikit-learn
export GROQ_API_KEY="your_key_here"
python baseline_groq.py
```

**Fine-tuning:**
Open `ai201_project3_takemeter_starter_clean.ipynb` in Google Colab with a T4 GPU runtime. Upload `takemeter_dataset_final.csv` when prompted. Run Sections 1–4 in order.
