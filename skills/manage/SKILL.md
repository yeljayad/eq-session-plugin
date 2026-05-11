---
name: manage
description: "Manage the eq-session plugin lifecycle — check status, sync new content from upstream, list bundled references/skills, or open an upstream PR with local edits. The plugin now lives inside the project repo at `<project>/.claude/plugins/eq-session/`, so its files are tracked by the project's git. Invoke when the user says 'update the eq-session plugin', 'sync eq-session', 'check eq-session status', 'list eq-session references', 'open an upstream PR for my eq-session edits', or types /eq-session:manage. Does NOT remove the plugin — uninstall is intentionally out of scope."
allowed-tools: [Read, Bash, Edit, Glob, Grep]
---

# Manage the eq-session Plugin

Self-management for the `eq-session` Claude Code plugin. The plugin lives **inside** the project repo, so its files are tracked by the project's git (not by a separate plugin repo). Upstream is `yeljayad/eq-session-plugin` on GitHub — we sync from it but never let it own the working tree.

**Does not remove** — uninstall is out of scope by design.

## Constants

- **Plugin dir** (inside the project repo): `${PROJECT_ROOT}/.claude/plugins/eq-session`
    - Resolve at runtime via `git rev-parse --show-toplevel` from any path inside the project.
- **Upstream repo**: `https://github.com/yeljayad/eq-session-plugin.git` (branch `main`)
- **Project settings**: `${PROJECT_ROOT}/.claude/settings.json` — must contain an `eq-team` marketplace pointing to a `directory` source at `${PROJECT_ROOT}/.claude`
- **User settings**: `~/.claude/settings.json` — must contain `"eq-session@eq-team": true` under `enabledPlugins`
- **Plugin manifest**: `${PLUGIN_DIR}/.claude-plugin/plugin.json` — `version` field is the source of truth

## Workflow

If the user's intent is already explicit ("update the plugin" → `update`), run that action directly. Otherwise ask which action they want.

| Action              | When to use                                                                                       |
| ------------------- | ------------------------------------------------------------------------------------------------- |
| `status`            | Show version, upstream drift, uncommitted edits to plugin files, settings wiring                  |
| `update`            | Pull latest content from upstream into the project plugin dir, preserving local-only files        |
| `list-refs`         | List bundled references and skills; flag MEMORY.md drift                                          |
| `upstream-pr`       | Take the project's local plugin edits and open a PR upstream against `yeljayad/eq-session-plugin` |
| `reset-to-upstream` | **Destructive** — discard local plugin edits and replace with a clean upstream snapshot           |

End every action with a one-line summary.

---

### Resolve paths

Before doing anything, set:

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel)
PLUGIN_DIR="$PROJECT_ROOT/.claude/plugins/eq-session"
test -d "$PLUGIN_DIR" || { echo "Plugin missing at $PLUGIN_DIR. Run 'reset-to-upstream' to install fresh."; exit 0; }
```

If `git rev-parse` fails (not inside a repo), tell the user to `cd` into the project first.

---

### Action: `status`

Gather everything in parallel; format a brief report.

```bash
# Local state
echo "--- VERSION ---"
cat "$PLUGIN_DIR/.claude-plugin/plugin.json" | jq -r '.name + " v" + .version'

echo "--- UNCOMMITTED EDITS TO PLUGIN FILES ---"
git -C "$PROJECT_ROOT" status --porcelain -- .claude/plugins/eq-session

echo "--- PROJECT SETTINGS MARKETPLACE ---"
jq '.extraKnownMarketplaces["eq-team"]' "$PROJECT_ROOT/.claude/settings.json"

echo "--- USER SETTINGS ENABLED ---"
grep -E '"eq-session@eq-team"' ~/.claude/settings.json || echo "NOT enabled in ~/.claude/settings.json"

# Upstream comparison (read-only: shallow clone to a temp dir)
TMP=$(mktemp -d)
git clone --quiet --depth 1 https://github.com/yeljayad/eq-session-plugin.git "$TMP"
echo "--- UPSTREAM VERSION ---"
jq -r '.name + " v" + .version' "$TMP/.claude-plugin/plugin.json"
echo "--- UPSTREAM HEAD ---"
git -C "$TMP" log -1 --oneline

echo "--- DIFF SUMMARY (project vs upstream) ---"
diff -rq "$PLUGIN_DIR" "$TMP" 2>&1 | grep -v -E '^Only in .*\.git$|/\.git/' | head -40

