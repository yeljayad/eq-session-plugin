---
name: Known Issues and Bugs
description: Remaining bugs, architectural gaps, and CI/CD issues (refreshed 2026-05-11) — check before working on related areas
type: project
---

## Recently shipped (May 2026)

Major architectural work landed since the previous known-issues snapshot (2026-04-13):

- **GKE + ASM migration complete on cips-test** (Plans 1–5, PRs #58 → #134). Cloud Run + ESPv2 fully decommissioned 2026-05-02 (PR #132). 35+ first-build defects catalogued in `packages/docs/Operations/gke-cutover-runbook.md` Appendix A.
- **Kafka domain-event spine LIVE** (Phases 1–5, PRs #139 → #144). Managed Kafka cluster + cert-manager mTLS + `@repo/kafka` + 36 event schemas + first end-to-end producer/consumer + auth-domain cohort + business/card producers. See `reference_kafka_spine.md`.
- **Per-org API keys MERGED** (PRs #145 → #148). Replaced the Lua EnvoyFilter + shared `gateway-api-key`. ApiKeyGuard now the first global guard on every service; iam-service holds the Prisma `ApiKey` table + argon2id + pepper. PCI scope reduction win. See `reference_per_org_api_keys.md`.
- **Cloud SQL Postgres provisioned** (PR #119) + initial Prisma migration (PR #123) + Cloud Build private-worker-pool migrator pattern. transaction-service + payment-gateway-service consume Cloud SQL via cloud-sql-proxy sidecar.
- **BuildKit migration** (PR #171) — Cloud Build moved from Kaniko to `docker buildx`, with a separate mutable `eqp-platform-cache` AR repo (PR #181). Worker pool bumped to `e2-highcpu-32` (PR #167).
- **Smart-routing report page** (PR #178) + tabbed admin dashboard (commit `b2094d3`) + 13 routing-engine endpoints + cache invalidation (commit `e1098a0`). See `reference_smart_routing.md`.
- **Public `/api/*` routing convention** (commit `07b5bf8`, 2026-05-08) — all backend APIs rewritten under `/api/<svc>/*` at the gateway; deny-no-jwt notPaths updated to match (commit `44db756`).
- **GrpcMetadataInterceptor wired globally** across all 11 services — was dead code in 2026-04-13 notes, now fully wired (cross-service idempotency-key propagation in gRPC metadata).

## Remaining Issues

### Backend

1. **CustomerRepository spread-order bug — STILL PRESENT.** `apps/business-service/src/customer/customer.repository.ts:92` uses `{ resource_id: doc.id, ...doc.data() }` (wrong order). If any stored customer doc has a `resource_id` field, it silently overwrites the authoritative Firestore doc id. Retailer and `FirestoreBaseRepository` use the correct `{ ...rest, resource_id: doc.id }` spread. Listed as "fixed" in earlier known_issues but the fix was only applied to base + Retailer — Customer's own `applyFilters` override was missed. Same bug as 2026-04-13.

### Infrastructure

2. **Cloud SQL IAM auth deferred.** Dev runs on password auth (`cloudsql-postgres-password` in Secret Manager). Module is already prepped (`cloudsql.iam_authentication = on`). Switch to IAM auth (PCI 8.x) BEFORE staging/prod cutover — TF + deployment.yaml wiring change, no infra churn. See `reference_cloud_sql_prisma.md` §8.

3. **Prisma migrator job not in CI.** `infra/cloudbuild-migrate.yaml` exists but is run ad-hoc by the operator. Add to the tag-build pipeline (gated on `db:migrate:deploy` having migrations to apply) before staging/prod. Until then, schema changes require an operator submit per `reference_cloud_sql_prisma.md` §5–6.

4. **3 services with gRPC-only health** in the legacy gateway health-check list — notification-service, payment-gateway-service, card-service. Was relevant on the old ESPv2 gateway; verify whether the ASM `RequestAuthentication` + VirtualService chain has any gRPC-only routing today, otherwise this note is closeable.

### CI/CD

5. **Portal Firebase secrets on new envs.** Staging/prod first-build will fail if `portal-firebase-*` secrets are not pre-provisioned in Secret Manager. They're inlined into the JS bundle at `next build`. Document placeholder values for first apply, then rotate.

6. **GitHub rulesets `file-path-restrictions.json`** — Enterprise-only rules, can't be imported via API. Must be imported manually via Settings → Rules → Rulesets → Import. No change since 2026-04-13.

7. **Lint-staged race in parallel subagent dispatch.** When multiple implementer subagents touch `bun.lock` concurrently, lint-staged's stash-restore can capture another agent's unstaged files with the wrong commit message. Observed in Kafka Phase 5 Cohort 2 (commit `7f4f86d`'s message didn't match its content) and Per-org API Keys PR-C (commit `dcac5ae` same pattern). Mitigation: serialise bun.lock-touching tasks, or use per-implementer git worktrees. Squash-merge collapses these mis-labeled commits anyway.

### Frontend

8. **No unit tests** in eqp-portal beyond smart-routing's `mock(getRoutingReportAction)` test. Vitest config is set up with `passWithNoTests`. Same as 2026-04-13.

### Documentation

9. **`docs/firestore-schema.json` path reference in `reference_firestore_schema.md` is stale.** The 18K-line raw schema export no longer lives at that path. Either restore the file or strip the reference.

## What's closed

- ~~`GrpcMetadataInterceptor` dead code~~ — now wired globally across all 11 services.
- ~~OIDC Workload Identity Federation~~ — fully cut over.
- ~~`@repo/jest-config`~~ — deleted.
- ~~ESPv2 gateway / Cloud Run~~ — decommissioned 2026-05-02 (Plan 5).
- ~~Shared `gateway-api-key`~~ — replaced by per-org API keys (PR #147).
- ~~Portal-in-PCI-scope (held `API_GATEWAY_KEY`)~~ — portal exited PCI scope when the env was removed.

**How to apply:** Check this list before working on customer-business code, Cloud SQL wiring, CI/CD, portal tests, or anything mentioning Firestore schema docs.
