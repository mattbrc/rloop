# rloop

Autonomous research loop for code optimization. AI agents iteratively improve your codebase
against a metric you define, while preserving correctness.

Four-phase loop: **Research** (find the bottleneck) → **Plan** (design the fix) →
**Build** (implement it) → **Evaluate** (test and measure). Accepts improvements,
rejects regressions, stops when it hits your target or gets stuck.

## Install

### With Claude Code

Tell Claude Code:

> Install rloop from this repo — copy the two command files to ~/.claude/commands/

Or manually:

```bash
git clone https://github.com/fortyau/rloop.git
cp rloop/commands/rloop-init.md ~/.claude/commands/
cp rloop/commands/rloop.md ~/.claude/commands/
```

### Verify

In any project, type `/rloop-init` in Claude Code. If it responds, you're set.

## Usage

### 1. Initialize in your project

```
/rloop-init
```

This will:
- Explore your repo to understand the stack
- Ask what you're optimizing and how to test it
- Generate `rloop.config.json`, `test_prompt.md`, and `experiments/` directory

### 2. Run the loop

```
/rloop
```

The loop runs autonomously:
- Finds optimization opportunities in your code
- Plans and implements changes
- Tests them using your `test_prompt.md` procedure
- Accepts improvements, rejects regressions
- Commits accepted changes (if `auto_commit` is true)
- Stops at your target, iteration limit, or when stuck

### 3. Tune as needed

Edit `rloop.config.json` to change constraints:

```json
{
  "src_dir": "src",
  "test_prompt": "test_prompt.md",
  "optimize": "minimize",
  "target_metric": 10.0,
  "max_iterations": 5,
  "max_consecutive_rejections": 3,
  "allowed_categories": ["parallelism", "algorithm_complexity"],
  "allowed_languages": ["go"],
  "focus_phase": "data_processing",
  "auto_commit": true
}
```

Edit `test_prompt.md` to change how testing works.

## What goes in your project

After `/rloop-init`, your project gets:

| File | Purpose | You edit it? |
|------|---------|-------------|
| `rloop.config.json` | Loop settings and constraints | Yes |
| `test_prompt.md` | How to test your project (agent follows this) | Yes |
| `experiment_log.jsonl` | History of every experiment (created during loop) | No (append-only) |
| `experiments/` | Artifacts from each iteration (created during loop) | No |

## How it works

```
/rloop
  │
  ├── 1. Check stopping conditions (iteration limit, target, stuck?)
  │
  ├── 2. Research agent
  │      Reads experiment log + source code
  │      Identifies highest-impact optimization
  │      Writes: experiments/current/research.md
  │
  ├── 3. Plan agent
  │      Reads research memo
  │      Designs concrete implementation
  │      Writes: experiments/current/plan.md
  │
  ├── 4. Build agent
  │      Reads plan
  │      Implements the changes
  │      Writes: experiments/current/build_report.md
  │
  ├── 5. Evaluate agent
  │      Reads YOUR test_prompt.md
  │      Runs your test procedure
  │      Writes: experiments/current/eval_result.json
  │
  ├── 6. Accept or reject
  │      Metric improved? → commit + accept
  │      Metric worse?    → git reset + reject
  │      Tests failed?    → git reset + fail
  │
  ├── 7. Archive artifacts
  │
  └── 8. Loop back to 1
```

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
| `max_consecutive_rejections` | `3` | Pause after N failures |
| `allowed_categories` | `[]` | Restrict optimization types |
| `allowed_languages` | `[]` | Restrict languages |
| `focus_phase` | `null` | Prioritize an area |
| `auto_commit` | `true` | Auto-commit accepted changes |

## Optimization categories

These are the categories the research agent uses to classify optimization approaches:

- **caching** — memoization, precomputation, lookup tables
- **parallelism** — concurrent processing, goroutines, threads
- **algorithm_complexity** — better big-O, smarter data structures
- **data_structure** — hash maps vs lists, adjacency lists, indexes
- **allocation_reduction** — pooling, in-place mutation, fewer copies
- **lazy_evaluation** — defer work until needed
- **batch_processing** — vectorize, bulk operations
- **architectural_restructuring** — reorder pipeline, early termination
- **language_rewrite** — rewrite in a faster language
- **io_optimization** — faster serialization, streaming, buffering

Use `allowed_categories` in config to restrict which ones the agent explores.
