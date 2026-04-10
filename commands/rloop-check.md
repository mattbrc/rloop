# rloop-check — Preflight Validation

Run a preflight check on the rloop configuration before starting the loop.
Validates that everything is properly set up and the loop will work.

## Your Task

Run each check below in order. Track results as PASS, WARN, or FAIL.
At the end, print a summary report.

### 1. Config File

- Check that `rloop.config.json` exists in the current directory
  - FAIL if missing: `"rloop.config.json not found. Run /rloop-init first."`
- Read the file and verify it's valid JSON
  - FAIL if invalid: `"rloop.config.json is not valid JSON."`
- Check that all fields are recognized (warn on unknown fields — could be typos):
  Known fields: `src_dir`, `test_prompt`, `optimize`, `target_metric`, `max_iterations`,
  `max_consecutive_rejections`, `allowed_categories`, `allowed_languages`, `focus_phase`,
  `auto_commit`, `auto_push`, `branch_per_experiment`, `branch_prefix`
  - WARN for each unknown field: `"Unknown config field: '<field>'. Typo?"`
- Validate field values:
  - `optimize` must be `"minimize"` or `"maximize"` → FAIL if not
  - `max_iterations` must be null or a positive integer → FAIL if not
  - `target_metric` must be null or a number → FAIL if not
  - `max_consecutive_rejections` must be a positive integer → FAIL if not
  - `allowed_categories` must be an array → FAIL if not
  - `allowed_languages` must be an array → FAIL if not
  - `auto_commit` must be a boolean → FAIL if not
  - `auto_push` must be a boolean → FAIL if not
  - If `auto_push` is true and `auto_commit` is false → WARN: `"auto_push has no effect when auto_commit is false"`
  - `branch_per_experiment` must be a boolean → FAIL if not
  - `branch_prefix` must be a string → FAIL if not

### 2. Source Directory

- Check that the `src_dir` directory exists
  - FAIL if missing: `"Source directory '<src_dir>' not found."`
- Check that it contains source files (not empty)
  - WARN if empty: `"Source directory '<src_dir>' is empty."`
- List the languages detected (look for .cs, .go, .rs, .py, .ts, .js, etc.)
  - If `allowed_languages` is set, check that at least one detected language matches
  - WARN if no match: `"Source is <detected> but allowed_languages only permits <allowed>."`

### 3. Test Prompt

- Check that the test prompt file exists (path from `test_prompt` config field, default `test_prompt.md`)
  - FAIL if missing: `"Test prompt '<path>' not found."`
- Read the file and check it's not empty
  - FAIL if empty: `"Test prompt is empty."`
- Check that it contains actionable content:
  - WARN if no code blocks or command references found: `"Test prompt has no code blocks or commands. The eval agent needs specific commands to run."`
  - WARN if no mention of `eval_result.json` or `metric_value`: `"Test prompt doesn't mention eval_result.json or metric_value. The eval agent may not know how to report results."`
  - WARN if no mention of pass/fail criteria: `"Test prompt doesn't describe pass/fail criteria."`

### 4. Git State

- Check that the current directory is a git repo
  - FAIL if not: `"Not a git repository. rloop needs git for commit/rollback."`
- Check for uncommitted changes
  - If `branch_per_experiment` is false:
    - WARN if dirty: `"Working tree has uncommitted changes. rloop uses git reset --hard on failures, which will discard these changes. Commit or stash first."`
  - If `branch_per_experiment` is true:
    - WARN if dirty: `"Working tree has uncommitted changes. Commit or stash before starting — rloop will create branches from the current HEAD."`
- If `branch_per_experiment` is true:
  - Check current branch name and confirm it's not detached HEAD
  - WARN if detached: `"HEAD is detached. rloop needs a branch to return to between experiments."`
- If `auto_push` is true:
  - Check that a remote is configured: `git remote -v`
  - WARN if no remote: `"auto_push is enabled but no git remote is configured."`

### 5. Experiment Log

- If `experiment_log.jsonl` exists:
  - Check it's valid (each line is valid JSON)
  - WARN if any lines are invalid: `"experiment_log.jsonl has invalid entries."`
  - Report current state: how many experiments, how many accepted, current best metric
- If it doesn't exist:
  - PASS: `"No experiment log yet. First run will create it."`

### 6. Experiments Directory

- Check that `experiments/current/` exists (or can be created)
- WARN if `experiments/current/` contains leftover files from a previous run:
  `"experiments/current/ has leftover files. These will be overwritten. Consider archiving them."`

### 7. Build Smoke Test (Quick)

- Try to identify the build system in `src_dir`:
  - Look for .csproj, go.mod, Cargo.toml, package.json, Makefile, pyproject.toml, etc.
  - PASS if found: `"Build system detected: <type>"`
  - WARN if not found: `"No recognized build system in '<src_dir>'. Make sure your test_prompt.md includes build instructions."`

### 8. Permissions

- Check if `.claude/settings.json` exists in the project root
  - If missing: WARN: `"No permissions configured. rloop will prompt for every action. Run /rloop-init to set up permissions, or create .claude/settings.json manually."`
  - If present, read it and check:
    - Are `Read`, `Write`, `Edit`, `Glob`, `Grep` in the allow list? WARN if any missing.
    - Are `Bash(git *)` and `Bash(mkdir *)` allowed? WARN if missing.
    - Are build commands for the detected language allowed? (e.g. `Bash(dotnet *)` for C#)
      WARN if missing: `"Build command 'dotnet' not in permissions. The loop will prompt for approval."`
    - Scan `test_prompt.md` for commands that may need permissions and check those too.
  - PASS if all needed permissions are configured: `"Permissions configured for autonomous operation."`

## Report

Print the summary report:

```
═══════════════════════════════════════════════════════════════
  rloop Preflight Check
═══════════════════════════════════════════════════════════════

  Config file          PASS   Valid JSON, all fields recognized
  Source directory      PASS   src/ — C# (.csproj detected)
  Test prompt          PASS   test_prompt.md — 4 steps, has commands
  Git state            WARN   Working tree has uncommitted changes
  Experiment log       PASS   No log yet, first run will create it
  Experiments dir      PASS   experiments/current/ ready
  Build system         PASS   Detected: dotnet (.csproj)
  Permissions          PASS   Configured for autonomous operation

  ─────────────────────────────────────────────────────────────
  Result: 6 passed, 1 warning, 0 failed

  Warnings:
    Git state: Working tree has uncommitted changes.
    rloop uses git reset --hard on failures, which will
    discard these changes. Commit or stash first.

═══════════════════════════════════════════════════════════════
```

If there are any FAILs:
```
  Result: BLOCKED — 1 failure must be fixed before running /rloop
```

If all PASS (with or without warnings):
```
  Ready to go. Run /rloop to start the loop.
```

Keep the report concise. Only show the Warnings/Failures detail section if there are any.
