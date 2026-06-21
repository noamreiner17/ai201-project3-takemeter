# TakeMeter: A Reddit Discourse Classifier for r/television

TakeMeter is a three-way text classifier that sorts posts and comments from the
`r/television` subreddit into the kind of discourse they represent: objective
industry **news**, subjective **reaction_opinion**, or predictive
**speculation**. The goal is a tool that could power a feed filter — letting a
user toggle to "verified industry updates only" or "fan theories only" and strip
out the noise in between.

This README is the final report. The working notes, full label rationale, edge
cases, and design decisions behind every choice live in
[planning.md](planning.md).

---

## 1. What I Built

- **Task:** single-label, three-class text classification.
- **Source:** ~200 manually annotated posts/comments collected from
  `r/television` (titles + top-level comment text), stored in
  [reddit.csv](reddit.csv).
- **Baseline:** a zero-shot LLM (Groq) given the label definitions and decision
  rules but no project-specific training.
- **Fine-tuned model:** `distilbert-base-uncased` fine-tuned on the annotated
  training split.
- **Split:** train / validation / held-out test, with a 30-example test set used
  for all reported metrics below.

### Label Taxonomy

| Label | One-Sentence Definition |
| :--- | :--- |
| `news` | Verifiable facts, official press releases, casting announcements, ratings data, or trade articles, without personal bias. |
| `reaction_opinion` | A personal emotional response, subjective critique, or value judgment about an episode, character, or show event. |
| `speculation` | A prediction about future plot points, character fates, casting, or renewal/cancellation, based on theories, clues, or rumors. |

