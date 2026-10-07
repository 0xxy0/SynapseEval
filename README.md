# SynapseEval

**SynapseEval** is an evaluation and analysis platform for measuring the real-world code-generation capabilities of local and self-hosted language models.

It combines controlled benchmark execution, automated repair loops, isolated code execution, model cascading, prompt-variation analysis, contamination testing, hardware profiling, and interactive reporting into a single evaluation workflow.

The system is designed to answer a more useful question than _“What is this model's benchmark score?”_:

> **How reliably, quickly, and efficiently can a model produce working code under realistic execution constraints?**

---

## Why SynapseEval?

Traditional code-generation benchmarks usually reduce model performance to a single pass-rate number. That number is useful, but it hides much of what happens during an actual coding workflow.

SynapseEval records the complete evaluation process.

It can measure:

- Whether the initial solution works.
- How many correction attempts are required.
- Whether compiler, linter, or test feedback helps the model recover.
- How response time changes across attempts.
- How performance changes with different problem descriptions.
- Whether benchmark familiarity may be influencing results.
- How much GPU memory a model requires.
- Whether a smaller model can solve most tasks before a larger model is needed.
- How much accuracy can be achieved within a fixed latency budget.

The result is a multidimensional view of model performance rather than a single benchmark percentage.

---

# Core Capabilities

### Automated Code Evaluation

Run coding tasks against any model accessible through the configured model-serving interface and automatically execute the generated solutions against the benchmark test suite.

### Sandboxed Execution

Generated programs are executed inside isolated Docker environments, keeping evaluation workloads separated from the host system.

### Iterative Self-Repair

When a generated solution fails, SynapseEval can feed the resulting error back to the model and request a correction. Multiple repair attempts can be performed automatically.

### Local Code Cleanup

Generated Python is passed through formatting and linting before execution. Automatically fixable issues are corrected without consuming another model request.

### Reasoning-Model Support

Models capable of extended reasoning can be evaluated using thinking-enabled inference.

### Adaptive Context Management

The context window can grow automatically when additional reasoning or error information is required.

### Multi-Model Cascades

Tasks do not necessarily need to be solved by the largest model.

SynapseEval can begin with an inexpensive, fast model and escalate only unsuccessful tasks to increasingly capable models.

### Cascade Optimization

Given benchmark measurements, the optimizer searches for an efficient sequence of models and iteration budgets subject to latency and VRAM constraints.

### Hardware Profiling

Measure model memory consumption and generation throughput across multiple context sizes.

### Prompt Sensitivity Analysis

Evaluate identical programming tasks using alternative descriptions to determine how strongly performance depends on wording.

### Contamination Analysis

Generate semantically equivalent but structurally different tasks to investigate whether benchmark familiarity may be contributing to model performance.

### Interactive Analytics

Produce HTML reports containing model comparisons, task-level analysis, error distributions, latency measurements, yield curves, cascade performance, and hardware information.

### Persistent Result Caching

Evaluation results and generated artifacts can be cached to prevent unnecessary repeated inference.

---

# Architecture

SynapseEval is organized around a staged evaluation pipeline:

```text
                    ┌──────────────────────┐
                    │   Benchmark Dataset  │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   Prompt Construction│
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │      Model Inference │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Formatting / Linting │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Isolated Execution  │
                    └──────────┬───────────┘
                               │
                    ┌──────────┴───────────┐
                    │                      │
                   PASS                  FAIL
                    │                      │
                    ▼                      ▼
               Record Result       Generate Repair
                                           │
                                           ▼
                                  Retry / Escalate
                                           │
                                           ▼
                                    Final Result
````

 For cascade evaluations, the model-selection layer sits above this pipeline and determines which model should receive each unresolved task.

---

 # Evaluation Workflow

 Every benchmark problem moves through a controlled sequence of processing stages.

 ### 1\. Prompt Preparation

 The benchmark specification is converted into the model-facing prompt.

 ### 2\. Initial Generation

 The selected model receives the problem and produces a candidate implementation.

 ### 3\. Static Cleanup

 The generated source is formatted and automatically corrected where possible.

 ### 4\. Lint Validation

 Remaining static-analysis errors can be returned to the model for correction.

 ### 5\. Sandboxed Execution

 The candidate implementation is executed together with its tests inside the evaluation sandbox.

 ### 6\. Failure Diagnosis

 If execution fails, the resulting error information becomes feedback for the next attempt.

 ### 7\. Automated Repair

 The model is asked to revise its previous implementation rather than starting from an unrelated blank response.

 ### 8\. Strategy Escalation

 When repeated repairs fail, a more aggressive repair prompt can request a substantially different implementation strategy.

 ### 9\. Final Classification

 The task is ultimately recorded as successful or unsuccessful together with timing, iteration, and diagnostic information.

---

 # Self-Correction Engine

 SynapseEval does more than test the first answer produced by a model.

 A failed solution can enter a repair cycle:

```
Initial Solution
      │
      ▼
  Static Checks
      │
      ▼
    Execute
      │
 ┌────┴────┐
 │         │
