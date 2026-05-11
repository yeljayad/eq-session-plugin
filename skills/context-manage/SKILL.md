---
name: context-manage
description: "Orchestrate context loading across the eq-apps repo — shared references, personal memory, project agents, skills, and hooks — to keep the right signal in the window without burning tokens. Use when: (a) you're about to start a task and need the right context loaded, (b) you're mid-session and unsure which skill/agent to dispatch, (c) the user asks 'what's loaded?', 'what should I use for X?', 'how do I save context?', 'route this task', 'find a reference about Y', or types /eq-session:context-manage, (d) the conversation feels context-pressured and you need to plan offload via subagent. Read-only by default — only loads or audits context, never deletes."
allowed-tools: [Read, Bash, Glob, Grep]
---

# Context Manage — Memory · Plugins · Skills · Agents · Hooks

The router and librarian for the eq-apps repo. This skill knows what context exists, where it lives, and when each piece earns its tokens. Invoke it before pulling content into the main window — especially for narrow tasks where a subagent should carry the load.

## What this repo loads (eager session-start, current state)

| Tier          | Source                        | Path                                                      | Count | ~Tokens           |
| ------------- | ----------------------------- | --------------------------------------------------------- | ----- | ----------------- |
| **Eager**     | Shared references             | `${PLUGIN_DIR}/references/reference_*.md`, `project_*.md` | 28    | ~57K              |
| **Eager**     | Personal memory               | `~/.claude/projects/<sanitized-cwd>/memory/*.md`          | 27    | ~21K              |
| **Eager**     | Project agents (descriptions) | `${PROJECT_ROOT}/.claude/agents/*.md`                     | 8     | ~13K              |
| **On-demand** | Local skills                  | `${PROJECT_ROOT}/.claude/skills/*/SKILL.md`               | 9     | varies            |
| **On-demand** | Plugin skills                 | `${PLUGIN_DIR}/skills/*/SKILL.md`                         | 2     | varies            |
| **Runtime**   | Hooks                         | `${PROJECT_ROOT}/.claude/hooks/*.sh`                      | 23    | 0 (don't eat ctx) |

Eager loading at session start ≈ **~91K tokens** before the user speaks. Use this skill to decide what truly belongs in the main window vs. what should be lazy or offloaded.

## Actions

| Action             | Purpose                                                                                     |
| ------------------ | ------------------------------------------------------------------------------------------- |
| `audit`            | What's loaded right now? What's been pulled mid-session? Token estimate.                    |
| `route <task>`     | Given a task, list the right skills + agents + references to use. **Highest-value action.** |
| `find <topic>`     | Search references and memory for a topic. Use before pulling a file.                        |
| `load <ref>`       | Read a specific reference into context (returns the file).                                  |
| `dispatch <agent>` | Dispatch a project reviewer agent with scoped context.                                      |
| `budget`           | Token budget status and offload recommendations.                                            |
| `lazy-mode`        | Recommend converting session-start from eager → tiered loading.                             |

Resolve paths first:

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel)
PLUGIN_DIR="$PROJECT_ROOT/.claude/plugins/eq-session"
MEM_DIR="$HOME/.claude/projects/$(printf %s "$PROJECT_ROOT" | tr / -)/memory"
```

---

### Action: `route <task>`

The core action. Given a task description, output a routing plan: which skill to invoke first (process), which references to pull, which agent to dispatch for review. Use the matrix below.

**Routing matrix** — task → (skill first, references, reviewer agent):

| Task pattern                                     | First skill (process)                                          | References to pull                                                                                                                                              | Reviewer agent                                                                        |
| ------------------------------------------------ | -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| New feature, page, or component                  | `superpowers:brainstorming` → `feature-dev:feature-dev`        | depends on layer (see below)                                                                                                                                    | `feature-dev:code-reviewer`                                                           |
| New endpoint / new RPC method (existing service) | `superpowers:brainstorming`                                    | `reference_backend_service_blueprint.md`, `reference_guard_chain_security.md`, `reference_nestjs_deep_dive.md`                                                  | `eqp-guard-reviewer` (+ `openapi-sync-checker` if REST)                               |
| New backend service                              | `superpowers:brainstorming`                                    | `reference_backend_service_blueprint.md`, `reference_di_injection_patterns.md`, `reference_guard_chain_security.md`, `reference_nestjs_deep_dive.md`            | `eqp-guard-reviewer`                                                                  |
| Fraud / AI risk / anti-fraud                     | `superpowers:brainstorming` (for new logic)                    | `reference_backend_service_blueprint.md`, `reference_guard_chain_security.md`, `reference_per_org_api_keys.md`                                                  | **`eqp-payment-reviewer`** + `security-reviewer` (fraud surfaces touch payment paths) |
| Callback / webhook / notification                | —                                                              | `reference_backend_service_blueprint.md`, `reference_guard_chain_security.md`                                                                                   | `eqp-payment-reviewer` if PSP callback, else `security-reviewer`                      |
| New frontend page                                | `superpowers:brainstorming`                                    | `reference_frontend_blueprint.md`, `reference_frontend_page_blueprint.md`, `reference_nextjs_portal_gotchas.md`, `reference_portal_gateway_integration.md`      | —                                                                                     |
| Payment / transaction work                       | `superpowers:brainstorming`                                    | `reference_backend_service_blueprint.md`, `reference_per_org_api_keys.md`, `reference_cloud_sql_prisma.md` (money columns), `reference_guard_chain_security.md` | **`eqp-payment-reviewer`** (P0 — mandatory)                                           |
| Database / migration / Prisma                    | —                                                              | `reference_cloud_sql_prisma.md`, `reference_firestore_schema.md` (cutover), `reference_coding_standards.md`                                                     | **`eqp-db-reviewer`** (mandatory)                                                     |
| Auth / guard / permission / scope                | —                                                              | `reference_guard_chain_security.md`, `reference_permission_matrix.md`, `reference_static_permissions.md`, `reference_per_org_api_keys.md`                       | **`eqp-guard-reviewer`** (mandatory)                                                  |
| gRPC / proto contract                            | —                                                              | `reference_backend_service_blueprint.md`                                                                                                                        | `proto-contract-checker`                                                              |
| OpenAPI / gateway routes                         | —                                                              | `reference_smart_routing.md`, `reference_gke_asm_architecture.md`, `reference_asm_deny_no_jwt_paths.md`, `reference_gclb_firewall_data_port.md`                 | `openapi-sync-checker`                                                                |
| Kafka / event / stream                           | —                                                              | `reference_kafka_spine.md`                                                                                                                                      | —                                                                                     |
| CI / deploy / release                            | —                                                              | `reference_cicd_gke_pipeline.md`, `reference_release_please_tagging.md`, `reference_github_rulesets.md`, `reference_claude_code_hooks.md`                       | —                                                                                     |
| GKE / ASM / runtime env                          | —                                                              | `reference_gke_asm_architecture.md`, `reference_gke_runtime_env_pattern.md`, `reference_gke_spc_bootstrap_optional.md`                                          | —                                                                                     |
| i18n (locale keys)                               | —                                                              | —                                                                                                                                                               | `i18n-completeness-checker`                                                           |
| Test gap / missing specs                         | —                                                              | `reference_test_config.md`                                                                                                                                      | `test-gap-finder`                                                                     |
| **Bug or failure**                               | `superpowers:systematic-debugging` (mandatory)                 | task-dependent                                                                                                                                                  | task-dependent                                                                        |
| **Code review request**                          | `superpowers:requesting-code-review`                           | —                                                                                                                                                               | task-dependent (see above)                                                            |
| **Plan execution**                               | `superpowers:executing-plans` or `subagent-driven-development` | —                                                                                                                                                               | —                                                                                     |

**Algorithm:**

1. Match the task description against the patterns (lowercase the prompt and `case` it against keywords).
2. Print the routing plan in 3-5 lines:
    - First skill to invoke
    - References to read (use `load` action to actually read them)
    - Reviewer agent to dispatch when implementation is done
3. **Do not pre-load references unless the user confirms.** Just list them. Loading is on-demand.
4. If multiple patterns match, list them all but mark P0 reviewers (`eqp-payment-reviewer`, `eqp-db-reviewer`, `eqp-guard-reviewer`) as mandatory.

---

### Action: `audit`

Inventory what context exists vs. what's likely been pulled this session.

```bash
echo "--- INVENTORY ---"
find "$PLUGIN_DIR/references" -maxdepth 1 -name 'reference_*.md' -o -name 'project_*.md' | wc -l
echo "shared refs"
find "$MEM_DIR" -maxdepth 1 -name '*.md' 2>/dev/null | wc -l
echo "personal memory"
ls "$PROJECT_ROOT/.claude/agents" | wc -l
echo "agents"

