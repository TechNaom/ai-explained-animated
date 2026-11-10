# Video pipeline design

Status: **draft for owner approval**. No pipeline code is written until this is approved.
Owner: Manohar Papasani. Created 2026-11-10.

## 1. Goal

Turn each animated explainer in this repo into two YouTube videos with as little manual work as possible:

- a **long video** (16:9, 2 to 4 minutes) with chapters and captions;
- a **Short** (9:16, under 60 seconds) cut from the same animation.

The narration is the owner's own voice. Everything else (rendering, timing, audio cleanup, captions, chapters, thumbnail, metadata, upload) is automated.

## 2. Decisions already made (2026-11-10)

| Decision | Choice |
|---|---|
| Repo and series name | `TechNaom/ai-explained-animated`, public, GitHub Pages |
| Narration | Owner's own voice |
| Formats | Long 16:9 video and a 9:16 Short from the same source |
| Channel | Existing channel; uploads are automated but **every video is uploaded as private** and the owner makes it public |
| Licence | Code Apache-2.0, content CC BY 4.0 |

## 3. The workflow, end to end

```
 Claude                         Owner                        GitHub Actions (free)                 YouTube
 ------                         -----                        ---------------------                 -------
 1. explainer page  ─┐
 2. narration script ─┼──────▶  3. records one audio file
                     │             per scene (phone is fine)
                     │          4. drops files in the repo ─▶ 5. clean audio
                                                              6. align audio to animation steps
                                                              7. render 1080p video + vertical Short
                                                              8. captions, chapters, thumbnail, metadata
                                                              9. draft release for review ──────────▶ 10. upload as PRIVATE
                                                                                                      11. owner checks in
                                                                                                          YouTube Studio and
                                                                                                          clicks Public
```

The owner's time per video: about 15 to 20 minutes to read and record the script, plus a final check.

## 4. Components

### 4.1 Explainer page (exists)

Each explainer is one self-contained HTML file with a scene and step engine: every step has a duration, a caption and a state change. For video, the engine gets two additions:

- **Render mode** (`?render=long` or `?render=short`): hides the controls, fills the frame (1920x1080 or 1080x1920), and exposes `window.EXPLAINER.goTo(step)` and the step list so the renderer controls timing.
- **Narration field** per step: the spoken text, which can be fuller than the on-screen caption.

The interactive page stays exactly as it is for learners.

### 4.2 Narration script

Generated from the explainer as `videos/<topic>/script.md`: one section per scene, a target length per scene, and plain spoken English. A long video script is roughly 350 to 500 words; the Short script is a condensed 110 to 140 words, recorded separately.

**Recording guide for the owner:** a quiet room, phone voice memo or laptop mic, phone held 15 to 20 cm away, one file per scene named `01-problem.m4a`, `02-retrieve-wide.m4a` and so on. Re-record any scene on its own; nothing else changes.

### 4.3 Audio cleanup

Using ffmpeg: trim leading and trailing silence, light noise reduction, and loudness normalisation to about -14 LUFS (what YouTube targets). Optional background music at about -22 dB from the YouTube Audio Library, so there are no copyright claims.

### 4.4 Alignment: matching the animation to the voice

- **Phase 2 (simple):** each scene's audio length sets that scene's duration, split across its steps in proportion to each step's narration word count.
- **Phase 2b (precise):** run open-source Whisper on the audio to get word timestamps, then start each step exactly when its first words are spoken. Whisper runs on the free Actions CPU in a few minutes for a 3-minute video.

### 4.5 Rendering

A headless Chromium (Playwright) opens the page in render mode at 1920x1080 (or 1080x1920), steps through it on the computed timeline, and captures the frames. ffmpeg encodes H.264 at 30 fps and muxes the narration.

**Open technical question, settled by a spike in Phase 1:** which capture method gives smooth, sharp video:

1. Playwright's built-in video recording (simplest; quality to be checked);
2. Chrome DevTools screencast frames;
3. deterministic frame-by-frame capture with Chrome's virtual time (sharpest; most work).

