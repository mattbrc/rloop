# rloop-update — Update rloop Commands

Update rloop to the latest version from GitHub.

## Your Task

### 1. Check current state

List the rloop command files currently installed:
```bash
ls -la ~/.claude/commands/rloop*.md
```

### 2. Download the latest versions

Try fetching via curl first. If curl fails (blocked network, 403, timeout), fall back to git clone.

**Primary method (curl):**
```bash
curl -sL https://raw.githubusercontent.com/mattbrc/rloop/main/commands/rloop.md -o /tmp/rloop.md
curl -sL https://raw.githubusercontent.com/mattbrc/rloop/main/commands/rloop-init.md -o /tmp/rloop-init.md
curl -sL https://raw.githubusercontent.com/mattbrc/rloop/main/commands/rloop-check.md -o /tmp/rloop-check.md
curl -sL https://raw.githubusercontent.com/mattbrc/rloop/main/commands/rloop-update.md -o /tmp/rloop-update.md
```

**Fallback (git clone):**
If any curl command fails or the downloaded files are empty/HTML error pages:
```bash
git clone https://github.com/mattbrc/rloop.git /tmp/rloop-repo
cp /tmp/rloop-repo/commands/rloop.md /tmp/rloop.md
cp /tmp/rloop-repo/commands/rloop-init.md /tmp/rloop-init.md
cp /tmp/rloop-repo/commands/rloop-check.md /tmp/rloop-check.md
cp /tmp/rloop-repo/commands/rloop-update.md /tmp/rloop-update.md
rm -rf /tmp/rloop-repo
```

### 3. Check for differences

For each file, compare the downloaded version against the installed version:
```bash
diff ~/.claude/commands/rloop.md /tmp/rloop.md
diff ~/.claude/commands/rloop-init.md /tmp/rloop-init.md
diff ~/.claude/commands/rloop-check.md /tmp/rloop-check.md
diff ~/.claude/commands/rloop-update.md /tmp/rloop-update.md
```

### 4. Report and install

If no differences found:
```
rloop is already up to date.
```

If there are differences, summarize what changed in plain language (don't dump raw diffs).
Then copy the new files into place:

```bash
cp /tmp/rloop.md ~/.claude/commands/rloop.md
cp /tmp/rloop-init.md ~/.claude/commands/rloop-init.md
cp /tmp/rloop-check.md ~/.claude/commands/rloop-check.md
cp /tmp/rloop-update.md ~/.claude/commands/rloop-update.md
```

### 5. Handle new commands

Check if there are any new command files in the repo that aren't installed locally.
Fetch the directory listing:
```bash
curl -sL https://api.github.com/repos/mattbrc/rloop/contents/commands
```

If there are new `.md` files not in `~/.claude/commands/`, download and install them too.
Tell the user about any new commands that were added.

### 6. Clean up

```bash
rm -f /tmp/rloop*.md
```

### 7. Summary

Print a report:

```
═══════════════════════════════════════════════════
  rloop updated
═══════════════════════════════════════════════════

  rloop.md          updated  — <brief change summary>
  rloop-init.md     unchanged
  rloop-check.md    updated  — <brief change summary>
  rloop-update.md   unchanged
  rloop-new-cmd.md  NEW      — <description>

  Note: Updates apply to new sessions. Restart
  Claude Code if you're in the middle of a session.
═══════════════════════════════════════════════════
```
