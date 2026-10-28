# Traffic Analysis Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the approved local video-to-statistics pipeline, API, and dashboard, then integrate the teammate's selected ML adapter when ready.

**Architecture:** A native Python processor reads a preselected video and incrementally stores crossing events in PostgreSQL. FastAPI serves consistent run statistics to a React dashboard polling every four seconds. The teammate will fine-tune a pretrained model; his specific model, data, and training configuration remain open, while labeled fixtures allow independent pipeline development.

**Tech Stack:** Python, OpenCV, Pydantic, PostgreSQL, Psycopg, FastAPI, React, TypeScript, Vite, Docker Compose; pytest and Vitest/React Testing Library for meaningful behavior checks.

**Spec:** [Approved design](../specs/2026-10-28-traffic-analysis-design.md). Read the spec together with this plan.

**Status:** Written for execution review; no product implementation has started. Fine-tuning requirement incorporated on 2026-10-28.

## Global Constraints

- Process preselected recorded traffic videos on a local machine.
- Detect and classify car, truck, motorcycle, and pickup truck as distinct classes.
- Count each tracked vehicle once when it crosses a configured line.
- Refresh statistics and processing progress approximately every 3-5 seconds.
- Make an annotated video available after processing completes.
- Fine-tune a pretrained vehicle model; the user confirmed this is the ML developer's intended contribution. Keep detector, tracker, model framework, dataset, and detailed training configuration open for him.
- Do not report sample or fixture results as model performance.
- Keep video assets and credentials out of Git; document data provenance and reuse conditions once footage is selected.
- The chosen run determines all dashboard totals. Counts from other runs are never implicitly added to it.
- GPU processing is launched separately from Compose initially.

## Review Focus

1. Lost tracker identities and unsupported vehicle labels: never silently coerce unsupported vehicles into a supported class; report identity fragmentation as an ML limitation. Pin mapping behavior in Task 1 and crossing behavior in Task 3.
2. Concurrent database writes during a statistics request: totals, buckets, and processed position must describe the same committed snapshot. Pin this with two connections in Task 2.
3. No traffic, short clips, and incomplete minutes: represent processed empty intervals as zero and partial intervals as partial; never plot future video as observed. Pin in Tasks 2 and 6.
4. Run changes and temporary API failures: late responses cannot replace another run's data, and last successful results remain visible with a stale indicator. Pin with deferred requests and fake timers in Task 6.
5. Abrupt processor exit and video-output failure: stale processing stays incomplete, and a missing/unfinalized artifact cannot appear as a completed run. Pin in Tasks 4 and 5.

## Scope, ownership, and effort

This is one integrated delivery, not independent subsystem projects: each task exposes the interface consumed by the next. Tasks 1-7 deliver a verifiable fixture-backed pipeline; Task 8 is required for the full real-video product. Fine-tuning was confirmed by the user during planning. Framework-specific training commands must be added after the teammate chooses the model/data; do not invent those choices on his behalf.

Budget targets for Taranon are approximately 1, 3, 2, 3, 2, 3, 2, and 2 hours for Tasks 1-8, respectively: 18 hours plus up to 3 hours of contingency. These are planning targets, not measured estimates or a promise. The teammate's ML work is additional and unestimated. If available time is closer to 12 hours, report the achieved milestone and remaining work rather than claiming the entire first phase is done. Keep the accepted React interface small.

Taranon owns Tasks 1, 2, 4-7; Task 3 and Task 8 are joint integration work. The teammate supplies the real adapter, supported/unsupported class mapping, data, model artifact identity, and evaluation decisions for Task 8.

### ML workstream: teammate-owned, alongside pipeline development

- [ ] Establish the dataset's provenance, labels, class counts, and whether pickup trucks are explicitly distinguishable; record the result in `docs/ml-experiments.md`.
- [ ] Select a pretrained detector/framework and tracker, inspect the available GPU memory, and verify one short inference sample before committing to training settings. Record dependency/environment versions.
- [ ] Save train/validation/test video or sequence IDs and hashes in `config/ml/splits.csv` (`split,source_id,content_sha256`). Keep all frames of a source sequence in one split; use unseen views for a stronger generalization check when feasible. Keep test footage untouched during tuning.
- [ ] Evaluate the pretrained baseline and document its supported categories. If its label space lacks pickups, report that limitation separately rather than fabricating a four-class baseline score.
- [ ] Fine-tune using only training data, select configurations using validation data, and save seeds, settings, experiment results, and model artifact hashes. Keep large data/weights out of Git; document retrieval/storage references. The selected framework determines the reproducible training commands added to `docs/ml-experiments.md`.
- [ ] Evaluate the selected tuned model on held-out footage, including per-class detection/classification performance and line-count errors; hand off the versioned `TrackerAdapter` artifact for Task 8. Agree quality targets with the team before final evaluation.

