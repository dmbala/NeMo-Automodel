# Reliability-Aware Training Implementation Checklist

This checklist turns the two-phase research plan into concrete repository work.

Research goal:
- Phase 1: use `cluster_bench/` as a pre-flight reliability screen before training.
- Phase 2: use lightweight runtime monitoring to trigger restart or fallback policies during training.

Recommended execution order:
1. Fix measurement-path blockers in `cluster_bench/`.
2. Normalize all bench and training outputs into one schema.
3. Build Phase 1 feature/label tables and a first risk model.
4. Add a pre-flight gate for training launch.
5. Add a runtime monitor plus one intervention policy.
6. Generate experiment reports and paper figures.

## 0. Measurement Blockers To Fix First

### Existing file: `cluster_bench/sbatch/accept_node.slrm`
- [ ] Replace `BASIC_EXAMPLES_ROOT` usage with a defined path rooted from `CLUSTER_BENCH_ROOT`.
- [ ] Compute the `basic_examples` root once near the top of the script.
- [ ] Make Stage 6 degrade to `WARN` only when data or helper scripts are missing, not when the shell aborts.
- [ ] Preserve final `verdict.json` generation even when MTTR is skipped.

Done when:
- Running the script with `set -u` no longer aborts on the MTTR stage path setup.

### Existing file: `cluster_bench/sbatch/mttr.slrm`
- [ ] Replace `BASIC_EXAMPLES_ROOT` with an explicit path derived from the repo layout.
- [ ] Keep the early user-facing error if `kill_and_resume.sh` is actually absent.
- [ ] Leave the output schema unchanged unless a new history field is intentionally added.

Done when:
- The script reaches its input checks instead of dying during variable expansion.

### Existing file: `cluster_bench/analysis/scrape_metrics.py`
- [ ] Add ingestion support for `cluster_bench/nccl_pair_matrix/`.
- [ ] Decide on one canonical row shape for pairwise NCCL rows:
  `kind=nccl_pair_matrix`, `src_host`, `dst_host`, `collective`, `size_bytes`, `busbw_gbps`, `timestamp_utc`.
- [ ] Add ingestion support for inter-node NCCL raw outputs if CSVs are not emitted.
- [ ] Restrict rescans so the scraper only processes fresh directories for the active bucket, not all historical directories.

Done when:
- Pair-matrix outputs and inter-node NCCL runs both show up in `history.jsonl`.

### Existing file: `cluster_bench/analysis/correlate.py`
- [ ] Either implement the documented `--snapshots` / fabric-verdict path, or remove the unsupported claims from the module docs.
- [ ] If implementing fabric correlation, consume `counter_delta.log` or snapshot-derived error indicators.
- [ ] Add a clear rule for `DEGRADED_FABRIC` versus `DEGRADED_NCCL`.

Done when:
- The CLI and the module header describe the same functionality.

## 1. Shared Schema And Research Scaffold

### New file: `research/reliability/README.md`
- [ ] Describe the project scope, the two phases, and the exact experiment loop.
- [ ] Document where raw data, normalized tables, trained models, and reports live.
- [ ] Include one short “quick start” flow for collecting a pilot dataset.

Done when:
- A collaborator can understand the research workflow without reading code first.

### New file: `research/reliability/results_schema.md`
- [ ] Define the canonical normalized row shapes for:
  `preflight_feature`, `training_outcome`, `runtime_event`, `policy_action`, `experiment_summary`.
- [ ] Define stable IDs and join keys:
  `run_id`, `node_id`, `allocation_id`, `timestamp_utc`, `workload_id`, `policy_id`.
- [ ] Specify which fields come from `cluster_bench` and which come from training logs.

Done when:
- Every downstream script can target one documented schema instead of ad hoc parsing.

### New directory: `research/reliability/configs/`
- [ ] Add `phase1_thresholds.yaml` for hand-tuned screening baselines.
- [ ] Add `phase2_policy.yaml` for runtime trigger thresholds and cooldown windows.
- [ ] Add `workloads.yaml` enumerating the supported pilot workloads and labels.
- [ ] Add `fallback_recipes.yaml` mapping risky workloads to safer restart configs.

Done when:
- Thresholds and policy settings no longer live only inside scripts.

## 2. Phase 1: Pre-Flight Feature Pipeline

### Existing file: `cluster_bench/analysis/scrape_metrics.py`
- [ ] Ensure all Phase 1 signals write normalized rows with stable IDs.
- [ ] Add a `source_path` field for traceability back to raw artifacts.
- [ ] Add optional `job_name` or `bench_name` tags to disambiguate benchmark provenance.

