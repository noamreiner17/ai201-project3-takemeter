# Project Plan: Reddit Discourse Classifier (r/television)

## 1. Community & Taxonomy Design

### Community Description
The chosen community is **r/television**, a text-heavy subreddit focused on television industry news, episode reviews, and viewer discussion. The discourse ranges from objective industry announcements to passionate, subjective debates. Distinguishing between facts, feelings, and theories matters immensely because low-effort emotional outbursts often clutter the feed, making it difficult for users to isolate verified industry updates or engaging narrative theories. This community is a perfect fit for a classification task because its natural lifecycle maps directly onto clean discourse boundaries: a studio drops a factual announcement (`news`), fans instantly express how they feel about it (`reaction_opinion`), and then they try to figure out what happens next (`speculation`).

---

### Label Taxonomy

| Label | One-Sentence Definition |
| :--- | :--- |
| `news` | The post shares verifiable facts, official press releases, casting announcements, ratings data, or industry trade articles without inserting personal bias. |
| `reaction_opinion` | The post expresses a user's personal emotional response, subjective critique, or value judgment regarding an episode, character, or show event. |
| `speculation` | The post predicts future plot points, character fates, casting choices, or cancellation/renewal outcomes based on theories, clues, or behind-the-scenes rumors. |

#### `news` Examples
* **Example 1:** *"HBO has officially renewed 'The Last of Us' for a third season, with production slated to begin in Vancouver this fall."*
* **Example 2:** *"The final episode of the limited series pulled in 4.2 million viewers across streaming and linear cable on Sunday night."*

#### `reaction_opinion` Examples
* **Example 1:** *"I'm sorry, but that finale felt incredibly rushed. They spent three seasons building up that villain just to defeat him in five minutes."*
* **Example 2:** *"The chemistry between the two lead actors is easily the best part of the show. I could watch them just talk in a room for an hour."*

#### `speculation` Examples
* **Example 1:** *"Based on the comic book storyline, I am 90% sure that the main character's brother is going to betray the group in the mid-season finale."*
* **Example 2:** *"The showrunner just posted a cryptic photo of a clock on Instagram—I bet they are planning a massive time-jump for season 4."*

---

### Edge Cases and Decision Rules

#### Edge Case 1: The Opinionated Rumor (`reaction_opinion` vs. `news` / `speculation`)
* **Uncertain Post:** *"Rumors are circulating online that Marvel is looking to recast the lead actor for the upcoming series because of scheduling conflicts, which would honestly ruin the entire project."*
* **The Ambiguity:** It mentions a real-world industry situation (`news`), guesses at the future of an unreleased project (`speculation`), and ends with a heavy value judgment (`reaction_opinion`).
* **Decision Rule:** If a post contains a personal, subjective value judgment (e.g., "ruin", "bad", "terrible", "love"), classify it as `reaction_opinion` — unless the post is a direct, unaltered link to a reputable trade article. If a user paraphrases a rumor and adds their emotional take, the text signal is fundamentally an opinion.
* **Final Verdict:** `reaction_opinion`

#### Edge Case 2: The Soft Announcement (`news` vs. `speculation`)
* **Uncertain Post:** *"A writer on the show just tweeted an emoji of a ghost. Looks like a major character is dying next week."*
* **The Ambiguity:** It anchors itself on a verifiable fact/action (`news`), but immediately uses it to guess a narrative outcome (`speculation`).
* **Decision Rule:** If the primary intent of the post text is to guess an unconfirmed future event based on a clue, classify it as `speculation`. It only qualifies as `news` if the event or announcement is confirmed as factual by the source.
* **Final Verdict:** `speculation`

---

## 2. Data Collection Plan

### Sourcing Strategy
The data will be collected from active threads within the `r/television` subreddit. Post titles and top-level comment text will be compiled and organized into a structured spreadsheet/CSV pipeline. 

### Volume Goals
The goal is to annotate a minimum baseline dataset of **200 distinct examples**. To ensure balance across the categories, the target breakdown is **70-80 examples per label**.

### Mitigation for Underrepresented Labels
If certain categories (such as `speculation` or `news`) are lagging after an initial collection pass:
1. **Targeted Subreddit Keyword Search:** I will query historical threads using high-signal search triggers. For example, filtering by keywords like *"theory"*, *"predict"*, or *"leak"* to isolate `speculation`, and searching for terms like *"deadline"*, *"variety"*, or *"renewed"* to boost `news` counts.
2. **Neighboring Sources:** Instead of artificially duplicating rows—which would cause the model to overfit on identical text patterns—I will gather text samples from highly identical sister communities such as `r/television_news`, `r/streaming`, or `r/TrueFilm` if a class remains sparse.