PASS      FAIL
 │         │
 ▼         ▼
Done    Diagnose
           │
           ▼
       Model Repair
           │
           ▼
       Re-format
           │
           ▼
        Execute
           │
           └──────► repeat
```

 The repair system retains previous attempts and error information to reduce repetitive corrections.

 Sampling temperature can also increase progressively between repair attempts. The default behavior begins conservatively and becomes more exploratory as the model continues to fail.

 This provides a practical mechanism for escaping repeated incorrect solutions.

---

 # Cascade Evaluation

 Large models are often capable of solving more problems, but using the largest model for every task can be unnecessarily expensive and slow.

 SynapseEval therefore supports **tiered evaluation**.

 A typical cascade might look like:

```
                    ┌───────────────┐
                    │   Task Input  │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │   Tier 1      │
                    │ Fast / Small  │
                    └───────┬───────┘
                            │
                     ┌──────┴──────┐
                     │             │
                   PASS          FAIL
                     │             │
                     ▼             ▼
                   Done       ┌───────────────┐
                              │   Tier 2      │
                              │ Medium Model  │
                              └───────┬───────┘
                                      │
                               ┌──────┴──────┐
                               │             │
                             PASS          FAIL
                               │             │
                               ▼             ▼
                             Done      ┌───────────────┐
                                      │   Tier 3      │
                                      │ Large Model   │
                                      └───────┬───────┘
                                              │
                                              ▼
                                           Result
```

 The objective is to achieve the desired benchmark yield while minimizing total inference time and resource consumption.

---

 # Cascade Optimization

 The optimizer can derive a model sequence from previously collected evaluation data.

 It considers factors including:

 - Success probability.
- Average generation time.
- Model loading overhead.
- VRAM requirements.
- Number of repair attempts.
- Per-tier iteration limits.
- Probability of succeeding before a specified latency threshold.

 For example:

```
uv run python scripts/optimize_cascade.py \
    --target-time 15 \
    --vram-budget 28000
```

 This allows the evaluation strategy to be optimized around a particular hardware environment or latency requirement.

 Instead of asking:

 > Which model has the highest pass rate?

 you can ask:

 > Which sequence of models produces the most successful solutions within my hardware and latency budget?

---

 # Yield-Oriented Evaluation

 A benchmark pass rate alone does not describe the experience of using a model.

 SynapseEval therefore supports **yield analysis**.

 For a selected patience threshold `τ`, yield represents the probability that a task is successfully completed within the allowed time budget.

 Conceptually:

```
Yield(τ) =
    successful tasks completed within τ
    ----------------------------------
              total tasks
```

 This makes it possible to compare models that have different accuracy/latency trade-offs.

 A slower model may eventually outperform a smaller model in raw accuracy while still being inferior when the acceptable response-time budget is tight.

---

 # Benchmark Variants

 SynapseEval can evaluate more than the original benchmark representation.

 ## Prompt Rewrites

 The same programming problems can be rewritten by another language model while preserving:

 - Function signatures.
- Expected behavior.
- Test cases.
- Underlying programming requirements.

 Example:

```
Original task
     │
     ├──────────────► Model evaluation
     │
     ▼
Rewritten task
     │
     └──────────────► Model evaluation
