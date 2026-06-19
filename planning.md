## Milestone 1: Community & Taxonomy Design

### Community Description
The chosen community is **r/television**, a text-heavy subreddit focused on television industry news, episode reviews, and viewer discussion. The discourse ranges from objective industry announcements to passionate, subjective debates. Distinguishing between facts, feelings, and theories matters immensely because low-effort emotional outbursts often clutter the feed, making it difficult for users to isolate verified industry updates or engaging narrative theories.

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
* **Decision Rule:** If a post contains a personal, subjective value judgment (*e.g., "good", "bad", "ruined", "love"*), classify it as `reaction_opinion`—unless the post is a direct, unaltered link to a reputable trade article. If a user paraphrases a rumor and adds their emotional take, the text signal is fundamentally an opinion.
* **Final Verdict:** `reaction_opinion`

#### Edge Case 2: The Soft Announcement (`news` vs. `speculation`)
* **Uncertain Post:** *"A writer on the show just tweeted an emoji of a ghost. Looks like a major character is dying next week."*
* **The Ambiguity:** It anchors itself on a verifiable fact/action (`news`), but immediately uses it to guess a narrative outcome (`speculation`).
* **Decision Rule:** If the primary intent of the post text is to guess an unconfirmed future event based on a clue, classify it as `speculation`. It only qualifies as `news` if the event or announcement is confirmed as factual by the source.
* **Final Verdict:** `speculation`