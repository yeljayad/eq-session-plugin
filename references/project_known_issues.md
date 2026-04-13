---
name: Known Issues and Bugs
description: Remaining bugs, architectural gaps, and CI/CD issues (updated 2026-04-13) — check before working on related areas
type: project
---

## Fixed on 2026-04-13

- `@repo/jest-config` package deleted entirely — no consumers remained after the full Vitest migration, 6 files removed. Commit `f83c8e9` on dev
- Dependabot alert #22 resolved (GHSA-q4gf-8mx6-v5v3 / CVE-2026-23869, Next.js DoS via Server Components, high/CVSS 7.5): jest-config was pinning `next@16.1.5` (vulnerable); eqp-portal's `^16.2.1` was also resolving to vulnerable 16.2.1 in lockfile — bumped to `^16.2.3` (patched). Alert still shows open on GitHub because Dependabot scans `main` branch, not `dev` — will auto-close at next release
- `bun.lock` synced with `next-themes@^0.4.6` that commit `70c2d2c` added to root `package.json` but never installed. `@repo/theme` was failing `tsc --noEmit` with `Cannot find module 'next-themes'`. Commit `65b491f`
- `packages/i18n/dist/locales/*/common.json` untracked — 4 files that were tracked historically but matched the root `.gitignore:36 dist` pattern. Would regenerate stale vs source on every install. Commit `88500bc`
- `packages/docs/packages/jest-config.md` renamed to `vitest-config.md` (the file already documented vitest-config, only the filename was stale). Cleaned up stale references in `README.md`, `packages/docs/README.md`, `packages/docs/Tech-Stack/overview.md`. Historical migration tables in `vitest-config.md` and `architecture-meeting-summary.md` preserved
- `packages/docs/packages/decorators.md` expanded from 1109 → 2206 lines with a comprehensive "Request Lifecycle — Every Data Mutation" section covering every guard/interceptor/pipe/filter/service/repository transformation with file:line evidence, plus a "Known Bugs and Gotchas" section with 17 entries. Commit `33b16c3`

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

### Backend

2. **CustomerRepository spread-order bug** — `apps/business-crud-service/src/customer/customer.repository.ts:92` uses `{ resource_id: doc.id, ...doc.data() }` (wrong order). If any stored customer doc has a `resource_id` field, it silently overwrites the authoritative Firestore doc id. Retailer and `FirestoreBaseRepository` use the correct `{ ...rest, resource_id: doc.id }` spread. Listed as "fixed" in earlier known_issues but the fix was only applied to base + Retailer — Customer's own `applyFilters` override was missed. Documented in `packages/docs/packages/decorators.md` §Known Bugs #1.

3. **`GrpcMetadataInterceptor` is dead code** — `packages/grpc/src/grpc-metadata.interceptor.ts` defines a class that propagates `idempotency-key` from HTTP header to gRPC metadata, but is never registered as `APP_INTERCEPTOR` in any service's `AppModule`. Either wire it globally (and enable cross-service idempotency key propagation) or delete.

### CI/CD

4. **Portal build in CI** — CI needs Firebase `NEXT_PUBLIC_*` env vars set via GitHub environment secrets.

### Infrastructure

5. **`NEXT_PUBLIC_DEFAULT_WORKSPACE_ID` missing from Dockerfile** — Not declared as Docker ARG, never baked into portal build.

6. **GitHub rulesets `file-path-restrictions.json`** — Enterprise-only rules, can't be imported via API. Must be imported manually via Settings > Rules > Rulesets > Import.

7. **Deploy staging pattern overly broad** — `deploy.yml` staging grep `'^v\d+\.\d+\.\d+-.+$'` matches any prerelease, not just `rc`. Could be narrowed to `'-rc\.\d+$'`.

8. **3 missing health endpoints in gateway** — notification-service, payment-gateway-service, card-service not in health.yaml (they only serve gRPC).

**How to apply:** Check this list before working on portal, CI/CD, or testing.