This workstream has no assumed hour estimate because his availability, dataset readiness, and training setup are unknown. Dataset annotation for a distinct pickup class is a material dependency. If the training effort does not fit the available window, report the limitation and agree a scope change rather than quietly substituting a pretrained-only final model.

## Proposed file structure

All paths below are repository-relative. The repository currently has only README.md plus the approved design; do not replace existing user changes if that changes before execution.

```text
pyproject.toml                         Python package, dependencies, CLI, pytest config
requirements.lock                     Resolved Python dependencies for the selected environment
.gitignore / .env.example              Ignore assets/secrets; document local configuration
config/sources.example.json            Manifest format, provenance, counting line
src/traffic_analysis/
  contracts.py                        Validated shared types and adapter protocol
  settings.py                         Database URL, artifact root, timeouts
  db.py                               Connections, versioned SQL application, repository
  statistics.py                       Consistent snapshot query and minute buckets
  video.py                            Decode frames, validate timing, write annotations
  counter.py                          Stateful finite-line crossing logic
  processor.py                        Run lifecycle, batching, failure/finalization
  cli.py                              migrate, process, and evaluate commands
  adapters/fixture.py                  Explicitly synthetic tracked detections
  api.py                              FastAPI routes and response/error mapping
migrations/001_initial.sql             Tables, constraints, indexes
tests/conftest.py                      Isolated test database schema and helpers
tests/test_contracts.py                Boundary validation and adapter mapping
tests/test_repository.py               Transactions, uniqueness, reruns
tests/test_statistics.py               Buckets, partial intervals, concurrent reads
tests/test_video.py                    Media timing and invalid input behavior
tests/test_counter.py                  Crossing geometry and tracker behavior
tests/test_processor.py                Fixture run, failures, finalized output
tests/test_api.py                      API snapshots and artifact access
tests/test_evaluation.py               Per-class count-error calculations
tests/fixtures/tracks.json             Four-class synthetic tracks, no video binary
scripts/create_fixture_video.py        Deterministic synthetic video for pipeline checks
frontend/
  package.json / package-lock.json     React/Vite and test dependencies
  vite.config.ts / tsconfig*.json      Build, tests, /api development proxy
  index.html / src/main.tsx            App entry
  src/api.ts                          Typed HTTP client and cancellation
  src/types.ts                        Types matching response models
  src/useRunStats.ts                   Four-second serial polling and cleanup
  src/App.tsx                         Source/run selection and empty/loading states
  src/RunDashboard.tsx                Totals, volume chart/table, status, artifact link
  src/styles.css                      Compact responsive accessible layout
  src/useRunStats.test.tsx             Polling, failures, and request races
  src/RunDashboard.test.tsx            Partial/failed/complete/empty presentation
  Dockerfile                          Local frontend container
Dockerfile.api / compose.yaml          API, frontend, PostgreSQL with durable volume
.github/workflows/checks.yml           CPU-only checks using PostgreSQL service
docs/demo.md                          Reproduction and observed limitations
docs/ml-handoff.md                    Adapter contract and deferred ML decisions
docs/ml-experiments.md                Teammate's data, baseline, and fine-tuning evidence
config/ml/splits.csv                  Reproducible source-sequence split manifest
docs/evaluation.md                    Real-data provenance and measured count errors
```

Pin compatible versions at execution time after checking the available Python/Node runtimes. Preserve a separate ML extra/environment if the teammate's framework imposes version constraints. Use a supported Node version satisfying the installed Vite/Vitest requirements. Do not install or select a detector as part of setup.

## Shared contract decisions

These are implementation choices within the approved design, not additional product features:

