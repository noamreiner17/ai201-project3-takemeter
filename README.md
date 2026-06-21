# TakeMeter: A Reddit Discourse Classifier for r/television

TakeMeter sorts `r/television` posts/comments into three discourse types:
objective **news**, subjective **reaction_opinion**, or predictive
**speculation** — the basis for a feed filter that strips noise. This README is
the final report; full design notes and edge cases are in [planning.md](planning.md).

## 1. What I Built

- **Task:** single-label, three-class text classification.
- **Data:** 200 manually annotated posts from `r/television` ([reddit.csv](reddit.csv)).
- **Baseline:** zero-shot Groq `llama-3.3-70b-versatile`, given the label rules but no training.
- **Fine-tuned:** `distilbert-base-uncased`, fine-tuned on Google Colab (free T4 GPU), evaluated on a 30-example held-out test set.

### Community

I chose **r/television**: a high-traffic, text-heavy subreddit where discourse
naturally cycles through all three labels — a studio drops a factual update
(`news`), fans react (`reaction_opinion`), then theorize about what's next
(`speculation`). That lifecycle makes the boundaries meaningful to participants
and gives a real use case: a feed filter for "facts only" vs. "theories only."

### Label Taxonomy

| Label | Definition |
| :--- | :--- |
| `news` | Verifiable facts, announcements, ratings — no personal bias. |
| `reaction_opinion` | A personal emotional response or value judgment. |
| `speculation` | A prediction about future plot/casting/renewal from clues or theories. |

**`news`** — *"Mark Hamill Joins Peacock's 'Twisted Metal' For Season 3."* ·
*"The final episode pulled in 4.2 million viewers across streaming and cable Sunday."*

**`reaction_opinion`** — *"That finale felt incredibly rushed — three seasons of
buildup wasted in five minutes."* · *"The chemistry between the two leads is easily
the best part of the show."*

**`speculation`** — *"Based on the comics, I'm 90% sure the brother betrays the
group in the mid-season finale."* · *"The showrunner posted a cryptic clock photo —
I bet they're doing a time-jump for season 4."*