```

 Comparing the two results provides a measure of **prompt sensitivity**.

 A large performance difference indicates that the model may depend significantly on how the problem is worded.

 ### Existing Rewrite Datasets

 Example datasets include:

```
data/
├── humaneval-rewritten-gpt-oss20b.jsonl.gz
├── humaneval-rewritten-ministral-314b.jsonl.gz
└── humaneval-rewritten-qwen34b.jsonl.gz
```

 ### Create a New Rewrite Set

```
uv run python scripts/generate_rewrites.py --model qwen3:14b
```

 The generation process uses persistent caching so interrupted jobs can continue without regenerating completed rewrites.

 ### Evaluate a Rewritten Dataset

```
SynapseEval eval qwen3:4b \
    --dataset humaneval-rewritten-qwen3-14b
```

 The rewrite results are available for dedicated analysis without automatically changing the primary benchmark aggregate.

---

 # Contamination Analysis

 Benchmark contamination can make a model appear more capable than it actually is on unseen programming problems.

 SynapseEval includes an experimental mechanism for investigating this effect.

 Instead of evaluating only the original task, the system can create an **isomorphic variant** that preserves the underlying computational structure while changing its surface representation.

 The transformation can include:

 - Domain changes.
- Renamed concepts.
- New input examples.
- Newly generated descriptions.
- Input/output pairs generated through a sandbox oracle.

 The resulting task is structurally related to the original while being substantially different in presentation.

 ## Generate Variants

```
uv run python -m SynapseEval.contamination.generate_variants
```

 For a smaller development run:

```
uv run python -m SynapseEval.contamination.generate_variants \
    --limit 20
```

 The generated dataset is stored as:

```
data/variants.jsonl.gz
```

 ## Evaluate Variants

```
SynapseEval eval qwen3:4b \
    --dataset variants
```

 ## Calculate Contamination Indicators

```
uv run python -m SynapseEval.contamination.contamination_report \
    output/results_original.jsonl \
    output/results_variant.jsonl
```

 The analysis compares changes in success and failure rates between original and transformed tasks.

 A positive asymmetry can indicate that benchmark familiarity may be contributing to the model's performance.

---

 # Hardware Benchmarking

 Model quality is only one part of deployment suitability.

 SynapseEval can profile model resource usage across multiple context sizes:

```
uv run python scripts/vram_bench.py \
    --models "qwen3:4b,llama3.1:8b" \
    --contexts "2048,4096,8192,16384"
```

 The benchmark records information such as:

 - VRAM consumption.
- Generation throughput.
- Context-size effects.
- Model-specific performance characteristics.

 Results are stored under:

```
output/vram_bench_*.json
```

 These measurements can subsequently be consumed by the cascade optimizer and model-selection reports.

---

 # Reports

 Running:

```
SynapseEval report
```

 generates an interactive HTML analysis suite.

 The reporting system is designed to make large evaluation runs easier to inspect without manually processing JSONL output.

 ## Report Sections

 ### Overview

 Provides a high-level comparison of evaluated models.

 ### Model Analysis

 Shows detailed results for individual models, including attempt-level behavior.

 ### Task Explorer

 Breaks down performance by individual programming problem and exposes signals associated with task difficulty.

 ### Context Analysis

 Examines context growth, token usage, and context-related failure behavior.

 ### Cascade Analysis

 Shows how tasks move between model tiers and where each tier contributes to the final result.

 ### Yield Analysis

 Visualizes the probability of successful completion as the permitted response time changes.

 ### Model Selector

 Allows models to be filtered according to resource constraints and urgency requirements.

 ### Hardware Profile

 Documents the evaluation environment, including relevant CPU, GPU, container, Python, and model-serving information.

---

 # Visual Analytics

 Generated reports include interactive visualizations such as:

 - Model pass-rate comparisons.
- Error-category distributions.
- Response-time distributions.
- Generated-token distributions.
- Performance envelopes.
- Yield curves.
- Rewrite comparisons.
- Attempt-by-attempt progression.
- Cascade contribution analysis.
- Context-window behavior.
- Task difficulty distributions.

 The goal is to make it possible to understand _why_ models differ rather than simply observing that they differ.

---

 # Command-Line Interface

 ## Evaluate One Model

```
SynapseEval eval qwen3:4b
```

 ## Evaluate Several Models

```
SynapseEval eval "qwen3:4b,gemma4:26b"
```

 ## Enable Model Reasoning

```
SynapseEval eval qwen3:4b --think
```

 ## Use Another Inference Server

```
SynapseEval eval qwen3:4b \
    --host http://remote-host:11434