- `VehicleClass` values: `car`, `truck`, `motorcycle`, `pickup_truck`; unknown/unsupported adapter labels become `None` explicitly, never `car` by default.
- `RunStatus`: `running`, `completed`, `failed`; `RunKind`: `fixture`, `model`.
- `Point(x: float, y: float)`, `BBox(x1: float, y1: float, x2: float, y2: float)`; finite values, ordered box coordinates. Image-coordinate bounds are checked against each frame.
- `LineConfig(line_id: str, start: Point, end: Point, tolerance_px: float = 3.0)`; endpoints distinct and inside the frame, tolerance nonnegative.
- `TrackedDetection(track_id: str, vehicle_class: VehicleClass | None, confidence: float | None, bbox: BBox)`; nonempty ID and finite confidence in [0, 1] when present.
- `VideoFrame(index: int, video_ms: int, image: numpy.ndarray)`; zero-based frame index and nonnegative media time.
- `CrossingEvent(track_id: str, line_id: str, vehicle_class: VehicleClass, video_ms: int, frame_index: int, confidence: float | None)`.
- `SourceConfig(source_id: str, name: str, path: Path, line: LineConfig, provenance: str, timing_mode: Literal['decoder', 'cfr'])`.
- `SourceMetadata(content_sha256: str, width: int, height: int, fps: float, duration_ms: int, frame_count: int)`; validate positive finite metadata. Decoder time must progress; CFR timing is an explicit manifest declaration, not an automatic guess.
- `ModelIdentity(name: str, artifact_sha256: str, config: dict[str, JSONValue], class_mapping: dict[str, VehicleClass | None])`; `JSONValue` is an alias of `pydantic.JsonValue`.
- `TrackerAdapter` protocol: `identity: ModelIdentity`, `kind: RunKind`, `reset() -> None`, `predict(frame: VideoFrame) -> list[TrackedDetection]`, `close() -> None`.
- `RunRecord`: `run_id: UUID`, `source_id: str`, `kind: RunKind`, `status: RunStatus`, `duration_ms: int`, `processed_through_ms: int`, `started_at: datetime`, `heartbeat_at: datetime`, `finished_at: datetime | None`, `error: str | None`, `artifact_relative_path: str | None` plus model/config/source provenance.
- `VolumeBucket(start_ms: int, end_ms: int, observed_ms: int, count: int, partial: bool)`; include only observed intervals. A running minute ending exactly at the progress boundary is complete; a short terminal minute is partial.
- `RunStats(run_id: UUID, status: RunStatus, kind: RunKind, duration_ms: int, processed_through_ms: int, heartbeat_at: datetime, is_stale: bool, error: str | None, totals: dict[VehicleClass, int], total: int, volume: list[VolumeBucket], artifact_url: str | None)`; all four total keys are present, including zeros.
- UI polling interval `4000` ms; processor commit interval at most `1.0` seconds between completed frame operations, also bounded to `100` processed frames per batch. Stale heartbeat warning threshold `30` seconds, not automatic failure.
- `processed_through_ms` is exclusive observed time coverage, capped by video duration; saved event times must be strictly below it. Do not confuse a frame's presentation timestamp with the end of its observed interval.
- FastAPI path prefix `/api`; timestamps in UTC ISO 8601, media times as integer milliseconds; local paths never appear in frontend responses.

---

### Task 1: Validated contracts and a fixture adapter

**Files:** Create `pyproject.toml`, `.gitignore`, `.env.example`, `config/sources.example.json`, `src/traffic_analysis/contracts.py`, `src/traffic_analysis/settings.py`, `src/traffic_analysis/adapters/fixture.py`, `tests/test_contracts.py`, `tests/fixtures/tracks.json`, `docs/ml-handoff.md`.

**Interfaces:** Consumes no product code. Produces the shared types above, `load_sources(path: Path) -> dict[str, SourceConfig]`, `Settings.from_env() -> Settings`, and `FixtureTracker(fixture_path: Path)` implementing `TrackerAdapter`. Fixture tracks have known frame positions, cover all four classes, and are explicitly synthetic.

- [ ] **Step 1: Write contract tests.** Define `test_rejects_invalid_detection` for NaN coordinates, reversed boxes, confidence 1.1, and empty IDs; `test_rejects_invalid_line` for coincident endpoints; `test_preserves_unsupported_label` asserting `vehicle_class is None`; `test_fixture_is_explicit_and_repeatable` asserting `adapter.kind == 'fixture'` and equal output after `reset()`; `test_load_sources_resolves_manifest_relative_paths` asserting manifest-relative source paths and rejection of duplicate IDs.
- [ ] **Step 2: Install only package/test prerequisites and run `python -m pytest tests/test_contracts.py -q`.** Expect failure because contracts/fixture adapter are not implemented. Use an editable install and a package rooted at `src/`.
- [ ] **Step 3: Implement the defined types, settings, source loader, and fixture adapter.** Use Pydantic validation, reject malformed/unsupported manifest fields clearly, and identify fixture provenance. Add the `traffic` console entry point configuration for later tasks; no real model dependency.
- [ ] **Step 4: Run `python -m pytest tests/test_contracts.py -q`.** Expect all assertions to pass. Document adapter inputs/outputs and deferred model/dataset decisions in `docs/ml-handoff.md`.
- [ ] **Step 5: Commit only this task's files.** Suggested message: `feat: define validated traffic and model adapter contracts`.