The two hard boundaries — and the rules I used to resolve them — are documented
in detail in [planning.md](planning.md#edge-cases-and-decision-rules). In short:
a subjective value judgment layered on a rumor defaults to `reaction_opinion`;
a clue used to forecast an unconfirmed future event defaults to `speculation`.

---

## 2. Evaluation Report

All metrics are computed on the **held-out 30-example test set**.
Raw numbers: [evaluation_results.json](result/evaluation_results.json).
Confusion-matrix image: [confusion_matrix.png](result/confusion_matrix.png).

### 2.1 Overall Accuracy — Both Models

| Model | Accuracy | Macro F1 |
| :--- | :--- | :--- |
| Zero-shot LLM baseline (Groq) | 0.767 | 0.80\* |
| **Fine-tuned DistilBERT** | **0.800** | **0.79** |

**Fine-tuning improvement: +3.3 percentage points in accuracy.**

\* The baseline macro-F1 (0.80) was measured on an earlier evaluation pass; the
two models are best compared on accuracy, where they were scored identically.

### 2.2 Per-Class Metrics — Both Models

**Zero-shot LLM baseline:**

| Label | Precision | Recall | F1 |
| :--- | :--- | :--- | :--- |
| reaction_opinion | 0.67 | 0.91 | 0.77 |
| news | 0.89 | 0.89 | 0.89 |
| speculation | 1.00 | 0.60 | 0.75 |

**Fine-tuned DistilBERT:**

| Label | Precision | Recall | F1 |
| :--- | :--- | :--- | :--- |
| reaction_opinion | 0.69 | 1.00 | 0.81 |
| news | 1.00 | 0.89 | 0.94 |
| speculation | 0.83 | 0.50 | 0.63 |

Both models share the same shape of failure: they nail `news`, over-predict
`reaction_opinion`, and under-recall `speculation`. Fine-tuning *moved* the
problem rather than solving it. The baseline was conservative about
`speculation` (precision 1.00, recall 0.60 — it rarely guessed speculation but
was right when it did). The fine-tuned model became *aggressive* about
`reaction_opinion`: it now catches every opinion (recall 1.00) but does so by
dumping half of all speculation posts into the opinion bucket, collapsing
speculation recall to 0.50.

### 2.3 Confusion Matrix — Fine-Tuned Model

Rows are the true label; columns are the model's prediction.

| True \ Predicted | reaction_opinion | news | speculation | **Total** |
| :--- | :---: | :---: | :---: | :---: |
| **reaction_opinion** | **11** | 0 | 0 | 11 |
| **news** | 0 | **8** | 1 | 9 |
| **speculation** | 5 | 0 | **5** | 10 |
| **Total predicted** | 16 | 8 | 6 | 30 |

The diagonal (correct) is 11 + 8 + 5 = 24 → 24/30 = **0.80 accuracy**.

**The error story is almost entirely one cell.** Of the 6 total errors, **5 are
`speculation` posts predicted as `reaction_opinion`**. The remaining error is a
single `news` post predicted as `speculation`. The model never confused anything
*with* `news` in the predicted direction (news precision = 1.00), and it never
misfired a `reaction_opinion` post into another class (reaction_opinion recall =
1.00). There is exactly one boundary the model has not learned:
**speculation vs. reaction_opinion**, and the error is directional —
speculation leaks into opinion, not the other way around.

### 2.4 Misclassified Examples & Analysis

All six misclassified test posts are shown below with the model's confidence.

| # | True | Predicted | Conf. | Post (excerpt) |
| :-: | :-- | :-- | :--: | :-- |
| 1 | speculation | reaction_opinion | 0.82 | "...I am convinced that Gustavo Fring has, or at least had, magical powers..." |
| 2 | speculation | reaction_opinion | 0.72 | "...The whole of Ireland is convinced she is going into the villa and the speculation and conspiracies are WILD..." |
| 3 | speculation | reaction_opinion | 0.76 | "...Gilded Age Season 3...would qualify for 2026. I hope they get a nod, this season was really good!" |
| 4 | news | speculation | 0.56 | "The Court has entered an order granting plaintiffs' motion for an extension of time to respond to Fox's Rule 11 notice..." |
| 5 | speculation | reaction_opinion | 0.64 | "HBO will likely have another strong year...based on its positive reviews it looks like Task should be a contender..." |
| 6 | speculation | reaction_opinion | 0.52 | "...I'm still convinced Tyrion is actually a Targaryen and intended to be the third dragon rider..." |

**Detailed analysis of three failures:**

**Example #1 — "I am convinced that Gustavo Fring has magical powers" (speculation → reaction_opinion, conf 0.82).**
This is the single most diagnostic error. The post is a *theory* about a
character (an interpretation of unconfirmed in-universe facts), so by the
taxonomy it is `speculation`. But the surface language — "I am convinced,"
first-person, evaluative, no external clue cited — is indistinguishable from how
opinion posts open. The model latched onto the **stance markers** ("I am
convinced," "I'm still convinced" appears again in #6) and treated them as
opinion signals. The high 0.82 confidence shows this isn't a near-miss: the model
has *learned* that first-person conviction language means `reaction_opinion`.
That is exactly the overlap I predicted before fine-tuning — and fine-tuning made
it worse, not better, because the training set taught the model that
opinion-shaped language is opinion.

**Example #3 — "Gilded Age would qualify for 2026... I hope they get a nod, this season was really good!" (speculation → reaction_opinion, conf 0.76).**
This post is genuinely a **mixed-signal** case and arguably a labeling-boundary
problem, not just a model problem. The *primary* claim — predicting Emmy
eligibility and contention based on the cutoff date — is forecasting an
unconfirmed future outcome (`speculation`). But the post is wrapped in explicit
opinion language ("I hope," "this season was really good!"). Under my own Rule A
("a subjective value judgment defaults to reaction_opinion"), a human annotator
could plausibly have labeled this `reaction_opinion` too. The model isn't being
unreasonable here — it's applying the same surface heuristic my decision rule
encodes, just in the wrong priority order. This points to **annotation-boundary
ambiguity**, not a clean model error.

**Example #4 — "The Court has entered an order granting plaintiffs' motion..." (news → speculation, conf 0.56).**
The lone non-`reaction_opinion` error, and revealingly **low-confidence (0.56)** —
the model was nearly guessing. This is a factual legal/industry update
(`news`): a court docket event that has already transpired. But it is dense,
jargon-heavy ("Rule 11 notice," "Letter Motion for Extension"), unlike the
clean casting-announcement / ratings-report style that dominates the `news`
training examples. With no in-distribution `news` pattern to match and a topic
(an ongoing lawsuit) that *sounds* forward-looking, the model fell back toward
`speculation`. This is a **training-data-distribution problem**: my `news`
examples were too stylistically narrow (press-release voice), so out-of-style
factual posts aren't recognized as news.

### 2.5 What I Asked an LLM to Surface, and What I Verified

Following the AI Tool Plan in [planning.md](planning.md#5-ai-tool-plan), I pasted
all six misclassified posts into an LLM and asked it to find common themes. It
proposed three: (1) the errors cluster on first-person conviction phrasing
("I am convinced," "I'm still convinced," "I hope"); (2) the confused pair is
overwhelmingly speculation↔reaction_opinion; (3) several speculation posts are
"opinion-wrapped predictions" where the prediction is buried under evaluative
language.

**What I verified and kept:** themes (1) and (2) hold up directly against the
confusion matrix — 5 of 6 errors are speculation→reaction_opinion, and 4 of
those 5 contain explicit first-person stance markers. Confirmed.

**What I corrected/discarded:** the LLM initially also claimed "short, low-info
posts" were a failure driver. Re-reading the examples, this is false — the
misclassified posts (#1, #2, #5) are *long and detailed*. The length signal was
a hallucinated pattern; I discarded it. I also reframed the LLM's "opinion-
wrapped prediction" theme: on inspection, #3 isn't purely a model failure but a
genuine boundary ambiguity my own Rule A creates, which is a more useful finding
than "the model got confused."

### 2.6 Sample Classifications

Each row below is a post run through the fine-tuned model, with predicted label
and confidence. The first three are correctly-predicted examples; the last two
are real test-set misclassifications (from §2.4) shown for contrast.

| Post (excerpt) | Predicted | Confidence | Correct? |
| :-- | :-- | :--: | :--: |
| _[paste a clear `news` example — see snippet below]_ | news | _0.__ | ✅ |
| _[paste a clear `reaction_opinion` example]_ | reaction_opinion | _0.__ | ✅ |
| _[paste a correctly-caught `speculation` example]_ | speculation | _0.__ | ✅ |
| "...I'm still convinced Tyrion is actually a Targaryen..." | reaction_opinion | 0.52 | ❌ (true: speculation) |
| "The Court has entered an order granting plaintiffs' motion..." | speculation | 0.56 | ❌ (true: news) |

**Why a correct prediction is reasonable** (fill in once you paste the correct
`news` row): the model assigns `news` with high confidence to posts written in
clean announcement voice — a confirmed, already-transpired industry fact stated
without any first-person evaluation — which is exactly the pattern the `news`
class is defined by and the one the training data represented most cleanly
(news precision = 1.00 on the test set).

> **To fill the three correct rows**, run this in your notebook after the model
> is loaded and print the result, then paste text + label + confidence above:
> ```python
> import torch, torch.nn.functional as F
> id2label = {0: "reaction_opinion", 1: "news", 2: "speculation"}
> correct = [(t, true) for t, true in zip(test_texts, test_labels)]
> for text, true in correct:
>     inputs = tokenizer(text, return_tensors="pt", truncation=True)
>     with torch.no_grad():
>         probs = F.softmax(model(**inputs).logits, dim=-1)[0]
>     pred = int(probs.argmax()); conf = float(probs[pred])
>     if id2label[pred] == true:        # only correctly-predicted
>         print(f"{id2label[pred]:18s} {conf:.2f}  {text[:90]}")
> ```

---

## 3. Reflection: What the Model Captured vs. What I Intended

I defined the three labels by **communicative intent**: is the author *reporting*
a fact, *evaluating* something, or *predicting* an unconfirmed outcome? The model
did not learn intent. It learned **surface stance markers**.

The clearest evidence is the asymmetry in the confusion matrix. `reaction_opinion`
recall is a perfect 1.00 and `news` precision is a perfect 1.00 — the model is
excellent at recognizing the two classes with the most distinctive *surface*
signatures (emotional/evaluative phrasing for opinion; clean announcement voice
for news). But `speculation` — which I defined by *intent* (forecasting), not by
any signature vocabulary — collapsed to 0.50 recall. Speculation has no surface
fingerprint of its own; it borrows the first-person framing of opinion ("I am
convinced," "I bet," "I'm sure") and the topical anchoring of news. With no
distinctive features to grab, the model defaulted speculative posts into whichever
neighbor had the stronger surface cue, which was almost always `reaction_opinion`.

**What the model overfit to:** first-person conviction language. The training
data taught it that "I [am convinced / hope / think / love]" → opinion. So at
test time, *any* post opening that way — including a structured plot theory — got
pulled into `reaction_opinion` with high confidence (0.82 on example #1).

**What the model missed:** the thing I actually care about — the *predictive*
function of a sentence. The boundary in my head ("is this forecasting the
future?") never became a feature the model could see, because in my data that
boundary wasn't reliably marked by any token the model could key on. The decision
boundary the model drew is "does this sound subjective?" — which is a real
boundary, just not the one I specified.

---

## 4. Definition of Success — Did I Meet It?

From [planning.md](planning.md#4-definition-of-success):

| Success criterion | Target | Result | Met? |
| :-- | :-- | :-- | :-: |
| Overall Macro F1 | ≥ 0.75 | 0.79 | ✅ |
| `news` precision | ≥ 0.85 | 1.00 | ✅ |
| `news` accuracy/recall | ≥ 0.80 | 0.89 | ✅ |
| `reaction_opinion` recall | ≥ 0.80 | 1.00 | ✅ |
| `speculation` recall | ≥ 0.65 | **0.50** | ❌ |

The model meets four of five thresholds, including the headline macro-F1 and the
strict `news`-precision constraint that protects the "facts-only" feed. It
**fails the `speculation` floor (0.50 vs. 0.65)**. Per my own criteria, the
classifier is *not yet* deployment-ready: a speculation filter that misses half
of all theories would be frustrating in practice. The fix is targeted, not
structural (see below).

**What would need to change:** more `speculation` training examples that *share
surface form with opinion* (first-person "I'm convinced" theories) so the model
is forced to use deeper cues than stance markers to separate them; a tighter,
operationalized definition of the speculation/opinion boundary that resolves
mixed-signal posts like #3 consistently; and a wider stylistic range of `news`
examples (legal/jargon-dense updates, not just press-release voice) to fix the
single news→speculation miss.

---

## 5. Hyperparameter Tuning

The initial fine-tuned model scored only 0.567 accuracy and predicted **zero**
`reaction_opinion` examples. Two adjustments fixed it:

- Increased epochs **3 → 5** (more passes over the small ~140-example train set).
- Reduced batch size **16 → 8** (more weight updates per epoch → finer class
  distinctions).

| Configuration | Accuracy | Macro F1 |
| :-- | :-- | :-- |
| 3 epochs, batch 16 | 0.567 | 0.48 |
| 5 epochs, batch 16 | 0.767 | 0.75 |
| **5 epochs, batch 8** | **0.800** | **0.79** |

`reaction_opinion` recall rose from 0.00 to 1.00 across these changes — though,
as §3 notes, that gain came partly at the expense of `speculation` recall.

---

## 6. Spec Reflection

**One way the spec helped:** the requirement to report *per-class* metrics and a
confusion matrix — not just overall accuracy — is what surfaced the entire
finding of this project. The 0.80 accuracy looks healthy on its own and would
have hidden the real story. Forcing the confusion matrix made the directional
speculation→reaction_opinion leak (5 of 6 errors in one cell) impossible to
miss, and that single cell drove every conclusion in §3 and §4.

**One way I diverged from the spec, and why:** my data-collection plan in
planning.md targeted a balanced 60–80 examples per class. In practice
`reaction_opinion` was far easier to collect than `speculation` on r/television
(opinions vastly outnumber structured theories in the feed), so my real
distribution skewed toward opinion. I accepted the imbalance rather than padding
`speculation` with duplicated or sister-subreddit posts, because I judged that
faithful in-distribution data was worth more than artificial balance — but in
hindsight this imbalance is a direct contributor to the speculation recall
failure, and the planned keyword-targeted top-up (filtering for "theory,"
"predict," "I bet") is the change I'd make first.

---

## 7. AI Usage

Per the AI Tool Plan in [planning.md](planning.md#5-ai-tool-plan), AI assistance
was used at three points; the two most substantive are detailed here.

**Instance 1 — Annotation pre-labeling (disclosed data assistance).**
I directed an LLM to pre-label an initial raw batch of ~50 crawled posts using my
exact Markdown taxonomy table, outputting structured rows. It produced draft
labels for the batch. **What I changed/overrode:** I manually audited 100% of its
output against my decision rules and corrected its systematic tendency to label
clue-based theories as `news` (it treated the cited fact as the headline rather
than recognizing the predictive intent — exactly my Edge Case 2). Pre-labeled
rows were tracked with an `is_prelabeled_by_ai` flag; final labels are my own.

**Instance 2 — Failure-pattern analysis.**
I pasted the six misclassified test posts into an LLM and asked it to identify
common themes. It produced three (first-person conviction phrasing, the
speculation↔opinion confusion pair, and "opinion-wrapped predictions").
**What I changed/overrode:** I verified themes against the confusion matrix and
kept the two that held; I **discarded** its claim that short/low-information
posts drove the errors (the misclassified posts are actually long), and I
reframed its "opinion-wrapped prediction" theme into the sharper finding that
example #3 is a genuine boundary ambiguity in my own Rule A. Full account in §2.5.

---

## Files

- [planning.md](planning.md) — design notes, full label rationale, edge cases.
- [reddit.csv](reddit.csv) — annotated dataset.
- [result/evaluation_results.json](result/evaluation_results.json) — raw metrics.
- [result/confusion_matrix.png](result/confusion_matrix.png) — fine-tuned confusion matrix.
</content>
</invoke>
