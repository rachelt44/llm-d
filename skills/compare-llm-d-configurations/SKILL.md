---
name: compare-llm-d-configurations
description: Compare the benchmark performance of two llm-d stack configurations end-to-end. For each configuration, deploys the stack, runs a benchmark, tears down, then generates a side-by-side results comparison. One configuration may already have been benchmarked previously — in that case, only the new configuration is deployed and benchmarked, and its results are compared against the existing ones. Use this skill whenever the user wants to A/B test llm-d configurations, compare models or serving strategies, evaluate the effect of a configuration change, run two benchmarks back-to-back for comparison, or compare a new run against a previous one — even if they say "which is faster", "test two setups", "compare these configs", "compare against my last run", or don't use "A/B" or "benchmark" terminology explicitly.
---

# Compare llm-d Configurations

## Purpose

Orchestrate two benchmark runs against different llm-d stack configurations, then compare results side by side. Each new run follows the sequence: deploy → benchmark → save state → teardown. If one configuration was already benchmarked in a previous session, its results can be loaded directly — only the other configuration goes through the full deploy/benchmark/teardown cycle. The comparison report is written to disk and displayed inline.

---

## Phase 0: Pre-flight Setup

### 0.1 Create Comparison Workspace

Create a timestamped local directory to hold state for both runs:

```bash
COMPARISON_DIR="llm-d-comparison-$(date +%Y%m%d-%H%M%S)"
mkdir -p $COMPARISON_DIR/run-a/results
mkdir -p $COMPARISON_DIR/run-b/results
echo "Comparison workspace: $COMPARISON_DIR"
```

### 0.2 Check for Pre-existing Run

Ask the user: *"Do you have results from a previous benchmark run you'd like to compare against, or are both configurations being run fresh?"*

- **Both fresh** — proceed normally through Phases 1 and 2.
- **One pre-existing** — ask which one (A or B) is pre-existing, then collect its details now (see below). Skip the deploy/benchmark/teardown phases for that run; only execute those phases for the new configuration.

**Collecting pre-existing run details:**

Ask the user for:
1. A short label for the pre-existing run (e.g. "baseline-qwen2.5-7b")
2. The path to its results directory (will be used for metric extraction in Phase 3)
3. A brief summary of its configuration — guide used, model, hardware, harness, workload (whatever they remember; used to populate the config table in the report)

Write a `run_state.json` for the pre-existing run directly into the comparison workspace, using the information provided:

```json
{
  "label": "<user-provided label>",
  "timestamp": "<timestamp of original run, or 'unknown'>",
  "stack": {
    "guide": "<guide or 'unknown'>",
    "namespace": "<namespace or 'unknown'>",
    "hardware": "<hardware or 'unknown'>",
    "gateway": "<gateway or 'unknown'>",
    "model": "<model or 'unknown'>"
  },
  "benchmark": {
    "harness": "<harness or 'unknown'>",
    "workload": "<workload or 'unknown'>",
    "load_stages": "<load stages or 'unknown'>"
  },
  "results_path": "<absolute path to existing results directory>"
}
```

Copy or symlink the existing results into the comparison workspace so everything is in one place:

```bash
# Copy (preferred — keeps the workspace self-contained)
cp -r <existing-results-path> $COMPARISON_DIR/run-{a,b}/results/

# Or symlink if the results are large
ln -s <existing-results-path> $COMPARISON_DIR/run-{a,b}/results
```

### 0.3 Name the New (or Both) Configurations

Ask the user for a short label for each run that still needs one. Labels should capture what differs between the two (e.g., model name, scheduling strategy, hardware tier). Examples:

- "qwen2.5-7b-inference-scheduling" vs "llama3.1-8b-inference-scheduling"
- "baseline" vs "prefill-decode-disagg"
- "a100-2gpu" vs "a100-4gpu"

Save them for use throughout:

```bash
RUN_A_LABEL="<label>"
RUN_B_LABEL="<label>"
```

### 0.4 Namespace

Only needed if at least one run is being deployed fresh. Both runs share the same namespace by default. Identify it using the standard detection order:

1. If `NAMESPACE` is already set, use it.
2. Check for an active `oc project`: `oc project -q 2>/dev/null`
3. Otherwise ask the user.

---

## Phase 1: Run A

> **If Run A was pre-existing**: its `run_state.json` and results were already set up in Phase 0. Skip directly to Phase 2.

Work through the following steps for the first configuration.

### 1.1 Deploy Stack

Tell the user: *"Starting Run A: $RUN_A_LABEL — deploying the stack."*

Follow the **llm-d-kubernetes-deployment** skill workflow. The namespace is already set from Phase 0.

### 1.2 Run Benchmark

Tell the user: *"Stack is up — running benchmark for Run A."*

Follow the **run-llm-d-benchmark** skill workflow. When the skill asks where to save results, use:

```
$COMPARISON_DIR/run-a/results
```

### 1.3 Save Run A State

After the benchmark completes (before teardown), capture all configuration and results metadata. This is the only opportunity to record deploy details before the stack is removed.

Write `$COMPARISON_DIR/run-a/run_state.json` with values collected during Steps 1.1 and 1.2:

