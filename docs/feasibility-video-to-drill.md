# Feasibility: Turning DCI High-Cam Video into Interactive Drill Pages

*Research date: November 2026*

## 1. The idea

Take wide "high cam" footage of a drum corps show (the elevated press-box view that shows the whole field) and automatically produce:

1. **Drill charts**: a top-down dot chart of every performer at each set.
2. **Coordinate sheets**: per-performer pages ("2 steps inside the 35, side 1, 4 behind front hash") like the ones Pyware exports.
3. **An interactive viewer**: scrub through the show, tap a dot, follow one performer's path, step through sets count by count.

## 2. Verdict

| Part | Feasibility | Notes |
|---|---|---|
| Field registration (map video pixels to field yards) | **High** | A solved problem in sports analytics. Yard lines and hashes are ideal keypoints. |
| Detecting performers in each frame | **High** | Off-the-shelf detectors (YOLO family, SAM2) work. Someone has already trained a marching band detector on Roboflow. |
| Approximate positions at holds/sets | **Medium-High** | With 4K footage, about ±0.5 yd (roughly one 8-to-5 step) is realistic. Accuracy drops on the far side. |
| Keeping each performer's identity through the whole show | **Low-Medium (hardest part)** | Identical uniforms, dense crossings, flags and props blocking the view. Needs drill-specific motion priors plus a human correction UI. |
| Finding set boundaries and counts | **Medium** | Can be found from moments when the whole ensemble stops or changes direction. Many post-show videos have **no audio**, so count timing has to come from visual step cadence. |
| Interactive viewer and coordinate sheets | **High** | Plain front-end work. Existing apps (UDBapp, Drillbook, Drill Studio) show the format. |
| **Legal and commercial use of DCI footage** | **Risky** | DCI and FloMarching footage is copyrighted and licensed for personal viewing. Drill designs are creative work by paid designers. |

**Bottom line:** A working prototype is technically achievable with today's open-source computer vision, at "good enough to learn from" accuracy rather than "replace the designer's Pyware file" accuracy. The biggest risks are **identity tracking** (technical) and **rights to DCI footage** (legal and business). The strongest product is a general **video-to-drill engine** that a band runs on **its own footage**. The same engine can then also check how closely a rehearsal matches the intended drill.

## 3. What already exists (market check)

- **Pyware**: the industry-standard drill design suite. Designers write the drill there and export coordinate sheets.
- **UDBapp Pro (Ultimate Drill Book)**: imports a Pyware export and shows each member their coordinates set by set, with playback. This is the dominant learning app.
- **Drillbook Next**: reader and director versions. It imports Pyware, EnVision and Field Artist coordinate sheets. Reviews say it has not been updated in years.
- **Drill Studio**: drill creation with 3D preview. It generates coordinate sheets and exports PDFs and MP4s.
- **DrillVision (Roboflow Universe)**: a YOLO object-detection model trained on about 6,400 images of marching bands. It is evidence that detection is tractable, but it does **not** produce coordinates.

**Gap:** No tool found that goes **from video to drill**. Every existing tool starts from a designer's file. That gap is the opportunity, and also a sign the problem is not trivial.

## 4. How it would work (technical pipeline)

The design borrows from soccer "game state reconstruction", which already turns broadcast video into a top-down minimap.

```
Video ─► 1. Field registration ─► 2. Performer detection ─► 3. Tracking + identity
      ─► 4. Project to field coords ─► 5. Set/count segmentation
      ─► 6. Snap to drill grammar ─► 7. Human review UI ─► 8. Export + interactive viewer
```

1. **Field registration (homography).** Detect yard lines, hash marks and numbers, then fit a homography to a field template with RANSAC. Recompute it every frame if the high cam pans or zooms. Open references: SoccerNet Game State Reconstruction, PnLCalib (used by the open-source MatchVision project), Roboflow's football pitch keypoint tutorials. Watch out: **hash positions differ** between high school, college (NCAA) and NFL fields, and drill coordinates are defined relative to hashes. The user (or auto-detection) must pick the hash standard per stadium.
2. **Detection.** A YOLO model fine-tuned on high-cam frames, or SAM2 segmentation. The foot point of each box is what gets projected. Classify each performer by section (brass, battery, guard, front ensemble), because equipment silhouettes are distinct. That gives a free identity cue.
3. **Tracking and identity.** This is the main technical risk. General trackers (ByteTrack, SAM2-based MOT) swap identities when similar-looking people cross. 2026 research on dense sports scenes (SAMIDARE, LiAM-SAM) is working on exactly this but is not solved. **Drill-specific advantages to exploit:**
   - Members move in near-straight lines or arcs at a constant step size between sets. That is a very strong motion prior compared with soccer.
   - Each set is a recognizable form (lines, arcs, blocks, company fronts), so the solver can match identities across sets with an assignment algorithm (Hungarian) rather than only frame by frame.
   - Section labels cut down the possible swaps.
   - A human can fix the remaining swaps in a correction UI ("these two dots swapped at set 14").
4. **Projection.** Pixel foot points go through the homography to give (x, y) in yards, smoothed over time.
5. **Set segmentation.** Sets are where most of the ensemble's velocity drops to zero or changes direction. Counts between sets come from tempo. Audio is often stripped from archived DCI video, so estimate tempo from the visual stepping rhythm (periodic leg motion).
6. **Snap to drill grammar.** Turn noisy positions into the kind of coordinates a designer would write: round to quarter or half steps, fit lines and arcs to forms, keep intervals equal.
7. **Review UI.** Unavoidable for good quality. Overlay the dots on the video, drag to correct, fix swaps, adjust set boundaries.
8. **Outputs.** An interactive web viewer, per-member PDF coordinate sheets, and optionally a CSV or Pyware-style coordinate export.

