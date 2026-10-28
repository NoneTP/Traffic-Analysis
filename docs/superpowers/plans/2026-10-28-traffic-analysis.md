# Traffic Analysis: Central Team Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans to implement the selected workstream task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. Updating this plan does not start implementation or authorize messaging either developer.

**Goal:** NoneTP and Puntawit jointly deliver a local vehicle-type detection, counting, storage, and visualization system, with a fine-tuned model and clear portfolio evidence for both roles.

**Architecture:** A Python processor runs Puntawit's detector/tracker and writes vehicle crossing events to PostgreSQL. FastAPI serves run-scoped statistics to a React dashboard that refreshes every four seconds. The developers work against a shared adapter contract and integrate an initial model early, before final training and UI polish.

**Tech Stack:** Python, OpenCV, Pydantic, PostgreSQL, Psycopg, FastAPI, React, TypeScript, Vite, Docker Compose. Puntawit selects the detector, tracker, and training framework in M2.

**Spec:** [Phase-one design](../specs/2026-10-28-traffic-analysis-design.md).

**Status:** Team planning only. Ownership names supplied by NoneTP on 2026-10-28; no product code, trained model, dataset selection, or completed tasks are implied.

## Start here

This document is the shared roadmap and task-status source for both developers. It sets ownership, dependencies, integration checkpoints, and acceptance criteria. Detailed technical checklists live in two equally important workstream plans:

| Developer | Target role | Detailed plan |
| --- | --- | --- |
| **NoneTP** (Taranon) | Pipeline engineering | [Pipeline tasks N1-N8](2026-10-28-pipeline-workstream.md) |
| **Puntawit** | Machine learning | [ML tasks M1-M5](2026-10-28-ml-workstream.md) |
| **Both** | One working product | Joint checkpoints J1-J3 below |

Start with J1. Each developer can then progress in their own workstream. Use N8/J3 as the final joint acceptance exercise; the separate workstreams are not separate finished products.

## Global Constraints

- Process preselected recorded traffic videos on a local machine.
- Detect and classify car, truck, motorcycle, and pickup truck as distinct classes.
- Count each tracked vehicle once when it crosses a configured line.
- Refresh statistics and processing progress approximately every 3-5 seconds.
- Make an annotated video available after processing completes.
- Fine-tune a pretrained vehicle model; Puntawit owns the model and training choices.
- Do not report sample or fixture results as model performance.
- Keep video assets and credentials out of Git; document data provenance and reuse conditions once footage is selected.
- The chosen run determines all dashboard totals. Counts from other runs are never implicitly added to it.
- GPU processing is launched separately from Compose initially.
- Public deployment, browser uploads, live CCTV input, and fuel-type classification remain outside this phase.

## Review Focus

1. The chosen data cannot distinguish pickups from other cars/trucks: detect this in M1/J1 before investing in training; do not silently merge classes.
2. Adjacent frames, duplicate source files, or shared sequences cross train/test boundaries: M1/M4 checks and preserves grouped splits before evaluation.
3. The detector produces boxes but integration needs stable tracks and agreed coordinates/time: M2/M5 and N1/N3 test the exact shared contract before J2.
4. Statistics disagree during processing or a rerun: N2/N5 test transaction consistency and run isolation; J2/J3 check actual model-driven results.
5. One workstream is labeled complete without the other: J3 requires measured ML results and an integrated system; fixtures, training loss, and screenshots alone do not satisfy the shared finish line.

## Ownership and review boundaries

| Area | Implementation lead | Review/contribution |
| --- | --- | --- |
| Dataset search, annotation audit, splits | Puntawit | NoneTP verifies source identity, paths, and integration footage |
| Detector selection, baseline, fine-tuning | Puntawit | NoneTP checks artifact/config reproducibility |
| Tracking and model adapter | Puntawit | NoneTP checks contract and lifecycle integration |
| Video reader, timestamps, crossing counter | NoneTP | Puntawit checks geometry, tracker behavior, and counting line |
| Database, API, incremental writes | NoneTP | Puntawit checks that outputs preserve ML meaning |
| React dashboard, local services, CI | NoneTP | Puntawit tests real-model results and presentation |
| ML metrics and error analysis | Puntawit | NoneTP reviews provenance and pipeline-related errors |
| Per-minute count comparison and final demo | Both | NoneTP owns the SQL/count report tool; Puntawit owns annotations and model analysis |
| Documentation and portfolio evidence | Both | Each explains their own decisions and links their contributions |

These are the working allocations requested by NoneTP, not a claim that Puntawit has reviewed every detail. Confirm workload and open ML decisions together at J1. A review request listed here is a suggested human workflow, not authorization for the assistant to send messages.

Shared files (`contracts.py`, `pyproject.toml`, `cli.py`, source/config schemas) need explicit coordination. NoneTP maintains the common contracts and CLI; Puntawit supplies model integration changes against them. Puntawit owns `ml/`, `config/ml/`, ML tests/reports, and `adapters/model.py`. Avoid concurrent incompatible edits to shared files.

