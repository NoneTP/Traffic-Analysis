# ML Workstream Implementation Plan: Puntawit

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans to implement the selected tasks. Steps use checkbox (`- [ ]`) syntax for tracking. Model/framework choices belong to Puntawit; confirm those choices before writing framework-specific implementation.

**Goal:** Prepare suitable data, establish a pretrained baseline, fine-tune a four-class vehicle detector, and hand off an evaluated detector/tracker adapter to the shared traffic-analysis system.

**Architecture:** Reproducible data manifests and experiment configuration drive baseline/training/evaluation scripts. A tracker wraps detector output and implements the same `TrackerAdapter` consumed by NoneTP's processor. The adapter owns model inference and tracking; video timing, counting, database writes, and API/UI remain in NoneTP's workstream.

**Tech Stack:** Python plus the model/tracker/training framework selected in M2. Shared NumPy/Pydantic contracts come from N1. Avoid selecting a detector framework or exact training settings without Puntawit's decision.

**Spec:** [Phase-one design](../specs/2026-10-28-traffic-analysis-design.md).

**Central plan:** [Team roadmap, task board, and joint checkpoints](2026-10-28-traffic-analysis.md). [NoneTP's pipeline details](2026-10-28-pipeline-workstream.md) define the exact shared types.

**Status:** Planned; all tasks are unchecked. Fine-tuning is confirmed. Model, dataset, tracker, available training memory, and Puntawit's weekly availability are not yet selected/confirmed.

## Global Constraints

- Process preselected recorded traffic videos on a local machine.
- Detect and classify car, truck, motorcycle, and pickup truck as distinct classes.
- Count each tracked vehicle once when it crosses a configured line.
- Fine-tune a pretrained vehicle model; Puntawit owns the model and training choices.
- Do not report sample or fixture results as model performance.
- Keep video assets and credentials out of Git; document data provenance and reuse conditions once footage is selected.
- Keep train, validation, and test source sequences separate; adjacent frames from the same sequence cannot become independent evidence of generalization.
- Freeze the selected checkpoint and inference settings before final held-out evaluation. Use validation results for tuning.
- Live CCTV and fuel-type classification remain outside phase one.

## Review Focus

1. Missing pickup labels or ambiguous car/truck mapping: M1 verifies labels and records the team's agreed taxonomy before training.
2. Duplicate footage and adjacent-frame split leakage: M1 validates source hashes, sequence groups, and split membership.
3. A pretrained baseline cannot emit all four labels: M2 reports unsupported classes explicitly; it does not fabricate equivalent scores.
4. Good image metrics hide tracking/count failures: M2/M5 validate IDs, reset behavior, ordering, and geometry; J2/J3 compare crossings.
5. A saved checkpoint cannot be reproduced or integrated: M3 saves settings/hashes; M4 freezes the selection; M5 reloads the delivered artifact in a clean documented environment.

## Planned files and ownership

Puntawit owns the files below. They are proposed outputs of implementation, not files created by this plan.

```text
ml/data.py                            Manifest/annotation audit and grouped split validation
ml/experiment.py                      Validated experiment config and artifact metadata
ml/baseline.py                        Chosen framework's reproducible pretrained evaluation
ml/train.py                           Chosen framework's fine-tuning entry point
ml/evaluate.py                        Held-out detection metrics and diagnostic exports
config/ml/classes.json                Agreed class names/IDs and unsupported-label policy
config/ml/data-manifest.csv            Source, hashes, annotation references, sequence groups
config/ml/splits.csv                   Reproducible source-sequence split membership
config/ml/experiments/                 Small baseline/training/inference configuration files
tests/ml/test_data.py                  Invalid labels/boxes and split leakage
tests/ml/test_experiment.py            Config validity and model identity
tests/ml/test_metrics.py               Metric wiring and class-support reporting
tests/ml/test_model_adapter.py         Box mapping, IDs, reset, unsupported labels, cleanup
src/traffic_analysis/adapters/model.py Real detector/tracker adapter shared with the pipeline
docs/ml-experiments.md                 Data card, baseline, experiments, model selection
docs/ml-handoff.md                     Shared contract and model retrieval/inference instructions
docs/evaluation.md                     Joint final metric/count report
```

NoneTP owns `contracts.py`, `video.py`, `counter.py`, `processor.py`, storage/API/frontend, and the shared CLI. Coordinate changes to those files. Framework-specific dependencies may use a separate ML environment or optional extra; keep the processor/adapter runtime compatible with the agreed shared Python contracts.

## Shared inputs and outputs

Use the exact N1 contract rather than inventing a second box/class schema:

- Input: `VideoFrame(index: int, video_ms: int, image: numpy.ndarray)`. Agree at J1 that decoded images are OpenCV BGR; any RGB conversion happens once inside the model adapter.
- Output: `list[TrackedDetection]`, each with `track_id`, `vehicle_class`, `confidence`, and `bbox(x1,y1,x2,y2)` in original frame pixel coordinates. Return an empty list when there are no supported detections; do not emit a placeholder vehicle.
- Canonical classes: `car`, `truck`, `motorcycle`, `pickup_truck`. Keep explicit raw-model-label-to-canonical mapping; unsupported/uncertain mapping is `None` or is filtered under the agreed policy.
- Adapter: `identity: ModelIdentity`, `kind='model'`, `reset()`, `predict(frame)`, `close()`. Fresh runs reset tracker state; track IDs need only be unique/stable within a run.
- Identity: model name, exact weights SHA-256, complete inference/tracker configuration, class mapping, and retrieval reference recorded in the handoff.
- Tracking errors remain visible as tracking errors. SQL uniqueness cannot repair ID switches or two identities for one physical vehicle.

Metric policy: report per-class precision, recall, and AP using the selected framework's documented IoU/aggregation definitions, plus support/example counts and failure cases. Record `AP50` and `mAP50-95` when supported by the chosen implementation; mark unavailable metrics explicitly. Counts are separately compared using the shared count report from N8. Do not treat missing reference counts as zero.

### Task M1: Establish data suitability and leakage-safe splits

**Owner:** Puntawit. **Reviewer:** NoneTP for source identity/paths; both agree the class policy. **Depends on:** J1. **Unblocks:** M2; real-video selection for J2.

**Files:** Create `ml/data.py`, `config/ml/classes.json`, `config/ml/data-manifest.csv`, `config/ml/splits.csv`, `tests/ml/test_data.py`; begin `docs/ml-experiments.md`.

**Interfaces:** `audit_dataset(manifest_path: Path, classes_path: Path) -> DataAuditReport`; `validate_splits(manifest_path: Path, splits_path: Path) -> SplitAuditReport`. Reports contain class counts, source/sequence counts, invalid-label/box entries, missing annotation references, and leakage errors; split membership keys are source IDs and content hashes, not machine-specific absolute paths.

- [ ] **Step 1: Audit candidate data before adopting it.** Record original source, access/license terms, annotation format, camera views, conditions, class definitions, and whether pickups are explicitly annotated. Verify sample images/frames and label quality manually. Isolated image datasets can help training, but grouped video footage is also needed for tracking/counting evaluation. Record any extra pickup annotation effort as work, not an assumed free input.
- [ ] **Step 2: Write audit tests.** `test_rejects_unknown_class_id` rejects IDs absent from `classes.json`; `test_rejects_invalid_boxes` rejects reversed/nonfinite/out-of-bounds boxes; `test_no_duplicate_content_across_splits` rejects identical hashes in train/test; `test_sequence_group_stays_in_one_split` rejects adjacent segments/frames from the same sequence group assigned to different splits; `test_reports_missing_pickup_support` makes missing target-class support visible.
- [ ] **Step 3: Run `python -m pytest tests/ml/test_data.py -q`.** Expect failure before audit functions exist. Implement the audit/split validators and rerun; expect the malformed fixtures rejected and a valid grouped manifest accepted. Unit fixtures should be tiny metadata/annotation examples; do not download an entire dataset in tests.
- [ ] **Step 4: Publish the chosen data manifest and split assignment.** Group by original camera/recording sequence, not just filenames; keep original-content identities after transcoding. Keep test sequences out of all tuning. Record per-split class counts and ensure all four classes have evaluation coverage; if not feasible, resolve the gap jointly before claiming readiness. Have NoneTP verify access to a short integration clip.
- [ ] **Step 5: Commit manifests, audit code/tests, and the data-card section.** Keep dataset binaries out of Git. Suggested message: `feat(ml): audit vehicle labels and preserve grouped dataset splits`.

**Exit evidence:** Usable data references, explicit four-class feasibility, reproducible split membership, and passing audit tests.

### Task M2: Baseline detector and an initial tracked adapter

**Owner:** Puntawit. **Reviewer:** NoneTP for interface compatibility. **Depends on:** M1 and N1. **Unblocks:** M3 and J2.

**Files:** Create `ml/experiment.py`, `ml/baseline.py`, `config/ml/experiments/baseline.json`, `tests/ml/test_experiment.py`, `tests/ml/test_metrics.py`, `tests/ml/test_model_adapter.py`; begin `src/traffic_analysis/adapters/model.py`. Coordinate dependencies with NoneTP; extend `docs/ml-experiments.md` and `docs/ml-handoff.md`.

**Interfaces:** `BaselineConfig` records model/checkpoint identity, data/split references, image size, device, seed, inference thresholds, and raw class mapping. `evaluate_baseline(config_path: Path, report_dir: Path) -> Path` writes a structured metric report. The initial `ModelTrackerAdapter` implements the N1 `TrackerAdapter`; its concrete constructor/configuration is documented after the framework choice.

- [ ] **Step 1: Select the model/framework/tracker and verify runtime compatibility.** Check actual available GPU memory, a small inference sample, dependency versions, license, and checkpoint access. Keep a CPU-capable smoke path if the framework supports it. Record the selection and why it fits the available data/hardware. No specific framework or model has already been agreed.
- [ ] **Step 2: Write boundary/config tests.** `test_baseline_config_records_class_support` asserts unsupported pickups remain unsupported rather than mapped into cars; `test_identity_changes_with_weights_or_inference_settings` verifies provenance changes; `test_box_mapping_uses_original_frame_pixels` checks scaling after preprocessing; `test_empty_frame_output_contains_no_fake_vehicle` expects `[]`; `test_reset_clears_tracker_state` prevents identities leaking between runs. Tests for normal conversions can use tiny deterministic model outputs, clearly labeled as test doubles.
- [ ] **Step 3: Run `python -m pytest tests/ml/test_experiment.py tests/ml/test_metrics.py tests/ml/test_model_adapter.py -q`.** Expect missing behavior failures, implement the config/metric wiring/initial adapter, and rerun. Use the framework's metric implementation rather than implementing AP from scratch; test the report wiring and class mapping.
- [ ] **Step 4: Run the baseline on validation footage and record supported per-class metrics.** Record preprocessing, thresholds, latency/throughput measurement conditions, class counts, and example errors. If the pretrained label set lacks pickup, expose the missing support and compare only genuinely comparable categories; never relabel unsupported output to improve the report.
- [ ] **Step 5: Give NoneTP the initial adapter and a short real clip for J2.** Verify frame order, BGR/RGB conversion, original pixel boxes, stable tracker IDs, reset/cleanup, and weight/config hashes together. The initial model can have known class limitations, clearly stated; this milestone checks integration.
- [ ] **Step 6: Commit code/config/tests and baseline evidence.** Suggested message: `feat(ml): establish pretrained baseline and tracked inference adapter`.

**Exit evidence:** Reproducible baseline report, verified initial adapter, and a J2-ready real inference sample.

### Task M3: Fine-tune with reproducible experiment records

**Owner:** Puntawit. **Reviewer:** NoneTP for saved artifact/config reproducibility. **Depends on:** M2. **Runs alongside:** N5-N7.

**Files:** Create `ml/train.py`, `config/ml/experiments/finetune-001.json`; extend `ml/experiment.py`, `tests/ml/test_experiment.py`, `docs/ml-experiments.md`. Further experiment configs are added only when their result informs a decision.

**Interfaces:** `TrainConfig` references the chosen pretrained checkpoint, dataset/grouped splits, four-class map, image size, epochs, batch size, optimizer/learning rate, augmentation policy, seed, device, and output location. `train_model(config_path: Path, output_dir: Path) -> Path` returns a model-artifact manifest containing checkpoint reference/hash, configuration hash, framework/dependency versions, and validation result references.

- [ ] **Step 1: Add configuration tests.** `test_training_uses_train_and_validation_only` rejects test-split inputs in training/selection configuration; `test_training_requires_all_four_class_definitions` rejects a silently dropped pickup class; `test_artifact_manifest_has_reproducible_identity` requires checkpoint/config hashes and environment versions. Run `python -m pytest tests/ml/test_experiment.py -q`; expect the new requirements to fail first.
- [ ] **Step 2: Implement the selected framework's training entry point/config translation.** Preserve the shared four-class meaning. Add a very short training smoke configuration to verify annotation parsing, loss computation, checkpoint saving, and reloading before a longer GPU run. Write exact runnable training commands into the experiment notes once the framework is chosen.
- [ ] **Step 3: Run the smoke configuration and the experiment tests.** Expect a real, reloadable checkpoint artifact and passing configuration assertions. Record the actual memory/runtime; if it does not fit the device, change batch/image settings explicitly and save the revised configuration rather than pretending the original succeeded.
- [ ] **Step 4: Run a bounded fine-tuning experiment and inspect validation results.** Inspect losses plus per-class precision/recall/AP and examples of car/pickup confusion and small motorcycles. If a follow-up experiment is warranted, change one interpretable factor and record the hypothesis/result. The number of experiments depends on actual time/compute, not an invented required sweep.
- [ ] **Step 5: Record selected configuration/checkpoint and validation-based reasoning.** Do not inspect final test results to pick thresholds or checkpoints. Save weights through the agreed artifact storage and record retrieval/hash instructions; commit the small configs, code, and experiment notes. Suggested message: `feat(ml): fine-tune four-class vehicle detector reproducibly`.

**Exit evidence:** At least one completed fine-tuning run, a reloadable checkpoint, validation evidence, and reproducible configuration/identity.

### Task M4: Freeze the tuned model and evaluate held-out footage

**Owner:** Puntawit. **Reviewer:** NoneTP; both discuss quality gaps. **Depends on:** M3. **Unblocks:** M5 and J3.

**Files:** Create `ml/evaluate.py`, `config/ml/experiments/selected.json`; extend `tests/ml/test_metrics.py`, `docs/ml-experiments.md`, `docs/evaluation.md`.

**Interfaces:** `evaluate_model(config_path: Path, split: Literal['validation','test'], report_dir: Path) -> Path` writes per-class support/precision/recall/AP, declared metric definitions, error examples, source/split/model hashes, and timing conditions. The frozen configuration includes detector and tracker/inference settings. A report must distinguish unsupported classes from supported classes with zero detections.

- [ ] **Step 1: Freeze the chosen checkpoint, preprocessing, confidence/NMS thresholds, class mapping, and tracker settings.** Confirm the test-set manifest is untouched and the team has recorded quality criteria before running final evaluation. Record any baseline/tuned label-space differences in the comparison design.
- [ ] **Step 2: Add metric-report tests.** `test_report_preserves_zero_support_and_unsupported_classes` distinguishes the cases instead of inventing AP; `test_report_records_model_and_split_hashes` requires exact identities; `test_baseline_comparison_marks_incomparable_classes` prevents a misleading four-class improvement claim. Run `python -m pytest tests/ml/test_metrics.py -q`, implement report assembly around the framework metrics, and rerun.
- [ ] **Step 3: Evaluate the frozen model once on the held-out test split.** Report per-class metrics and support, not just a combined score. Include distant/small objects, occlusion, lighting/view differences present in the chosen data, and confusion between pickups and other classes. Record unsupported or untested conditions explicitly.
- [ ] **Step 4: Compare with the recorded baseline under compatible settings and data.** Separate improvements in existing categories from newly added pickup support. Do not claim general improvement if only training loss improved or category definitions differ. If the test set is used to guide a further model revision, record that it has become development feedback and arrange independent final evaluation evidence.
- [ ] **Step 5: Commit the selected config, evaluator/tests, and honest reports.** If criteria are not met, list the failure and agree the next experiment or scope adjustment together. Suggested message: `docs(ml): evaluate frozen tuned model on held-out traffic footage`.

**Exit evidence:** Frozen model identity, held-out report, baseline comparison with explicit limitations, and a model-quality decision.

### Task M5: Deliver the tuned adapter and integration handoff

**Owner:** Puntawit. **Reviewer:** NoneTP for processor compatibility. **Depends on:** M4 and J2 feedback. **Unblocks:** N8/J3.

**Files:** Finalize `src/traffic_analysis/adapters/model.py`, `tests/ml/test_model_adapter.py`, `docs/ml-handoff.md`; reference `config/ml/experiments/selected.json`. Coordinate shared CLI/config wiring with NoneTP rather than creating a second processor.

**Interfaces:** `ModelTrackerAdapter` remains the same N1 `TrackerAdapter`. Its identity now points to the selected tuned weights and complete inference/tracker config. It does not write SQL, calculate line-crossing totals, alter media time, or update dashboard state.

- [ ] **Step 1: Add final adapter contract tests.** `test_real_adapter_reports_model_kind_and_selected_hash` checks real provenance; `test_track_ids_persist_within_run` checks the selected tracker's expected continuity on a short labeled clip; `test_reset_prevents_cross_run_state` and `test_close_releases_resources` cover lifecycle; `test_unsupported_labels_follow_agreed_policy` verifies the joint class policy. Keep heavy real-model smoke tests opt-in and separate from lightweight contract tests.
- [ ] **Step 2: Run `python -m pytest tests/ml/test_model_adapter.py -q` for lightweight checks.** Fix contract regressions. Then run the documented opt-in real-model smoke command in the selected ML environment; record its actual results and resource requirements.
- [ ] **Step 3: Give NoneTP a reproducible handoff.** Include exact dependency setup, selected config, accessible weight reference, SHA-256, class mapping, tracker settings, expected input convention, a permitted short smoke clip/reference, known failure cases, and observed GPU/CPU behavior. Verify checkpoint retrieval/reloading from these instructions.
- [ ] **Step 4: Run N8 together.** NoneTP connects the tuned adapter to the existing worker/CLI, database/API/UI, and count-evaluation tool. Puntawit supplies manually annotated crossing references and diagnoses detector/tracker errors. Both review how those errors differ from counting/persistence bugs. Do not silently change thresholds from the frozen configuration during this check.
- [ ] **Step 5: Commit final adapter/config/handoff changes and complete J3 jointly.** Suggested message: `feat(ml): hand off versioned tuned detector and tracker adapter`.

**Exit evidence:** Reproducible tuned artifact handoff, passing contract checks, and a real integrated system/count report.

## Execution and completion

Puntawit's model task completion is evidenced by the data/split manifests, reproducible fine-tuning, held-out evaluation, and a working tuned adapter. The project's completion is J3 in the central plan. GPU speed and four-second dashboard refresh are separate measurements.

Framework-specific command syntax is deliberately decided in M2 and recorded in the experiment notes before M3 execution. This plan does not claim a model/dataset has already been chosen or that either developer has completed a task. Keep source code tests tied to the relevant boundary and use actual training/evaluation runs for model-quality evidence.
