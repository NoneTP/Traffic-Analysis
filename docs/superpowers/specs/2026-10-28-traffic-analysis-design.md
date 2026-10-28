# Traffic Analysis: Phase-One Design

Date: 2026-10-28
Status: Approved by the user on 2026-10-28 ("LGTM"). Updated the same day: Puntawit wants to fine-tune a pretrained model, and the shared plan identifies both developers as NoneTP and Puntawit. Specific model, data, and training settings remain deferred.

## Intent and context

Build a local traffic-analysis application as a shared internship portfolio project.
NoneTP (Taranon) is targeting pipeline engineering at Oxygen AI; Puntawit is
targeting an ML role. The available internship brochure does not establish a
specific employer technology stack or assessment rubric.

The project should demonstrate individual engineering decisions, measurable
results, and an integrated working system. NoneTP's stated availability is
6-7 hours per week over roughly 2-3 weeks before finals, interpreted as about
12-21 hours of his time. Puntawit's availability and hardware are unknown.
NoneTP has an RTX 3060 laptop; GPU memory and actual throughput are unverified.
At initial discovery, the repository contained only a title in README.md. The
[central team plan](../plans/2026-10-28-traffic-analysis.md) now tracks both
developers' tasks and joint checkpoints, with separate detailed workstream plans.

## Agreed scope

- Process preselected recorded traffic videos on a local machine.
- Detect and classify car, truck, motorcycle, and pickup truck as distinct classes.
- Count each tracked vehicle once when it crosses a configured line.
- Store results in a database.
- Show totals by vehicle type and traffic volume over video time in a dashboard.
- Refresh statistics and processing progress approximately every 3-5 seconds.
- Make an annotated video available after processing completes.
- Use FastAPI and React for the application.

The first phase includes the full system, not just a model demonstration.

## Future goals and exclusions from phase one

Future goals are live CCTV input and experimental fuel-type classification.
Phase one does not include public deployment, browser uploads, accounts,
multi-camera identity matching, live annotated-video streaming, speed estimation,
or congestion scoring. These exclusions are proposed scope boundaries based on
the agreed local, preselected-video workflow.

Fuel type is separate from vehicle body type. EV/petrol classification is a
stretch goal requiring suitable labeled data and an explicit treatment of
hybrids, diesel, and uncertain predictions; it is not an acceptance criterion.

## Constraints and anti-goals

- Keep the project achievable alongside three university projects.
- Fine-tune a pretrained vehicle model; the user confirmed this is the ML
  developer's intended contribution. Keep detector, tracker, model framework,
  dataset, and detailed training configuration open for him.
- Do not report sample or fixture results as model performance.
- Do not claim that UI refresh frequency proves real-time inference capability.
- Do not combine counts from repeated analyses of the same video into one total.
- Keep processing speed independent of the definition of traffic volume.
- Keep video assets and credentials out of Git; document data provenance and reuse
  conditions once footage is selected.

## Approaches considered

### Python processor, PostgreSQL, and Streamlit dashboard

Recommended initially for a small Python-only dashboard and lower UI integration
effort. The user preferred FastAPI and React for longer-term portfolio value.

### Python processor, PostgreSQL, FastAPI, and React: selected

Separates processing, storage, API, and presentation. Adds API and frontend work,
but matches the user's choice and makes integration responsibilities explicit.
Use an independently launched processor and simple HTTP polling for this phase.

For dashboard updates, periodic polling was proposed and accepted. Persistent
push connections and live annotated-video streaming are unnecessary for the
accepted refresh behavior.

## Proposed stack

The following completes the selected FastAPI/React approach. Supporting tools are
assistant recommendations to be checked by the user, not independently confirmed
preferences.

| Responsibility | Proposed choice |
| --- | --- |
| Video ingestion and processing | Python and OpenCV |
| Model execution and tracking | Puntawit's choice behind a Python adapter |
| Boundary validation | Pydantic |
| Durable storage and aggregation | PostgreSQL and Psycopg |
| Read API | FastAPI |
| Dashboard | React and TypeScript |
| Local service setup | Docker Compose for PostgreSQL, API, and frontend |
| GPU processor | Native Python environment on the laptop initially |
| Verification | Focused Python integration tests and frontend checks |