### Task 2: Durable events and trustworthy statistics

**Files:** Create `migrations/001_initial.sql`, `src/traffic_analysis/db.py`, `src/traffic_analysis/statistics.py`, `tests/conftest.py`, `tests/test_repository.py`, `tests/test_statistics.py`; modify `pyproject.toml`, `.env.example`.

**Interfaces:** Consumes Task 1 contracts. Produces `apply_migrations(conn: psycopg.Connection) -> None` and `Repository(conn)` methods `register_source(config: SourceConfig, metadata: SourceMetadata) -> None`, `create_run(source_id: str, metadata: SourceMetadata, line: LineConfig, identity: ModelIdentity, kind: RunKind, software_version: str) -> UUID`, `commit_batch(run_id: UUID, events: Sequence[CrossingEvent], processed_through_ms: int) -> None`, `finish_run(run_id: UUID, artifact_relative_path: str) -> None`, `fail_run(run_id: UUID, error: str) -> None`, `get_run(run_id: UUID) -> RunRecord | None`, `list_sources() -> list[SourceSummary]`, `list_runs(source_id: str | None) -> list[RunRecord]`. Define `SourceSummary(source_id: str, name: str)` in contracts. Produces `read_stats(conn: psycopg.Connection, run_id: UUID, now: datetime) -> RunStats | None`.

- [ ] **Step 1: Write real-PostgreSQL repository tests.** `test_duplicate_batch_does_not_double_count` writes one event twice and asserts one stored row; `test_invalid_batch_rolls_back_progress_and_events` inserts a valid then invalid row and asserts no event/progress changes; `test_progress_cannot_regress_or_precede_event` asserts invalid batches fail; `test_rerun_has_independent_totals` asserts the second run starts at zero; `test_finished_run_rejects_more_writes` checks terminal state guards.
- [ ] **Step 2: Write statistic tests.** `test_four_class_keys_and_short_partial_bucket` inserts car@1000, pickup@61000 with observed time 65000 and asserts total=2, all four keys, bucket `(0,60000,1,False)` and `(60000,120000,1,True)`, with `observed_ms=5000` in the latter. `test_zero_traffic_stops_at_observed_time` asserts processed empty minutes show zero and no future buckets. `test_snapshot_does_not_mix_concurrent_commits` commits an extra event/progress on a second connection between reads and asserts the first response uses one snapshot. `test_stale_heartbeat_is_warning_not_completion` asserts a running run remains running after 31 seconds and `is_stale is True`.
- [ ] **Step 3: Start a dedicated test PostgreSQL instance and run `python -m pytest tests/test_repository.py tests/test_statistics.py -q`.** Before Compose exists, use `docker run --name traffic-test-db -e POSTGRES_USER=traffic_test -e POSTGRES_PASSWORD=local-test-only -e POSTGRES_DB=traffic_test -p 127.0.0.1:5433:5432 -d postgres:18`, and set `TEST_DATABASE_URL=postgresql://traffic_test:local-test-only@127.0.0.1:5433/traffic_test` in the test environment. Wait for `pg_isready` before tests. An existing test instance may be reused. Expect missing implementation failures. Create isolated test schemas and never truncate the development database.
- [ ] **Step 4: Implement schema, repository, and SQL summaries.** Tables are `sources`, `runs`, `crossing_events`, and a migration ledger. Preserve the run's input path/hash/timing snapshot even if the source manifest later changes. Use parameterized SQL, checks/FKs, `UNIQUE(run_id, track_id, line_id)`, and `(run_id, video_ms)` index. Use `ON CONFLICT DO NOTHING` only on the event identity constraint. Validate event/progress consistency inside the same transaction; update heartbeat even for zero-event batches. Use an explicit read-only REPEATABLE READ transaction for multi-query statistics; generate buckets only up to observed coverage. A failed run keeps its committed events.
- [ ] **Step 5: Run the Task 2 test command against PostgreSQL.** Expect all assertions to pass, including rollback and two-connection consistency. Add schema constraints for valid class/status/kind values and terminal timestamps.
- [ ] **Step 6: Commit this task's files.** Suggested message: `feat: persist isolated runs and consistent traffic statistics`.

### Task 3: Video timing and one-crossing-per-track logic

**Files:** Create `src/traffic_analysis/video.py`, `src/traffic_analysis/counter.py`, `tests/test_video.py`, `tests/test_counter.py`, `scripts/create_fixture_video.py`; modify `pyproject.toml`.

