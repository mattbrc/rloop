# rloop

Autonomous research loop for code optimization. AI agents iteratively improve your codebase
against a metric you define, while preserving correctness.

Four-phase loop: **Research** (find the bottleneck) → **Plan** (design the fix) →
**Build** (implement it) → **Evaluate** (test and measure). Accepts improvements,
rejects regressions, stops when it hits your target or gets stuck.

## Install

### With Claude Code

Tell Claude Code:

> Fetch the rloop commands from https://github.com/mattbrc/rloop and install them to ~/.claude/commands/

### Or manually (curl)

```bash
mkdir -p ~/.claude/commands
curl -sL https://raw.githubusercontent.com/mattbrc/rloop/main/commands/rloop.md -o ~/.claude/commands/rloop.md
curl -sL https://raw.githubusercontent.com/mattbrc/rloop/main/commands/rloop-init.md -o ~/.claude/commands/rloop-init.md
curl -sL https://raw.githubusercontent.com/mattbrc/rloop/main/commands/rloop-check.md -o ~/.claude/commands/rloop-check.md
curl -sL https://raw.githubusercontent.com/mattbrc/rloop/main/commands/rloop-update.md -o ~/.claude/commands/rloop-update.md
```

### If curl is blocked (clone + copy)

```bash
git clone https://github.com/mattbrc/rloop.git /tmp/rloop
cp /tmp/rloop/commands/*.md ~/.claude/commands/
rm -rf /tmp/rloop
```

### Update

```
/rloop-update
```

Fetches the latest command files from GitHub, shows what changed, and installs them.

### Verify

In any project, type `/rloop-init` in Claude Code. If it responds, you're set.

## Commands

| Command | Purpose |
|---------|---------|
| `/rloop-init` | Interactive setup — explores your repo, asks questions, generates config + test prompt + permissions |
| `/rloop-check` | Preflight validation — checks config, source dir, test prompt, git state, permissions, lock file |
| `/rloop` | Run the loop |
| `/rloop-update` | Self-update from GitHub |

## Usage

### 1. Initialize in your project

```
/rloop-init
```

This will:
- Explore your repo to understand the stack
- Ask what you're optimizing and how to test it
- Generate `rloop.config.json`, `test_prompt.md`, and `experiments/` directory
- Configure Claude Code permissions for autonomous operation
- Walk you through every config option

### 2. Preflight check (recommended)

```
/rloop-check
```

Validates 9 areas before you start: config syntax, source directory, test prompt quality,
lock file, git state, experiment log, experiments directory, build system, and permissions.
Reports PASS/WARN/FAIL with actionable messages.

### 3. Run the loop

```
/rloop
```

### 4. Tune as needed

Edit `rloop.config.json` anytime — the orchestrator re-reads it at the start of each
iteration, so changes take effect mid-loop.

## What goes in your project

After `/rloop-init`, your project gets:

| File | Purpose | You edit it? |
|------|---------|-------------|
| `rloop.config.json` | Loop settings and constraints | Yes |
| `test_prompt.md` | Your test procedure (eval agent follows this) | Yes |
| `.claude/settings.json` | Permissions for autonomous operation | Optional |
| `experiment_log.jsonl` | History of every experiment (created during loop) | No (append-only) |
| `experiments/` | Artifacts from each iteration (created during loop) | No |

The framework itself lives in `~/.claude/commands/` — invisible to your repo.

## How it works

```
/rloop
  │
  ├── Setup
  │   ├── Acquire lock file (prevents concurrent runs)
  │   ├── Create safety snapshot (git tag)
  │   └── Start keep-awake (caffeinate/systemd-inhibit)
  │
  ├── For each iteration:
  │   │
  │   ├── 1. Check stopping conditions
  │   │      Iteration limit? Target achieved? Stuck?
  │   │
  │   ├── 2. Re-read config (picks up mid-loop changes)
  │   │
  │   ├── 3. Research agent (skippable via mode)
  │   │      Reads experiment log + source code
  │   │      Identifies highest-impact optimization
  │   │
  │   ├── 4. Plan agent (skippable via mode)
  │   │      Designs concrete implementation
  │   │
  │   ├── 5. Create experiment branch (if configured)
  │   │
  │   ├── 6. Build agent (skippable via mode)
  │   │      Implements the changes
  │   │
  │   ├── 7. Evaluate agent
  │   │      Follows YOUR test_prompt.md
  │   │      Runs tests, analyzes logs, extracts metric
  │   │
  │   ├── 8. Accept or reject
  │   │      Tests passed + metric improved → accept + commit
  │   │      Tests passed + metric worse   → reject + rollback
  │   │      Tests failed                  → fail + rollback
  │   │
  │   └── 9. Archive artifacts
  │
  └── Cleanup
      ├── Print session summary (table of all experiments + token usage)
      ├── Release lock file
      └── Stop keep-awake
```

## How experiments are accepted or rejected

Every experiment must clear **two gates** to be accepted:

**Gate 1 — Qualitative (test_prompt.md)**

The eval agent follows your test prompt and uses its judgment to decide `passed: true/false`.
This is where you define what "correct" means for your project:

- Did all commands complete without errors?
- Do the logs show expected output patterns?
- Are there any warnings about data corruption, missing records, regressions?
- Does the output look reasonable?

The agent can analyze messy log output, interpret ambiguous results, and catch subtle
issues that a simple script would miss. If anything looks wrong, it sets `passed: false`
regardless of the metric.

**Gate 2 — Quantitative (rloop.config.json)**

The orchestrator compares `metric_value` from the eval result against the previous best.
If `optimize` is `"minimize"`, the metric must go down. If `"maximize"`, it must go up.
If the metric didn't improve, the experiment is rejected even though tests passed.