Keep SQL schema changes versioned in the repository. The builder can choose the
migration mechanism, package managers, chart library, frontend build tooling, and
test framework. Pin compatible dependencies during implementation rather than
selecting exact versions in this design.

## Responsibilities and integration boundary

Proposed ownership:

- NoneTP: source/run metadata, validated data boundaries, persistence, API,
  dashboard integration, logs, repeatability, and local setup.
- Puntawit: dataset investigation, class definitions, detector/tracker
  selection, pretrained baseline, fine-tuning, evaluation, and model adapter.
- Joint: crossing-count behavior, representative demo footage, end-to-end
  validation, and documentation of individual contributions.

Puntawit owns the detector/tracker adapter; NoneTP owns the video reader, media
timestamps, and line-crossing counter, with Puntawit reviewing counting behavior.
Both agree class policy and interface changes together. Track identities must
remain stable within a run. The pipeline may be developed using clearly labeled fixture
events while the actual ML adapter is undecided.

The proposed model adapter accepts video frames and returns detections with a
bounding box, normalized vehicle class, confidence, and a stable track identifier
after tracking. An integration smoke check with the chosen model is required
before calling phase one complete.

## Processing behavior

1. A command-line entry point selects a video from a local source manifest and
   supplies the counting-line configuration and model adapter.
2. Validate that the file can be read, its timing information is usable, and the
   line is valid for the frame dimensions. Reject unsupported inputs clearly.
3. Create a processing run with its input identity and configuration provenance.
4. Read frames, run detection/tracking, and identify qualifying line crossings.
5. Write crossing events and progress in bounded transactional batches.
6. Write an annotated output video locally and finalize the run.

Command-line initiation is an assistant proposal to keep the first UI focused on
viewing statistics. A browser start button is not required by the agreed scope.
One video is processed at a time in the first version. No message broker or
distributed scheduler is needed for the proposed workflow.

The source-reading code should have a clear boundary so a live-stream reader can
be added later. Live input will still require new work for reconnects, dropped
frames, clock handling, and longer-running storage; it is not promised as a URL
replacement alone.

## Counting and time semantics

- Configure one line per camera view/video source for phase one. Coordinates and
  crossing rules are versioned as part of the run configuration.
- Proposed crossing reference: bounding-box bottom center. Require an observed
  transition across the line, with a small tolerance to reduce jitter. Exact
  thresholds are builder/ML integration decisions to evaluate on sample clips.
- Count at most once per track and line within a run. A vehicle that stops on the
  line must not accumulate repeated counts.
- Track fragmentation and ID switches can still produce count errors. Database
  uniqueness protects repeated writes, not errors in vehicle identity tracking.
- Derive event time from the video timeline. Store elapsed video milliseconds,
  frame index, and available source timing metadata. Do not present elapsed video
  time as an actual date/time when the recording start time is unknown.
- A run records the furthest video time actually processed.
- Display volume as crossing counts in one-minute video-time buckets. Mark the
  currently incomplete bucket and a short final bucket as partial. Do not infer
  zero traffic for unprocessed video or extrapolate partial buckets to a full
  minute by default.
- The chosen run determines all dashboard totals. Counts from other runs are
  never implicitly added to it.

## Proposed data model

### Sources

Store a source ID, display name, local video reference, content hash, dimensions,
duration, frame/timing metadata, and provenance/license reference when known.

### Processing runs

Store run ID, source ID, status, start/finish times, processed video position,
model artifact identifier/hash, relevant software version, complete inference and
counting configuration, configuration hash, error summary, and output artifact
reference. Fixture-backed runs are explicitly identified.

Statuses are running, completed, and failed. Running means the processor has
started, not that all video time is already represented in the database.
Store a progress/heartbeat timestamp. A stale heartbeat is shown as a warning;
it must not be represented as successful completion.

### Crossing events

Store event ID, run ID, track ID, line ID, vehicle class, event video time, frame
index, and confidence where available. Enforce valid classes and nonnegative
time, plus uniqueness on run ID, track ID, and line ID.

Keep source and annotated videos in local files; store their references in the
database. Per-frame boxes need not be persisted in PostgreSQL for phase one.
Compute summaries from crossing events using SQL; a separate aggregation system
is not required at this scale.

## Reliability and reruns

