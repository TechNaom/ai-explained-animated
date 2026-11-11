# Course-to-video framework: design

Status: **draft for owner approval.** No framework code is written until this is approved.
Owner: Manohar Papasani. Created 2026-11-10. Replaces the earlier single-video pipeline draft.

## 1. Goal

Give the framework a **topic or a whole course** plus requirements (audience, depth, length). It produces, for every lesson:

- an interactive animated explainer page (GitHub Pages);
- a long YouTube video (16:9) and a Short (9:16), narrated in the owner's cloned voice;
- captions, chapters, thumbnail, title and description;
- cuts and posts for YouTube, Instagram, Facebook, LinkedIn and X, published only when the owner approves;
- all driven from a chat-style **Studio UI** where the owner types, pastes or speaks the requirement.

**Quality bar:** every lesson must be factually correct, clearly taught, visually precise and well paced. **Cost bar:** a 20-lesson course should cost **under Rs 500** in model and voice usage.

## 2. Decisions already made

| Decision | Choice |
|---|---|
| Repo | `TechNaom/ai-explained-animated`, public, Pages; code Apache-2.0, content CC BY 4.0 |
| Voice | Owner's own voice, cloned once with **Sarvam Bulbul V3** (30 to 60 s consented sample); ElevenLabs as fallback |
| Formats | Long 16:9 + Short 9:16 from the same source |
| Channel | Existing channel; uploads are always private; the owner makes each video public |
| Visual style | Our own animation engine and component library for all teaching visuals; OpenArt optional for cinematic hooks only |

## 3. The core design choice: specs, not free-form code

Agents never write a page from scratch. They write a **scene spec** (JSON) that our tested engine plays using a fixed **component library** (cards, boxes, arrows, meters, pipelines, code blocks, charts, flows, callouts). This is what makes cheap models usable and every lesson consistent:

- a spec is short and checkable (JSON Schema validation, then layout checks);
- the engine guarantees smooth animation, theming, captions and both aspect ratios;
- new visual components are built rarely, by Claude in Claude Code (covered by the subscription, not per-lesson API spend).

## 4. Pipeline and agents

```
Course input (syllabus / PDF / notes / URL / one topic)
  │
  ▼
1 Course Planner ── modules, lessons, learning goals, prerequisite order        [once per course]
  │
  ▼  for each lesson
2 Research ─────── fetch sources, extract facts with citations
3 Teacher ──────── learning goals, story arc, long script + Short script
4 Fact Checker ─── script vs sources, a different model family from the Teacher
5 Storyboard ───── script -> scene spec JSON (positions, steps, captions, timing hints)
6 Layout Checker ─ CODE: schema, overlaps, text overflow, contrast, min font size
7 Spec Fixer ───── applies the exact error list from step 6 (max 2 rounds, then escalate)
8 Voice ────────── pronunciation pass -> Sarvam TTS per step -> exact step durations
9 Voice QA ─────── speech-to-text on the audio, compare with the script, regenerate bad lines
10 Render ──────── CODE: Playwright + ffmpeg -> 1080p long video and 1080x1920 Short
11 Video QA ────── a video-capable model watches the render: pacing, readability, sync
12 Packaging ───── title, description, chapters, tags, thumbnail text
13 Quality gate ── rubric score per lesson; below the bar -> back to the failing step
14 Publish ─────── upload as PRIVATE; owner approves -> public
```

Owner approval points: (a) the course plan, optional; (b) each finished video, required before public.

## 5. Model routing: the best quality per rupee

**Principle: spend on leverage, save on volume.**

- Steps that are **small in tokens but decide quality** (course plan, teaching script) get the strongest models: they cost little because the token counts are small.
- Steps that are **large in tokens** (research reading, layout fixing, visual checks) go to cheap models, free tiers or plain code.
- Hard steps use a **cascade**: try the strong-but-cheap model first; escalate to a premium model only when validation fails.

