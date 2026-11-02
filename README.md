# Jachai (যাচাই)

**A browser extension that tags Facebook posts with evidence, not verdicts.**

> **Status:** Early stage: scaffold only, nothing implemented yet.

---

## Table of contents

1. [What Jachai is (and is not)](#what-jachai-is-and-is-not)
2. [Motivation](#motivation)
3. [Badges](#badges)
4. [How it works](#how-it-works)
5. [Planned tech stack](#planned-tech-stack)
6. [Repository structure](#repository-structure)
7. [Datasets](#datasets)
8. [Evaluation plan](#evaluation-plan)
9. [Privacy design](#privacy-design)
10. [Limitations and risks](#limitations-and-risks)
11. [Roadmap](#roadmap)
12. [Getting started](#getting-started)
13. [Contributing](#contributing)
14. [License](#license)
15. [About the name](#about-the-name)

---

## What Jachai is (and is not)

Jachai is a Chrome extension (Manifest V3) that runs while you browse Facebook and adds a small badge to each post. It works on Bangla, English, and romanized Bangla (Banglish).

What it does: it looks at the text of a post, decides whether the post makes a factual claim that someone could check, and, if so, looks for a published fact-check that matches that claim. The badge tells you what it found and links to the evidence.

What it deliberately is **not**: a "fake" / "not fake" classifier. I don't want to build a tool that stamps a post as fake and expects people to trust it. Three reasons:

- **A verdict I can't back up is worse than no verdict.** If the extension says "fake" and is wrong, the cost lands on a real person's post and on the user's trust in the tool. False flags are the costly error, so the design avoids making that call.
- **Most posts can't be settled by their text alone.** Whether a claim is true depends on the world, not on how the sentence is written. A model that scores "fakeness" from style is mostly learning the style of its training sources.
- **Evidence is something a reader can check themselves.** A link to an existing fact-check lets the reader judge. A confidence score with no source doesn't.

So Jachai is a tag-and-evidence tool. It says "this is checkable", "this matches a published fact-check (here it is)", "this reads like clickbait", or "this is checkable and I found nothing". It never says "this is false" on its own authority.

## Motivation

Plenty of fake-news classifiers report high accuracy on a public dataset. I expect them to fall apart on a real Facebook feed, for reasons I want to measure rather than assume:

- **Domain shift.** Public datasets are mostly news articles or political statements. A Facebook feed is shares, memes, rants, ads, and screenshots of other posts. A model trained on one distribution has no reason to transfer to the other.
- **Short, informal text.** Many posts are a sentence or two, with slang, abbreviations, emoji, and missing punctuation. Models trained on full articles lean on structure that isn't there.
- **Banglish.** A large share of Bangla social media is written in Latin script ("ei khobor ta shotti?"). Tokenizers and pretrained models built for Bangla script handle this poorly, and there is little labeled data for it.
- **Image-heavy posts.** For many posts the claim is in the image, not the caption. A text-only system sees almost nothing. I'm treating this as a known gap rather than pretending to solve it.
- **Unbacked "fake" labels cost trust.** One confident wrong flag teaches a user to ignore the tool. That is why the primary metric is precision and why the output is a tag with evidence.

## Badges

Every badge carries a confidence value and, where relevant, a link. None of them is a truth verdict.

| Badge | Meaning | When it appears |
| --- | --- | --- |
| `Checkable claim` | The post makes a factual claim that could be verified against a source. | The in-browser classifier decides the post contains a verifiable claim. Opinions, jokes, greetings, and personal updates should not get this badge. |
| `Matches fact-check` | The claim closely matches a claim that a fact-checking organization has already published a review of. The link to that review is shown. | Retrieval finds a candidate fact-check, and the verification step (NLI or LLM) agrees the match is real. The badge shows the source, not my own verdict. |
| `Clickbait / sensational` | The wording is written to provoke a reaction (exaggeration, urgency, curiosity gaps) regardless of whether the content is true. | The in-browser classifier flags the style. This is about phrasing, not accuracy. |
| `Unverified` | The post makes a checkable claim, but no matching fact-check was found. | A claim was detected and retrieval returned nothing that passed verification. This is not a statement that the claim is false. It often just means nobody has checked it. |
| `Possibly AI-written` *(optional, off by default)* | The text has statistical features associated with machine-generated writing. Low confidence. | Only when the user enables the flag. See [limitations](#limitations-and-risks): this signal is weak, especially in Bangla. |

## How it works

Per post, the pipeline is:

1. **Language ID.** Decide whether the text is Bangla, English, or Banglish.
2. **Cheap in-browser classifier.** Two questions: is there a checkable claim, and is the post clickbait? This runs locally and decides whether anything needs to leave the browser.
3. **Retrieval.** For posts with a checkable claim, search fact-check sources: the Google Fact Check Tools API, plus Bangla fact-check sites where their terms permit it.
4. **Verification.** Use NLI or an LLM to judge whether a retrieved fact-check really covers the claim in the post, rather than just sharing keywords.
5. **Badge.** Render a tag with a confidence value and an evidence link.

```
 Facebook feed
      |
      v
+---------------------------------------------------------------+
| Extension (Manifest V3)                                       |
|                                                               |
|  content script                                               |
|  - observes feed, extracts post text                          |
|  - (all Facebook selectors live in one file)                  |
|        |                                                      |
|        v                                                      |
|  service worker                                               |
|  - language ID (Bangla / English / Banglish)                  |
|  - local classifier: checkable claim? clickbait?              |
|  - cache of results                                           |
|        |                                                      |
|        +---- no claim ----------------------+                 |
|        |                                    |                 |
|        | claim text only                    v                 |
+--------|------------------------------ badge UI -------------+
         |                                  ^
         v                                  |
+--------------------------------+          |
| Retrieval + verification       |          |
| (optional stateless backend,   |          |
|  or direct API calls)          |          |
|                                |          |
|  1. Google Fact Check Tools    |          |
|  2. Bangla fact-check sites    |          |
|     (where terms permit)       |          |
|  3. NLI / LLM: does the        |          |
|     fact-check match the claim?|          |
+--------------------------------+          |
         |                                  |
         +---- tag + confidence + link -----+
```

Badges that only depend on the local classifier (`Checkable claim`, `Clickbait / sensational`) don't need the network at all.

## Planned tech stack

Everything below is planned, not final. I may change it as I learn what works.

- **Extension:** TypeScript, Vite, Manifest V3.
- **In-browser inference:** ONNX Runtime Web or transformers.js.
- **ML / training:** Python 3.11+, Hugging Face transformers, export to ONNX.
- **Models to compare:** BanglaBERT and XLM-R (fine-tuned), a zero-shot LLM, and a retrieval + NLI pipeline.
- **Backend (optional):** stateless FastAPI service for retrieval and verification.

## Repository structure

```
jachai/
  extension/
    src/
      content/        # feed observer, post extraction, badge injection
      background/     # service worker: model loading, messaging, caching
      ui/             # badge component, popup/options page
    public/           # icons, static assets
  ml/
    data/
      raw/            # downloaded public datasets (git-ignored)
      processed/      # cleaned and split data (git-ignored)
      labeled/        # hand-labeled real-feed eval set (local only, git-ignored)
    notebooks/        # exploration
    src/              # training, eval, ONNX export
    eval/             # eval harness + results
  backend/            # optional stateless FastAPI service
    app/              # service code
    tests/            # backend tests
  docs/
    architecture.md   # components and data flow
    datasets.md       # candidate datasets, licenses, quirks
    evaluation.md     # metrics, protocol, results
    privacy.md        # what stays local, what is sent
    labeling-guide.md # label schema and rules for the eval set
    roadmap.md        # detailed plan and progress
  .gitignore
  .env.example        # variable names only
  LICENSE
  README.md
```

## Datasets

These are candidates, not commitments. I haven't verified licenses or current availability for any of them, and I'll do that before using anything. I'm not listing sizes here because I haven't checked them.

| Dataset | Notes |
| --- | --- |
| LIAR | English short political statements with fact-check-based labels. Short text, which is useful, but the domain is political speech, not social media posts. |
| FakeNewsNet | English news and social context collected around fact-checking sources. Availability depends on re-collecting content, which may be incomplete. |
| ISOT | English news articles labeled by source. **Known to be trivially separable by source style** (the real and fake articles differ in formatting and boilerplate), so a high score on it says very little. I would only use it as a cautionary baseline, if at all. |
| BanFakeNews | Bangla news dataset. Likely the closest public starting point for Bangla, but it is news text rather than social posts. |

The main dataset for this project is the one I build myself: a hand-labeled set of roughly 200-300 real posts from a Facebook feed, kept local and never published. The labeling rules will live in [`docs/labeling-guide.md`](docs/labeling-guide.md).

## Evaluation plan

**Real-feed eval set.** Roughly 200-300 posts that I label by hand, covering Bangla, English, and Banglish. Labels follow the guide in `docs/labeling-guide.md`. The set stays on my machine.

**Metrics.** Precision first, because a false flag is the costly error. Then recall and F1. All metrics are broken down per language (Bangla, English, Banglish).

**Public vs. real.** One of the main results I want is the measured drop between performance on a public dataset and performance on the real-feed set, for the same model. I expect a large drop but I don't have the number yet.

**Model comparison.** Every result is TBD.

| Approach | Precision | Recall | F1 | Notes |
| --- | --- | --- | --- | --- |
| Fine-tuned BanglaBERT | TBD | TBD | TBD | TBD |
| Fine-tuned XLM-R | TBD | TBD | TBD | TBD |
| Zero-shot LLM | TBD | TBD | TBD | TBD |
| Retrieval + NLI | TBD | TBD | TBD | TBD |

| Setting | Result |
| --- | --- |
| Best model on public dataset | TBD |
| Same model on real-feed set | TBD |
| Drop | TBD |

## Privacy design

- **In-browser inference where possible.** Language ID and the first-stage classifier run locally. Posts with no checkable claim never leave the browser.
- **No wholesale feed upload.** The extension does not send the feed, page HTML, or post lists anywhere.
- **Claim text only.** When retrieval or verification needs the network, only the extracted claim text is sent. No author names, profile links, post URLs, or other identifying data.
- **No storage of author names or profile data.** Not locally, not on a backend.
- **Labeled set is opt-in.** Saving a post to the eval set happens only when I do it on purpose, and the file stays local and git-ignored.

The backend, if I build it, is stateless. Details will go in [`docs/privacy.md`](docs/privacy.md).

## Limitations and risks

- **Facebook DOM brittleness.** Facebook's markup is obfuscated and changes often. All selectors will be isolated in a single file so a break is a one-file fix, but breaks will happen.
- **Meta's terms of service.** Reading and annotating a page you are viewing is different from scraping, but I haven't concluded that this is within Meta's terms. I'll read them closely and keep the tool strictly to what the logged-in user already sees.
- **AI-text detection is weak.** Detectors are unreliable in English and worse in Bangla, with little training data. That is why the badge is optional, off by default, and low confidence.
- **Fact-check coverage gaps in Bangla.** Far fewer Bangla claims have published fact-checks than English ones. Many real claims will end up as `Unverified` simply because nobody has reviewed them.
- **Text-only.** Claims carried in images or video are mostly invisible to this design.
- **Banglish.** Romanized Bangla has no standard spelling, which makes language ID and classification harder.
- **No Chrome Web Store release planned.** This is a portfolio and research project. If you want to try it, you'll load it unpacked.

## Roadmap

A four-week plan. Details will go in [`docs/roadmap.md`](docs/roadmap.md).

- [ ] **Week 1: skeleton.** Extension scaffold, feed observer, post extraction, badge injection with hardcoded tags.
- [ ] **Week 2: labeling and baseline.** Build the real-feed labeled set, train a first baseline classifier, measure it on public data and on the real set.
- [ ] **Week 3: retrieval and verification.** Fact-check retrieval, NLI / LLM verification, evidence links on badges.
- [ ] **Week 4: evals, writeup, demo.** Model comparison, public-vs-real analysis, fill in this README with real numbers, record a demo video.

## Getting started

Not available yet.

Unpacked-install steps will go here once there is something to install:

<!-- TBD: build instructions -->
<!-- TBD: load unpacked in chrome://extensions -->
<!-- TBD: configure .env (see .env.example) -->

## Contributing

Nothing to contribute to yet. Once the skeleton is in place I'll document how to file issues and what kind of help is useful. The most valuable contribution later is likely to be Bangla and Banglish labeling feedback.

## License

MIT. See [LICENSE](LICENSE).

## About the name

যাচাই (*jachai*) is Bangla for "verify" or "check". It's the verb you'd use for checking a claim before believing it, which is what I want the tool to nudge people toward.
