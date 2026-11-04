# Labeling Guide

Draft v0. This guide defines how I hand-label real posts and images for the evaluation set. It will change after the pilot round. Real Bangla and Banglish examples will be added from the pilot.

## Ground rules

- **The unit is one post** (its text) or **one image**. Label what is on the screen, not what you guess about the author.
- **Label the wording, not the person.** The tags describe the text. They are not verdicts on anyone.
- **Never save author names, profile links, or post URLs** in the labeled set. Save text and image content only.
- **Saving is deliberate.** Only posts I choose to save enter the set. The set stays on my machine and is git-ignored.
- **When unsure, use `unclear`** instead of guessing. Count how often it is used.
- **Do not label religion as a topic.** There is no religion tag. Hostility aimed at a religion is labeled as Group hate.

## Text tags

A post can have several tags. A post with no tag is `neutral`.

| Tag | Use it when the post... | Do not use it for |
| --- | --- | --- |
| **Threat** | says or wishes that someone should be hurt or killed, or says the writer will do it | News reporting a threat; quoting a threat to criticize it; jokes and idioms ("I'd kill for a biryani") |
| **Personal attack** | insults, mocks, or degrades an individual (name-calling, slurs, mocking appearance, pile-ons) | Plain disagreement or harsh but fair criticism |
| **Group hate** | demeans or dehumanizes a whole group because of identity | Criticizing a policy, an event, or a single act without targeting the group |
| **Accusation** | claims a person or group did something wrong, usually without evidence | Reporting a verified court case or an official finding |

**Group hate target (optional).** If a post is Group hate, record who it is aimed at: `gender` (add `toward women` or `toward men`), `politics`, `origin` (ethnic or national), or `other`. Posts that demean men are treated the same as posts that demean women.

### Examples (illustrative, in English)

| Post | Tags |
| --- | --- |
| "He should be shot." (about a named person) | Threat |
| "News: man jailed for threatening to kill neighbor." | none |
| "You're a worthless idiot." | Personal attack |
| "Women are all [demeaning generalization]." | Group hate (gender, toward women) |
| "Men are all [demeaning generalization]." | Group hate (gender, toward men) |
| "The minister stole millions from the fund." (no source) | Accusation |
| "Someone like him should be thrown out of the country, and he's a thief." | Group hate or Personal attack depending on the target, plus Accusation |

### Edge cases to settle in the pilot

- **Quotes and counter-speech.** A post quoting hostile text in order to condemn it is not hostile itself.
- **News and reporting.** Neutral unless the wording itself is hostile.
- **Sarcasm and jokes.** Label the likely reading; mark `unclear` when it can't be decided.
- **Banglish spelling variants and deliberate evasion.** Label by meaning, not spelling.
- **Hostility inside a long, mostly neutral post.** Tag the post, and note the share of text that is hostile.
- **Replying to a hostile comment.** Label the reply on its own text.

## Language

Record one of: `bangla`, `english`, `banglish`, `mixed`. This is needed for per-language results.

## Image labels (blur people)

| Field | Values |
| --- | --- |
| `person_present` | yes / no / unclear |
| `face_visible` | yes / no / unclear |
| `kind` | photo / drawing or cartoon / screenshot / other |

For the experimental women-only mode, I additionally record, for each person: `perceived_as_woman` / `perceived_as_man` / `unclear`. This is my perception of the image, not a fact about the person, and the mode's error rate is reported separately.

## Ad labels (if the ad highlight is built)

| Field | Values |
| --- | --- |
| `sponsored_label_shown` | yes / no |
| `looks_promotional` | yes / no (organic posts that read like ads) |

## Process

1. **Self-sort.** Collect about 50 posts I wish I hadn't seen and sort them into piles. Check that the tags fit what really appears.
2. **Pilot.** Two or three Bangla speakers from different backgrounds label the same 50-100 posts independently, without talking.
3. **Compare.** Count disagreements per tag. Every disagreement points to a vague definition. Rewrite the definition, then relabel.
4. **Scale up.** Label about 200-300 posts and a set of images, across Bangla, English, and Banglish.
5. **Report.** State the number of posts per tag and per language, and how often annotators agreed.

## Open questions

- Should Accusation be a tag or a separate flag that sits beside the hostile tags?
- Is a single "hostile" label enough for the first model, with types added later?
- How should hostility in memes (text inside an image) be handled? Out of scope for now.