**Interfaces:** Consumes Task 1 contracts. Produces `probe_video(source: SourceConfig) -> SourceMetadata`, `iter_frames(source: SourceConfig, metadata: SourceMetadata) -> Iterator[VideoFrame]`, `LineCounter(line: LineConfig)`, and `LineCounter.update(frame: VideoFrame, detections: Sequence[TrackedDetection]) -> list[CrossingEvent]`. A later processor uses `frame_end_ms(frame: VideoFrame, metadata: SourceMetadata, next_video_ms: int | None) -> int` to advance exclusive coverage from observed frame intervals.

- [ ] **Step 1: Write crossing tests.** `test_crossing_counts_once_across_stop_and_return` feeds stable sides, line jitter, a stop, and a return; assert one event for the track. `test_no_crossing_at_infinite_extension` asserts zero outside the finite segment. `test_first_seen_on_line_is_not_counted` asserts no event until two stable sides exist. `test_unknown_class_is_not_coerced` asserts zero for `None`. `test_track_ids_are_run_local` creates a new counter and asserts the same ID can count in a new run. Parameterize all four supported classes.
- [ ] **Step 2: Write timing tests.** `test_media_time_is_independent_of_wall_clock` asserts identical event times despite injected processing delays; `test_decoder_time_regression_is_rejected` expects a clear error; `test_cfr_mode_uses_explicit_fps` asserts frame 30 at 30 FPS has `video_ms == 1000`; `test_frame_coverage_includes_last_interval` verifies exclusive progress covers the last frame. Reject missing/empty files, zero/nonfinite FPS, malformed dimensions, out-of-bounds line/detections, and decode ending materially before expected frame count.
- [ ] **Step 3: Run `python -m pytest tests/test_video.py tests/test_counter.py -q`.** Expect failures for missing reader/counter logic.
- [ ] **Step 4: Implement the reader and finite-line counter.** Hash source bytes; use decoder timestamps when declared, explicit frame-index/FPS in declared CFR mode. Do not silently use wall-clock time or assume CFR when decoder timing fails. Compute signed distance of the box bottom center to the line; points within `tolerance_px` do not replace the previous stable side. Emit only when stable opposite sides also intersect the finite segment. Keep a counted track-ID set for the run. The event class/confidence come from the crossing detection. Generate a tiny synthetic CFR clip with corresponding fixture tracks for pipeline verification.
- [ ] **Step 5: Run the Task 3 tests.** Expect all assertions to pass; confirm OpenCV can reopen the generated clip and the expected frame count/dimensions. Counting cannot repair tracker ID switches; document that limitation.
- [ ] **Step 6: Commit this task's files.** Suggested message: `feat: count finite-line crossings on the video timeline`.

### Task 4: Incremental processing with reliable finalization

**Files:** Create `src/traffic_analysis/processor.py`, `src/traffic_analysis/cli.py`, `tests/test_processor.py`; extend `video.py`, `settings.py`, `.env.example`, `README.md`, `pyproject.toml`.

**Interfaces:** Consumes Tasks 1-3. Produces `process_video(source: SourceConfig, adapter: TrackerAdapter, repository: Repository, artifact_root: Path, software_version: str) -> UUID`; `AnnotatedWriter(output_path: Path, metadata: SourceMetadata)` with `write(frame: VideoFrame, detections: Sequence[TrackedDetection], line: LineConfig) -> None` and `finalize() -> Path`. CLI: `traffic migrate`; `traffic process --manifest PATH --source SOURCE_ID --adapter fixture --tracks PATH`. Settings include `DATABASE_URL` and `ARTIFACT_ROOT`.

- [ ] **Step 1: Write processor integration tests.** `test_fixture_video_reaches_completed_with_expected_counts` asserts an explicit fixture run, four expected class counts, saved progress during processing, and an artifact that OpenCV can reopen. `test_adapter_failure_preserves_partial_run` injects failure after a committed batch and asserts failed status plus retained counts. `test_failed_artifact_never_completes_run` injects writer finalization failure and asserts no completed/artifact-ready state. `test_zero_events_still_advances_heartbeat` checks progress during quiet footage. `test_processor_kill_leaves_observable_stale_run` simulates no final status update and checks stale running status at 31 seconds. `test_cleanup_on_failure` asserts adapter and video/writer resources close.
- [ ] **Step 2: Run `python -m pytest tests/test_processor.py -q` against the test database.** Expect missing lifecycle/CLI/writer failures.
- [ ] **Step 3: Implement the lifecycle and CLI.** Validate input before a successful run is possible; report input-validation errors with nonzero CLI exit. Catch failures after run creation and record failed state while retaining committed events. Reset the adapter and counter for each new run. Commit after completed frame operations at the earlier of 1 second or 100 frames; update progress even without crossings. No exact resume. Write to a temporary run-scoped file under `ARTIFACT_ROOT`; close/reopen to verify and atomically rename before recording completed status. Return CLI exit 0 only for completion; log run ID and errors for failures. Include class boxes, IDs, and counting line in annotated output.
- [ ] **Step 4: Run the Task 4 tests and a fixture CLI demo.** Expected: counts appear before completion, rerunning creates a different run UUID, and a completed run has a readable video. Preserve a supported output codec with video download as the baseline; test browser playback separately and document when remux/transcoding is needed. No raw or annotated video goes into Git.
- [ ] **Step 5: Commit this task's files.** Suggested message: `feat: process video with incremental writes and finalized artifacts`.