| # | Agent | Primary model | Fallback / escalation | Why |
|---|---|---|---|---|
| 1 | Course Planner | **Claude Opus 5.5** | Sonnet 5.5 | Runs once per course and shapes everything; about $0.60 per course |
| 2 | Research | **DeepSeek V4-Flash** + a web search API | Qwen3.8 Flash, then Grok 4 Fast | Reading many sources is high-volume; the cheapest strong reader |
| 3 | Teacher | **Claude Sonnet 5.5** | Opus 5.5 | The script is the heart of learning quality; only about 13K tokens per lesson, so premium is affordable |
| 4 | Fact Checker | **Gemini Flash** (free tier) | Grok 4 Fast | A different family from the Teacher catches its blind spots |
| 5 | Storyboard | **DeepSeek V4-Pro** or **Qwen3.7 Plus** (the bench decides) | The other one, then Sonnet 5.5 on validation failure | Large structured output; both are strong at structured JSON at a fraction of Sonnet |
| 6 | Layout Checker | **Code** (no model) | none | Overlaps, overflow and contrast are exact geometry; free and deterministic |
| 7 | Spec Fixer | **DeepSeek V4-Flash** | Qwen3.8 Flash, then Sonnet 5.5 after 2 failed rounds | Fixing a precise error list is easy work |
| 8 | Voice | **Sarvam Bulbul V3**, cloned voice | ElevenLabs | Owner's voice, Indian English, 11 Indian languages later |
| 8a | Pronunciation | **Dictionary in code** + Claude Haiku 5.5 for new terms | none | Most terms repeat across lessons |
| 9 | Voice QA | **Whisper** (open source, free) | Sarvam speech-to-text | Free and accurate enough to catch skipped or misread words |
| 11 | Video QA | **Gemini Flash** (free tier) | none | Watches video natively |
| 12 | Packaging | **Claude Haiku 5.5** | DeepSeek V4-Flash | Short, simple outputs |
| 15 | Translator (later: Telugu, Hindi and other Indian languages) | **Qwen3.7 Plus** | Sonnet 5.5 | Qwen's broad multilingual coverage; Sarvam voices the result in the same cloned voice |
| 13 | Quality gate judge | **Claude Sonnet 5.5** | none | Scores the finished lesson against the rubric; small input |
| — | New components | **Claude in Claude Code** | none | Rare; subscription, not API |

All model calls go through one router (OpenRouter or direct APIs) so that any agent's model is a one-line config change.

**Provider diversity:** DeepSeek and Qwen back each other up on every high-volume step, so an outage, a price rise or a model retirement at one provider never stops a course run.

## 6. Cost estimate for a 20-lesson course

Estimates from the token budget per agent; Claude prices from Anthropic's reference, other prices from third-party pricing sites, all to be confirmed on each provider's own page before the first run.

| Agent | Course cost (approx.) |
|---|---|
| Course Planner (Opus 5.5, once) | $0.60 |
| Research (DeepSeek V4-Flash) | $0.15 |
| Teacher (Sonnet 5.5) | $1.60 |
| Storyboard (DeepSeek V4-Pro, about 20% escalated to Sonnet) | $2.50 |
| Spec Fixer (DeepSeek V4-Flash) | $0.30 |
| Fact Checker, Video QA (Gemini free tier) | $0 |
| Packaging + pronunciation (Haiku 5.5) | $0.10 |
| Quality gate (Sonnet 5.5) | $0.60 |
| Layout Checker, Render, Whisper (code, GitHub Actions) | $0 |
| **Models total** | **about $5.85, Rs 515** |
| **With Claude batch pricing (half price) and prompt caching** | **about $3.50, Rs 310** |
| Sarvam voice (about 150,000 characters per course) | to be confirmed from Sarvam's pricing |

For comparison: running every agent on Sonnet 5.5 costs about Rs 2,190 per course, and on Opus 5.5 about Rs 4,380, without better results on the high-volume steps.

### Cost controls