### New file: `cluster_bench/analysis/build_feature_table.py`
- [ ] Read normalized `history.jsonl`.
- [ ] Group related pre-flight measurements into one feature row per candidate node or allocation.
- [ ] Export a tabular file under `research/reliability/data/features_phase1.jsonl` or `.csv`.
- [ ] Start with features from:
  GPU/NIC affinity, topology class counts, NCCL bandwidth at 1 GB, compute MFU baseline, storage throughput, MTTR summary.

Done when:
- One command builds a model-ready feature table from `cluster_bench` history.

### New file: `cluster_bench/analysis/build_labels.py`
- [ ] Parse training outputs and convert them into labels such as:
  `healthy`, `degraded`, `failed`.
- [ ] Define at least one numeric target:
  mean MFU, wall-clock to target step, MTTR, or checkpoint throughput.
- [ ] Allow threshold-based relabeling through CLI flags or a config file.

Done when:
- You can join a training run to both a class label and a numeric outcome.

### New file: `cluster_bench/analysis/join_features_and_labels.py`
- [ ] Join pre-flight rows to training outcomes using `run_id`, host info, and time windows.
- [ ] Emit one training example per run.
- [ ] Surface unmatched rows as warnings or a sidecar report.

Done when:
- There is a single supervised-learning dataset for Phase 1.

### New file: `cluster_bench/analysis/train_risk_model.py`
- [ ] Train one simple baseline model first:
  logistic regression or gradient-boosted trees.
- [ ] Support both binary risk prediction and multi-class health prediction.
- [ ] Emit:
  metrics JSON, confusion matrix, ROC/PR data, and a serialized model artifact.
- [ ] Save outputs under `research/reliability/models/phase1/`.

Done when:
- The repo can reproduce a first trained risk model from normalized data.

### New file: `cluster_bench/analysis/score_run.py`
- [ ] Load the serialized Phase 1 model.
- [ ] Score one candidate node or one feature row.
- [ ] Emit a compact decision object:
  `risk_score`, `predicted_class`, `top_features`.

Done when:
- A launch wrapper can ask for a pre-flight decision without retraining the model.

## 3. Training Outcome Collection

### Existing file: `basic_examples/02_pretrain/pretrain.sh`
- [ ] Add optional env vars or flags for a research `run_id`, `policy_id`, and structured output location.
- [ ] Append a compact metadata line or JSON artifact that records:
  config path, node info, start/end time, exit code, checkpoint dir.

### Existing file: `basic_examples/04_finetune/finetune.sh`
- [ ] Add the same metadata hooks as `pretrain.sh`.
- [ ] Ensure restart-driven runs can preserve the same logical `run_id` while incrementing an `attempt_id`.

### Existing file: `basic_examples/14_scaling/ladder.sh`
- [ ] Add the same metadata hooks for scaling experiments.
- [ ] Record the selected scaling step and effective world size in a machine-readable artifact.

### New file: `research/reliability/collect_training_outcome.py`
- [ ] Parse metadata and selected logs from the chosen training workloads.
- [ ] Emit normalized `training_outcome` rows.
- [ ] Record:
  `exit_code`, `success`, `mean_mfu`, `avg_iter_time_s`, `time_to_first_ckpt_s`, `final_step`, `num_restarts`.

Done when:
- Pilot workloads produce structured outcomes without manual log scraping.

## 4. Phase 1 Launch Gate

### New file: `research/reliability/policies/preflight_gate.py`
- [ ] Accept a candidate node or allocation descriptor plus a feature row.
- [ ] Support two modes:
  threshold baseline and learned model.
- [ ] Emit one of:
  `launch`, `warn`, `skip`.
- [ ] Record the decision as a normalized `policy_action` row.

Done when:
- Pre-flight screening can be evaluated as a reproducible baseline policy.

### New file: `research/reliability/policies/launch_with_gate.sh`
- [ ] Wrap a selected training entrypoint.
- [ ] Run the feature-scoring step first.
- [ ] Respect the gate decision.
- [ ] If launching, write a run manifest tying the decision to the training job.

Done when:
- Phase 1 experiments can be run with and without screening using the same wrapper.

## 5. Phase 2: Runtime Monitoring And Intervention

### New file: `research/reliability/policies/runtime_monitor.py`
- [ ] Tail structured training artifacts plus selected bench signals.
- [ ] Compute rolling indicators:
  MFU drop, step-time drift, checkpoint-latency spike, NCCL degradation, error-counter change.
- [ ] Emit normalized `runtime_event` rows at a fixed interval.

Done when:
- There is one machine-readable runtime stream for policy logic.