## Central task board

All tasks start unchecked. Check a task only when its evidence exists; record the relevant commit, report, or test output alongside it during execution.

| Done | ID | Deliverable | Owner | Depends on | Acceptance evidence |
| --- | --- | --- | --- | --- | --- |
| [ ] | J1 | Shared labels, boundaries, and evaluation agreement | Both | None | Agreed class policy, adapter contract, ML quality criteria, availability note |
| [ ] | N1 | Validated contracts and synthetic adapter | NoneTP | J1 | Contract tests; fixture data clearly labeled |
| [ ] | M1 | Usable dataset and grouped split manifest | Puntawit | J1 | Data audit, four-class support, provenance, no split leakage |
| [ ] | N2 | Database and consistent statistics | NoneTP | N1 | PostgreSQL uniqueness, rollback, run isolation, bucket tests |
| [ ] | M2 | Pretrained baseline and initial tracking adapter | Puntawit | M1, N1 | Baseline report, truthful class mapping, frame-to-track smoke check |
| [ ] | N3 | Reader, video time, and crossing counter | NoneTP; Puntawit reviews | N1 | Finite-line, jitter, timestamp, and one-count tests |
| [ ] | N4 | Incremental processor and annotated artifact | NoneTP | N2, N3 | Fixture run writes progress, handles failure, finalizes output |
| [ ] | J2 | Early real-model integration | Both | M2, N4 | Baseline or provisional adapter produces tracked events and a completed annotated clip |
| [ ] | N5 | Read API and run/artifact endpoints | NoneTP | N2; N4 for full smoke check | API contracts, consistent response, completed artifact access |
| [ ] | M3 | Reproducible fine-tuning experiments | Puntawit | M2 | Training config, seed/environment, validation results, checkpoint hashes |
| [ ] | N6 | React statistics and four-second updates | NoneTP | N5 | Partial/stale/failed/completed states and race-safe polling |
| [ ] | M4 | Frozen tuned model and held-out ML evaluation | Puntawit | M3 | Per-class metrics, baseline comparison, failures, untouched test set |
| [ ] | N7 | Reproducible local stack and pipeline CI | NoneTP | N4, N5, N6 | Clean local setup, backend/frontend checks, durable database |
| [ ] | M5 | Versioned tuned adapter and model handoff | Puntawit | M4; J2 feedback | Exact artifact retrieval, model hash, class map, adapter tests |
| [ ] | N8 | Tuned-model system run and count report | Both; NoneTP leads integration | N7, M5 | Real footage reaches UI; manual vs predicted counts by class |
| [ ] | J3 | Shared acceptance and portfolio package | Both | N8, M4 | Reproducible demo, quality criteria, separate individual contributions |

M3 can continue while NoneTP builds N5-N7. J2 uses the initial baseline/provisional adapter and does not wait for the selected tuned model. If the baseline lacks the pickup label, J2 explicitly records that limitation; J3 still requires the accepted four-class model.

## Joint checkpoints

### J1: Agree what crosses the boundary

**Owners:** NoneTP and Puntawit. **Files:** `docs/ml-handoff.md`, `docs/ml-experiments.md`, `config/sources.example.json`. These are planned outputs, not files already created by this planning update.

- [ ] Agree the four label IDs/names and treatment of SUVs, vans, buses, unknowns, and ambiguous pickups. Document unsupported labels as unsupported; do not silently map them into a supported type.
- [ ] Agree `TrackerAdapter` consumes video frames and emits normalized boxes, classes, confidence, and stable run-local IDs, using the exact contracts in the pipeline plan. Puntawit owns tracker state; NoneTP owns video time and crossing state.
- [ ] Agree how selected datasets and checkpoints are shared through reproducible references without committing large files; confirm license/provenance and source hashes.
- [ ] Confirm Puntawit's weekly availability, hardware access, and whether NoneTP's GPU is shared. Keep time budgets distinct; neither developer's available time is assumed for the other.
- [ ] Record the metric set and minimum per-class quality criteria before final evaluation. M1/M2 may establish feasibility; freeze the criteria before evaluating the final test set. Define what improvement or limitation the fine-tuning experiment should demonstrate.
- [ ] Create one small shared smoke-test input or reference to it. NoneTP can use labeled synthetic tracks until M2 produces real output. Each developer checks that they can consume the other's agreed file/type format.

**Exit:** The interface and responsibilities are agreed. Dataset/model selection remains Puntawit's work in M1/M2. Do not make pipeline scaffolding wait for a completed training run.

### J2: Integrate early

**Owners:** Both. **Files:** `docs/demo.md`, `docs/ml-handoff.md`; adapter changes owned by Puntawit, processor fixes owned by NoneTP.

- [ ] Run M2's initial adapter through N4 on a short real clip; retain the real model identity and label coverage.
- [ ] Check box coordinates, image color convention, frame order, timestamp use, GPU memory, `reset()`, resource cleanup, and tracker IDs through the crossing counter.
- [ ] Compare a few manually observed line crossings to stored events. Diagnose label/tracker errors separately from timing/counter/database errors.
- [ ] Reprocess the clip into a separate run; verify that the first run's totals remain unchanged and an annotated artifact can be opened/downloaded.
- [ ] Record failures and agree any contract correction before fine-tuning or dashboard integration depends on it.