### Accuracy math (rough)

- Field length including end zones is about 120 yd. At **1080p** that is roughly 16 px per yard horizontally, worse in depth on the far side because of perspective. At **4K** it is about 32 px per yard.
- One 8-to-5 step is 22.5 in, about 0.625 yd.
- Foot-point error of 3–6 px gives roughly ±0.2–0.4 yd at 1080p near side and worse far side, so **roughly ±1 step**. The snapping step (6) helps a lot because real drill sits on a grid.
- **Recommendation:** ask for 4K, a locked-off camera if possible, and a full-field view.

## 5. App-building options

| Approach | Strengths | Weaknesses | Fit |
|---|---|---|---|
| **Web app** (React/Next.js plus Canvas/WebGL or Three.js viewer) and a **Python GPU backend** (PyTorch, Ultralytics, SAM2) on Modal, RunPod, AWS or GCP | Fastest to build. Heavy computer vision belongs on GPUs. Works on every device. Easy to share links to drill pages. | GPU hosting costs. Uploading large video files. | **Recommended for MVP** |
| **Native iOS/iPadOS** (Swift, Core ML, Vision) | iPad is the main device for bands (UDBapp is iPad-first). On-device viewer works offline on the field. | Full-show tracking on device is heavy. Slower to build. Apple-only. | Phase 2 viewer |
| **Cross-platform mobile** (React Native or Flutter) as the viewer, sharing the cloud backend | One codebase for iOS and Android. Good for the interactive viewer. | Native computer vision still runs in the cloud. | Good Phase 2 alternative |
| **Desktop app** (Electron or Python + Qt) with local GPU | No hosting costs. Handles big files. Suits directors and staff. | Needs install and setup. Not everyone has a GPU. | Niche or power-user option |
| **No-code / low-code** (Bubble, Glide, FlutterFlow) | Quick landing page and waitlist. | Cannot do the computer vision. Fine only for the shell around it. | Marketing only |

**Suggested stack:** Next.js front end, PixiJS or Three.js for the field and dot viewer, FastAPI or Python workers for the pipeline (OpenCV, Ultralytics YOLO, SAM2, a tracker, SciPy for assignment), a job queue, S3-style storage for video, and Postgres for shows, sets and performers.

## 6. Legal and business risk (important)

- DCI states that its content is copyrighted and that recordings, **even for personal use**, are not allowed without permission. FloMarching licensing is limited, and audio is often removed for copyright reasons.
- Drill is the creative work of a designer, and choreography can be copyrighted. Publishing reconstructed DCI drill pages, especially for sale, could invite takedowns. Get real legal advice before building a business on DCI footage. *(This is not legal advice.)*
- **Safer, larger market:** high school and college bands film their **own** rehearsals and shows from the press box. Uses for them:
  - **Drill accuracy checker:** import the official Pyware file and compare it with what was actually marched. Show who is off their dot, by how much, at which set. Nothing in the market does this today, and directors would value it.
  - **Reconstruct lost drill:** for bands that have only video of an old show.
  - **Fan and education mode with DCI's blessing:** a licensing partnership with DCI or FloMarching, or with the designers themselves.

## 7. Phased plan and rough effort

| Phase | Scope | Rough effort (1–2 developers with computer vision experience) |
|---|---|---|
| **0. Spike** | One high-cam clip. Manual 4-point field calibration. Off-the-shelf YOLO. Plot dots per frame on a top-down field. | 1–2 weeks |
| **1. Prototype** | Automatic field registration. Fine-tuned detector (label about 1–2k high-cam frames). Tracking with drill motion priors. Set detection. Basic web viewer. | 1–2 months |
| **2. MVP** | Correction UI, snapping to drill grammar, PDF coordinate sheets, accounts, uploads, GPU job queue. | 2–3 more months |
| **3. Product** | Pyware import and the accuracy checker, iPad viewer, multi-camera support, section and instrument classification, handling props. | Ongoing |

**Biggest unknowns to test in the Phase 0/1 spike:**

1. How often do identities swap in a dense DCI show, and how many minutes of manual fixing does one show need?
2. What foot-point accuracy is actually achievable at 1080p versus 4K?
3. Can visual cadence alone recover counts and tempo reliably?

## 8. Sources

- Roboflow DrillVision model: https://universe.roboflow.com/drillvision-324/drillvision-3-24
- Pyware: https://pyware.com/about
- UDBapp Pro: https://apps.apple.com/us/app/udbapp-pro/id1455826724
- Drillbook Next: https://apps.apple.com/us/app/drillbook-next/id547116306
- Drill Studio: https://apps.apple.com/app/id1485526505
- Soccer video to 2D field view (2026): https://arxiv.org/html/2604.08722
- SoccerNet Game State Reconstruction: https://arxiv.org/html/2404.11335v1 and https://arxiv.org/html/2504.06357v1
- MatchVision (YOLO + ByteTrack + PnLCalib homography): https://github.com/BlazeWild/MatchVision-AI-Sports-Video-Analytics-Tracking-Pipeline
- Football-PitchVision: https://github.com/DesusLove/Football-PitchVision
- Roboflow, tracking football players: https://blog.roboflow.com/track-football-players
- Field registration with RANSAC (MDPI Sensors): https://mdpi.com/1424-8220/23/18/7938
- SAMIDARE, dense-scene tracking: https://arxiv.org/abs/2604.22162
- LiAM-SAM: https://arxiv.org/pdf/2609.28078
- SAM-MT: https://arxiv.org/pdf/2607.08688
- DCI photo and video policy FAQ: https://www.dci.org/news/clarification-of-dci-photo-policy-an-faq
- DCI on music licensing: https://www.dci.org/?p=33078
- FloMarching FAQ: https://www.flomarching.com/articles/5067014-flomarching-faqs