- Commit crossing events and corresponding progress together so readers do not
  see progress beyond the events that were durably saved.
- Make retried event writes safe using the database unique constraint and
  conflict handling. An invalid batch rolls back and produces a useful error.
- Explicitly reprocessing a video creates a new run with separate results. The
  dashboard selects one run and labels its configuration/provenance.
- A failed run retains its committed partial results and error information.
- Proposed recovery is a full reprocess into a new run. Exact frame-level resume
  is outside phase one because restoring tracker state is additional work.
- Structured logs include run ID, source ID, processing position, and error
  context. Keep failure details available locally.
- A run becomes completed only after event writes finish and its annotated
  artifact has been finalized and checked as available.

## API and dashboard behavior

FastAPI serves source/run listings, run status/progress, per-class totals,
time-bucketed volume, and the selected run's finalized annotated artifact. Exact
endpoint naming is left to the builder. Statistics responses identify the run
and include the processed-through position so charts and status describe the
same progress snapshot.

React offers source/run selection and displays the four class totals, total
crossings, volume over elapsed video time, and processing status/progress. It
refreshes active-run statistics every 3-5 seconds with at most one refresh request
in flight. Stop active-run polling on completion/failure or navigation away.
Switching runs must not let an old response overwrite the new selection.

While a run is active, label statistics as partial. On an API failure, retain the
last successful data, mark it stale, and allow subsequent refresh attempts. A
failed processing run and an unreachable API are different UI states.

After successful completion, provide the annotated video for browser playback or
download. The builder must check browser-compatible encoding for playback;
OpenCV output is not assumed to be browser-compatible without verification.
Serve artifacts by run identity, not by accepting arbitrary filesystem paths.

Bind local services to the local machine by default. Document configuration and
service startup. GPU processing is launched separately from Compose initially.

## Evidence required for completion

- A real recorded-video demo traverses model/tracking, crossing logic, storage,
  API, and dashboard, including all four supported class definitions.
- The dashboard updates during a run, identifies partial results, and exposes a
  finalized annotated video after completion.
- Known fixture crossings demonstrate correct aggregation, time buckets, and
  safe repeated writes independently of ML accuracy.
- A failed input/run is visible and diagnosable; an intentional rerun has
  separate totals. Unexpected processor interruption is observable as stale.
- Database-backed checks cover invalid records, transactional behavior, and
  duplicate-event handling. Check the API response contracts and the important
  dashboard loading, partial, stale, completed, and failed states.
- Document setup, architecture decisions, data provenance, individual ownership,
  observed throughput, and measured limitations.
- Record a pretrained baseline and fine-tuning experiments, then evaluate the
  selected tuned model on held-out footage. Keep footage/vehicle sequences
  separate between training and evaluation so adjacent frames do not leak
  across splits. Report baseline class limitations explicitly, including whether
  it can represent pickup trucks; do not invent comparable scores for unsupported
  labels. Experiment configuration and model artifact identities must be saved.
- Compare counts against manual counts on a small representative held-out clip.
  Report errors by class. Label fixture checks and model evaluation separately.

Model quality targets are intentionally not invented here. Puntawit must
help select the dataset, evaluation split, metrics, and acceptable quality before
the full project's model acceptance criteria can be finalized. A fixture-only
pipeline does not establish model readiness.

## Decisions led by Puntawit

- Dataset and video selection, including whether it has explicit pickup labels
  and sufficient examples of all four classes.
- Treatment of vans, SUVs, buses, ambiguous/unsupported vehicles, and class
  mapping. Unsupported vehicles must not be silently forced into truck/car.
- Detector/framework, tracker, fine-tuning configuration, pretrained baseline
  procedure, and model artifact interface details. Fine-tuning is confirmed.
- Class confidence policy, evaluation split and metrics, and measurable quality
  targets. Validate performance across selected views/conditions.
- Actual GPU capacity and processing throughput on available hardware.

These decisions remain visible dependencies. Pipeline/API/dashboard development
can proceed against the proposed boundary after design approval; integration and
full phase-one completion depend on resolving the ML decisions.

## Approval boundary

This document records agreed requirements and explicitly marked assistant
proposals, accepted by the user on 2026-10-28. Proceed to implementation planning.
Approval does not settle the intentionally deferred ML decisions or authorize
public deployment.