**Exit:** A genuine model-to-database path works, with any baseline class limitations clearly recorded. This is an integration milestone, not final model-quality acceptance.

### J3: Finish one product and two portfolio stories

**Owners:** Both. **Files:** `docs/demo.md`, `docs/evaluation.md`, `docs/ml-experiments.md`, `README.md`.

- [ ] From documented local setup, process held-out footage using M5's tuned model and show progress/counts/volume through FastAPI and React.
- [ ] Demonstrate all four class definitions in the evaluation evidence. A clip may lack a class, but the complete held-out evaluation must report its coverage and metrics; insufficient data is a gap to resolve together.
- [ ] Compare manually annotated crossing counts with predicted counts by class and video-time interval. Report observed errors and agreed quality criteria honestly.
- [ ] Show completed annotated output, separate rerun results, and a useful failure/stale-state demonstration. Confirm the service and processing checks from N7 still pass.
- [ ] Puntawit records data decisions, baseline limitations, fine-tuning settings, held-out metrics, and model/tracking failure cases.
- [ ] NoneTP records contracts, SQL/storage decisions, transactional/duplicate handling, API integration, deployment reproduction, and reliability evidence.
- [ ] Both record individual commits/PRs or task contributions and rehearse a short explanation of the full architecture and their own work.

**Exit:** A reproducible shared demo plus distinct, evidence-backed ML and pipeline contributions. Training loss alone, fixture-only output, or a dashboard screenshot is not the final product.

## Suggested cadence and capacity

This is a milestone order, not a fixed deadline or an assumption that both developers have equal free time.

| Window | NoneTP | Puntawit | Shared checkpoint |
| --- | --- | --- | --- |
| Opening sessions | N1, then N2/N3 | M1, then M2 | J1; agree labels, time, and contract |
| Middle sessions | N4/N5, then N6 | Baseline/tracking, then M3 | J2 as soon as N4/M2 are ready |
| Closing sessions | N7 and integration fixes | M4/M5 and ML report | N8/J3; compare counts and record demo |

NoneTP has stated roughly 12-21 hours over 2-3 weeks. The pipeline plan carries an approximately 18-hour target plus contingency, not a guarantee. Puntawit's time and training setup are still unknown. At each joint checkpoint, update this board with actual progress and remaining work; annotation, pickup-label coverage, GPU compatibility, and training runtime may change the schedule.

If time becomes tight, simplify styling and nonessential UI interactions first. Changing the four classes, removing fine-tuning, or dropping the integrated dashboard changes the accepted scope and must be agreed together. Record an incomplete milestone accurately rather than calling it phase-one completion.

## Working agreements

- Each task has one implementation lead; involve the other developer when its interface or meaning changes.
- Use short branches such as `codex/pipeline-events` and `codex/ml-baseline` unless the team chooses another convention. Keep shared schema/label changes explicit in reviews.
- In a task's PR or commit notes, record owner, task ID, behavior changed, and verification. For ML work, attach the data/config/checkpoint identity and metric report reference.
- Merge or share the small common contract early. NoneTP's fixture checks and Puntawit's training work can then proceed independently against it.
- Before integrating shared-file changes, check the other workstream's consumers. Avoid one developer independently changing class names, box conventions, or run metadata.
- Keep credentials, raw videos, datasets, annotated outputs, and model weights out of ordinary Git commits. Commit configuration, small permitted fixtures, manifests, reports, and retrieval instructions.
- Keep central task status here. Put detailed implementation notes/tests in the workstream plans and measurements in the referenced reports; do not maintain contradictory duplicate status boards.

## Open decisions and who resolves them

| Decision | Lead | Needed by |
| --- | --- | --- |
| Data provenance, four-class annotations, annotation effort | Puntawit; NoneTP reviews source handling | M1 |
| Detector, pretrained checkpoint, framework, tracking method | Puntawit | M2 |
| Detailed class/unsupported-label policy | Puntawit proposes; both agree | J1/M1 |
| Dataset splits and evaluation metrics/quality criteria | Puntawit proposes; both agree | M1/M2; freeze before M4 |
| Input/box/time/reset contract corrections | Both | J2 |
| Training hyperparameters and checkpoint selection | Puntawit | M3 |
| Model/GPU dependency compatibility | Puntawit; NoneTP checks local worker integration | M2/J2 |
| Individual availability and shared GPU access | Both | J1 |

## Execution handoff

Read this central plan first, then the relevant workstream plan. Review the allocation together before execution; no coding starts as part of this documentation request. An assistant implementing either lane must preserve the task owners and the decisions assigned to Puntawit rather than treating the central plan as permission to select his model for him.

The documents were checked for matching task IDs, links, owners, and adapter boundaries. Verification commands inside the workstream plans describe future checks; they have not been run against product code.