echo "--- PERSONAL MEMORY INDEX ---"
head -50 "$MEM_DIR/MEMORY.md" 2>/dev/null | sed -n 's/^- \[\(.*\)\].*/\1/p'

echo "--- SHARED MEMORY INDEX ---"
head -80 "$PLUGIN_DIR/references/MEMORY.md" | sed -n 's/^- \[\(.*\)\].*/\1/p'

echo "--- LOCAL SKILLS ---"
ls "$PROJECT_ROOT/.claude/skills"

echo "--- AGENTS ---"
ls "$PROJECT_ROOT/.claude/agents"
```

Report:

- Eager total: ~91K tokens
- Skills available: count
- Agents available: count
- Hint: anything Claude has explicitly read this session vs. just had injected. (Claude knows what's in its window — be honest about it.)

---

### Action: `find <topic>`

Search references and personal memory for a topic before pulling a whole file.

```bash
TOPIC="$1"
echo "--- HITS IN SHARED REFS ---"
grep -lir --include='*.md' "$TOPIC" "$PLUGIN_DIR/references" 2>/dev/null
echo
echo "--- HITS IN PERSONAL MEMORY ---"
grep -lir --include='*.md' "$TOPIC" "$MEM_DIR" 2>/dev/null
echo
echo "--- FIRST 3 MATCHES (with context) ---"
grep -rin --include='*.md' -m 3 "$TOPIC" "$PLUGIN_DIR/references" "$MEM_DIR" 2>/dev/null | head -20
```

Output the file list. The user (or Claude) can then `load <ref>` for the one(s) they want.

---

### Action: `load <ref>`

Pull a specific reference into context using the Read tool. **Do not** open the whole file if you only need a section — use Read with `offset`/`limit` after a `grep -n` to find the right range.

Steps:

1. Resolve full path: `${PLUGIN_DIR}/references/${ref}` if not absolute. Try `.md` suffix if missing.
2. Confirm file exists.
3. If file is large (> 5K tokens, ~20KB), prompt Claude to **read only the section** matching the user's intent rather than the whole thing. Use grep to find the heading anchor first.
4. Read with the Read tool.

---

### Action: `dispatch <agent>`

Dispatch a project agent in an **isolated subagent context** so its findings don't bloat the main window. This is the biggest context-saving lever — let the agent read the diff, the references, and report back a short summary.

Agents (from `${PROJECT_ROOT}/.claude/agents/`):

| Agent                       | Dispatch when                                                                                                                                                               |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `eqp-payment-reviewer`      | After any change in `apps/payment-gateway-service/**`, `apps/transaction-service/**`, anything touching `amount`, `currency`, `settlement_*`, `acquirer_*`, PSP credentials |
| `eqp-db-reviewer`           | After any change in `packages/database/prisma/**`, `*.repository.ts`, `@repo/redis` cache code, Firestore `applyFilters` overrides                                          |
| `eqp-guard-reviewer`        | After any change in `*.guard.ts`, AppModule guard wiring, `@RequirePermission`/`@ValidateScope`/`@Public`/`@Idempotent` decorators, gRPC controllers                        |
| `security-reviewer`         | Cross-cutting sensitive change — use as a backstop, the EQP-specialised reviewers are sharper                                                                               |
| `proto-contract-checker`    | After `packages/protos/src/**` or any `@GrpcMethod` change                                                                                                                  |
| `openapi-sync-checker`      | After `apps/api-gateway/openapi/**` or NestJS controller endpoint changes                                                                                                   |
| `i18n-completeness-checker` | After `packages/i18n/src/locales/**` or new `t.xxx.yyy` keys                                                                                                                |
| `test-gap-finder`           | After new `*.controller.ts` / `*.service.ts` without a sibling spec                                                                                                         |

**Dispatch pattern**:

```
Use the Agent tool with subagent_type="<agent-name>". The prompt MUST be self-contained:
- What changed (paths + brief summary)
- What to verify (specific rules from the agent's domain)
- What format to report back ("punch list under 200 words")
```

The agent runs in its own context window. The cost to the main thread is just the prompt + the summary it returns — typically 1-3K tokens instead of the 10-30K it would cost to do the same review in the main thread.

---

### Action: `budget`

Recommend offloading or compression strategies.

Heuristic:

- If main-context spend on completed work > 30K tokens → suggest spawning a subagent to continue, summarising the prior conversation
- If a single task pulls > 3 references → suggest dispatching a domain agent instead of reading them in-thread
- If the user is finishing implementation → recommend `superpowers:verification-before-completion` + the right `eqp-*-reviewer` (P0 paths mandatory)
- If the user is about to start something new → strongly suggest `audit` + `route` first, not "let's just see"

Output: 3 concrete recommendations, each one a single sentence with the command to run.

---

### Action: `lazy-mode`

Recommend converting `session-start` to tiered loading. Outputs a diff plan; **does not** modify session-start automatically (that's a meaningful architecture change — needs user opt-in).

Proposed tiers:

- **T1 (eager, ~5K tokens)**: `references/MEMORY.md`, all `feedback_*.md` personal memory, `user_profile.md`, project `MEMORY.md`. Critical workflow rules — must always be in context.
- **T2 (smart-injected per prompt via `inject-context.sh`)**: 1-3 references matching keywords in the user's prompt.
- **T3 (on-demand via this skill)**: full reference reads when Claude or the user explicitly pulls one.

Trade-off: reduces session-start cost from ~91K → ~5K (-95%), at the cost of one extra hop per task to load the right reference. Net win for most tasks; loss for tasks that need 4+ references upfront.

---

## Coordination with `inject-context.sh`

The hook at `${PROJECT_ROOT}/.claude/hooks/inject-context.sh` runs on every UserPromptSubmit and injects the first 60 lines of references matching keywords in the prompt. This is the **per-prompt T2 channel**.

Keyword → reference mapping (current):

| Keywords                                          | Reference                                                               |
| ------------------------------------------------- | ----------------------------------------------------------------------- |
| `guard`, `permission`, `scope`, `auth`            | `reference_guard_chain_security.md`                                     |
| `payment`, `transaction`, `charge`                | `reference_backend_service_blueprint.md`                                |
| `firebase`, `firestore`                           | `reference_firestore_schema.md`                                         |
| `gateway`, `openapi`, `asm`, `ingress`, `routing` | `reference_gke_asm_architecture.md`, `reference_smart_routing.md`       |
| `prisma`, `database`, `postgres`, `cloud sql`     | `reference_cloud_sql_prisma.md`                                         |
| `kafka`, `event`, `stream`                        | `reference_kafka_spine.md`                                              |
| `api key`, `x-api-key`, `server-to-server`        | `reference_per_org_api_keys.md`                                         |
| `hook`, `claude code`                             | `reference_claude_code_hooks.md`                                        |
| `deploy`, `ci`, `cd`, `pipeline`, `release`       | `reference_cicd_gke_pipeline.md`, `reference_release_please_tagging.md` |

If `route` recommends a reference that the hook already injected for this prompt, skip the `load` step — it's already in context (check the system reminder injected by the hook).

To extend the mapping, edit `${PROJECT_ROOT}/.claude/hooks/inject-context.sh`'s `case "$lc" in` block.

---

## Coordination with `session-start` and `manage` skills

This skill (`context-manage`) is the **runtime** counterpart to:

- `session-start` — bulk-loads at session start (currently eager).
- `manage` — manages the plugin lifecycle (install/update/PR upstream).

Relationship:

- `session-start` decides **what loads upfront**.
- `context-manage` decides **what loads on demand** and **how to route mid-session**.
- `manage` decides **what's installed at all**.

If the user wants smaller upfront load → run `lazy-mode`, then edit `session-start` per its plan (separately, the user approves).

---

## Safety rules

- **Read-only by default.** This skill never deletes memory, references, or settings. It only reads, audits, and routes.
- **Never pre-load all references** — that defeats the purpose. Always start with `route` to identify the 1-3 needed.
- **Prefer subagent dispatch** for any review or narrow-scope task to keep the main window lean.
- **Respect the inject-context.sh hook** — don't re-inject what the hook already pulled (the system reminder shows what it injected).
- If the user is in a long session and asks "why are you slow?" — that's often context bloat. Suggest `budget`.

## End-of-action summary

Always end with ONE line:

- `route` → "Routing: <skill> → <refs count> reference(s) → <agent> for review."
- `audit` → "Inventory: X refs, Y memories, Z agents. Eager load ~91K tokens."
- `find` → "Found N file(s). Use `load <name>` to read."
- `load` → "Loaded `<ref>` (~K tokens)."
- `dispatch` → "Dispatched `<agent>` in subagent context; awaiting report."
- `budget` → "3 offload candidates listed."
- `lazy-mode` → "Diff plan generated; edit session-start to opt in."