```json
{
  "label": "<RUN_A_LABEL>",
  "timestamp": "<ISO-8601 timestamp>",
  "stack": {
    "guide": "<well-lit-path-guide-name>",
    "namespace": "<namespace>",
    "hardware": "<accelerator type and node count>",
    "gateway": "<gateway provider>",
    "model": "<model name>"
  },
  "benchmark": {
    "harness": "<harness name>",
    "workload": "<workload description>",
    "load_stages": "<stages summary, e.g. '5/10/20 req/s × 60s each'>"
  },
  "results_path": "<absolute path to $COMPARISON_DIR/run-a/results>"
}
```

### 1.4 Teardown Run A Stack

Tell the user: *"Benchmark complete — tearing down Run A stack to prepare for Run B."*

Follow the **teardown-llm-d-stack** skill workflow with these fixed constraints (do not ask the user about the optional steps):

- **Uninstall** all Helm releases (required)
- **Remove** HTTPRoutes and Gateways — needed to avoid routing conflicts when Run B deploys
- **Keep** PVCs — they may contain benchmark data and will be reused for results storage
- **Keep** the namespace — it will be reused for Run B

---

## Phase 2: Run B

> **If Run B was pre-existing**: its `run_state.json` and results were already set up in Phase 0. Skip directly to Phase 3.

Repeat the same sequence for the second configuration.

### 2.1 Deploy Stack

Tell the user: *"Starting Run B: $RUN_B_LABEL — deploying the stack."*

Follow the **llm-d-kubernetes-deployment** skill workflow in the same namespace.

### 2.2 Run Benchmark

Tell the user: *"Stack is up — running benchmark for Run B."*

Follow the **run-llm-d-benchmark** skill workflow. Results path:

```
$COMPARISON_DIR/run-b/results
```

### 2.3 Save Run B State

Write `$COMPARISON_DIR/run-b/run_state.json` with the same structure as Run A.

### 2.4 Teardown Run B Stack

Same constraints as Phase 1.4: remove Helm releases and routing, keep PVCs and namespace.

---

## Phase 3: Generate Comparison Report

### 3.1 Extract Key Metrics

Read the benchmark result files from both `run-a/results` and `run-b/results`. The exact file format depends on the harness used:

- **inference-perf** / **vllm-benchmark**: typically CSV or JSON with per-request timing
- **guidellm**: produces a summary JSON with aggregated percentiles
- **inferencemax**: check for a results summary file

For each run, extract these metrics where available:

| Metric | Description |
|--------|-------------|
| `throughput_req_s` | Requests per second |
| `throughput_tok_s` | Output tokens per second |
| `latency_p50_ms` | End-to-end latency, 50th percentile |
| `latency_p90_ms` | End-to-end latency, 90th percentile |
| `latency_p99_ms` | End-to-end latency, 99th percentile |
| `ttft_p50_ms` | Time to first token, 50th percentile |
| `ttft_p90_ms` | Time to first token, 90th percentile |
| `itl_mean_ms` | Inter-token latency, mean |
| `error_rate_pct` | Percentage of failed requests |

If extraction requires parsing, write and run a short Python script rather than reading manually — it's faster and reproducible. Save extracted metrics back into each `run_state.json` under a `"metrics"` key.

If a metric is not available for a given harness, record it as `"N/A"`.

### 3.2 Write and Display Comparison Report

Write the report to `$COMPARISON_DIR/comparison_report.md`, then display it inline.

Use this structure:

```markdown
# llm-d Configuration Comparison

**Generated**: <timestamp>
**Workspace**: <COMPARISON_DIR>

---

## Configuration Summary

| | Run A: <label> | Run B: <label> |
|---|---|---|
| **Guide** | | |
| **Model** | | |
| **Hardware** | | |
| **Gateway** | | |
| **Harness** | | |
| **Workload** | | |
| **Load stages** | | |

---

## Performance Results

| Metric | Run A: <label> | Run B: <label> | Delta (B − A) |
|---|---|---|---|
| Throughput (req/s) | | | |
| Throughput (tok/s out) | | | |
| Latency P50 (ms) | | | |
| Latency P90 (ms) | | | |
| Latency P99 (ms) | | | |
| TTFT P50 (ms) | | | |
| TTFT P90 (ms) | | | |
| ITL mean (ms) | | | |
| Error rate (%) | | | |

> For throughput, positive delta means Run B is better.
> For latency/TTFT/ITL/error rate, negative delta means Run B is better.

---

## Summary

<2–3 sentences: which configuration performed better overall, in which dimensions,
and any notable tradeoffs (e.g., higher throughput at the cost of tail latency).>
```

### 3.3 Announce Completion

Tell the user where everything is saved:

```
Comparison complete. Results are in: $COMPARISON_DIR/
  ├── comparison_report.md     ← side-by-side comparison (displayed above)
  ├── run-a/
  │   ├── run_state.json       ← Run A config + extracted metrics
  │   └── results/             ← raw benchmark output
  └── run-b/
      ├── run_state.json       ← Run B config + extracted metrics
      └── results/             ← raw benchmark output
```

---

## Key Rules

- **Runs are consecutive** — complete all of Phase 1 before starting Phase 2.
- **Save state before teardown** — `run_state.json` must be written in Step 1.3 / 2.3, before any resources are removed.
- **Minimal teardown between runs** — Helm releases and routing only; PVCs and namespace are preserved.
- **Same namespace for both runs** — both runs deploy into the same namespace; the deploy skill will find it already exists, which is expected.
- **Don't ask about optional teardown steps** — HTTPRoutes/Gateways are always removed between runs (to prevent conflicts), PVCs and namespace are always kept. These are not user choices in this workflow.
