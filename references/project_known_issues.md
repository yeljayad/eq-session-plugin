---
name: Known Issues and Bugs
description: Remaining bugs, architectural gaps, and CI/CD issues (updated 2026-04-01) — check before working on related areas
type: project
---

## Fixed on 2026-04-01

- Deep code review: all critical/important issues across backend, frontend, CI/CD, packages resolved
- FirestoreBaseRepository spread order bug (resource_id overwrite) — fixed in base class + RetailerRepository
- UserGuard device mismatch check guarded when DeviceGuard disabled
- Idempotency race condition (lock return value now checked)
- Firebase provider uses GCP_PROJECT_ID env var (not hardcoded)
- ApiExceptionFilter added to business-crud-service
- Auth cookie now HttpOnly via API route handler + JWT structure validation
- Firebase token auto-refresh via AuthRefreshProvider (onIdTokenChanged)
- AppCheck token forwarded as X-Firebase-AppCheck header
- Full i18n for transactions + dashboard (4 locales: en/fr/pt/ar)
- OIDC Workload Identity Federation replaces GCP_SA_KEY in deploy.yml
- Release-please configured with PAT token (enterprise PR restriction bypass)
- Release-please tag format fixed (no component prefix, beta/rc prerelease types)
- Health endpoints public (security: [] in OpenAPI)
- Tag protection covers all tags, rulesets re-imported via API
- format:check added to all branch ruleset required checks
- Bun 1.3.10 aligned across all configs
- Build job added to CI pipeline
- TransactionController + TransactionService specs added (27 tests)
- Version reset to 0.1.0 (pre-1.0)
- feat/iam-auth-me merged and branch deleted
- All docs updated (iam-service, eqp-portal, api-gateway, cloud-run, development-workflow)
- Claude code review workflow fixed (removed non-existent context7 plugin)
- .prettierignore: CHANGELOG.md + .release-please-manifest.json

## Fixed on 2026-03-29

- ESPv2 ENDPOINTS_SERVICE_NAME, gateway API key, portal server actions, OpenAPI camelCase params

## Fixed on 2026-03-27

- force-dynamic removed, proxy.ts route protection, workspace cookie SSR safety, Suspense boundary

## Fixed on 2026-03-26

- Jest→Vitest migration, static USER_PROFILES, GET /iam/users/:uid, portal UserProvider, rulesets

## Fixed on 2026-03-25

- Customer repository data corruption, Workspace/Space simplified, eslint-disable eliminated, Zod v4, gRPC foundation

## Remaining Issues

### Frontend (eqp-portal)

1. **No unit tests** — Zero test files in the portal. Vitest config is set up with `passWithNoTests`.

### CI/CD

2. **`@repo/jest-config` package still exists** — No consumers remain after full Vitest migration. Should be deleted and removed from turbo.json pipeline.

3. **Portal build in CI** — CI needs Firebase `NEXT_PUBLIC_*` env vars set via GitHub environment secrets.

### Infrastructure

4. **`NEXT_PUBLIC_DEFAULT_WORKSPACE_ID` missing from Dockerfile** — Not declared as Docker ARG, never baked into portal build.

5. **GitHub rulesets `file-path-restrictions.json`** — Enterprise-only rules, can't be imported via API. Must be imported manually via Settings > Rules > Rulesets > Import.

6. **Deploy staging pattern overly broad** — `deploy.yml` staging grep `'^v\d+\.\d+\.\d+-.+$'` matches any prerelease, not just `rc`. Could be narrowed to `'-rc\.\d+$'`.

7. **3 missing health endpoints in gateway** — notification-service, payment-gateway-service, card-service not in health.yaml (they only serve gRPC).

**How to apply:** Check this list before working on portal, CI/CD, or testing.