### New file: `research/reliability/policies/restart_policy.py`
- [ ] Consume `runtime_event` rows and `phase2_policy.yaml`.
- [ ] Implement one minimal intervention first:
  `checkpoint_and_restart_on_healthier_node`.
- [ ] Add cooldown logic to avoid repeated thrashing.
- [ ] Emit a normalized `policy_action` row each time the policy fires.

Done when:
- Phase 2 has one real intervention policy, not just passive monitoring.

### New file: `research/reliability/policies/slurm_requeue_wrapper.sh`
- [ ] Translate a policy action into Slurm-friendly behavior.
- [ ] Preserve the logical `run_id` across attempts.
- [ ] Attach restart metadata such as `attempt_id`, `restart_reason`, and prior node list.

Done when:
- A runtime intervention can be executed reproducibly by the training wrapper.

### Existing file: `basic_examples/08_fault_tolerance/kill_and_resume.sh`
- [ ] Reuse this script as the reference restart flow.
- [ ] Avoid hard-coding it as the only intervention path; keep policy hooks separate.
- [ ] Optionally expose one machine-readable summary line for MTTR experiments.

### Existing file: `basic_examples/08_fault_tolerance/elastic_restart.slrm`
- [ ] Use this as a runtime-policy experiment harness, not as the policy implementation itself.
- [ ] Add optional research metadata hooks if needed for `run_id` and `attempt_id`.

## 6. Experiment Orchestration

### New file: `research/reliability/configs/workloads.yaml`
- [ ] Enumerate the pilot workloads:
  small pretrain, LoRA finetune, one scaling case.
- [ ] Record expected resource shape and success criteria for each workload.

### New file: `research/reliability/run_phase1_experiment.py`
- [ ] Launch repeated pre-flight-plus-training experiments.
- [ ] Support experiment modes:
  `no_screening`, `threshold_screening`, `model_screening`.
- [ ] Save a manifest of all runs and decisions.

### New file: `research/reliability/run_phase2_experiment.py`
- [ ] Launch runtime-policy experiments.
- [ ] Support modes:
  `static`, `monitor_only`, `monitor_plus_restart`.
- [ ] Save intervention counts and final outcomes per run.

Done when:
- The same experiment can be rerun with only a config change.

## 7. Analysis And Reporting

### New file: `research/reliability/analysis/phase1_prediction_report.py`
- [ ] Read Phase 1 datasets and model metrics.
- [ ] Output tables for:
  AUC, F1, calibration, feature importance, false positives, false negatives.

### New file: `research/reliability/analysis/phase2_policy_report.py`
- [ ] Compare policy modes on:
  completion rate, wall-clock to target step, mean MFU, MTTR, wasted GPU-hours.
- [ ] Emit both per-workload and aggregate summaries.

### New file: `research/reliability/analysis/plot_mfu_vs_risk.py`
- [ ] Plot predicted risk against realized MFU or slowdown.

### New file: `research/reliability/analysis/plot_gpu_hours_saved.py`
- [ ] Plot intervention benefit as avoided failures or reduced wasted compute.

### Existing file: `cluster_bench/analysis/report.py`
- [ ] Optionally add a reliability-focused section once normalized research rows are stable.
- [ ] Keep generic cluster history reporting separate from paper-specific reporting if the file becomes too broad.

Done when:
- The repo can produce paper-ready Phase 1 and Phase 2 summaries from raw results.

## 8. Recommended Pilot Dataset

Use this first before scaling up:

- [ ] `basic_examples/02_pretrain/pretrain.sh` on 1 GPU
- [ ] `basic_examples/04_finetune/finetune.sh` on 4 GPUs
- [ ] `basic_examples/14_scaling/ladder.sh` for one 2-node point
- [ ] `cluster_bench/sbatch/per_node.slrm` before each training batch
- [ ] `cluster_bench/nccl_tests/intra_node.slrm` and one inter-node NCCL path for multi-GPU cases
- [ ] Repeat each condition at least 5 times

Labels to start with:
- [ ] `healthy`: completed and MFU above chosen threshold
- [ ] `degraded`: completed but below MFU or above latency threshold
- [ ] `failed`: crashed, timed out, or required restart

## 9. Success Criteria For A First Paper

Phase 1 minimum:
- [ ] Predict degraded or failed runs better than hand-written screening thresholds.

Phase 2 minimum:
- [ ] Show that one runtime policy improves at least one of:
  completion rate, MFU, time-to-target-step, wasted GPU-hours, MTTR.

Repo minimum:
- [ ] All scripts emit enough structured data to reproduce the experiment tables.
