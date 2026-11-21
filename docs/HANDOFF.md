# Handoff: Video-to-Drill App (paste this into a new Claude Code session on the director's PC)

You are taking over a project from an earlier cloud session. Read this whole file before doing anything.

## 1. Roles

- **The user is the director, not the builder.** They set priorities, supply footage, label frames, review results and make go/no-go calls. They don't write code.
- **You (Claude) build everything:** code, models, evaluation, UI. Explain things in plain language and keep questions to the decisions only the director can make.
- This session runs **on the director's own PC**, which has an **NVIDIA RTX 30/40/50-series GPU**. Use it for training and processing. That's why the work moved here.

## 2. The project

An app that turns **high-cam marching band footage** into **interactive drill pages**:

- A **top-down dot chart** of every performer at each set.
- **Per-performer coordinate sheets** in Pyware style, for example "2 steps inside the 35, side 1, 4 behind front hash".
- An **interactive viewer**: scrub through the show, tap a performer, follow their path, step through sets count by count.

High cam means the wide, elevated press-box view of the whole field, as in DCI (Drum Corps International) broadcasts.

Pyware is the industry-standard drill design software. Apps like UDBapp import Pyware files, but **no existing tool goes from video to drill**. That's the gap this project fills.

## 3. Repository

- GitHub: `vnkjghbtw7-commits/FrontierEngine-`
- Branch: **`claude/video-drill-app-feasibility-k9zk4o`**. Clone it, check it out, and work and push here.
- Existing docs on that branch. Read them first:
  - `docs/feasibility-video-to-drill.md`: feasibility study, technical pipeline, accuracy math, market check, legal notes, sources.
  - `docs/roadmap.md`: the phased roadmap with a pass target for each phase, plus the director's decisions.
  - `docs/HANDOFF.md`: this file.
- No code exists yet. You start at **Phase 0**.
- Commit often with clear messages and push to the branch. GitHub is the project's memory between sessions.

## 4. Director's decisions (already made, don't re-ask)

| Topic | Decision |
|---|---|
| Footage | Support **both DCI footage and bands' own footage** from the start |
| Field types | **High school, college (NCAA) and NFL** hash marks. Hash spacing differs, and drill coordinates reference hashes, so the field type is chosen per video. |
| Test data | Plan for **video only**. Pyware files may come later. Until then, accuracy is measured against **hand-checked tracks**, so build a tool for the director to check and fix tracks. |
| Compute | **This PC's RTX GPU**. Cloud GPU only if truly needed, and ask first. |
| Labeling | **The director corrects labels.** You pre-label automatically, they fix. |
| Check-ins | **A weekly report**: metrics, progress against targets, and the decisions needed |
| Tracking target | **20 or fewer manual fixes per full show** (stretch goal: 5) |
| Legal | DCI footage is copyrighted. Use it for **internal research and development only**. Never publish or distribute drill reconstructed from DCI footage, and don't put DCI footage in the Git repo. |

## 5. Technical approach (summary; details in the feasibility doc)

```
Video → 1 field registration (homography) → 2 performer detection → 3 tracking + identity tags
      → 4 project to field coordinates (yards) → 5 set and count detection
      → 6 snap to drill grammar → 7 correction UI → 8 export + interactive viewer
```

1. **Field registration:** first version is a manual 4-point click per clip. Then detect yard lines, hashes and numbers automatically, fit a homography with RANSAC, and follow pans and zooms frame to frame.
2. **Detection:** YOLO (Ultralytics) or SAM2. Fine-tune on about 1–2k high-cam frames, which you pre-label and the director corrects. Use each person's **foot point** for projection.
3. **Tracking and identity (the hardest part, which is research):** the director calls these "tags". The program gives each performer an ID and has to keep it on the right person despite identical uniforms and dense crossings. Layer these, and measure each one:
   - A motion-prediction baseline (Kalman filter / ByteTrack).
   - Section and instrument classification (brass, battery, guard, front ensemble).
   - A visual fingerprint (re-identification) for each tag.
   - Re-checking every tag at each set, when the form is spread out.
   - **Drill-rule matching from set to set:** everyone moves on the same counts, mostly in straight lines at a steady step size. Solve it as an assignment problem (Hungarian algorithm).
   - Physical sanity checks.
   - A manual "these two swapped" fix that carries through the rest of the show.
