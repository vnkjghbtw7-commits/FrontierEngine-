# Roadmap: Video-to-Drill App

**Roles:** You direct. You set priorities, supply footage, review results, and make go/no-go calls. Claude builds: code, models, evaluation, UI.

## How to read the timeline

- Writing the code for one component takes me **hours to a few days**. That is not the bottleneck.
- Calendar time depends on four things:
  1. **Data:** footage, plus hand-checked "correct answers" to measure accuracy against.
  2. **GPU compute:** this cloud workspace has **no GPU** (4 CPUs, 16 GB RAM). Training and full-show processing need a cloud GPU account such as Modal or RunPod.
  3. **Your review cycles:** how quickly you look at results and decide.
  4. **Research iteration:** the tracking accuracy work is trial and error. It's measured, not guessed.
- Estimates assume you respond within about a day and we work most weekdays. Each work session starts fresh, so **GitHub is the project's memory**. GitHub access has to be fixed first.

## Decisions (Nov 2026)

- **Footage:** support both DCI footage and bands' own footage from the start.
- **Field types:** high school, college (NCAA) and NFL hash marks.
- **Test data:** plan for **video only**. Pyware files may come later. Until then, accuracy is measured against hand-checked tracks.
- **Compute:** the director's home PC (NVIDIA RTX 30/40/50 series), connected to the session through Remote Control. Cloud GPU only if needed.
- **Labeling:** the director corrects the pre-labeled frames.
- **Check-ins:** a weekly report with metrics and the decisions needed.
- **Tracking target:** 20 or fewer manual fixes per full show (stretch goal: 5).

**Legal caution:** DCI footage is used for internal research and development only. Don't publish drill reconstructed from it until rights are cleared.

## Guiding principle: measure everything

Every phase has a **test set** (clips with known correct answers) and a **metric**. We only move on when the metric passes the gate.

The best test data is **your own band's footage plus its Pyware drill file**. The drill file tells us exactly where every performer was supposed to be, so it works as an answer key with no manual labeling.

---

## Phase 0: Setup and data (week 1)

**What I build**
- Repo structure.
- A pipeline to load video and pull out frames.
- A test harness for measuring accuracy.
- A simple labeling tool for clicking on frames.

**What you supply**
- Fixed GitHub access.
- A GPU provider account. I'll ask for its key through a secure form, never in chat.
- **3–5 high-cam clips** covering different stadiums and lighting, 4K if possible.
- Ideally, one show with its Pyware file.

**Decisions for you:** primary target (DCI footage, or bands' own footage), and which field types to support (high school, college or NFL hashes).

## Phase 1: Field registration (weeks 1–3)

**Goal:** map pixels to exact field coordinates.

**How**
- v1: click 4 field points by hand per clip. This works on day 1.
- v2: detect yard lines, hashes and numbers automatically and handle camera pans and zooms frame to frame.
- I generate an overlay video that draws the field grid on the footage, so you can check it by eye.

**Gate:** grid lines land within about 0.25 yd on the test clips.

## Phase 2: Performer detection (weeks 2–5)

**Goal:** find every performer in every frame, plus the point where their feet touch the ground.

**How**
- Start with off-the-shelf detection models.
- Then fine-tune on about 1–2k labeled high-cam frames. I pre-label the frames automatically, and you or a paid labeling service correct them. Labeling is the main human time cost in this phase.

**Gate:** at least 97% of performers found at holds, and foot-point error of about 0.3 yd or less.

## Phase 3: Tracking and identity tags, the core research (weeks 4–12)

**Goal:** each performer's tag stays on the right person for the whole show.

**How**
- Build a ground-truth test set: about 3–5 minutes of hand-checked tracks, or the Pyware answer key.
- Then add techniques one at a time, measuring each:
  1. Motion prediction (baseline tracker).
  2. Section and instrument classification (brass, battery, guard, front ensemble).
  3. A visual fingerprint for each tag.
  4. Re-checking all tags at each set, when the form is spread out.
  5. Set-to-set matching using drill rules: same counts, straight paths, steady step size.
  6. Physical sanity checks: no teleporting, no impossible step sizes.

**Metric:** tag swaps per minute, and **manual fixes needed per full show**.

**Gate:** 20 or fewer fixes per show, with a stretch goal of 5 or fewer.

**Risk:** this phase is the least predictable and could take longer. You'll get weekly progress numbers so you can see the trend and decide whether to keep pushing.

## Phase 4: Sets and counts (weeks 8–12, alongside Phase 3)

- Find set boundaries from moments when the whole ensemble stops or changes direction.
- Find tempo from audio when present, otherwise from the visible stepping rhythm.
- Label each move with its count number.

**Gate:** 90% or more of set boundaries within 1 count of the truth.

## Phase 5: Drill cleanup and coordinate output (weeks 10–14)

- Snap positions to quarter or half steps.
- Fit lines, arcs and even intervals to forms.
- Output Pyware-style coordinates ("2 steps inside the 35, side 1, 4 behind front hash").
- Per-performer PDF coordinate sheets, plus CSV export.

**Gate:** 90% or more of coordinates within 1 step of the answer key.

## Phase 6: Viewer and correction screen (basic version from week 6, polished by week 16)

- A basic top-down dot viewer early on, so you can **see** progress throughout Phases 3–5.
- Then the full interactive app:
  - Scrub through the show.
  - Tap a performer to follow their path.
  - Step through sets count by count.
  - Watch the video side by side with the dots.
  - Correction tools: "these two swapped", drag a dot, move a set boundary.

**Gate:** you can fix one full show in under 30 minutes.

## Phase 7: Private beta (weeks 16–22)

- Accounts, video uploads, a GPU job queue, storage, and cost tracking (dollars per processed show).
- 2–3 real directors or bands use it. Collect their feedback and fix problems.

**Decisions for you:** pricing model, and whether to pursue DCI, FloMarching or designer partnerships before public use of DCI footage.

## Phase 8: v1 launch and follow-ons (week 22 onward)

- **Accuracy checker:** import a Pyware file and compare it with what the band actually marched. This is the strongest feature for bands.
- iPad viewer app.
- Multiple camera angles.
- Handling props.
- Faster processing.

---

## Summary timeline

| Weeks | Milestone you can see |
|---|---|
| 1 | Field grid overlaid on your footage |
| 3 | Automatic field mapping on all test clips |
| 5 | Every performer boxed on video, plus a top-down dot plot |
| 8 | First full show tracked, with a swap count measured |
| 12 | Tracking at or near target. Sets and counts detected. |
| 14 | Coordinate sheets generated from video |
| 16 | Interactive viewer and correction screen usable by you |
| 22 | Private beta with real bands |

**About 4–6 months to beta.** Most of the uncertainty is in Phase 3.

## Rough running costs during development

- **GPU:** roughly tens to low hundreds of dollars per month during training and testing.
- **Labeling:** optional. Your time, or a few hundred dollars for a labeling service.
- **Hosting in beta:** small. The main cost is GPU time per processed show, which we'll measure in Phase 7.