```

 ## Increase Concurrency

```
SynapseEval eval qwen3:4b \
    --workers 4
```

 ## Select a Dataset

```
SynapseEval eval qwen3:4b \
    --dataset humaneval-rewritten-qwen3-14b
```

 ## Specify an Output Location

```
SynapseEval eval qwen3:4b \
    --output-dir results \
    --tag experiment-02
```

 ## Enable Cascading

```
SynapseEval eval qwen3:4b \
    --cascade
```

 ## Reset Cached Results

```
SynapseEval eval qwen3:4b \
    --clear-cache
```

 Evaluation artifacts are written as JSONL records by default.

---

 # Installation

 SynapseEval requires:

 - Python 3.12 or newer.
- Docker.
- A locally available or remotely accessible model-serving endpoint.
- `uv`.

 Install the environment with:

```
uv sync --all-extras --dev
```

---

 # Execution Sandbox

 Generated code should never be treated as trusted host-level software.

 SynapseEval therefore executes benchmark programs inside a dedicated container.

 Build the execution environment with:

```
docker build \
    -t python-sandbox \
    -f Dockerfile .
```

 The sandbox is responsible for running generated implementations and their associated tests in a controlled environment.

---

 # Repository Layout

 The codebase is organized into several functional areas:

```
src/SynapseEval/
│
├── cli.py
├── runner.py
├── agent.py
├── cascade_agent.py
├── cascade_optimizer.py
├── cascade_report.py
│
├── prompts.py
├── sandbox.py
├── pytest_wrapper.py
├── apply_solution.py
├── code_formatter.py
│
├── data.py
├── config.py
├── jsonl_io.py
├── discover_model_stats.py
├── system_profile.py
├── kde.py
├── pocketflow.py
├── autocontext.py
│
├── contamination/
│   ├── generate_variants.py
│   ├── contamination_report.py
│   ├── dataset_diff_report.py
│   ├── oracle.py
│   ├── variant_utils.py
│   └── prompts.py
│
└── report/
    ├── _shared.py
    ├── html_report.py
    ├── html_index.py
    ├── html_model.py
    ├── html_tasks.py
    ├── html_context.py
    ├── html_violin.py
    ├── html_rewrite.py
    ├── html_cascade.py
    ├── html_yield.py
    ├── html_selector.py
    ├── html_setup.py
    ├── yield_score.py
    ├── analyse_difficulty.py
    ├── analyse_envelope.py
    ├── metrics.py
    └── results_parser.py
```

 Supporting utilities live under:

```
scripts/
├── generate_rewrites.py
├── vram_bench.py
├── optimize_cascade.py
├── analyse_bench.py
└── compare_fixharder.py
```

 Automated tests are maintained under:

```
tests/
├── test_agent_nodes.py
├── test_sandbox.py
├── test_pocketflow.py
├── test_pocketflow_async.py
├── test_autocontext.py
├── test_runner_eval.py
├── test_apply_solution.py
├── test_helpers.py
└── contamination/
```

---

 # Internal Agent Model

 The primary evaluation agent is implemented as a node-based workflow.

 Its conceptual state machine is:

```
                 ┌────────────────────┐
                 │ Prepare Task       │
                 └─────────┬──────────┘
                           ▼
                 ┌────────────────────┐
                 │ Generate Solution  │
                 └─────────┬──────────┘
                           ▼
                 ┌────────────────────┐
                 │ Format / Lint      │
                 └─────────┬──────────┘
                           ▼
                 ┌────────────────────┐
                 │ Execute Tests      │
                 └─────────┬──────────┘
                           │
                ┌──────────┴──────────┐
                │                     │
              PASS                   FAIL
                │                     │
                ▼                     ▼
             Complete            Repair Loop
                                      │
                                      ▼
                               Retry Execution
                                      │
                              ┌───────┴───────┐
                              │               │
                            Fixed          Still Failing
                              │               │
                              ▼               ▼
                           Complete       Harder Repair
                                              │
                                              └──► loop