### Task 5: Read API for run selection and consistent updates

**Files:** Create `src/traffic_analysis/api.py`, `tests/test_api.py`; extend `contracts.py`, `settings.py`, `pyproject.toml`.

**Interfaces:** Consumes `Repository` and `read_stats`. Produces `create_app(settings: Settings | None = None) -> FastAPI`, resolving environment settings when omitted; `GET /api/health` returns `{'status':'ok'}` when DB-ready (503 otherwise); `GET /api/sources` returns `SourceSummary[]`; `GET /api/runs?source_id=...` returns `RunSummary[]`; `GET /api/runs/{run_id}/stats` returns `RunStats`; `GET /api/runs/{run_id}/artifact` returns a finalized artifact download. Define `RunSummary` as `run_id`, `source_id`, `kind`, `status`, `started_at`, `finished_at` only, and use it instead of exposing storage records. All unavailable runs are 404; malformed UUIDs are 422; artifacts for incomplete/failed runs are 409, missing files are 404.

- [ ] **Step 1: Write API tests.** `test_stats_matches_database_snapshot` asserts response totals/buckets/progress and UTC timestamp format; `test_run_selection_does_not_leak_other_runs` checks source filtering; `test_artifact_requires_completion_and_existing_file` covers 409/404/success; `test_artifact_is_confined_to_root` asserts a stored path escaping the artifact root cannot be served; `test_public_payload_contains_no_local_paths` checks the returned field set; `test_database_outage_returns_503` checks useful errors rather than a fake empty result. Confirm fixture/model kind and stale/error information are present.
- [ ] **Step 2: Run `python -m pytest tests/test_api.py -q`.** Expect missing app/route failures.
- [ ] **Step 3: Implement routes with explicit response models.** Share the database-query layer with the processor without starting video work inside request handlers. Offload blocking SQL through synchronous handlers or an appropriate pool; close connections. Run-stat responses derive artifact URL only for completed available artifacts. Use Starlette/FastAPI file serving for downloads, validated canonical paths, and public IDs. Serve `/api` through the frontend proxy; do not add a permissive CORS wildcard by default.
- [ ] **Step 4: Run Task 5 tests and `python -m pytest tests -q`.** Expect all available CPU/DB tests to pass. Start `python -m uvicorn traffic_analysis.api:create_app --factory --host 127.0.0.1 --port 8000` locally and inspect OpenAPI for the concrete response schema.
- [ ] **Step 5: Commit this task's files.** Suggested message: `feat: expose run statistics and completed artifacts through FastAPI`.

### Task 6: Small React dashboard with resilient polling

**Files:** Create the listed frontend package/config/entry/type/client/hook/dashboard/style/test files; extend `.gitignore`.

**Interfaces:** Consumes Task 5 HTTP responses. Produces `fetchSources(signal?: AbortSignal): Promise<SourceSummary[]>`, `fetchRuns(sourceId: string, signal?: AbortSignal): Promise<RunSummary[]>`, `fetchRunStats(runId: string, signal?: AbortSignal): Promise<RunStats>` in `api.ts`; `useRunStats(runId: string | null): { data: RunStats | null; loading: boolean; error: string | null; stale: boolean }`; `RunDashboard({stats, connectionStale}: {stats: RunStats; connectionStale: boolean})`.

