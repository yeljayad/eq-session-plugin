## User
- [user_profile.md](user_profile.md) — YOUSSEFELJAYAD, lead dev, EQ-Pay-Solutions org, fast-paced workflow

## Feedback
- [feedback_no_coauthor.md](feedback_no_coauthor.md) — Never add Claude as co-author in git commits
- [feedback_superpowers_workflow.md](feedback_superpowers_workflow.md) — Never skip brainstorming/spec/plan/review steps — follow every phase
- [feedback_no_eslint_disable.md](feedback_no_eslint_disable.md) — Zero eslint-disable in source code — all relaxations in config files
- [feedback_fix_code_not_rules.md](feedback_fix_code_not_rules.md) — Refactor code to comply with lint rules, don't weaken the rules
- [feedback_no_skip_hooks.md](feedback_no_skip_hooks.md) — Never use --no-verify — fix the root cause instead
- [feedback_global_deps.md](feedback_global_deps.md) — Pin shared deps at root only, sub-packages use peerDependencies
- [feedback_use_context7.md](feedback_use_context7.md) — Use context7 MCP to fetch library docs before upgrades or unfamiliar API usage
- [feedback_auto_review_specs.md](feedback_auto_review_specs.md) — Always dispatch spec/plan reviewer subagents before presenting docs to user

## References
- [reference_nestjs_deep_dive.md](reference_nestjs_deep_dive.md) — NestJS architecture: bootstrap, guards, feature modules, controller/service/repo patterns, validation, errors, testing
- [reference_coding_standards.md](reference_coding_standards.md) — Where to find coding rules, patterns, and conventions
- [reference_backend_service_blueprint.md](reference_backend_service_blueprint.md) — NestJS service blueprint: DI wiring, controller, service caching, repository, DTO/schema, errors, idempotency
- [reference_guard_chain_security.md](reference_guard_chain_security.md) — Six-guard pipeline, request enrichment, auth decorators, scope validation, Redis caching
- [reference_di_injection_patterns.md](reference_di_injection_patterns.md) — Firebase/Redis DI, useFactory vs useClass, string tokens, value imports for DI
- [reference_frontend_blueprint.md](reference_frontend_blueprint.md) — Next.js portal blueprint (app router/components/auth/styling/i18n)
- [reference_frontend_page_blueprint.md](reference_frontend_page_blueprint.md) — Dashboard page file structure, hooks, next-safe-action
- [reference_firestore_schema.md](reference_firestore_schema.md) — Firestore DB schema: collection hierarchy, all fields/types
- [reference_permission_matrix.md](reference_permission_matrix.md) — RBAC permission matrix: 9 roles x resources with CRUD
- [reference_static_permissions.md](reference_static_permissions.md) — USER_PROFILES architecture, guard chain, GET /iam/users/:uid
- [reference_test_config.md](reference_test_config.md) — Vitest presets, inline configs, CI pitfalls, passWithNoTests
- [reference_ci_pipeline.md](reference_ci_pipeline.md) — CI/deploy/release workflows, OIDC auth, release-please, rulesets
- [reference_nextjs_portal_gotchas.md](reference_nextjs_portal_gotchas.md) — proxy.ts, SSR cookie guards, NODE_ENV conflict, .next cache
- [reference_github_rulesets.md](reference_github_rulesets.md) — 6 active rulesets, API management, OIDC, release-please PAT
- [reference_gateway_deploy.md](reference_gateway_deploy.md) — ESPv2 deploy flow, service names, API key, common pitfalls
- [reference_portal_gateway_integration.md](reference_portal_gateway_integration.md) — Portal→gateway: HttpOnly cookie, AuthRefreshProvider, AppCheck, server actions

## Project
- [project_known_issues.md](project_known_issues.md) — Remaining: portal tests, jest-config cleanup, file-path-restrictions, staging deploy pattern