- **Budget governor:** a hard spend cap per lesson and per course; the run stops and reports instead of overspending.
- **Prompt caching** for the shared system prompts and component schema.
- **Batch API** for all Claude calls, since no one waits on a course run.
- **Code before models:** layout checks, pronunciation dictionary and Whisper replace model calls wherever an exact check exists.
- **Retry caps:** at most 2 fix rounds before escalation, at most 1 escalation per step.
- **Measured, not guessed:** every call logs tokens and cost per agent; lesson 1 replaces these estimates with real numbers.

## 7. Quality controls

- **Rubric** per lesson, scored 1 to 5: factual accuracy, clarity for the stated audience, visual precision, pacing and voice. Publish bar: **no score below 4**.
- **Two-family check:** the Fact Checker never shares a model family with the Teacher.
- **Sources:** every factual claim in the script links to a source from the Research step; unsupported claims are cut.
- **Owner review:** nothing is public until the owner approves.

## 8. Choosing models by evidence: the model bench

Before the first course, run **3 pilot lessons** (easy, medium, hard topic) through the candidate models for the four roles where the choice matters most:

| Role | Candidates |
|---|---|
| Research | DeepSeek V4-Flash, Qwen3.8 Flash, Grok 4 Fast, Haiku 5.5 |
| Teacher | Sonnet 5.5, Opus 5.5, Qwen3.8 Max, DeepSeek V4-Pro |
| Storyboard | DeepSeek V4-Pro, Qwen3.7 Plus, Sonnet 5.5 |
| Fact Checker | Gemini Flash, Grok 4 Fast |

**Scoring:** spec validity rate, layout errors, fact errors found by cross-check, the owner's 1-to-5 rating of each script, and measured cost per lesson.
**Rule:** among models scoring at least 4 out of 5 on quality, pick the cheapest. Re-run the bench when providers release or reprice models.
**Bench budget:** estimated under Rs 300 in total; confirmed with the owner before it runs.

## 9. Voice setup

- The owner records a **60 to 90 second sample** from a provided script full of technical terms, in a quiet room on a phone.
- Sarvam creates the cloned voice; only the **voice ID** is stored, as a GitHub secret. The sample never goes in the public repo.
- Sarvam's cloning needs the `text_to_speech_voice_cloning` capability on the subscription; check this when the account is created.

## 10. Rendering, publishing and platform limits

- **Render:** headless Chromium (Playwright) plays the spec at the exact narration timings; ffmpeg encodes H.264 at 30 fps. Phase 1 compares three capture methods (Playwright video, DevTools screencast, frame-by-frame virtual time) and the owner picks by eye.
- **YouTube Data API:** uploads as private. Videos from unverified API projects stay private until Google's audit, which matches our approval rule. The OAuth consent screen must be **In production** or the refresh token expires after 7 days. Custom thumbnails need a phone-verified channel. One upload uses about 1,600 of the 10,000 daily quota units.
- **OpenArt (optional):** cinematic 5 to 8 second hooks and analogy clips; generated by hand at first since no API was confirmed.

## 10a. Studio UI: a chat interface to make videos

Concept preview (clickable, nothing generated): https://claude.ai/artifact/UEpongYJcB6ZVExkXY9oPT

**How it works for the owner:**

1. Open the studio. A chat screen asks "What should we teach today?" with example prompts.
2. **Type, paste or speak** the requirement: a topic, a full course, a problem statement. Long pastes become a document chip (like Claude). Files (PDF, DOCX, PPTX, Markdown) can be attached.
3. **Dictation:** the mic records the owner's voice and transcribes it with **Sarvam speech-to-text** (Indian English, Telugu, Hindi); the browser's built-in speech recognition is the instant fallback. The transcript lands in the box to edit before sending.
4. The **Producer agent** asks four quick choices as tap-to-pick chips: audience, length, languages and channels.
5. It shows a **plan card**: lessons or scenes, outputs per channel, voice, estimated cost and time. **Start production** or **Change the plan** in plain words.
6. A **live progress card** shows each agent step, its model and its cost, with a running total.
7. A **review card** shows the previews per channel, the quality scores and the cost. **Approve** schedules the posts; changes are requested in chat ("slow down scene 3", "add a Telugu version").