The spike renders explainer #1 silently with each method and the owner picks by eye.

### 4.6 Shorts

The Short needs a **portrait layout** of the same scenes (question at the top, chunk cards in the middle, LLM box below). Each explainer defines a second set of positions for portrait; the scene logic is shared. Captions are **burned in** for Shorts, because many viewers watch them muted.

### 4.7 Captions, chapters, thumbnail and metadata

- **Captions:** an SRT file built from the narration text and the step timings (Phase 2b: from Whisper's word timestamps). Uploaded as a caption track on the long video.
- **Chapters:** one per scene, written into the description (`00:00 The problem`, `00:42 Retrieve wide` and so on). YouTube needs the first at 00:00, at least three, each at least 10 seconds.
- **Thumbnail:** 1280x720 screenshot of a dedicated thumbnail scene with large title text.
- **Metadata:** title, description (summary, chapters, link to the live interactive page and the repo, licence credit) and tags, generated from the explainer and reviewed in the draft release.

### 4.8 Upload

YouTube Data API v3, from a GitHub Actions job that runs only when the owner approves (manual `workflow_dispatch`, or the `approve` label on the draft release).

- `videos.insert` uploads as **private**; `captions.insert` and `thumbnails.set` add the extras.
- **Known limits:**
  - **Videos from unverified API projects are locked to private** until Google's compliance audit is passed. That suits the approval rule here: the owner makes each video public in YouTube Studio.
  - **The OAuth consent screen must be set to "In production"**, not "Testing", otherwise the refresh token expires after 7 days and uploads silently stop.
  - **Custom thumbnails need a phone-verified channel.**
  - **Quota:** 10,000 units per day; one upload costs about 1,600, so about 6 uploads per day, far more than needed.
- The OAuth refresh token is stored only as a GitHub Actions secret, never in the repo.

## 5. Cost

| Item | Cost |
|---|---|
| GitHub Actions (public repo) and GitHub Pages | Free |
| ffmpeg, Playwright, Whisper | Free, open source |
| YouTube Data API | Free |
| Background music | Free (YouTube Audio Library) |
| **Total** | **Rs 0 per month** |

Nothing runs between videos: the pipeline starts when audio is pushed and stops when it finishes.

## 6. Isolation and safety

- Its own repo, its own Google Cloud project for the YouTube API, its own secrets. Nothing is shared with LBE, ACE or any other pipeline.
- Nothing goes public without the owner: uploads are private, and the owner flips them to public.
- No secrets in the repo; the OAuth token is a GitHub secret.

## 7. Phases

| Phase | What | Done when |
|---|---|---|
| **0** | Repo, GitHub Pages gallery, explainer #1, this design | Pages link works (done 2026-11-10) |
| **1** | Render spike: silent 1080p render of explainer #1 with the three capture methods | Owner picks the method by watching the three clips |
| **2** | Narration: script for #1, owner records it, audio cleanup, alignment, captions, chapters | A finished long MP4 with voice the owner is happy with |
| **3** | Shorts: portrait layout, short script, burned captions | A finished vertical Short |
| **4** | Upload automation as private, thumbnail, metadata | First video appears as private in YouTube Studio |
| **5** | Cadence: one new explainer and its two videos per week | Running |

## 8. Topic backlog (first ten)

1. RAG reranking (done as a page)
2. Chunking strategies: fixed, recursive, semantic, small-to-big
3. How vector search works: embeddings and HNSW
4. The agent loop: plan, call a tool, observe, repeat
5. MCP explained: one protocol between agents and tools
6. Prompt caching: why the second call is cheaper
7. LoRA fine-tuning in one picture
8. The KV cache and why long context is expensive
9. LLM evals: test sets, LLM-as-judge and regression gates
10. Guardrails: input, output and tool-call checks

## 9. Questions for the owner

1. Approve this design, or tell me what to change.
2. The channel name and handle, and whether the channel is phone-verified (needed for custom thumbnails).
3. Background music: yes or no?
4. Captions: burned into Shorts plus an SRT track on long videos. Is that OK?
5. Cadence: is one explainer per week right?
