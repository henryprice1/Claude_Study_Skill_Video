---
name: "study-video"
description: "Turn study material (textbook pages, notes, outlines) into a narrated, chaptered video-lesson web page with animated maps and charts, on-screen captions, fun facts, research-backed learning checks, and a narrator picker."
---

# Study Video

Build a narrated "video lesson" that teaches a student every section of their material as if they know none of it. It is a web page that plays like a video:

- A 16:9 screen with animated maps, charts, timelines and diagrams.
- Recorded AI narration.
- One-line captions shown on the video.
- A chapter for each homework section, each ending with a fun fact.
- Learning checks between chapters: guess first, quick quiz, explain it yourself.
- A narrator picker.

Publish it as an artifact. The audience is a high school student, so use plain, warm, concrete language. It is a web page, not an MP4; say so once in the reply.

## Research the design follows (keep every one; don't overclaim)

- **Multimedia and modality** (Mayer): spoken narration plus visuals beats on-screen text. Every scene is visual; never put walls of text on screen.
- **Temporal contiguity and signaling:** map items and chart rows appear as they are named. The reveal is staggered across the first 55–60% of the scene.
- **Segmenting:** short scenes and chapters the learner controls (pause, back, speed, jump).
- **Personalization:** a conversational "you" voice. Modern synthetic voices did about as well as a human voice in one 2017 study (Craig & Schroeder).
- **Pretraining:** define every term the first time it is used.
- **Redundancy effect:** reading every word while listening can hurt. Captions show only the current sentence, on the video, with a CC toggle.
- **Pretesting effect** (Richland, Kornell & Kao 2009): one guess-first question before each chapter. Show the answer right after the guess.
- **Interpolated testing / testing effect** (Szpunar, Khan & Schacter 2013; Roediger & Karpicke 2006): a 3-question quiz at the end of each chapter, with feedback and a memory hook.
- **Self-explanation** (Chi et al. 1994): after the quiz, the student explains it like teaching a friend. Then compare with a model answer, with key-point chips lit green when they appear in the student's text.
- **Spacing** (Cepeda et al. 2006): the end screen and the research panel send the student to spaced practice in a study trainer (the study-train skill).
- **Interest:** one "Fun fact" scene per chapter. Each is true, specific and surprising, and labeled "From your book", "Book + beyond" or "Beyond your book". Prefer facts from the book; never invent facts.
- On the page, put the research in a collapsible "Why the video is built this way" panel with one line per technique and its citation. In the reply, say "well-supported techniques", never "guaranteed".

## 1. Read the material

