
# Defining the labels

| Label | Scope | Grounded Criteria |
| :--- | :--- | :--- |
| **`news`** | Objective Facts | Verifiable industry data, official press releases, casting confirmations, ratings, or network announcements. Must be devoid of personal bias. |
| **`reaction_opinion`** | Subjective Feelings | Emotional responses, reviews, critiques, and value judgments regarding episodes, characters, or real-world show events. |
| **`speculation`** | Predictive Theories | Hypotheses regarding unreleased plotlines, future character arcs, cancellation/renewal odds, or interpretations of cryptic teaser clues. |

### 🛠️ Concrete Annotation Examples

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

### 🔮 Strict Decision Boundaries (Edge Cases)

When a post is borderline and could reasonably fit multiple labels, annotators must strictly apply the following logic tree:

#### Rule A: The Opinionated Rumor (`reaction_opinion` vs. `news`/`speculation`)
* **The Problem:** A post references a real industry rumor, guesses at its outcome, but expresses a clear emotional bias.
* **Example:** *"Rumors are circulating online that Marvel is looking to recast the lead actor for the upcoming series because of scheduling conflicts, which would honestly ruin the entire project."*
* **Operational Rule:** Look for subjective adjective triggers (*e.g., "ruin", "awesome", "terrible", "love"*). If the post text explicitly inserts a personal value judgment on top of a rumor, it defaults to **`reaction_opinion`**. It can only be `news` if it is an objective, unedited headline or official quote.

#### Rule B: The Clue-Driven Theory (`speculation` vs. `news`)
* **The Problem:** A post cites a verified real-world fact to immediately make a guess about an unreleased project.
* **Example:** *"A writer on the show just tweeted an emoji of a ghost. Looks like a major character is dying next week."*
* **Operational Rule:** Evaluate the primary intent of the text. If the text shifts from stating a fact to drawing an unconfirmed conclusion or prediction, classify it as **`speculation`**. It remains `news` if and only if the event itself has already factually transpired or been formally confirmed by the network.