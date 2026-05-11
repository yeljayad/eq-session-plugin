## User
- [user_profile.md](user_profile.md) — YOUSSEFELJAYAD, lead dev, EQ-Pay-Solutions org, fast-paced workflow

## Feedback
- [feedback_no_coauthor.md](feedback_no_coauthor.md) — Never add Claude as co-author in git commits
- [feedback_superpowers_workflow.md](feedback_superpowers_workflow.md) — Never skip brainstorming/spec/plan/review steps — follow every phase
- [feedback_no_eslint_disable.md](feedback_no_eslint_disable.md) — Zero inline lint-suppression in source; all relaxations in config files
- [feedback_fix_code_not_rules.md](feedback_fix_code_not_rules.md) — Refactor code to comply with lint rules, don't weaken the rules
- [feedback_no_skip_hooks.md](feedback_no_skip_hooks.md) — Never use --no-verify — fix the root cause instead
- [feedback_global_deps.md](feedback_global_deps.md) — Pin shared deps at root only, sub-packages use peerDependencies
- [feedback_use_context7.md](feedback_use_context7.md) — Use context7 MCP to fetch library docs before upgrades or unfamiliar API usage
- [feedback_auto_review_specs.md](feedback_auto_review_specs.md) — Always dispatch spec/plan reviewer subagents before presenting docs to user

## References — Platform Architecture (live state)
- [reference_gke_asm_architecture.md](reference_gke_asm_architecture.md) — GCLB → ASM IngressGateway → NestJS (HTTP) → gRPC (mTLS) on GKE Autopilot. Workload Identity, BinAuthz, secret CSI. Replaced Cloud Run+ESPv2.
- [reference_cicd_gke_pipeline.md](reference_cicd_gke_pipeline.md) — Tag-based Cloud Build + Cloud Deploy pipeline. BuildKit, immutable AR, BinAuthz sign, GKE rollout. GitHub Actions runs lint/test only.
- [reference_cloud_sql_prisma.md](reference_cloud_sql_prisma.md) — Cloud SQL Postgres provisioning, Prisma migrations via Cloud Build private worker pool, IAM-auth follow-up.
- [reference_kafka_spine.md](reference_kafka_spine.md) — Managed Kafka domain-event spine. mTLS via cert-manager, CloudEvents 1.0 JSON, @repo/kafka. Topics + ACLs + per-service certs.
- [reference_per_org_api_keys.md](reference_per_org_api_keys.md) — Per-org X-API-Key validated via gRPC at iam-service. Argon2id + pepper, Redis cache, Pub/Sub→BigQuery audit, per-org rate limit.
- [reference_smart_routing.md](reference_smart_routing.md) — Portal smart-routing report page + 13 orchestration-service endpoints + CIPS legacy `/v1c86ea3/*` bridge.

## References — Backend Patterns
- [reference_nestjs_deep_dive.md](reference_nestjs_deep_dive.md) — NestJS architecture: bootstrap, 7-guard chain, modules, controllers, services, repos, validation, testing
- [reference_backend_service_blueprint.md](reference_backend_service_blueprint.md) — Canonical NestJS service blueprint (DI, caching, CRUD, errors, idempotency)
- [reference_guard_chain_security.md](reference_guard_chain_security.md) — 7-guard pipeline (ApiKey + Token + Device + User + Permission + RoleScope + Scope), enrichment, error propagation
- [reference_static_permissions.md](reference_static_permissions.md) — USER_PROFILES architecture (in-memory, not Firestore), GET /iam/users/:uid
- [reference_di_injection_patterns.md](reference_di_injection_patterns.md) — Firebase/Redis DI, useFactory vs useClass, string tokens, `API_KEY_VALIDATOR` port pattern (in-process vs gRPC)
- [reference_test_config.md](reference_test_config.md) — Vitest presets, inline configs, passWithNoTests, CI pitfalls

## References — Frontend Patterns
- [reference_frontend_blueprint.md](reference_frontend_blueprint.md) — Next.js portal blueprint (app router, components, auth state machine, styling, i18n)
- [reference_frontend_page_blueprint.md](reference_frontend_page_blueprint.md) — Dashboard page file structure, hooks, next-safe-action server-action pattern
- [reference_portal_gateway_integration.md](reference_portal_gateway_integration.md) — Portal → ASM IngressGateway: HttpOnly cookie, SSR auth headers, server actions, scope cookies
- [reference_nextjs_portal_gotchas.md](reference_nextjs_portal_gotchas.md) — proxy.ts, SSR cookie guards, .next cache, GKE CSI mounted-file pattern

## References — Domain Schemas
- [reference_firestore_schema.md](reference_firestore_schema.md) — Firestore collection hierarchy + all fields (mid-migration to Postgres)
- [reference_permission_matrix.md](reference_permission_matrix.md) — RBAC matrix: 9 roles × resources (including api_keys)
- [reference_packages_inventory.md](reference_packages_inventory.md) — One-liner per @repo/* package + @repo/observability and @repo/kafka deep dives

## References — Specific Gotchas
- [reference_asm_deny_no_jwt_paths.md](reference_asm_deny_no_jwt_paths.md) — `deny-no-jwt` AuthorizationPolicy (inverted to positive `paths:` list, PR #130). How to add public/backend paths.
- [reference_gclb_firewall_data_port.md](reference_gclb_firewall_data_port.md) — GCLB→ingressgateway firewall MUST open both 15021 AND 80. Opening only 15021 → backend HEALTHY + every request 503.
- [reference_gke_runtime_env_pattern.md](reference_gke_runtime_env_pattern.md) — Per-env ConfigMap `platform-env` (REDIS_URL, GCP_PROJECT_ID, KAFKA_BOOTSTRAP_SERVERS, …). Two ConfigMaps per env (services + frontend namespaces).
- [reference_gke_spc_bootstrap_optional.md](reference_gke_spc_bootstrap_optional.md) — GKE CSI has NO sync-controller. Read secrets from the mounted file, or use `optional: true` on secretKeyRef.
- [reference_release_please_tagging.md](reference_release_please_tagging.md) — release-please owns the `v*` namespace. Merge the bot's open PR; never `git tag v*` manually.

## References — Standards & Tools
- [reference_coding_standards.md](reference_coding_standards.md) — Where to find coding rules, patterns, conventions
- [reference_claude_code_hooks.md](reference_claude_code_hooks.md) — 22 Claude Code hooks (security blocks, semantic LLM review, audit). `CLAUDE_TEAM_LEAD_BYPASS=1` for the bypass-with-audit path.
- [reference_github_rulesets.md](reference_github_rulesets.md) — 6 active rulesets, API management, OIDC, release-please PAT

## Project
- [project_known_issues.md](project_known_issues.md) — Remaining: customer.repository spread order, Cloud SQL IAM auth follow-up, Prisma migrator in CI, portal tests, lint-staged race in parallel subagents
