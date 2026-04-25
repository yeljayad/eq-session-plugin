---
name: Claude Code Hooks Stack
description: Payment-grade Claude Code hooks (22 hooks across 8 events) — security blocks, quality enforcement, audit logging, semantic LLM review. Bypass via CLAUDE_TEAM_LEAD_BYPASS=1. Live on dev since 2026-04-25.
type: reference
---

## Overview

equitipay-apps enforces 22 Claude Code hooks defined in .claude/settings.json and .claude/hooks/*.sh. Three goals: block categorical errors, enforce repo conventions, audit AI activity. Source of truth: .claude/hooks/README.md in the repo.

## Hook inventory by event

| Event | Hooks |
|-------|-------|
| SessionStart | session-start (eq-session loader) |
| PreToolUse Edit/Write | block-protected-files, block-secrets, block-eslint-disable, block-console-log, block-bc-sentinels, block-settings-edit, semantic-review |
| PreToolUse Bash | block-bad-bash, block-infra-bash, enforce-conv-commits |
| PostToolUse Edit/Write | format-and-lint, typecheck-touched, audit-sensitive |
| UserPromptSubmit | inject-context |
| SubagentStart / SubagentStop | log-subagent-start / verify-subagent |
| ConfigChange (project_settings) | block-config-change |
| FileChanged (.env) | watch-env-files |

User-level (optional, in ~/.claude/settings.json): audit-user, audit-bash, notify-stop, session-end-summary. See .claude/hooks/USER_SETTINGS_INSTRUCTIONS.md for the JSON snippet.

## Categories of what gets blocked

- Hardcoded secrets (cloud API key prefixes, PSP live key prefixes, JWT shape, Luhn-valid card numbers, service-account JSON shape, PEM private-key armor)
- Dangerous bash (verify-bypass flags, alternate package managers, force push to protected branches, PR-merge-to-main, broad rm -rf)
- Production infra commands (deploy/apply/destroy across the cloud CLIs)
- Convention violations (inline lint-disable comments, browser-console logging in source files, backwards-compat marker comments, underscored deprecation renames, non-conventional commit messages)
- Self-protection (settings.json, hooks/*, .husky/*, commitlint.config.ts, .semgrep.yml — must be human-edited; ConfigChange blocks in-session reload)

For exact patterns, read the individual scripts under .claude/hooks/.

## Semantic review (the LLM hook)

semantic-review.sh fires PreToolUse on payment-critical (apps/payment-gateway-service/**, apps/transaction-service/src/**, apps/api-gateway/openapi.yaml) and auth-critical (*.guard.ts, */auth/*, packages/api/src/**, */firebase-app.module.ts) paths.

Implementation: spawns `claude -p --model opus --effort high --bare --no-session-persistence --output-format json --max-budget-usd 0.10`. The --bare flag is essential (prevents recursive hook firing). Uses the team's existing Claude Code billing — no per-developer API key required.

Failure mode: any LLM error → fail-open (exit 0) with LLM_REVIEW_UNAVAILABLE advisory. Deterministic hooks already ran; semantic is a bonus check.

Mock for tests: set CLAUDE_HOOK_LLM_PATH=lib/llm-mock.sh and LLM_MOCK_VERDICT=allow/deny/fail.

## Bypass — CLAUDE_TEAM_LEAD_BYPASS=1

Run as: CLAUDE_TEAM_LEAD_BYPASS=1 claude code

When set: every hook short-circuits via lib/bypass-check.sh. Each skipped rule logs an NDJSON entry to BOTH .claude/audit-sensitive.log (project) AND ~/.claude/audit/equitipay-apps.log (personal, when user-level hooks wired). SessionEnd writes a summary listing every rule that fired-but-was-skipped.

The bypass is per-session, never persistent. All-or-nothing — no partial bypass mode.

## Path classifier — single source of truth

lib/is-sensitive-path.sh is the only place where "what is sensitive" is defined. 4 tiers: payment-critical, auth-critical, sensitive, normal. Adding new sensitive paths requires only editing this one file.

Used by: semantic-review.sh, audit-sensitive.sh, audit-user.sh.

## Test infrastructure

bash .claude/hooks/test/run-tests.sh runs all 16 test files. CI: hooks-lint-test job in .github/workflows/ci.yml runs shellcheck (severity=warning) + tests. Tests use stdin-fed JSON fixtures, assert exit codes + stderr substrings + audit-log writes.

Each hook test includes positive (block), negative (allow), and bypass+audit cases. Bypass tests use PROJECT_AUDIT_OVERRIDE and USER_AUDIT_OVERRIDE env vars + mktemp to isolate from the real logs.

## How to apply

- Before editing apps/payment-gateway-service/** or apps/transaction-service/**, expect a 3–10s pause for semantic review. If it blocks, read the reasoning in stderr — it usually flags a real issue.
- Commit messages must follow conventional commits. Don't bypass via verify flags — fix the message.
- Don't ask Claude to edit .env, lock files, or .claude/settings.json. Edit those manually.
- Don't introduce browser-console logging outside test/spec/scripts/config files. Use the NestJS Logger.
- If a hook blocks legitimately — restart with CLAUDE_TEAM_LEAD_BYPASS=1 (audited) rather than disabling the hook.
- After editing .claude/settings.json itself, restart Claude Code (ConfigChange prevents partial reloads).

## Known false-positive on Markdown docs

Markdown files are not in the path-exemption list of block-console-log and block-secrets. Writing docs that quote literal trigger strings is blocked unless bypassed. Recommended follow-up: extend the path exemption to include *.md.