Also in the UI: a project list with status, a **Best value / Max quality** switch, the monthly spend against the budget, and the cloned-voice status.

**Architecture (free tiers, scales to zero):**

| Part | Choice |
|---|---|
| Front end | Static web app on Cloudflare Pages |
| API | Cloudflare Worker |
| Projects, chat history, costs | Cloudflare D1 (own database, isolated) |
| Media files | Cloudflare R2 (10 GB free, no download fees) |
| Heavy jobs (agents, render) | GitHub Actions, started by the Worker |
| Live progress | The UI polls the Worker every few seconds |
| Sign-in | Cloudflare Access: owner only |
| Producer agent | Claude Sonnet 5.5 (short conversations, about Rs 1 to 2 per project) |

## 10b. One source, every channel

Each lesson is rendered once from the spec in **three frame shapes**, then cut and packaged per channel. The Packaging agent writes channel-specific text (YouTube description with chapters, Instagram caption and hashtags, LinkedIn post, X post).

| Channel | Formats | Notes |
|---|---|---|
| YouTube | Long 16:9 with chapters; Short 9:16 | Upload as private; owner makes public |
| Instagram | Reel 9:16, captions burned in | Needs a Business or Creator account linked to a Facebook Page |
| Facebook | Reel 9:16; feed video 16:9 | Posted to a Facebook Page |
| LinkedIn | Square 1:1 or 16:9 video with post text | Separate app and token from LBE, so LBE is never touched |
| X | 16:9 clip with post text | API posting needs a suitable X API plan; manual posting until then |

**Approval:** not every platform supports drafts, so the studio holds every post and publishes only when the owner approves it, per channel. Platform limits (lengths, posting quotas, API plan needs) are confirmed on each platform's own documentation before Phase 6.

## 11. Isolation, privacy and safety

- Own repo, own API keys, own Google Cloud project, own budget cap; nothing shared with other pipelines.
- China-hosted APIs (DeepSeek, Qwen) receive only public course content, never private or employer documents.
- Secrets only in GitHub Actions secrets; the voice sample stays private.
- Runs on GitHub Actions and stops when done; nothing is always on.

## 12. Build phases

| Phase | What | Done when |
|---|---|---|
| 0 | Repo, Pages, explainer #1, this design | Done 2026-11-10 |
| 1 | Engine + component library + scene spec schema; rebuild explainer #1 from a spec; render spike | Explainer #1 plays from a spec; owner picks the render method |
| 2 | Agents 2 to 7 and the model router with cost logging; model bench on 3 pilot lessons | Bench table with quality and real cost; owner approves the routing |
| 3 | Voice: Sarvam clone, pronunciation, Voice QA, timing from audio | A narrated long video in the owner's voice |
| 4 | Shorts layout, captions, chapters, thumbnail, packaging, quality gate | Long video and Short both pass the rubric |
| 5 | Course Planner, budget governor, private YouTube upload | A course produces private videos end to end |
| 6 | Studio UI: chat, paste, dictation, plan and progress cards, review | The owner makes a video from the studio alone |
| 7 | Multi-channel: 1:1 render, per-channel packaging, Instagram, Facebook, LinkedIn and X publishing on approval | One approval posts to every chosen channel |

## 13. Questions for the owner

1. Approve this design, or tell me what to change.
2. Approve the bench budget (estimated under Rs 300).
3. Accounts to create when we reach them: Sarvam (Phase 3), OpenRouter or the individual providers (Phase 2), the YouTube API project (Phase 5).
4. Channel name, and whether the channel is phone-verified.
5. Background music: yes or no.
6. Framework code: keep it in this public repo, or in a separate private repo (the explainers stay public either way)?
7. Accounts for channels: an Instagram Business or Creator account linked to a Facebook Page, a Facebook Page, and whether to pay for X API posting.