- [ ] **Step 1: Create a minimal React/TypeScript Vite scaffold and write behavior tests before the hook/dashboard logic.** `test_polls_serially_every_four_seconds` uses fake timers to assert no overlapping requests and no polling after completed/failed responses; `test_run_change_cannot_be_overwritten_by_old_response` uses deferred promises; `test_failed_refresh_retains_data_and_recovers` asserts previous totals remain visible, stale state appears, and a later success clears it; `test_unmount_cancels_polling` asserts timer/request cleanup. Avoid testing the implementation's exact React render count.
- [ ] **Step 2: Add dashboard tests.** `test_partial_and_complete_volume_labels` asserts partial buckets are identified; `test_zero_traffic_and_no_runs_are_distinct` checks empty state versus zero counts; `test_processing_failure_is_distinct_from_connection_failure` checks useful state labels; `test_fixture_banner_and_artifact_availability` asserts synthetic provenance is visible and the download link appears only when supplied. Include labeled selectors and accessible data tables as assertions.
- [ ] **Step 3: Run `npm --prefix frontend test -- --run`.** Expect failing behavior tests. Configure Vitest's DOM test environment and React Testing Library; no GPU dependencies.
- [ ] **Step 4: Implement the client, serial polling, selection, and dashboard.** Fetch immediately on selection and schedule the next request 4000 ms after a response/error. Use cancellation and a selection token to reject stale responses. Keep the last successful result on connection failure; stop after terminal processing status. Show source/run selectors, fixture banner, total/four class counts, elapsed-video minute volume, progress/status, and artifact download. Use a small native SVG or CSS volume chart paired with a table instead of adding a chart framework. Keep partial bars visually distinct and label units. Do not display a fabricated recording date.
- [ ] **Step 5: Run `npm --prefix frontend test -- --run` and `npm --prefix frontend run build`.** Expect tests to pass and TypeScript/Vite build to succeed. Inspect the local browser with an active fixture run, failure/stale states, narrow layout, keyboard navigation, and artifact download. Mark any browser-playback codec limitation explicitly; downloading remains supported.
- [ ] **Step 6: Commit this task's files and lockfile.** Suggested message: `feat: show traffic statistics with resilient four-second updates`.

### Task 7: Reproducible local setup and independent pipeline evidence

**Files:** Create `compose.yaml`, `Dockerfile.api`, `frontend/Dockerfile`, `.github/workflows/checks.yml`, `requirements.lock`, `docs/demo.md`; modify `.env.example`, `README.md`, frontend proxy config as needed.

**Interfaces:** Consumes Tasks 1-6. Produces a local three-service stack, native-worker instructions, and CPU-only automated checks. Local ports: web `127.0.0.1:5173`, API `127.0.0.1:8000`, PostgreSQL `127.0.0.1:5432`; make host ports configurable for collisions. The worker uses the host DB URL; containers use service-name URLs. Mount the same artifact directory into API read-only and retain PostgreSQL data in a named volume.

- [ ] **Step 1: Define a clean-clone acceptance exercise in `docs/demo.md`.** It creates local config, starts services, applies migrations once, generates fixture video, runs fixture processing, observes partial-to-completed UI state, downloads the annotated file, reruns, and verifies separate totals. Explicitly label this as pipeline evidence, not ML evaluation. Include an invalid-input example and an API interruption/recovery example.
- [ ] **Step 2: Run the documented commands against the unfinished packaging.** Expect missing Compose/build/config failures; record them rather than treating the scaffold as verified.
- [ ] **Step 3: Implement packaging and CI.** Supply health checks and startup ordering; API readiness depends on migrated schema. Run `traffic migrate` explicitly before serving the API (do not race schema setup in every request). Frontend proxies `/api` to the API container and offers the local Vite development server; this is local tooling, not public hosting. Pin Python dependency resolutions and npm lockfile. CI uses a disposable PostgreSQL service, backend tests, frontend behavior tests, and frontend build; it never downloads a model or evaluates synthetic accuracy as real accuracy.
- [ ] **Step 4: Run `docker compose config --quiet`, build/start the stack, and execute the documented fixture workflow.** Run `python -m pytest tests -q`, `npm --prefix frontend test -- --run`, and `npm --prefix frontend run build`. Expect passing checks, persistent results across ordinary service restart, correct host/container URLs, and successful artifact download. Do not run destructive volume cleanup on a database containing user results.
- [ ] **Step 5: Record observed setup commands, runtime versions, processing throughput, limitations, and contribution ownership.** The dashboard may refresh every four seconds even when video inference is slower than playback. Update docs with that distinction.
- [ ] **Step 6: Commit this task's files.** Suggested message: `chore: package local services and verify fixture pipeline in CI`.

### Task 8: Joint real-model integration and portfolio acceptance

**Files:** Extend `docs/ml-handoff.md`, `docs/demo.md`, `README.md`; create `docs/evaluation.md`, `tests/test_evaluation.py`, `src/traffic_analysis/evaluation.py`; extend `cli.py` for `traffic evaluate`. The teammate creates `src/traffic_analysis/adapters/model.py` only after choosing the model/framework and records any optional dependencies separately.