4. **Projection** to yards, with smoothing.
5. **Sets and counts:** set boundaries are where ensemble-wide speed drops or direction changes. Tempo comes from audio if present. Archived DCI footage is often silent, so otherwise estimate it from the visible stepping rhythm.
6. **Snapping:** round to quarter or half steps (an 8-to-5 step is 22.5 in, about 0.625 yd) and fit lines, arcs and even intervals.
7. **Correction UI** and **viewer**.
8. **Exports:** Pyware-style coordinates, per-performer PDFs, CSV.

**Accuracy reality:** 1080p gives about 16 px per yard and 4K about 32 px per yard, so expect roughly ±1 step before snapping. Far-side accuracy is worse.

**Suggested stack:**
- Python for the pipeline: OpenCV, PyTorch with CUDA, Ultralytics, SAM2, SciPy, ffmpeg.
- Next.js front end with PixiJS or Three.js for the viewer and correction UI.
- FastAPI between them.
- SQLite or Postgres for data later.

Start local-first. Hosting comes in Phase 7.

## 6. Roadmap (targets to pass; full detail in docs/roadmap.md)

| Weeks | Phase | Target |
|---|---|---|
| 1 | 0. Setup: repo structure, video and frame pipeline, accuracy-test tools, labeling tool | — |
| 1–3 | 1. Field registration | Grid lines within about 0.25 yd |
| 2–5 | 2. Detection | At least 97% of performers found at holds; foot point within about 0.3 yd |
| 4–12 | 3. Tracking and identity | 20 or fewer fixes per show |
| 8–12 | 4. Sets and counts | 90% of set boundaries within 1 count |
| 10–14 | 5. Snapping and coordinates | 90% of coordinates within 1 step of hand-checked truth |
| 6–16 | 6. Viewer and correction UI (basic viewer early) | Director fixes a show in under 30 minutes |
| 16–22 | 7. Private beta | Real bands get usable drill |
| 22+ | 8. v1: Pyware import and accuracy checker, iPad viewer, multiple camera angles | — |

Principle: **measure everything.** Every phase has a test set and a metric. Show the director visual output early (overlay videos, dot plots) so they can judge progress by eye.

## 7. Your first steps in this session

1. **Check the machine:** run `nvidia-smi` and report the GPU model and memory, the OS, and whether Python, Git and ffmpeg are installed.
2. **Set up the environment.** Explain each install in one line and ask before installing system-wide software. The setup:
   - A Python virtual environment.
   - **PyTorch with CUDA**, matching the installed driver.
   - OpenCV and Ultralytics.
   - Confirm `torch.cuda.is_available()` is true.
3. **Clone the repo and check out the branch** if you aren't already in it. Read the docs in `docs/`.
4. **Footage:** ask the director where their high-cam clips are, or have them put the clips in a `footage/` folder. Add `footage/` to `.gitignore` so video never goes into Git. Record each clip's resolution, frame rate, length, stadium and field type.
5. **Build Phase 0:**
   - Project structure: `pipeline/`, `tools/`, `viewer/`, `eval/`.
   - Video ingest and frame extraction.
   - A manual 4-point field-calibration tool that writes a **field-grid overlay video**, the first visible milestone.
   - A simple labeling and correction tool.
6. Commit, push, and give the director a short plain-language summary of what's done, what they should look at, and what you need from them.

## 8. Working style

- Lead with results and keep reports short and plain. The director isn't technical.
- Ask only for decisions only the director can make, and offer a recommended option.
- Report test results honestly. If something misses its target, say so with the numbers.
- Never commit footage, secrets or API keys.