rm -rf "$TMP"
```

Report format:

- Installed: `eq-session vX.Y.Z`
- Upstream: `eq-session vX.Y.Z` (HEAD: `<short-sha> <subject>`)
- Drift: `N files differ`, `M files only in project`, `K files only in upstream`
- Uncommitted edits: list paths (or "none")
- Marketplace wired: yes/no
- Enabled in user settings: yes/no

If upstream has newer content, **recommend** running `update` but do not run it automatically.

---

### Action: `update`

Sync new content from upstream into the project plugin dir. **Preserves local-only files** (like the `manage` skill that may not exist upstream yet). Leaves changes uncommitted in the project repo so the user can review and commit.

1. **Pre-flight** — check that the plugin dir is clean OR stash uncommitted plugin edits in the project repo:
    ```bash
    if [ -n "$(git -C "$PROJECT_ROOT" status --porcelain -- .claude/plugins/eq-session)" ]; then
      git -C "$PROJECT_ROOT" stash push -m "eq-session manage: pre-update plugin edits" -- .claude/plugins/eq-session
      STASHED=1
    fi
    ```
2. **Fetch upstream** to a temp dir (shallow clone):
    ```bash
    TMP=$(mktemp -d)
    git clone --quiet --depth 1 https://github.com/yeljayad/eq-session-plugin.git "$TMP"
    ```
3. **Sync** — `rsync` without `--delete` so local-only files survive:
    ```bash
    rsync -a --exclude='.git' --exclude='.github' "$TMP/" "$PLUGIN_DIR/"
    ```
    `rsync -a` does not delete files in the destination, so any local-only additions (`skills/manage/`, future local skills) remain in place.
4. **Cleanup the temp clone**:
    ```bash
    rm -rf "$TMP"
    ```
5. **Restore stash** if applicable:
    ```bash
    if [ "${STASHED:-}" = "1" ]; then
      git -C "$PROJECT_ROOT" stash pop
    fi
    ```
    If `stash pop` conflicts, leave the stash in place. Report which files conflict and instruct the user to resolve manually (`git status`, `git diff`, `git stash show -p`).
6. **Report**:
    - New version (from `plugin.json`)
    - Files added / modified — pull from `git status --porcelain -- .claude/plugins/eq-session`
    - Whether local edits were preserved
    - Reminder to **review with `git diff`** before committing, then commit on a branch.
    - **Restart Claude Code** so the new content is loaded.

---

### Action: `list-refs`

Enumerate bundled content and cross-check `MEMORY.md` against the filesystem.

```bash
echo "--- SKILLS ---"
ls -1 "$PLUGIN_DIR/skills"

echo "--- REFERENCES (filesystem) ---"
FS_REFS=$(ls -1 "$PLUGIN_DIR/references" | grep -E '^(reference|project)_.*\.md$' | sort)
echo "$FS_REFS"

echo "--- INDEX (MEMORY.md) ---"
INDEX_REFS=$(grep -oE '\[.+\]\(.+\.md\)' "$PLUGIN_DIR/references/MEMORY.md" | sed -E 's/.*\((.+)\).*/\1/' | sort -u)
echo "$INDEX_REFS"

echo "--- DRIFT: in MEMORY.md but not on disk ---"
comm -23 <(echo "$INDEX_REFS") <(echo "$FS_REFS")