---

## 3. Evaluation Metrics

To thoroughly measure model performance, relying on global accuracy alone is insufficient. Because the dataset size is small and text patterns can vary wildly, I will track the following targeted multi-class evaluation parameters:
* **Per-Class Accuracy:** This will serve as our primary multi-class metric (calculated as `correct [class] predictions ÷ total [class] episodes in test set`). This reveals what global accuracy hides, ensuring that a high score in `reaction_opinion` doesn't mask a total failure to learn the `speculation` class.
* **Directional Error Analysis (via Confusion Matrix):** We will track the asymmetry of misclassifications (e.g., how often `speculation` leaks into `reaction_opinion` vs. vice versa). Asymmetrically skewed errors will diagnose exactly where our label boundaries are underspecified in the training data.

---

## 4. Definition of Success

### Deployment Requirements
For this classifier to be genuinely useful as a practical moderation or user-facing filtering tool on Reddit, it must reliably filter out low-effort chatter without accidentally suppressing genuine, breaking news. 

### Objective and Specific Criteria for Success
At the end of evaluation, success will be determined objectively by checking if the fine-tuned model hits or exceeds the following independent thresholds on the held-out test set:

1. **Overall Macro F1-Score:** **$\ge 0.75$**. This calculates the F1-score for all three classes individually and takes their unweighted mean. Hitting $\ge 0.75$ mathematically guarantees that performance remains stable across both common classes (`reaction_opinion`) and more elusive classes (`speculation`).

2. **`news` Class Precision:** **$\ge 0.85$**. Defined precisely as `True News Predictions ÷ (True News + False News Predictions)`. This constraint forces the model to keep its False Positive rate for news below 15%, ensuring that the "Facts-Only" feed option remains clean and completely free of subjective noise or rumor threads.

3. **Per-Class Accuracy Floors:**
   * `news` Accuracy: $\ge 0.80$
   * `reaction_opinion` Accuracy: $\ge 0.80$
   * `speculation` Accuracy: $\ge 0.65$ (Allowing a lower floor due to smaller sample representations and linguistic crossover with opinions).

### Operational Goal
If all three criteria are met, the classifier will be deemed "good enough" for production deployment. It will be implemented inside a user-facing dashboard where users can cleanly toggle checkboxes to see *only* verified industrial updates (`news`) or *only* narrative brainstorms (`speculation`), effectively eliminating generic internet noise.

---

## 5. AI Tool Plan

Because this is a data-centric and design-centric project, Gemini will be utilized at three specific bottlenecks to test, accelerate, and refine our data workflows.

### 1. Label Stress-Testing

* **Workflow:** Before annotating the full 200-post dataset, I will input our current label definitions, guidelines, and known edge cases into Gemini. I will prompt Gemini to leverage its integrated Google Search capabilities to find or generate 5–10 highly realistic, ambiguous posts that sit directly on the boundary lines between `news` vs. `speculation` and `reaction_opinion` vs. `speculation`.

* **Action Item:** If Gemini returns boundary examples that cannot be confidently sorted into a single category using our current decision boundaries, I will pause and tighten the text taxonomy *before* beginning human annotation.

### 2. Annotation Assistance
* **Workflow:** To establish an efficient data pipeline, I will use Gemini to pre-label an initial raw batch of 50 crawled examples. I will pass the text along with our exact Markdown taxonomy table and instructions to output labels in a structured CSV format.

* **Verification & Disclosure:** I will manually audit 100% of Gemini's outputs against our ground-truth rules to correct any systemic hallucinations or misclassifications. In the data spreadsheet, a tracking column titled `is_prelabeled_by_ai` (Boolean: `TRUE`/`FALSE`) will be maintained to explicitly log and disclose AI usage for data curation.

### 3. Failure Analysis
* **Workflow:** Following model evaluation, I will isolate all rows from our test set where the fine-tuned model's predictions mismatched the ground-truth labels. I will feed this collection of misclassified texts into Gemini.

* **Analysis & Verification:** I will instruct Gemini to perform an algorithmic review to discover underlying linguistic patterns—such as identifying specific trigger tokens, syntactic constructions, or hidden entity cross-overs that systematically caused the model to fail. I will manually cross-verify Gemini’s findings against our confusion matrix to check for directional error patterns before compiling the final performance report.