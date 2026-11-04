# Jachai: Product Requirements and Due Diligence

Last updated: 2026-11-04. Status: draft, based on desk research only. Nothing here has been tested yet.

## What is a PRD?

A PRD (product requirements document) is a single page that answers: what are we building, for whom, why, what is in and out of scope, how will we know it works, and what could go wrong. It is written before the code so the plan can be argued with while changing it is still cheap.

This one has three parts: **what changed** since the first idea, **due diligence** (is the plan sound, where do we stand), and **requirements** (what exactly gets built first). Skip to [Verdict](#verdict) and [Next steps](#next-steps) if you only want the answer.

---

## Summary

Jachai is a Chrome extension that makes my own time on Facebook better by filtering what I'd rather not see. It runs entirely in the browser, in Bangla, English, and Banglish, with no backend and no running cost.

Features, in priority order:
1. **Blur people** in images (core).
2. **Focus mode**: hide the feed with a toggle (core).
3. **Hostile-wording tags**: Threat, Personal attack, Group hate, Accusation (core).
4. **Ad highlight** (stretch).

It is a personal and portfolio project. The deliverable that matters is a measured, honest evaluation on real feed posts, not a store release.

## What changed

| Decision | Reason |
| --- | --- |
| Dropped fact-check retrieval, the LLM verification step, and the backend | Costs money to run, depends on fact-check coverage that may not exist for Bangla, and a text-only model can't verify truth. |
| Dropped the "probably fake / probably safe" verdict and the scam and misinformation flags | A style-based model learns writing patterns, not truth. Published scores fall sharply on data from other sources or later dates. |
| Dropped bot-like and bot-share detection | It needs other people's profile data, which breaks the privacy design and Meta's rules, and there is no ground truth to measure. |
| Added blur people and focus mode | Visual, useful, off-the-shelf models, and a good demo. |
| Added hostile-wording tags, with no religion tag | Personal filter. Decided to avoid religion as a topic. |
| Women-only blur kept as an optional experiment | Gender-from-appearance is error-prone and biased, so the main feature is "blur any person". |
| Precision-first replaced by "measure both, favor recall" | The tool is private and every blur is reversible. A miss costs more than a false blur. |

## Where we stand

| Area | State |
| --- | --- |
| Idea and scope | Defined (this document) |
| Repo | Scaffold: folders, README, doc stubs, `.gitignore`. The `backend/` folder and `.env.example` are leftovers from the earlier design. |
| Code | None |
| Data | None. The real-feed labeled set does not exist yet. |
| Models | None trained or evaluated |
| Research | First pass done (below) |

---

## Due diligence

Confidence: **High** means a primary source says it, **Medium** means a reasonable reading or one source, **Low** means unverified.

### 1. Hostile-wording data

| Dataset | What the sources say | Use |
| --- | --- | --- |
| BanTH | 37,350 YouTube comments in Banglish; hate vs not, plus targets (political, religious, gender, personal offense, abusive/violence, origin, body shaming). Best multi-label Macro-F1 was 30.17%; best subset accuracy 54.71%. | The only Banglish set found. License and download location not confirmed. |
| Bengali Hate Speech Dataset (DeepHateExplainer) | About 8,087 posts from Facebook, YouTube, newspapers (per a 2026 survey). Categories: political, personal, geopolitical, religious (the repo also mentions gender abusive). Reported per-category accuracy in the v2 abstract: 84%, 90%, 88%, 88%. | Closest to Facebook text. |
| Bangla multilabel cyberbully / sexual harassment / threat / spam | 12,557 comments after cleaning, from Facebook and TikTok. Version 3 is dated July 2024. | Has threat labels. |
| Facebook cyberbullying study | 44,001 Facebook comments on public figures' posts; classes include threat, troll, religious, sexual, not bully. | Label scheme reference. |
| BanHate | 19,203 YouTube comments, April 2024 to June 2025. | Hate benchmark. |
| TRAC 2020, OMANISHA, WoNBias | Bangla misogyny and aggression sets (TRAC: over 25,000 YouTube comments in three languages; OMANISHA: 7,017 samples, which one Mendeley entry says were authored rather than copied from real comments; WoNBias: 31,484 texts). | Gender-targeted hostility. |

Takeaways:
- Everything is comments or tweets, not Facebook posts. I found no large public set of Facebook posts.
- **Threats are rare in a normal feed**, so they are the hardest class. Public data will carry them.
- I found **no dataset on hostility toward men**. That class, if kept, needs hand labeling.
- Banglish has one dataset (BanTH), and its best model scored low. Treat Banglish as a hard case to measure, not a solved one.
- A 2026 paper, "Beyond Benchmarks: Exposing the Hidden Crisis in Bangla Hate Speech Detection" (<https://arxiv.org/pdf/2607.11597>), came up in my search. I only saw its title, so read it before trusting reported numbers.
- None of this was checked for licenses.

### 2. How reliable are such classifiers?

| Finding | Implication | Confidence |
| --- | --- | --- |
| Fake-news models: one study reports about 99.85% in-dataset and about 68% on external corpora, blamed on dataset shortcuts. Another reports a drop of 7-8 macro-F1 points on later time periods. | Scores on the training distribution say little about a live feed. | Medium (search summaries) |
| SMS spam: a transformer's recall on the old UCI set fell from 99.03% to under 80% on newer spam. | Wording shifts over time. | Medium |
| Phishing: benchmarks show 96-99%, but a benchmark paper builds sets at realistic rates of 0.05-5% because balanced benchmarks hide false-flag cost. | Report precision at realistic base rates, not accuracy on balanced data. | Medium |
| Bangla hate and abuse results vary widely: 77% accuracy on a seven-category hate set, 95% on a cyberbullying set, Macro-F1 of 30% on multi-label BanTH. | Results depend heavily on the task and the labels. Expect modest numbers for finer tags. | Medium |

This is why the project reports the measured drop from public data to real feed, and why it starts with one binary detector before splitting into types.

### 3. Image models (blur people)

| Finding | Implication | Confidence |
| --- | --- | --- |
| MediaPipe's object detector runs in the browser through the `@mediapipe/tasks-vision` package, with EfficientDet-Lite0 as the example model. Web workers are recommended to avoid blocking the main thread. | Person detection in the browser is practical. | High (docs) |
| NSFWJS is a client-side image classifier on TensorFlow.js. Its README claims about 90% accuracy for the small model and about 93% for the mid-sized one. | A related precedent for in-browser image filtering. Not needed for the core plan. | Medium (README claim) |
| Gender Shades: three commercial systems misclassified darker-skinned women at rates up to 34.7%, against up to 0.8% for lighter-skinned men. | A women-only mode will have uneven errors. Keep it experimental and report its error rate. | High (paper) |
| I believe MediaPipe is Apache 2.0 and Ultralytics YOLO is AGPL, but I could not confirm either. | Check licenses before choosing a detector, especially if the code is ever distributed. | Low |
| Reading pixels of images served from Facebook's CDN inside an extension may need extra work. | Prototype this first. | Low |

### 4. Ads

| Finding | Implication | Confidence |
| --- | --- | --- |
| Facebook breaks the "Sponsored" label into pieces and scrambles it, with random characters, extra nodes, SVG, and reported flexbox reordering. Open-source extensions rebuild the label, but they are small and lightly maintained. | Ad highlight is a DOM arms race, not an ML problem. A stretch goal at best. | Medium (blog posts, repos) |

### 5. Technical feasibility in Manifest V3

| Finding | Implication | Confidence |
| --- | --- | --- |
| MV3 service workers have no DOM. Chrome's answer is an offscreen document (one per extension, independent lifetime). One 2024 write-up says WebGPU was unavailable in the service worker, so inference moved to an offscreen page. | In-browser inference is feasible, probably hosted in an offscreen document. Verify against current Chrome. | Medium |
| BanglaBERT has about 110M parameters. The model page states no license. | Likely needs quantizing or a smaller model for the browser. Confirm the license. | Medium |
| Chrome Web Store policy limits data collection to a disclosed single purpose. | Moot with no store release, and consistent with a local-only design. | High |

### 6. Platform, legal, and naming

| Finding | Implication | Confidence |
| --- | --- | --- |
| Meta's terms say you may not "access or collect data from our Products using automated means (without our prior permission)". | A content script that reads a logged-in feed is a gray area. Account-level enforcement is possible. | High (quoted) |
| In Meta v. Bright Data (Jan. 2024) a court held the terms did not bar logged-off scraping of public data. In the BrandTotal case Meta won on this clause. I did not check whether BrandTotal used a browser extension. | Case law is mixed and does not cleanly cover a local, logged-in extension. | Medium |
| One report says Bangladesh replaced its cyber law in 2026 and that an amendment would criminalize spreading rumors online. Only one source, not confirmed. | Not aimed at a private filter. Confirm the current law if the tool is ever shared. | Low |
| A search summary suggested a Bangladeshi fact-checking initiative called "Jachai" exists. I could not confirm it. | Possible name collision, and the project's scope no longer matches the name anyway. | Low |

---

## Verdict

**The plan is sound for a personal and portfolio project, and it is cheaper and more honest than the first idea.**

What holds:
- Everything runs locally, so there is no running cost and no privacy problem with uploading feeds.
- Blur people and focus mode rely on mature, off-the-shelf pieces and make a strong demo.
- Hostile-wording tags have real Bangla and Banglish data, and the real-feed set plus the public-vs-real drop makes a credible research result.

What to watch:
1. **Pixel access to Facebook images** and **person detection speed** are the first technical unknowns. Test them in Week 0.
2. **Fine-grained tags will score low.** Start with one binary detector.
3. **Ad highlight will keep breaking.** Keep it a stretch goal.
4. **Gender-specific blurring is unreliable.** Keep it optional and report the error rate.

What I would not claim: that the tool detects fake news, that it judges people, that it works on text inside images, or that Banglish works until measured.

---

## Requirements

### Goals
1. Blur images containing people, with reveal-on-click.
2. Provide a focus mode that hides the feed.
3. Tag posts for Threat, Personal attack, Group hate, and Accusation, in Bangla, English, and Banglish.
4. Build a hand-labeled real-feed set and measure the drop from public-data results.
5. Keep everything local.
6. Publish an honest README with real numbers and limitations.

### Non-goals
- True/false verdicts or fact-checking.
- Bot or fake-engagement detection.
- Any backend, search service, or paid API.
- A religion tag or religion-topic filter.
- Chrome Web Store release.
- Other platforms.

### Users
- **Primary: me**, building and using it unpacked.
- **Secondary: reviewers** who read the README, results, and demo video.

### Functional requirements

Priority: **P0** is needed for the 4-week deliverable, **P1** is wanted, **P2** is only if time remains.

| ID | Requirement | Priority |
| --- | --- | --- |
| F1 | Detect posts as they appear and extract text and image references. All Facebook selectors in one file. | P0 |
| F2 | Focus mode: hide the feed with a toggle and a peek option. | P0 |
| F3 | Blur images by default, run person detection locally, and reveal on click once cleared or on request. | P0 |
| F4 | Sample frames from video for person detection. | P1 |
| F5 | Language ID: Bangla, English, Banglish. | P0 |
| F6 | Hostile-wording detector (binary). | P0 |
| F7 | Type tags: Threat, Personal attack, Group hate (with optional target), Accusation. A post may carry several. | P1 |
| F8 | Show tags as chips with a short reason; blur flagged posts with a click to reveal; per-feature and per-tag toggles; a sensitivity setting. | P0 |
| F9 | Save a post or image to the local labeled set on purpose. | P0 |
| F10 | Women-only blur mode, off by default, with its error rate reported separately. | P2 |
| F11 | Ad highlight. | P2 |
| F12 | Cache results per post so nothing is processed twice. | P1 |

### Non-functional requirements

| Area | Requirement |
| --- | --- |
| Privacy | No network use at runtime other than loading bundled models. No feed, author names, or profile data stored or sent. |
| Safety of display | Images are blurred before any check completes, so a slow model never shows a raw image. |
| Reversibility | Nothing is deleted or reported. Every blur can be revealed. |
| Performance | Scrolling stays smooth. Specific targets TBD after the first prototype. |
| Resilience | When Facebook's markup changes, only the selector file needs editing. |
| Wording | Tags describe the text. They never call a person a name or say "this is hate speech" as a verdict. |

### Success metrics

Targets are TBD until a baseline exists. I'm choosing what to measure now.

| Metric | Why |
| --- | --- |
| Precision and recall per tag, per language (Bangla, English, Banglish), with counts | Core result |
| Wrongly blurred and missed posts per 100 | What I feel when using it |
| Public-vs-real drop for the same model | Headline finding |
| Person-detection precision and recall on hand-labeled images | Blur people |
| Women-only mode error rate, reported separately | Honesty about the experiment |
| Sponsored posts caught and false highlights (if built) | Ad highlight |

### Milestones

| Week | Outcome |
| --- | --- |
| 0 | De-risk: image pixel access, person detector speed, how sponsored posts look today, dataset licenses |
| 1 | Skeleton, selectors file, focus mode, blur-all placeholder, hardcoded tag chips |
| 2 | Blur people with an off-the-shelf detector; pilot labeling with two or three people; first hostile-wording baseline on public data |
| 3 | Real-feed evaluation; split into types if data supports it; ad highlight if time allows |
| 4 | Public-vs-real analysis, README numbers, demo video |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| Cannot read Facebook image pixels from the extension | Unknown | High | Week 0 prototype; fetch images through the extension if needed |
| Person detection too slow for scrolling | Medium | Medium | Blur by default, detect in a worker, cache, use a small model |
| Hostile-wording model scores low on real posts | High | Medium | That is the research finding. Report it. Start binary. |
| Banglish performs poorly | High | Medium | Measure and report it. Try XLM-R or transliteration. |
| Labeling is slow or inconsistent | Medium | High | Short guide, pilot with two or three people, measure agreement |
| Threats too rare in the real set to evaluate | High | Medium | Lean on public data; report counts and wide uncertainty |
| Dataset or model licenses restrict use | Medium | Medium | Verify first; this is a non-commercial personal project |
| Facebook changes the page and breaks selectors | High | Medium | One selector file; keep saved HTML as test fixtures |
| Ad highlight keeps breaking | High | Low | Stretch goal only |
| Meta enforces its automated-access terms on my account | Low to medium | Medium | Test on a non-primary account; local reading only |
| Women-only blur is biased | High | Medium | Experimental, off by default, error rate published |
| Quotes, news, sarcasm cause false tags | High | Medium | Define them in the labeling guide; reveal-on-click softens the cost |
| Model too large for the browser | Medium | Medium | Quantize or distill; try smaller models |

## Open questions

1. Can the content script read image pixels, or must images be fetched through the extension?
2. Is the in-browser person detector fast enough, and which detector has a usable license?
3. Which hostile-wording datasets can I actually use, and under what licenses?
4. Is one binary detector enough for a useful personal filter, or do I need the types?
5. How do labelers from different backgrounds disagree on these tags?
6. How are sponsored posts marked in the page today?
7. Should the project be renamed, and does the name collide with an existing project?
8. Should the leftover `backend/` folder and `.env.example` be removed?

## Next steps

In order:

1. **Collect about 50 posts I wish I hadn't seen** and sort them into piles. Check that the four tags match what actually shows up.
2. **Week 0 prototype:** read one feed image from the extension, run a person detector on it, and time it.
3. **Check how sponsored posts look in the page today**, to decide whether ad highlight is worth trying.
4. **Verify licenses and availability** of BanTH, the Bengali Hate Speech Dataset, the Mendeley multilabel set, and the detector.
5. **Pilot labeling:** two or three Bangla speakers from different backgrounds label the same posts independently; compare.
6. **Read the "Beyond Benchmarks" paper** and BanTH's annotation guidelines.
7. **Decide on the name and remove leftovers** (`backend/`, `.env.example`).
8. **Start Week 1.**

---

## Sources

Primary sources unless noted.

- BanTH (hate speech, transliterated Bangla): <https://arxiv.org/pdf/2410.13281>
- Bengali Hate Speech Dataset: <https://www.github.com/rezacsedu/Bengali-Hate-Speech-Dataset>
- DeepHateExplainer: <https://arxiv.org/pdf/2012.14353.pdf>
- Bangla multilabel cyberbully, threat, spam set: <https://data.mendeley.com/datasets/sz5558wrd4/3>
- Cyberbullying in Bangla (44,001 Facebook comments): <https://arxiv.org/pdf/2106.04506>
- BanHate: <https://aclanthology.org/2025.banglalp-1.19>
- TRAC 2020 misogyny and aggression corpus: <https://arxiv.org/pdf/2003.07428>
- OMANISHA: <https://data.mendeley.com/datasets/x9fym689h4>
- Hate speech in low-resource languages (survey): <https://arxiv.org/pdf/2411.19017>
- Bangla hate speech detection, benchmark critique: <https://arxiv.org/pdf/2607.11597>
- Gender Shades: <https://proceedings.mlr.press/v81/buolamwini18a.html>
- MediaPipe object detector for web: <https://developers.google.com/edge/mediapipe/solutions/vision/object_detector/web_js>
- NSFWJS: <https://github.com/nsfw-filter/nsfwjs>
- Chrome offscreen documents: <https://developer.chrome.com/blog/Offscreen-Documents-in-Manifest-v3>
- Transformers.js in a Chrome extension: <https://huggingface.co/blog/transformersjs-chrome-extension>
- BanglaBERT model page: <https://huggingface.co/csebuetnlp/banglabert>
- Meta Terms of Service: <https://www.facebook.com/terms>
- Chrome Web Store program policies: <https://developer.chrome.com/docs/webstore/program-policies/policies>
- Fake news cross-dataset drop: <https://jaes.qu.edu.sa/index.php/jae/article/view/2733>
- Temporal shift study (GenBench 2023): <https://aclanthology.org/2023.genbench-1.6>
- Spam concept drift: <https://arxiv.org/pdf/2504.09776>
- Phishing benchmark at realistic base rates: <https://arxiv.org/pdf/2507.10854>

Secondary sources (blogs, news, and community projects; treat as less reliable):

- How Facebook avoids ad blockers: <https://www.dylanpaulus.com/how-fb-avoids-adblockers/>
- Open-source Facebook ad blocker example: <https://awesome.ecosyste.ms/projects/github.com%2Fz1nc0r3%2Ffacebook-ad-blocker>
- Meta v. Bright Data summary: <https://www.fbm.com/technology/publications/major-decision-affects-law-of-scraping-and-online-data-collection-meta-platforms-v-bright-data/>
- Bangladesh cyber law changes (Prothom Alo): <https://en.prothomalo.com/bangladesh/wbq9kji6z7>
