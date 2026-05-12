---
name: Sub-agent dispatch pattern — Sonnet for impl, Opus 4.7 for review
description: Non-trivial implementation runs in a Sonnet sub-agent; reviews run in a dedicated Opus 4.7 (claude-opus-4-7) reviewer sub-agent. Main thread orchestrates only.
type: feedback
---

For any non-trivial implementation work, dispatch to sub-agents as follows:

- **Implementation** — `Agent` tool with `model: "sonnet"`. Sub-agent gets a focused, self-contained brief (spec section + plan phase) and produces code changes + tests. Sonnet is the workhorse for editing, scaffolding, refactoring.
- **Review** — `Agent` tool with `model: "opus"` (latest Opus 4.7, max capability). A **dedicated reviewer thread** kept alive across phases via `SendMessage` so it accumulates context. Use a fresh Opus reviewer only when scope shifts (e.g. moving from data-layer to infra-layer).

**Why:** User instruction 2026-05-11. Sonnet is cost-efficient + fast for code production; Opus 4.7 max-capability catches subtle issues. Keeping reviewers dedicated avoids context loss between review passes.

**How to apply:**

- Main thread orchestrates only: writes specs/plans, dispatches sub-agents, integrates results, surfaces blockers. Main thread does NOT write feature code directly.
- Each implementation dispatch:
    - `Agent({ subagent_type: 'general-purpose', model: 'sonnet', description: '...', prompt: '...' })`
    - Prompt includes: spec section reference, plan phase reference, explicit acceptance criteria, project rules (no DROP, no inline lint suppression, no skip hooks, no co-author, enum DB-parity), and what to verify before claiming done.
- Each review dispatch:
    - First review of a new domain: spawn fresh `Agent({ subagent_type: 'general-purpose', model: 'opus', ... })`.
    - Subsequent reviews on same domain: `SendMessage({ to: '<prior-agent-id>', ... })` so reviewer keeps state.
- **Trust-but-verify**: after each Sonnet sub-agent reports done, main thread runs verification commands (`bun run check-types`, `bun run test`, etc.) before passing to the Opus reviewer. Don't relay Sonnet's self-assessment as truth.
- If a sub-agent stalls (two identical diffs / unable to resolve a TS error / hook failure), main thread debugs by reading actual files — do not dispatch a third identical sub-agent.

**Exceptions where main thread does work directly:**

- 1-line edits (typo, comment, version bump)
- Memory/spec/plan writes (these ARE the orchestration artifact)
- Git operations (branch create, commit, push, PR)
- Diagnosing sub-agent failures (main thread is the debugger)