Boundary rules: a value judgment on a rumor → `reaction_opinion`; a clue used to
forecast → `speculation`. Details in [planning.md](planning.md#edge-cases-and-decision-rules).

### Dataset

- **Source:** 200 public posts and top-level comments collected manually from
  `r/television` ([reddit.csv](reddit.csv)).
- **Labeling process:** each example was read and hand-labeled against the
  planning.md definitions; an LLM pre-labeled an initial ~50-post batch that I then
  reviewed and corrected (disclosed in §7), with hard cases logged in a notes column.
- **Split:** the notebook splits 70/15/15 into train/validation/test (30-example test set).

**Label distribution (no class exceeds 70%):**

| Label | Count | Share |
| :-- | :-: | :-: |
| reaction_opinion | 69 | 34.5% |
| speculation | 67 | 33.5% |
| news | 64 | 32.0% |
| **Total** | **200** | **100%** |

**3 genuinely difficult examples (full reasoning in [planning.md](planning.md)):**

1. *"With Russell T Davies leaving, I think the show will lose its identity."* —
   `news` vs. `reaction_opinion`. Cites a real departure but the point is a value
   judgment → **`reaction_opinion`** (evaluation takes precedence over the fact).
2. *"The clues were everywhere — I still think Marvel is setting up Mephisto next
   season."* — `reaction_opinion` vs. `speculation`. Opinion-shaped, but the core
   claim is an unconfirmed future prediction → **`speculation`**.
3. *"Now that the writers have changed, I bet the next season will be darker."* —
   `news` vs. `speculation`. The production fact is only evidence for a forecast →
   **`speculation`** (primary intent is predicting).
## 2. Hyperparameter Tuning

The initial fine-tuned model performed poorly, achieving only 0.567 accuracy and failing to predict any examples of the `reaction_opinion` class. To improve performance, I adjusted two training hyperparameters:

* Increased the number of training epochs from **3 to 5**
* Reduced the training batch size from **16 to 8**

Increasing the number of epochs allowed the model to make additional passes through the small training dataset (~140 training examples after the train/validation/test split). This helped the model learn category-specific language patterns, particularly those associated with subjective opinions.

Reducing the batch size increased the number of weight updates performed during each epoch. With a small dataset, the additional updates helped the model learn finer distinctions between classes, especially the difficult boundary between `reaction_opinion` and `speculation`.

These changes improved performance substantially:

| Configuration           | Accuracy | Macro F1 |
| ----------------------- | -------- | -------- |
| 3 epochs, batch size 16 | 0.567    | 0.48     |
| 5 epochs, batch size 16 | 0.767    | 0.75     |
| 5 epochs, batch size 8  | 0.800    | 0.79     |

The final configuration achieved 0.800 accuracy and a macro F1-score of 0.79, nearly matching the zero-shot LLM baseline. The largest improvement occurred in the `reaction_opinion` class, whose recall increased from 0.00 to 1.00 after the hyperparameter adjustments.


## 3. Evaluation Report

Metrics computed on the 30-example test set.
Raw numbers: [evaluation_results.json](result/evaluation_results.json) ·
Image: [confusion_matrix.png](result/confusion_matrix.png).

**Baseline setup:** Groq `llama-3.3-70b-versatile`, zero-shot, run on the same 30
test examples. The prompt supplied the three label definitions and decision rules
from planning.md and instructed the model to *output only one label name and
nothing else* (so the notebook could parse responses cleanly). No examples or
fine-tuning — purely the general model applying the written rules.

### 3.1 Overall — Both Models

| Model | Accuracy | Macro F1 |
| :--- | :--- | :--- |
| Zero-shot baseline (Groq) | 0.767 | 0.80 |
| **Fine-tuned DistilBERT** | **0.800** | **0.79** |

Fine-tuning improved accuracy by +3.3 points.

### 3.2 Per-Class — Both Models

**Baseline:**

| Label | P | R | F1 |
| :--- | :-: | :-: | :-: |
| reaction_opinion | 0.67 | 0.91 | 0.77 |
| news | 0.89 | 0.89 | 0.89 |
| speculation | 1.00 | 0.60 | 0.75 |

**Fine-tuned:**

| Label | P | R | F1 |
| :--- | :-: | :-: | :-: |
| reaction_opinion | 0.69 | 1.00 | 0.81 |
| news | 1.00 | 0.89 | 0.94 |
| speculation | 0.83 | 0.50 | 0.63 |

Fine-tuning *moved* the problem: it now catches every opinion (recall 1.00) but
does so by dumping half of all speculation into the opinion bucket.

### 3.3 Confusion Matrix — Fine-Tuned

Rows = true, columns = predicted.

| True \ Pred | reaction_opinion | news | speculation |
| :--- | :---: | :---: | :---: |
| **reaction_opinion** | **11** | 0 | 0 |
| **news** | 0 | **8** | 1 |
| **speculation** | 5 | 0 | **5** |

Diagonal = 24/30 = 0.80. **5 of 6 errors are `speculation` → `reaction_opinion`** —
one directional boundary the model never learned.

### 3.4 Misclassified Examples

| # | True | Pred | Conf | Post (excerpt) |
| :-: | :-- | :-- | :--: | :-- |
| 1 | speculation | reaction_opinion | 0.82 | "...I am convinced that Gustavo Fring has...magical powers..." |
| 2 | speculation | reaction_opinion | 0.72 | "...Ireland is convinced she is going into the villa..." |
| 3 | speculation | reaction_opinion | 0.76 | "...would qualify for 2026. I hope they get a nod..." |
| 4 | news | speculation | 0.56 | "The Court has entered an order granting plaintiffs' motion..." |
| 5 | speculation | reaction_opinion | 0.64 | "HBO will likely have a strong year...should be a contender..." |
| 6 | speculation | reaction_opinion | 0.52 | "...I'm still convinced Tyrion is actually a Targaryen..." |

**#1 (conf 0.82, speculation→opinion)** — A character theory, but the model keyed
on the stance marker "I am convinced" and read conviction as opinion. High
confidence shows it *learned* this wrong rule. → boundary/label-overlap problem.

**#3 (speculation→opinion)** — Genuinely mixed: the prediction (Emmy eligibility)
is wrapped in "I hope...this was really good." A human applying my own Rule A
could label it opinion too. → annotation-boundary ambiguity, not a clean error.

**#4 (news→speculation, conf 0.56)** — A real court-docket fact, but jargon-heavy
and unlike my clean press-release `news` examples; the model nearly guessed. →
training-data-distribution problem: my `news` examples were stylistically too narrow.

### 3.5 AI-Surfaced Patterns (verified)

I pasted the 6 errors into an LLM. **Kept:** the errors cluster on first-person
conviction phrasing and on the speculation↔opinion pair — both confirmed against
the matrix. **Discarded:** its claim that "short, low-info posts" drove the
errors — the misclassified posts are actually long.

### 3.6 Sample Classifications

| Post (excerpt) | Pred | Conf | Correct? |
| :-- | :-- | :--: | :--: |
| "Fox News' Chris Wallace, NBC's Kristen Welker, and C-SPAN's Steve Scully will each moderate one of three Presidential Debates" | news | 0.91 | ✅ |
| "Pickle Rick. It's not bad, but I'd never think of it when making a list of my favorite episodes." | reaction_opinion | 0.91 | ✅ |
| "We all saw how, in the latest episode of Peacemaker S2, the group evacuated the Earth X dimension, with the only open question being..." | speculation | 0.88 | ✅ |
| "...I'm still convinced Tyrion is a Targaryen..." | reaction_opinion | 0.52 | ❌ (true: speculation) |
| "The Court has entered an order..." | speculation | 0.56 | ❌ (true: news) |

The correct `news` prediction (conf 0.91) is reasonable: it's a confirmed,
already-decided industry fact in clean announcement voice, with no first-person
evaluation — exactly the `news` pattern, which the model learned cleanly (news
precision = 1.00). Note the contrast in confidence: the model is decisive on the
three correct in-pattern cases (0.88–0.91) but near-guessing on its errors
(0.52, 0.56) — its confidence is at least directionally meaningful.

## 4. Reflection: Captured vs. Intended

I defined labels by **intent** (report / evaluate / predict). The model learned
**surface stance markers** instead. `reaction_opinion` recall (1.00) and `news`
precision (1.00) are perfect because those classes have distinctive surface
signatures; `speculation` has none of its own — it borrows opinion's first-person
framing — so it collapsed to 0.50 recall. **Overfit to:** "I am convinced/hope/
think" → opinion. **Missed:** the predictive function of a sentence, which never
became a feature the model could see.

## 5. Definition of Success

| Criterion | Target | Result | Met? |
| :-- | :-- | :-- | :-: |
| Macro F1 | ≥ 0.75 | 0.79 | ✅ |
| `news` precision | ≥ 0.85 | 1.00 | ✅ |
| `news` recall | ≥ 0.80 | 0.89 | ✅ |
| `reaction_opinion` recall | ≥ 0.80 | 1.00 | ✅ |
| `speculation` recall | ≥ 0.65 | **0.50** | ❌ |

Meets 4/5, including the strict `news`-precision constraint, but fails the
`speculation` floor — so it is not yet deployment-ready. **Fix:** more
speculation examples that *share surface form with opinion*, a tighter
speculation/opinion definition, and a wider stylistic range of `news` examples.

## 6. Spec Reflection

**Helped:** requiring per-class metrics + a confusion matrix surfaced the whole
finding — 0.80 accuracy alone would have hidden the directional
speculation→opinion leak that drives every conclusion here.

**Diverged:** my plan targeted balanced 60–80 examples per class, but
`reaction_opinion` was far easier to collect than `speculation`, so my data
skewed toward opinion. I accepted the imbalance rather than padding with
duplicates — in hindsight, a direct cause of the speculation-recall failure.

## 7. AI Usage

**Annotation pre-labeling (disclosed):** I had an LLM pre-label ~50 raw posts with
my taxonomy. I audited 100% and overrode its tendency to label clue-based theories
as `news`. Pre-labeled rows tracked via an `is_prelabeled_by_ai` flag; final
labels are mine.

**Failure analysis:** I had an LLM find themes across the 6 errors; I kept the
two that matched the matrix and discarded the false "short posts" theme (§2.5).

## 8. Demo

Loom link: https://www.loom.com/share/d137f7421aee4273861aeda60acb35a4

## Files

- [planning.md](planning.md) — design notes & edge cases
- [reddit.csv](reddit.csv) — dataset
- [result/evaluation_results.json](result/evaluation_results.json) — raw metrics
- [result/confusion_matrix.png](result/confusion_matrix.png) — confusion matrix
</content>
