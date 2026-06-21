
### Defining the labels

| Label | Scope | Grounded Criteria |
| :--- | :--- | :--- |
| **`news`** | Objective Facts | Verifiable industry data, official press releases, casting confirmations, ratings, or network announcements. Must be devoid of personal bias. |
| **`reaction_opinion`** | Subjective Feelings | Emotional responses, reviews, critiques, and value judgments regarding episodes, characters, or real-world show events. |
| **`speculation`** | Predictive Theories | Hypotheses regarding unreleased plotlines, future character arcs, cancellation/renewal odds, or interpretations of cryptic teaser clues. |

### Concrete Annotation Examples

#### `news`
* **Text:** *"HBO has officially renewed 'The Last of Us' for a third season, with production slated to begin in Vancouver this fall."*
* **Text:** *"The final episode of the limited series pulled in 4.2 million viewers across streaming and linear cable on Sunday night."*

#### `reaction_opinion`
* **Text:** *"I'm sorry, but that finale felt incredibly rushed. They spent three seasons building up that villain just to defeat him in five minutes."*
* **Text:** *"The chemistry between the two lead actors is easily the best part of the show. I could watch them just talk in a room for an hour."*

#### `speculation`
* **Text:** *"Based on the comic book storyline, I am 90% sure that the main character's brother is going to betray the group in the mid-season finale."*
* **Text:** *"The showrunner just posted a cryptic photo of a clock on Instagram—I bet they are planning a massive time-jump for season 4."*

---

###  Strict Decision Boundaries (Edge Cases)

When a post is borderline and could reasonably fit multiple labels, annotators must strictly apply the following logic tree:

#### Rule A: The Opinionated Rumor (`reaction_opinion` vs. `news`/`speculation`)
* **The Problem:** A post references a real industry rumor, guesses at its outcome, but expresses a clear emotional bias.
* **Example:** *"Rumors are circulating online that Marvel is looking to recast the lead actor for the upcoming series because of scheduling conflicts, which would honestly ruin the entire project."*
* **Operational Rule:** Look for subjective adjective triggers (*e.g., "ruin", "awesome", "terrible", "love"*). If the post text explicitly inserts a personal value judgment on top of a rumor, it defaults to **`reaction_opinion`**. It can only be `news` if it is an objective, unedited headline or official quote.

#### Rule B: The Clue-Driven Theory (`speculation` vs. `news`)
* **The Problem:** A post cites a verified real-world fact to immediately make a guess about an unreleased project.
* **Example:** *"A writer on the show just tweeted an emoji of a ghost. Looks like a major character is dying next week."*
* **Operational Rule:** Evaluate the primary intent of the text. If the text shifts from stating a fact to drawing an unconfirmed conclusion or prediction, classify it as **`speculation`**. It remains `news` if and only if the event itself has already factually transpired or been formally confirmed by the network.

--- 

## Baseline Model Results -

Before fine-tuning, I evaluated a zero-shot LLM baseline on the held-out test set. The model was given the project label definitions and classification rules but had not been trained on any project-specific examples.

### Performance

| Metric   | Score |
| -------- | ----- |
| Accuracy | 0.80  |
| Macro F1 | 0.80  |

Per-class performance:

| Label            | Precision | Recall | F1   |
| ---------------- | --------- | ------ | ---- |
| reaction_opinion | 0.67      | 0.91   | 0.77 |
| news             | 0.89      | 0.89   | 0.89 |
| speculation      | 1.00      | 0.60   | 0.75 |

### Baseline Analysis

The baseline performed best on the `news` category, achieving an F1-score of 0.89. This suggests that factual reporting language, such as official announcements, ratings reports, and production updates, is relatively easy for a general-purpose language model to recognize.

The primary weakness was the `speculation` category, which achieved a recall of only 0.60. While the model was very precise when predicting `speculation` (precision = 1.00), it failed to identify many speculative posts.

### Hypothesis Before Fine-Tuning

I hypothesize that the model frequently confuses `speculation` and `reaction_opinion`. Many speculative posts contain phrases such as "I think," "I believe," or "my theory is," which resemble opinion language even when the primary purpose of the post is predicting an unconfirmed future event. Because of this overlap, the baseline likely labels some speculative posts as `reaction_opinion`.

Fine-tuning on the manually annotated dataset should help the model learn the project-specific distinction between expressing an opinion and making a prediction, improving performance on the `speculation` class.


### Hyperparameter Tuning

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


Evaluation files: 

## Evaluation Files

- [evaluation_results.json](evaluation_results.json)
- [confusion_matrix.png](confusion_matrix.png)

## Baseline vs Fine-Tuned Model

| Model | Accuracy |
|--------|----------|
| Zero-shot Baseline (Groq) | 0.767 |
| Fine-tuned DistilBERT | **0.800** |

**Fine-tuning Improvement:** +0.033 (+3.3 percentage points)