echo "--- DRIFT: on disk but not in MEMORY.md ---"
comm -13 <(echo "$INDEX_REFS") <(echo "$FS_REFS")
```

Report counts and any drift. If `MEMORY.md` is out of sync with the filesystem, mention which direction (missing entries vs. orphan files).

---

### Action: `upstream-pr`

Contribute the project's local plugin edits back to `yeljayad/eq-session-plugin` as a PR. Works for both committed and uncommitted edits in the project repo (filtered to `.claude/plugins/eq-session/**`).

1. **Detect local plugin changes** — anything in the working tree or recent project commits touching `.claude/plugins/eq-session/`:
    ```bash
    git -C "$PROJECT_ROOT" status --porcelain -- .claude/plugins/eq-session
    git -C "$PROJECT_ROOT" log --oneline -10 -- .claude/plugins/eq-session
    ```
    If nothing has changed since the last upstream sync, tell the user there's nothing to PR.
2. **Show the user the diff** (`git -C "$PROJECT_ROOT" diff -- .claude/plugins/eq-session`) and ask for:
    - branch name (in the upstream repo)
    - commit message
    - PR title
    - PR body
3. **Clone upstream** to a temp dir:
    ```bash
    TMP=$(mktemp -d)
    git clone --quiet https://github.com/yeljayad/eq-session-plugin.git "$TMP"
    cd "$TMP"
    git checkout -b "$BRANCH"
    ```
4. **Copy plugin files from project into the upstream clone**, mapping paths (`.claude/plugins/eq-session/` → `/`):
    ```bash
    rsync -a --exclude='.git' --exclude='.github' "$PLUGIN_DIR/" "$TMP/"
    ```
    This propagates everything currently in the project's plugin dir — including local-only files like `skills/manage/`. If the user only wants a subset, ask them which files to include and `rsync` per-file instead.
5. **Commit and push**:
    ```bash
    cd "$TMP"
    git add -A
    git commit -m "$MSG"
    git push -u origin "$BRANCH"
    gh pr create --repo yeljayad/eq-session-plugin --base main \
      --title "$TITLE" \
      --body "$BODY"
    ```
6. Return the PR URL. After merge, the user can run `update` to pull the changes back into the project (and the local-only file will then exist upstream too).
7. **Clean up**:
    ```bash
    rm -rf "$TMP"
    ```

**Never** include `Co-Authored-By: Claude` in the commit message (per repo feedback memory `feedback_no_coauthor`).

If `gh` is not authenticated, stop after `git push` and print the PR-creation URL or the manual `gh pr create` command for the user to run.

---

### Action: `reset-to-upstream`

**Destructive.** Discards all local plugin edits and replaces the plugin dir with a clean snapshot from upstream `main`. Use only when:

- the plugin dir is corrupted, or
- the user has confirmed they want to drop all local plugin customizations (e.g., they upstreamed their changes and now want a clean state).

**Always show what will be lost and ask for explicit confirmation before proceeding.**

```bash
echo "--- Files that will be lost (uncommitted) ---"
git -C "$PROJECT_ROOT" status --porcelain -- .claude/plugins/eq-session

echo "--- Local-only files that will be deleted ---"
TMP=$(mktemp -d)
git clone --quiet --depth 1 https://github.com/yeljayad/eq-session-plugin.git "$TMP"
diff -rq "$PLUGIN_DIR" "$TMP" 2>&1 | grep -E "^Only in $PLUGIN_DIR"
```

Wait for explicit user confirmation (e.g., "yes, reset"). On confirmation:

```bash
# Stash uncommitted changes to plugin files as a safety net (kept in stash, not lost)
git -C "$PROJECT_ROOT" stash push -m "eq-session manage: pre-reset safety stash" -- .claude/plugins/eq-session 2>/dev/null || true

# Wipe and replace
rm -rf "$PLUGIN_DIR"
mkdir -p "$PLUGIN_DIR"
rsync -a --exclude='.git' --exclude='.github' "$TMP/" "$PLUGIN_DIR/"
rm -rf "$TMP"

echo "Plugin reset to upstream. Safety stash kept in case you need to recover anything."
git -C "$PROJECT_ROOT" stash list
```

Tell the user the changes are now showing as `deletion + replace` in `git status` and need to be committed. **Restart Claude Code** afterward.

---

## Safety rules

- **Never** force-push.
- **Never** `rm -rf` the plugin dir except inside `reset-to-upstream`, and only after explicit user confirmation.
- **Never** edit settings.json with `Write` — always `Edit` so other settings (permissions, hooks, env, model) are preserved.
- **Always** show diffs and ask for confirmation before any write to the upstream repo (PR creation).
- **Always** use a stash as a safety net before destructive sync operations.
- If `gh` is not authenticated, fall back to printing the manual command.
- If `git rev-parse --show-toplevel` fails (not inside a repo), stop and ask the user to `cd` into the project.
- Temp directories created via `mktemp -d` MUST be cleaned up with `rm -rf` after use.

## After any successful update or reset

Tell the user: **"Restart Claude Code so the new plugin state loads. In-session reload of settings.json is blocked by the `ConfigChange` hook (`block-config-change.sh`)."**

Also remind them to commit the changes on a branch (the plugin lives inside the project repo, so changes ship with the project's normal git workflow):

```bash
git -C "$PROJECT_ROOT" checkout -b chore/eq-session-sync
git -C "$PROJECT_ROOT" add .claude/plugins/eq-session
git -C "$PROJECT_ROOT" commit -m "chore(eq-session): sync plugin from upstream vX.Y.Z"
```
