# rloop-init — Initialize Research Loop

Set up rloop in the current project. This scaffolds the configuration and test prompt
so you can run `/rloop` to start the autonomous optimization loop.

## Your Task

### 1. Check for existing setup

If `rloop.config.json` already exists, ask the user:
> "rloop is already initialized in this project. Do you want to reconfigure it?"

If they say no, stop. If yes, continue (you'll overwrite the existing files).

### 2. Explore the project

Before asking anything, explore the repo to understand it. Check:
- Language/framework (look for .csproj, go.mod, Cargo.toml, package.json, pyproject.toml, Makefile, etc.)
- Build system (how to compile/build)
- Test infrastructure (test directories, test scripts, CI config)
- What the project does (README, main entry points)
- Directory structure (where source code lives)

Summarize what you found in 3-5 bullet points so the user can confirm you understand
the project correctly.

### 3. Ask the user these questions

Ask all of these in a **single message**. Be concise.

1. **What are you optimizing?** (runtime, accuracy, memory usage, binary size, test pass rate, latency, throughput, etc.)
2. **Should the metric be minimized or maximized?** (e.g., runtime = minimize, accuracy = maximize)
3. **What directory contains the source code agents should modify?** Suggest what you found in step 2 as the default.
4. **How do you test/evaluate a change?** Ask them to walk you through:
   - What commands to run (in order — build, test, benchmark, etc.)
   - What to look for in the output (success messages, error patterns, metric values)
   - Where/how the metric value appears in the output
   - Beyond the metric number, what qualitative checks should pass?
     (e.g., no error lines in logs, no data corruption warnings, no regressions
     in specific categories)

   Explain the two-gate system:
   > "rloop uses two gates to accept an experiment. First, the eval agent follows your
   > test_prompt.md and uses its judgment to decide pass/fail — this is where you define
   > qualitative criteria like 'no errors in logs' or 'no regressions in any category.'
   > Second, the orchestrator checks whether the metric improved. Both must pass.
   > So even if accuracy goes up, the experiment fails if the agent spotted data corruption
   > in the logs. This gives you precise control over what 'good' means."

5. **Any constraints?** (languages to restrict to, optimization categories to focus on, iteration limit, target metric value)

### 4. Generate the files

Based on their answers, generate these files:

#### `rloop.config.json`

```json
{
  "src_dir": "<source directory>",
  "test_prompt": "test_prompt.md",
  "optimize": "<minimize or maximize>",
  "target_metric": null,
  "max_iterations": null,
  "max_consecutive_rejections": 3,
  "iteration_timeout_min": 120,
  "allowed_categories": [],
  "allowed_languages": [],
  "focus_phase": null,
  "auto_commit": true,
  "auto_push": false,
  "branch_per_experiment": false,
  "branch_prefix": "experiment",
  "mode": "full"
}
```

Fill in `src_dir`, `optimize`, and any constraints the user specified.
Set `target_metric`, `max_iterations`, `allowed_categories`, `allowed_languages`
based on their answers (leave as defaults if not specified).

#### `test_prompt.md`

This is the most important file. It must be specific enough for an AI agent to follow
literally. Structure it as:

```markdown
# Test Procedure — <Project Name>

## Overview
<One sentence: what we're testing and what the metric is>

## Steps

### 1. <First step name>
**Run:**
\`\`\`bash
<exact command>
\`\`\`
**Check:** <what to look for — success patterns, error patterns>
**If failed:** <what to do — set passed=false, note the error>

### 2. <Next step>
...

### N. Extract the metric
<How to find and parse the metric value from the output>

### N+1. Write eval_result.json
- `metric_value` = <what to use>
- `passed` = <all conditions that must be true>
- `notes` = <what to include — key log lines, metric context, etc.>
```

Guidelines for writing a good test prompt:
- Use exact commands (not pseudocode)
- Specify exact patterns to grep/look for in output
- Be explicit about what constitutes failure at each step
- Tell the agent exactly how to parse the metric (regex, line pattern, JSON field, etc.)
- Include the full pass/fail criteria in one place
- If multiple commands produce output that needs analysis, tell the agent what to save
  and reference between steps

#### `experiments/` directory

Create the directory structure:
```bash
mkdir -p experiments/current experiments/archive
```

#### `.gitignore` additions

Check if `.gitignore` exists. If it does, read it. If these entries are missing, add them:
```
experiments/current/
.rloop.lock
```
If `.gitignore` doesn't exist, create it with these entries.
Tell the user what was added.

### 5. Show the summary

After generating all files, show:

```
rloop initialized:

  rloop.config.json  — settings (optimize: <direction>, src: <dir>)
  test_prompt.md     — test procedure (<N> steps)
  experiments/       — artifact storage

To start the loop:  /rloop
To reconfigure:     edit rloop.config.json or test_prompt.md
```

Then walk the user through how to control the loop. Show them their actual config values
and explain what each one does in plain language:

```
How to control the loop (edit rloop.config.json anytime):

  Iterations:
    "max_iterations": null       ← runs forever until target or you stop it
    "max_iterations": 3          ← runs exactly 3 iterations then stops
    "max_iterations": 1          ← single-shot: try one optimization and stop

  Target:
    "target_metric": null        ← no target, just keep improving
    "target_metric": 10.0        ← stop when metric hits 10.0
                                   (respects "optimize" direction)

  Safety net:
    "max_consecutive_rejections": 3  ← pauses and asks you after 3 failures in a row
    "iteration_timeout_min": 120     ← abort an iteration if it takes longer than 2 hours
                                       (checked between phases, not during — set with headroom)

  Mode:
    "mode": "full"             ← (default) Research → Plan → Build → Evaluate
    "mode": "plan+build+eval"  ← you provide the research, agent plans + builds
    "mode": "build+eval"       ← you provide the plan, agent just builds + evaluates
    "mode": "eval-only"        ← just test current code (useful for testing your test_prompt)

  Focus:
    "focus_phase": null              ← agent chooses what to optimize
    "focus_phase": "data_processing" ← steer toward a specific area
    "allowed_categories": []         ← no restrictions on approach
    "allowed_categories": ["parallelism", "caching"]  ← only these strategies

  Git:
    "auto_commit": true              ← auto-commits accepted improvements
    "auto_commit": false             ← tells you and lets you decide (no commit, no push)
    "auto_push": false               ← (default) commits stay local
    "auto_push": true                ← pushes to remote after each commit
                                       (only works with auto_commit: true)

  Branching:
    "branch_per_experiment": false   ← (default) all work on current branch
    "branch_per_experiment": true    ← each experiment gets its own branch
    "branch_prefix": "experiment"    ← branches named experiment/1, experiment/2, ...
                                       accepted branches are kept, failed ones are deleted
                                       always returns to your starting branch between experiments

You can change these between runs or even mid-loop (the orchestrator
re-reads the config at the start of each iteration).
```

### 6. Configure permissions

rloop runs autonomously with subagents that read files, edit code, and run shell commands.
Without pre-configured permissions, Claude Code will prompt for approval on every action,
which defeats the purpose.

Ask the user:
> "rloop needs permissions to read/write files, run build and test commands, and manage git.
> Want me to configure these permissions for this project so the loop can run autonomously?"

If they say yes, check if `.claude/settings.json` exists in the project root.
If it exists, read it first and merge the permissions. If not, create it.

Determine which build/test commands are needed based on the detected language and
the commands in `test_prompt.md`. Then write `.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Write",
      "Edit",
      "Glob",
      "Grep",
      "Bash(git *)",
      "Bash(mkdir *)",
      "Bash(cp *)",
      "Bash(rm experiments/*)",
      "Bash(caffeinate *)",
      "Bash(kill *)"
    ]
  }
}
```

Then add language-specific permissions based on what you detected:
- C#/.NET: add `"Bash(dotnet *)"`
- Go: add `"Bash(go *)"`
- Rust: add `"Bash(cargo *)"`
- Python: add `"Bash(python *)"`  and `"Bash(python3 *)"`
- Node.js: add `"Bash(npm *)"` and `"Bash(node *)"`
- Makefile: add `"Bash(make *)"`

Also scan `test_prompt.md` for any other commands the eval agent will need to run
and add those too.

If they say no, tell them:
> "No problem. Claude Code will ask for permission as it goes. You can choose
> 'always allow' when prompted to avoid repeated approvals, or configure
> permissions later in `.claude/settings.json`."

Ask if they want to adjust anything before they start.