```

 The main processing components include:

 | Component | Responsibility |
| --- | --- |
| `FormatOriginalQuestionNode` | Converts benchmark data into the initial model prompt |
| `GenerateNode` | Performs model inference |
| `RuffFixNode` | Applies local formatting and automatic lint fixes |
| `LintFixNode` | Requests model corrections for unresolved lint issues |
| `ExecuteNode` | Runs the candidate against the sandboxed test suite |
| `FixNode` | Uses execution feedback to repair failed implementations |
| `FixHarderNode` | Requests a substantially different solution strategy |
| `RespondNode` | Finalizes the task result |
| `CascadeGenerateNode` | Performs generation within a selected cascade tier |

---

 # Experimental Design Principles

 SynapseEval is built around several principles.

 ## Measure Execution, Not Just Generation

 A plausible-looking answer is not equivalent to a working program. Evaluation therefore happens against executable tests.

 ## Preserve Diagnostic Information

 A final pass/fail score is insufficient for understanding model behavior. Intermediate attempts, failures, timing, and repair trajectories are retained.

 ## Separate Capability From Latency

 A model that eventually solves everything but requires excessive time has different practical characteristics from a model that solves slightly fewer tasks almost instantly.

 ## Account for Hardware

 Model selection is constrained by available memory and inference throughput. Hardware characteristics are therefore first-class evaluation data.

 ## Test Robustness

 Rewritten prompts and transformed tasks provide additional views of the same underlying capability.

 ## Optimize Systems, Not Isolated Models

 The best deployment configuration may consist of several models working in sequence rather than a single universally preferred model.

---

 # Typical Experiment

 A complete evaluation cycle can look like this:

```
# 1. Prepare the environment
uv sync --all-extras --dev

# 2. Build the execution sandbox
docker build -t python-sandbox -f Dockerfile .

# 3. Evaluate candidate models
SynapseEval eval "qwen3:4b,gemma4:26b" \
    --workers 4

# 4. Run a cascade experiment
SynapseEval eval qwen3:4b \
    --cascade

# 5. Profile hardware characteristics
uv run python scripts/vram_bench.py \
    --models "qwen3:4b,llama3.1:8b" \
    --contexts "2048,4096,8192,16384"

# 6. Optimize model routing
uv run python scripts/optimize_cascade.py \
    --target-time 15 \
    --vram-budget 28000

# 7. Build the analysis suite
SynapseEval report
```

 The resulting reports provide a complete view of accuracy, latency, resource consumption, repair behavior, and model-routing efficiency.

---

 # What SynapseEval Is Designed to Answer

 SynapseEval is particularly useful when comparing questions such as:

 - Which model produces the most correct code?
- Which model reaches a correct solution fastest?
- How often can a small model recover after receiving test feedback?
- When is a larger model actually necessary?
- How much does prompt wording affect performance?
- Does benchmark familiarity appear to influence results?
- How does context length affect memory and throughput?
- What model sequence gives the best accuracy under a fixed latency budget?
- Which model provides the best success rate for a given VRAM limit?
- How does performance evolve across multiple correction attempts?

 These questions turn benchmark evaluation into a more comprehensive engineering measurement system.

---

 # Output

 Evaluation runs produce machine-readable JSONL artifacts suitable for subsequent analysis.

 Typical output structure:

```
output/
├── *.jsonl
├── vram_bench_*.json
└── html/
    ├── index.html
    ├── models/
    ├── tasks/
    ├── cascade/
    ├── context/
    ├── yield/
    └── setup/
```

 The raw evaluation data can be retained independently of the generated HTML reports, allowing reports to be regenerated without repeating model inference.

---

 # Development

 Run the test suite with your preferred Python test runner after installing the development dependencies.

 The test suite covers:

 - Agent-node behavior.
- Sandbox execution.
- Workflow mechanics.
- Asynchronous execution.
- Context management.
- Evaluation orchestration.
- Solution extraction.
- Helper utilities.
- Contamination-analysis components.

 The project is structured so individual evaluation components can be tested independently without requiring a complete benchmark run.

---

 # Design Philosophy

 SynapseEval treats model evaluation as an engineering problem rather than a leaderboard exercise.

 A useful evaluation should capture at least four dimensions:

```
                 MODEL EVALUATION
                        │
        ┌───────────────┼────────────────┐
        │               │                │
    Correctness      Latency          Resources
        │               │                │
        └───────────────┼────────────────┘
                        │
                     Robustness
                        │
                        ▼
                 Practical Utility
```

 The objective is not simply to identify the model with the highest theoretical capability.

 The objective is to understand **which model, configuration, or model cascade is most effective for a particular operational constraint**.

