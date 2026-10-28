# Traffic Analysis

A computer-vision application that turns CCTV traffic recordings into vehicle
counts, a breakdown by vehicle type, and traffic volume over time.

Built as a shared internship portfolio project by **NoneTP** and **Puntawit**,
combining ML model development with a reliable video-to-database pipeline and a
local web dashboard.

**Current status: planning.** Implementation has not started. This README
describes the intended phase-one product; the preview and capabilities below are
design targets, not completed features or measured results.

For development ownership and task status, start with the
**[central team plan](docs/superpowers/plans/2026-10-28-traffic-analysis.md)**.

## What the finished first version will do

Given a preselected traffic recording, the system will identify vehicles, track
them through the camera view, and count each tracked vehicle once when it crosses
a configured line. Results will be saved in PostgreSQL and shown in a React
dashboard that updates during processing.

The four target vehicle types are:

- Car
- Truck
- Motorcycle
- Pickup truck

The result will let someone answer questions such as:

- How many vehicles crossed the selected point during this recording?
- How many belonged to each vehicle type?
- Which minutes had the highest traffic volume?
- How far has analysis progressed, and are the displayed results complete?

## Intended user experience

1. **Prepare a recording.** A developer registers a local video and configures a
   counting line for its camera view. Phase one uses preselected files.
2. **Start analysis.** The developer launches the processor from the command line.
   A new processing run records the source, model identity, and settings.
3. **Open the local dashboard.** Select the recording and processing run in the
   browser.
4. **Watch results arrive.** Counts, volume charts, and progress refresh roughly
   every four seconds. In-progress results are labeled partial.
5. **Review the finished run.** Inspect the statistics and download the annotated
   video showing vehicle boxes, class labels, track IDs, and the counting line.

The processor, API, and dashboard run separately, so viewing statistics does not
require keeping a video-processing request open in the browser.

## Dashboard preview

Conceptual layout only. All numbers below are illustrative and are not model
evaluation results; visual styling will be decided during implementation.

```text
+--------------------------------------------------------------------+
| Traffic Analysis                                                   |
| Recording: [Intersection 01 v]   Run: [run-002 v]                    |
+--------------------------------------------------------------------+
| PROCESSING - PARTIAL RESULTS                                       |
| Video analyzed: 06:20 / 10:00                      Progress: 63%      |
+--------------------------------------------------------------------+
| Total: 530 | Cars: 345 | Trucks: 51 | Motorcycles: 96 | Pickups: 38  |
+--------------------------------------------------------------------+
| Traffic volume                                                     |
|                                                                    |
|              [Crossings per minute: bar chart]                      |
|                                                                    |
| Horizontal axis: elapsed time in the recording                      |
| Current minute: marked partial                                     |
+--------------------------------------------------------------------+
| Minute-by-minute counts table                                      |
| Annotated video: available after successful completion              |
+--------------------------------------------------------------------+
```

| Dashboard element | Intended behavior |
| --- | --- |
| Recording and run selectors | Inspect one recording and one analysis run at a time. |
| Total and per-type counts | Show recorded line crossings for the selected run. |
| Traffic volume chart and table | Show crossings in one-minute intervals using the recording's timeline. |
| Progress and status | Distinguish processing, completed, and failed runs; identify partial data. |
| Connection/progress warnings | Keep the last successful statistics visible if refresh fails, and flag stale processing progress. |
| Annotated output | Offer a finalized video download after successful processing; browser playback depends on verified codec support. |

Live annotated-video streaming into the browser is outside phase one. The
dashboard updates statistics while the completed annotated video becomes
available afterward.

## What a vehicle count means

A count is a **line-crossing event**, not a count of every vehicle detection in
every frame. A tracked car visible for 100 frames should still contribute one
crossing when it passes the configured line.

Traffic volume uses time within the recording. Processing a ten-minute clip in
five minutes does not turn its traffic into five minutes of observations. The
current incomplete minute is marked partial, and unprocessed time is not shown
as observed zero traffic.

Each analysis has its own run identity. Reprocessing the same video creates a
separate run, allowing results to be inspected without adding repeat analyses
together. Detector mistakes and tracker identity switches can still cause count
errors; these will be measured and documented.

## How the system fits together

```text
Recorded video
    -> Fine-tuned detector + tracker
    -> Line-crossing counter
    -> PostgreSQL events and processing progress
    -> FastAPI statistics endpoints
    -> React dashboard

Processing also produces a finalized annotated video for download.
```

| Component | Planned technology and responsibility |
| --- | --- |
| Video processor | Python and OpenCV for video decoding, media timestamps, counting, and output annotation. |
| ML adapter | Puntawit's selected detector/tracker, with a pretrained baseline and fine-tuned four-class model. |
| Data validation | Pydantic contracts for model outputs, events, and API responses. |
| Storage | PostgreSQL and Psycopg for run metadata, crossing events, and consistent SQL summaries. |
| API | FastAPI for run selection, statistics, progress, and finalized artifacts. |
| Dashboard | React, TypeScript, and Vite for the local interface and periodic refreshes. |
| Local services | Docker Compose for the database, API, and frontend; the GPU processor initially runs in a native Python environment. |

## ML and pipeline evidence

The finished project will include evidence for both parts of the system:

- A dataset audit confirming support for all four vehicle classes, including
  explicit pickup labels.
- A documented pretrained baseline, reproducible fine-tuning experiments, and
  evaluation on held-out footage kept separate from training sequences.
- Per-class model metrics and a comparison between manual and predicted crossing
  counts, with limitations and failure examples.
- Checks for duplicate event writes, failed processing runs, consistent database
  summaries, independent reruns, and dashboard refresh failures.
- Reproducible local setup, model/configuration identity, and clear individual
  contributions.

No accuracy or processing-speed claim has been established yet. Dataset, model,
tracker, and training settings remain to be selected. The four-second dashboard
refresh target is separate from the model's video-processing throughput.

## Scope and future direction

| Phase-one target | Future direction |
| --- | --- |
| Preselected recorded footage | Live CCTV streams and reconnection handling |
| Car, truck, motorcycle, and pickup classification | Experimental fuel-type classification with an explicit uncertain/unknown outcome |
| Local dashboard and services | Broader deployment if the project later needs it |
| Statistics during processing; annotated output afterward | Additional video delivery capabilities if needed |

Fuel-type classification and live CCTV input are future goals. Browser uploads,
public hosting, speed estimation, and congestion scoring are outside the current
phase-one scope.

## Team and development plans

| Developer | Primary ownership | Detailed checklist |
| --- | --- | --- |
| **NoneTP** | Video/data pipeline, counting, PostgreSQL, FastAPI, React, reliability, and local setup | [Pipeline workstream](docs/superpowers/plans/2026-10-28-pipeline-workstream.md) |
| **Puntawit** | Dataset preparation, baseline, fine-tuning, tracking, model adapter, and ML evaluation | [ML workstream](docs/superpowers/plans/2026-10-28-ml-workstream.md) |
| **Both** | Class policy, shared contracts, early integration, count validation, and final demo | [Central team plan](docs/superpowers/plans/2026-10-28-traffic-analysis.md) |

The [phase-one design](docs/superpowers/specs/2026-10-28-traffic-analysis-design.md)
contains the agreed product behavior and technical constraints. Installation and
run instructions will be added as the implementation becomes runnable.