**Interfaces:** Consumes the stable `TrackerAdapter`, selected labeled footage, model identity/class mapping, and manual count CSV with `source_id,bucket_start_ms,vehicle_class,count`. Produces `evaluate_counts(predicted: Sequence[CountRow], reference: Sequence[CountRow]) -> EvaluationReport`, where `CountRow(source_id: str, bucket_start_ms: int, vehicle_class: VehicleClass, count: int)` is nonnegative and unique per key; `EvaluationReport` includes per-class predicted/reference totals, absolute count error, and mean absolute error across supplied buckets. CLI: `traffic evaluate --run RUN_UUID --reference PATH`. It rejects mismatched source IDs, missing/duplicate reference keys, or reference times beyond the completed run. Adaptation is selected via `--adapter model`; no model mode exists until implemented.

- [ ] **Step 1: Obtain the teammate's decisions and fine-tuning outputs before real-model implementation.** Confirm footage access/reuse, explicit pickup labels, unsupported-class mapping, model/tracker/framework, tuned artifact identity, source-sequence split manifest, pretrained baseline and fine-tuning evidence, held-out evaluation footage, and quality targets. These are required external inputs for Task 8, not a reason to block Tasks 1-7 or invent framework-specific training work now. Agree ownership of the real adapter and verify a short frame-to-track smoke check together.
- [ ] **Step 2: Write evaluation tests before the evaluation implementation.** `test_per_class_count_error` compares car predicted=12/reference=10 and motorcycle predicted=3/reference=4; assert absolute errors 2 and 1. `test_zero_reference_does_not_divide_by_zero` asserts absolute error is defined without a percentage denominator. `test_reference_alignment_is_required` rejects duplicate/missing keys and wrong sources. `test_partial_run_cannot_be_used_as_final_evaluation` rejects incomplete run data. Explicit reference zeroes are required; missing labels are not assumed zero.
- [ ] **Step 3: Run `python -m pytest tests/test_evaluation.py -q`.** Expect the missing evaluator to fail, then implement only the report/CLI interfaces specified above. Run again; expect all assertions to pass. Detection/classification metrics and training experiments remain the teammate's deliverable, recorded alongside count evaluation once agreed.
- [ ] **Step 4: Integrate the real adapter using the same contracts.** Require `kind='model'`, actual model artifact hash/config, stable run-local track IDs, supported-class mapping, explicit reset/cleanup, and no fallback to fixture detections. Run a real clip through CLI, database, API, and dashboard; verify the annotated artifact and count totals manually. If the chosen dataset cannot support all four categories, report the shortfall and revisit scope with the user rather than silently dropping a class.
- [ ] **Step 5: Record actual evidence in `docs/evaluation.md`.** Include source provenance, held-out source-sequence split, manual per-class counts, observed errors, unsupported vehicles, timing/throughput/hardware, ID-switch errors, and what each developer implemented. Link the baseline and fine-tuning experiment records and identify any incompatible baseline classes. Keep baseline and tuned processing runs separate. Do not invent results or pass/fail thresholds. Full completion requires the tuned model and agreed quality criteria, or an explicit scope revision.
- [ ] **Step 6: Run all applicable checks from Task 7 plus the real-video demo and evaluation command.** Expect truthful metrics, correct four-class storage/UI, one-run totals, incremental updates, and downloadable annotated output. Run a final diff review and commit the adapter/evaluation/docs changes separately from any large data/model files. Suggested message: `feat: integrate real vehicle model and document measured traffic results`.

## Completion and execution handoff

Completion is the real-video system in Task 8 meeting the agreed model-quality criteria. Tasks 1-7 alone establish a working pipeline demonstrator. No public deployment or fuel-type/live-CCTV work is implied.

This plan has been self-reviewed for spec coverage, matching names/types across tasks, the five review-focus conditions, and proportion. Its commands are future verification instructions; they have not been executed against an implementation because product code has not been written.

The next step is user review of this plan and selection of execution style. For the tightly coupled interfaces and limited time budget, native execution is recommended if the user wants agent implementation; subagent-driven execution adds independent task-level reviews. Preserve the user's chosen method and implement only after that review. If the user instead wants to build it personally, use this as the shared development checklist.

## Primary references checked during planning

- [Vite setup and runtime requirements](https://vite.dev/guide/)
- [Vitest setup](https://vitest.dev/guide/)
- [PostgreSQL transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [FastAPI file responses](https://fastapi.tiangolo.com/advanced/custom-response/#fileresponse)
- [React Effect cleanup and request synchronization](https://react.dev/reference/react/useEffect)