**Both gates must pass.** An experiment with great metrics but log errors → failed.
An experiment with clean logs but worse metrics → rejected. Only clean tests AND
an improved metric → accepted.

## Safety features

- **Safety snapshot** — git tag created before the first iteration. Always there to return to.
- **Lock file** — `.rloop.lock` prevents concurrent runs. Detects stale locks from crashed sessions.
- **Iteration timeout** — aborts an iteration if it exceeds `iteration_timeout_min` (checked between phases, default 2 hours).
- **Consecutive rejection pause** — after N failures in a row, stops and asks the user for guidance instead of burning tokens.
- **Branch isolation** — optional `branch_per_experiment` keeps each experiment on its own branch. Failed branches are deleted, accepted ones are kept.
- **Keep-awake** — auto-detects macOS/Windows/Linux and prevents machine sleep during long runs.
- **Context management** — experiment logs over 20 entries are automatically summarized to keep agent context lean.
- **Token tracking** — tracks input/output tokens per iteration and cumulative. Shown in session summary.

## Phase modes

Control which phases run via the `mode` config:

| Mode | Phases | Use case |
|------|--------|----------|
| `"full"` | Research → Plan → Build → Evaluate | Default — fully autonomous |
| `"plan+build+eval"` | Plan → Build → Evaluate | You provide the research (`experiments/current/research.md`) |
| `"build+eval"` | Build → Evaluate | You provide the plan (`experiments/current/plan.md`) |
| `"eval-only"` | Evaluate | Test current code state (great for validating your test_prompt) |

## Example test_prompt.md

### Simple (single command, runtime optimization)

```markdown
# Test Procedure

## Steps

### 1. Build
Run: `go build -o bin/myapp ./cmd/myapp`
If the build fails, set passed=false.

### 2. Run benchmark
Run: `./bin/myapp --benchmark --input testdata/large.json`
The last line of output contains: `Elapsed: 12.345s`
Parse the number as the metric value.

### 3. Write eval_result.json
metric_value = elapsed seconds
passed = build succeeded and benchmark completed without errors
```

### Complex (multi-step, accuracy optimization)

```markdown
# Test Procedure

## Steps

### 1. Build
Run: `dotnet build src/MyProject -c Release`
If build fails, set passed=false immediately.

### 2. Run ingestion
Run: `dotnet run --project src/MyProject -- ingest --input data/test.json`
Check log output for ERROR lines. If any, fail.

### 3. Run processing
Run: `dotnet run --project src/MyProject -- process`
Check for "Processing complete" in output.

### 4. Run comparison
Run: `dotnet run --project src/MyProject -- compare --previous`
Parse output for: `Accuracy: 96.2%`
The percentage is the metric value.

### 5. Pass/fail criteria
All steps must complete without ERROR lines.
Accuracy must be >= 90%.

### 6. Write eval_result.json
metric_value = accuracy percentage (e.g. 96.2)
passed = true only if all criteria met
notes = include comparison output and any warnings
```

## Configuration reference

| Field | Default | Description |
|-------|---------|-------------|
| `src_dir` | `"src"` | Source code directory agents modify |
| `test_prompt` | `"test_prompt.md"` | Path to test procedure |
| `optimize` | `"minimize"` | `"minimize"` or `"maximize"` |
| `target_metric` | `null` | Stop when metric reaches this |
| `max_iterations` | `null` | Stop after N iterations |
| `max_consecutive_rejections` | `3` | Pause after N failures in a row |
| `iteration_timeout_min` | `120` | Abort iteration after N minutes (checked between phases) |
| `mode` | `"full"` | `"full"`, `"plan+build+eval"`, `"build+eval"`, `"eval-only"` |
| `allowed_categories` | `[]` | Restrict optimization types |
| `allowed_languages` | `[]` | Restrict languages |
| `focus_phase` | `null` | Prioritize an area |
| `auto_commit` | `true` | Auto-commit accepted changes |
| `auto_push` | `false` | Push after committing |
| `branch_per_experiment` | `false` | Isolate each experiment on its own branch |
| `branch_prefix` | `"experiment"` | Branch naming (e.g. `experiment/1`) |

All fields are optional — defaults apply when omitted or null. Config is re-read each
iteration, so you can change settings mid-loop.

## Optimization categories

The research agent classifies each optimization into a category. Use `allowed_categories`
in config to restrict which ones the agent explores. These are not exhaustive — the agent
can define its own if none fit.

**Performance**
- **caching** — memoization, precomputation, lookup tables
- **parallelism** — concurrent processing, goroutines, threads
- **algorithm_complexity** — better big-O, smarter algorithms
- **data_structure** — hash maps vs lists, adjacency lists, indexes
- **allocation_reduction** — pooling, in-place mutation, fewer copies
- **lazy_evaluation** — defer work until needed
- **batch_processing** — vectorize, bulk operations
- **memory_optimization** — data layout, cache locality, memory-mapped I/O
- **profiling_guided** — optimizations driven by profiler or flame graph data

**Architecture**
- **architectural_restructuring** — reorder pipeline, early termination
- **language_rewrite** — rewrite in a faster language
- **io_optimization** — faster parsing, streaming, buffering
- **serialization** — schema changes, format optimization, compression
- **dependency_optimization** — replacing heavy libs, removing unnecessary deps
- **code_simplification** — removing dead code, reducing complexity

**Correctness & Reliability**
- **correctness_fix** — bug fixes that improve accuracy or pass rate
- **error_handling** — retry logic, fallback strategies, recovery paths
- **concurrency_safety** — fixing races, deadlocks, contention

**Tuning**
- **configuration_tuning** — adjusting thresholds, parameters, hyperparameters
- **model_tuning** — hyperparameters, feature selection, training strategies