- Read every upload in full: text files, PDFs (extract the text) and photos (view them).
- Use the book's own names, dates and numbers. If the book differs from common usage, follow the book and flag it (e.g. the book dates Unified Silla 688; most sources say 668).
- If the material is only an outline of headings, fill gaps from standard course knowledge and say which parts came from outside the pages.
- Organize chapters by the student's own units ("HWK 6", "Ch. 2 · Africa"), in the book's order. Put primary-source features (Historians' Voices, Working with Evidence, Looking Again, Zooming In) inside the chapter they belong to.

## 2. Write the script (scenes.js)

The script is an array `SC` of scenes. Each scene is `{ch, say, v}`:

- `ch`: the chapter id. A number for HWKs, or a string ("af", "am") for extra sections.
- `say`: the narration.
  - 2–6 short sentences of 25 words or fewer.
  - Define every term the first time it appears, and work in every key term from the material.
  - Address the student as "you". Avoid stock phrases, em-dash asides and colon reveals.
- `v`: the visual. Make every scene visual.

Each chapter has:

- a title card showing the unit's main-point question
- 3–7 teaching scenes
- a final fun-fact scene whose narration opens with "Fun fact!"

Cover everything and skip nothing. A full unit runs about 25–40 minutes, roughly 4,000–5,500 words.

Also define:

- `CHAPTERS`: `[[id, short name, on-screen tag]]`
- `P`: places as `[lon, lat]`
- `R`: approximate historical regions as `[lon, lat]` polygons with 8–20 points
- `RT`: routes, with ocean waypoints so sea routes stay on water

Visual types (`v.t`):

- `title`: `{hwk, title, pages, q}`.
- `fact`: `{head, src}`. A big "!" badge, the headline, and a source pill.
- `map`: `{bb:[w,s,e,n], regions:[[key,color,label,labelPos?]], pts:[[name,[lon,lat],sub,pos?]], routes:[{p,c,arrow,both,lbl}], bands?}`.
  - Colors are `a`–`e`.
  - Point label `pos` is `l`/`r`/`a` (above)/`b` (below).
  - `labelPos` is `[lon, lat]`; use it to move a region label off city labels.
- `bars`: `{title, unit, data:[[label,value]], note, log?}`. Label approximate figures as approximate.
- `timeline`: `{title, from, to, ev:[[year,label]], span?}`. Use negative years for B.C.E.
- `gantt`: `{title, from, to, rows:[[label,start,end,color]]}`. For overlapping empires or reigns.
- `cards`: `{title, items:[[term, short line]]}`, 4–6 items.
- `compare`: `{title, cols, rows, note?}`.
- `flow`: `{title, steps:[[step, line]]}`, 4 steps, for causes, processes and arguments.
- `pie`: `{title, value, label, reasons}`.
- `pyramid`: `{title, levels}`.
- Custom one-offs drawn with HTML or canvas: `fields` (crop rotation), `monsoon` (wind arrows that flip halfway through), `axis` (continent orientation).

## 3. Learning-check data (quiz.js)

Build `QZ = {chapterId: {pre, quiz:[3], ex}}` at build time.

- **Source:** if a study-train deck exists for the subject, draw the questions from it. Otherwise write them yourself.
- **Only taught facts:** only use items whose answer actually appears in that chapter's narration (normalize and match the text). Spread the picks in order of first mention.
- **Format:** each item is `{q, ans, d:[3 distractors], h:hook}`. Distractors must be real look-alikes.
- **Capitalization:** make the answer match its distractors so case never gives it away. Title-case proper names, keep small words like "and/of/al" lowercase, and capitalize hyphen parts.
- **Explain prompt (`ex`):** `{q, m: model answer of 30–45 words, k: 3–4 key points}`.
- **Skip** intro and wrap-up chapters.

## 4. Build the player (one HTML page)

Use a single-look dark "atlas at night" theme: navy sea, sand land, five accent colors, a serif display font plus a condensed label font. Load d3 7.8.5 from cdnjs and inline everything else.

**Map data**

- Run `npm i world-atlas@2.0.2 topojson-client@3.1.0 d3@7.8.5`.
- Convert `land-50m` to GeoJSON, round coordinates to 0.05°, and embed it as `const LAND` (about 800 KB).
- Fix the ring winding after rounding. Reverse an exterior ring whose `d3.geoArea` is over 2π, and reverse a hole whose area is under 2π.
- Check that land is about 0.287 of the globe. Otherwise land paints over the whole map.

**Map drawing**

- Canvas with `d3.geoMercator`, at device-pixel ratio up to 3.
- `fitExtent` to the scene's bounding box with a 3–7% margin. Leave the bottom 16% free when captions are on.
- Ease between bounding boxes over about 1.3 s.
- Regions: Catmull-Rom closed curves with a translucent fill and dashed stroke.
- Routes: drawn progressively, then marching dashes, arrowheads and a mid-route label.
- Points: pulsing dots with haloed labels.
- Stagger the reveals: regions, then routes, then points.
- Resize the canvas with a ResizeObserver on the screen element.

**HTML visuals**

- Elements start with the class `.rv` and are revealed in sequence. Size everything in `cqw` on the 16:9 container.
- Each scene fades in.
- Bars animate their width. Make `.fill` `position:relative`, and put the value label after short bars.
- Gantt bars under about 18% wide get an outside label.
- Timeline labels alternate above and below the axis, and the span label sits under everything.

**On-screen captions**

- An absolute caption bar at the bottom center of the screen shows the current sentence only. It has a dark translucent background, a balanced wrap and a size of about 2.35cqw.
- A CC toggle, default on.
- Hide the captions while a check is open. Put the chapter tag at the top right so the two never collide.

**Learning checks**

The check is an overlay covering the screen. While it is open, the clock holds, narration stops, and keys 1–4 and Enter work.

- **Guess first:** when a chapter's first scene loads during playback, show the `pre` question. After the guess, show the right answer ("Watch for it"), then a "Start chapter" button. A skip link is always there.
- **Quick check:** when the last scene of a chapter ends on its own, show the 3 quiz questions. Each answer gets right/wrong highlighting, the correct answer and the 💡 hook. Then show the explain prompt (a textarea), and after "Compare", the model answer and the key-point chips.
- Save scores in localStorage and show "✓ 2/3" on the chapter grid.
- A "✓ Checks on/off" toggle.
- Manual navigation closes any open check.

**Controls**

- A large ▶ play button over the screen, showing the real duration.
- Play/pause, back, next, a speed menu (0.85–1.5×), narration on/off, CC, checks, and ⛶ full screen (`requestFullscreen` on the screen element, with errors caught).
- A progress bar with ticks at chapter boundaries; click it to seek.
- A chapter grid, a narrator picker with ▶ Preview buttons, and the research panel.
- Space plays and pauses; ←/→ move between scenes.
- localStorage, in try/catch, holds the current scene, the voice and the scores.
- Add your own `[hidden]{display:none!important}` rule so overlays hide in local tests too.
- Before Play is pressed, show the current scene fully revealed, so the page looks complete at rest.

## 5. Narration

Record with Kokoro TTS. It runs offline; the model comes from GitHub releases.

- Run `pip install --break-system-packages kokoro-onnx soundfile`.
- Download `kokoro-v1.0.onnx` and `voices-v1.0.bin` from `github.com/thewh1teagle/kokoro-onnx/releases/download/model-files-v1.0/`.
- Voices:
  - `af_heart`: Heart, American woman
  - `am_michael`: Michael, American man
  - `bf_emma`: Emma, British woman (`lang='en-gb'`)
  - `bm_george`: George, British man (`lang='en-gb'`)
- The default is a two-voice mix: Heart and Michael alternating by chapter.
- Split sentences with this exact regex in both the generator and the page: `/[^.!?]+[.!?]+["’”]?\s*/g`.
- Cache each sentence clip as `.npy`, keyed by the md5 of the sentence. Script edits then only re-record new sentences, and a quality change only re-encodes.
- Put 0.28 s of silence between sentences.
- Encode each scene with ffmpeg as mono MP3 at 64 kbps, saved as `audio/<voice>/sNN.mp3`.
- Save `meta.json` with each scene's duration and each sentence's start offset.
- Record a one-line `sample.mp3` per voice ("Hi, I'm Emma…").
- Respell hard names for the voice only (HWK → homework, Chinggis, Huitzilopochtli, Tenochtitlán, Srivijaya, Quechua and similar). Captions keep the real spelling.
- **Time:** about 10–20 s of CPU per new scene on 2 cores, so roughly 25 min per voice for a 35-minute lesson.
  - Run it in the background with `run_in_background`, one voice after another.
  - Publish once the first two voices are done; add the others later.
- **Playback:**
  - One `Audio` element; seek to the sentence offset; `playbackRate` comes from the speed menu.
  - `timeupdate` drives the caption. On `ended`, wait about 0.9 s, then go to the next scene or the quiz.
  - The animation clock follows `au.currentTime`. Prefetch the next scene's audio.
  - Switching voices resumes from the same sentence.
  - **Fallbacks:** if the audio fails, use the browser's built-in speech one sentence at a time with a safety timeout. If narration is off, advance on a word-count timer (about 2.55 words/s).

## 6. Test, then publish

1. **Script checks**
   - Run `node --check` on the combined script.
   - Every region key exists, every point has coordinates, and every `ch` is in `CHAPTERS`.
   - Every fact scene is the last scene of its chapter.
2. **Visual checks**
   - Use Playwright with the pre-installed Chromium. Route the d3 CDN request to the local `node_modules/d3/dist/d3.min.js` and abort font requests.
   - Add a test hook `window.__V`: `load`, `seek`, `setVoice`, `setChecks`, `idx`, `t`, `playing`, `holding`, `SC`.
   - Screenshot every map scene and each chart type after `seek(est*0.95)`, and review them on contact sheets. Fix every overlapping or clipped label using label positions and `l`/`r`/`a`/`b`.
3. **Checks flow** (setTimeout sped up 20×)
   - A chapter jump opens the guess-first screen.
   - Answering with keys works, and Enter starts the chapter.
   - The quiz opens at the chapter end; three answers lead to the explain screen, then the compare screen, then the next chapter.
   - The score badge shows on the chapter grid. There are no page errors.
4. **Fast play-through** (setTimeout sped up 60×, checks off, voice off, 420 px wide)
   - It reaches the last scene and stops.
   - There is no horizontal scroll.
   - Prev, chapter jump and arrow keys all work.
5. **Audio test** (`--autoplay-policy=no-user-gesture-required`)
   - Captions follow the audio.
   - Pause holds the clock and resume continues it.
   - Switching voices keeps playing.
   - The last two scenes end cleanly. No page errors.
6. **Publish**
   - Use the Artifact tool, icon `map`. Pass the audio through `files` as a plain list of `{path}` entries with `root` set to the build folder.
   - Each publish takes at most 255 files and 64 MB, so publish voices in batches.
   - Reuse the same paths (`audio/<voice>/sNN.mp3`) when scenes change, so old files are overwritten. Remove leftovers with `null`.
   - Republish to the same file path to keep the URL.

## 7. Reply

- **What it is:** one line, with chapters, length and the narrator choice.
- **How to use it:** 2–3 lines on watching a chapter, answering the checks, and the CC, speed, full-screen and voice-preview controls.
- **Testing:** one line on what you tested.
- **Sources:** anything from outside the student's pages, and where the book's date or name differs from common usage.
- Recommend spaced practice in a study trainer afterward (offer the study-train skill if they don't have one). Don't recap the build steps.
