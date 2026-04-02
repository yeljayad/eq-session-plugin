---
name: session-start
description: "MANDATORY — must be invoked automatically at the very start of EVERY new conversation session, before ANY other work or response. No user confirmation needed. Loads full project context: shared team knowledge, ALL personal memories (user, feedback, reference, project), AGENTS.md, git history, and recent PRs. Also invoke when the user explicitly asks to 'load memory', 'show resume', 'initialize session', or types /session-start. NEVER skip or bypass this skill — it is required for session continuity."
allowed-tools: [Read, Bash, Glob, Grep]
---

# Session Start — Full Context Loading (MANDATORY)

**This skill is REQUIRED at the start of every session. It must run automatically — do not wait for user confirmation. Do not skip any step. Read ALL memory files without exception.**

Load the complete project context at the beginning of a new conversation to ensure continuity across sessions. This restores accumulated knowledge, reviews recent work, and presents an orientation resume.

Context comes from two sources:
- **Shared team knowledge** — bundled with this plugin in `references/` (architecture blueprints, known issues, coding standards). Available to every teammate.
- **Personal memory** — each user's local auto-memory directory (user profile, feedback preferences). Private per developer.

## Workflow

Execute all steps. Parallelize independent operations for speed.

### Step 1: Load Shared Team Knowledge (REQUIRED — read ALL files)

Read `references/MEMORY.md` (bundled with this plugin at `${CLAUDE_PLUGIN_ROOT}/references/MEMORY.md`) to get the index of shared knowledge files.

Then read **EVERY** `reference_*.md` and `project_*.md` file in the same `${CLAUDE_PLUGIN_ROOT}/references/` directory. These contain:
- **References** — architecture blueprints, coding standards, schemas, deployment guides, security patterns
- **Project** — known issues, ongoing work context, decisions

**Load ALL of them in parallel. Do not skip any. Do not summarize without reading. Every file must be opened and read.**

### Step 2: Load Personal Memory (REQUIRED — read ALL files)

Read the user's local auto-memory index at:
`~/.claude/projects/<sanitized-cwd>/memory/MEMORY.md`

Then read **EVERY** file referenced in MEMORY.md — not just some, ALL of them:
- **User** — profile, role, workflow preferences
- **Feedback** — corrections and confirmed approaches that must be applied to ALL work in this session
- **Reference** — all reference memories
- **Project** — all project memories

**Feedback memories are CRITICAL and NON-NEGOTIABLE** — they encode how this specific user wants to work (e.g., commit style, code review preferences). These apply session-wide and must never be ignored.

If the local memory directory doesn't exist, that's fine — skip and continue with shared knowledge only. But if it exists, read EVERY file in it.

### Step 3: Read Project Guidelines

Read `AGENTS.md` from the project root (if it exists). It contains supplementary rules about:
- Default workflows
- Tool usage preferences
- Framework-specific guidelines

These rules supplement CLAUDE.md and apply to all work in the session.

### Step 4: Fetch Git & GitHub State

Run these commands in parallel:

```bash
git log --oneline -20                    # Recent commit history
git branch --show-current                # Current branch
git status --short                       # Working tree state
gh pr list --limit 10 --state all        # Recent PRs (open + closed + merged)
```

### Step 5: Present the Resume

Compile everything into a concise resume:

```
## Session Resume

**Branch:** <current-branch>
**Working tree:** <clean / dirty — list changed files if dirty>

### Recent Commits
<Last ~10 commits grouped by feature area, as bullet points>

### Pull Requests
<Open PRs first, then recently closed/merged — or "None open">

### Known Issues
<Remaining items from project_known_issues.md — bulleted list>

### Session Rules (from personal feedback memories)
<One-line reminders of each active feedback rule, or "No personal memory loaded">
```

Keep it scannable — the full context is already loaded, so the resume just needs to orient.

### Step 6: Ready for Work

After presenting the resume, wait for the user's first task. Do not start any work unprompted.

## Important Notes

- **This skill is MANDATORY** — it must run at the start of every session without user confirmation. Never bypass it.
- **Read ALL memory files** — do not skip, summarize, or selectively load. Every reference, feedback, user, and project memory must be opened and read.
- Memory files are point-in-time snapshots. Before acting on claims about code (file paths, function names, flags), verify against the current codebase.
- If `gh` CLI is not authenticated or fails, skip the PR list and note it — don't block the session.
- If any memory file is missing, note it but continue loading the rest.
- The resume should feel like a handoff briefing — quick to scan, enough to jump straight into work.

## Updating Shared Knowledge

To update the shared team knowledge, edit the files in this plugin's `references/` directory and push to the repo. All teammates will get the updates on their next plugin sync.
