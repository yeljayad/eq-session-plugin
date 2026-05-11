---
name: Smart-routing report page + orchestration backend
description: Portal admin dashboard for transaction routing reports + 13 orchestration-service routing-engine endpoints + legacy CIPS bridge for /v1c86ea3/* calls.
type: reference
---

## 1. Purpose

Admin dashboard for the `routing-engine` (a Cloud Run service that decides which PSP to route each transaction to). The orchestration-service `SmartRoutingModule` proxies the routing-engine behind the standard auth chain, mints a Cloud Run ID token via `@repo/gcp`'s `getCloudRunIdToken`, and caches reads in Redis with explicit invalidation on writes. The portal renders a tabbed Suspense page that drives 13 endpoints across 6 controllers.

**Source files (orchestration-service):** `apps/orchestration-service/src/smart-routing/`
- `smart-routing.module.ts` — registers all 6 controllers + `RoutingEngineClient`, `RoutingEngineService`, `ReportService`
- `routing-engine.client.ts` — HTTP client (15s timeout, Zod parsing on every response, Cloud Run audience derived from URL origin)
- `routing-engine.service.ts` — Redis caching layer (read-through + DEL on writes)
- `report.service.ts` — historic BigQuery snapshot path

**Public surface (3 endpoints, `SmartRoutingController`):**
- `GET  /smart-routing/engine-health` — cached `CACHE_TTL.SMART_ROUTING_ENGINE_HEALTH`. `@RequirePermission('retailer', 'R')`
- `POST /smart-routing/route` — pass-through; NOT cached (routing engine memoises by request_id). `@RequirePermission('retailer', 'C')`
- `GET  /smart-routing/decision/:id` — pass-through; NOT cached (upstream owns the authoritative cache). `@RequirePermission('retailer', 'R')`

**Report surface (1 endpoint, `ReportController`):**
- `GET  /smart-routing/report?hours=N` — BigQuery snapshot + LLM prose. `1 <= hours <= 720`. `@RequirePermission('retailer', 'R')`

**Admin surface (9 endpoints across 4 controllers):**
- `AdminCapabilitiesController` (`/smart-routing/admin/capabilities`):
  - `GET  /snapshot` — PSP capability matrix. Read-only; operators edit the routing-engine sidecar JSON.
- `AdminHealthController` (`/smart-routing/admin/health`):
  - `GET  /snapshot` — rolling-window health. `'R'`
  - `POST /record` — append a health sample. `'U'`
  - `POST /reset` — wipe health window. `'D'`
- `AdminVolumeController` (`/smart-routing/admin/volume`):
  - `GET  /snapshot` — monthly volume counter (powers `volume_quota` pre-filter). `'R'`
  - `POST /record` — increment volume. `'U'`
  - `POST /reset` — zero volume. `'D'`
- `AdminDecisionCacheController` (`/smart-routing/admin/decision-cache`):
  - `GET  /snapshot` — FIFO of the last N upstream decisions. `'R'`
  - `POST /reset` — wipe upstream + local. `'D'`

Total: **13 endpoints**. Permission resource is `retailer` across the board — there is no dedicated `smart-routing` resource in the permission matrix today.

## 2. Portal page

`apps/eqp-portal/app/(dashboard)/smart-routing/page.tsx` is a thin Suspense wrapper around `SmartRoutingPage` (commit `7655524` — `useSearchParams` MUST be inside `Suspense` for the prerender to succeed):

```tsx
export default function Page(): ReactElement {
    return (
        <Suspense fallback={<ReportSkeleton />}>
            <SmartRoutingPage />
        </Suspense>
    );
}
```

`features/smart-routing/components/smart-routing-page.tsx` is a tabbed UI driven by `?tab=` in the URL with 6 tabs: `overview | health | volume | capabilities | cache | tryit`. The tab parser whitelists only those 6 values; anything else falls back to `overview`. An always-visible `EngineHealthCard` sits above the tabs.

## 3. Hook → action pattern

The page uses **server actions, not raw fetch**. Each tab's hook (`useRoutingReport`, `useRoutingAdmin*`) goes through `@tanstack/react-query` and calls a `'use server'` action in `features/smart-routing/actions/`:

- `get-routing-report.ts` → `getRoutingReportAction`
- `admin-health.ts`, `admin-volume.ts`, `admin-capabilities.ts`, `admin-decision-cache.ts`, `engine-health.ts`, `route-decision.ts`

The actions use `next-safe-action`'s `actionClient` (from `apps/eqp-portal/lib/safe-action.ts`) which injects `{ token, gatewayUrl, cipsBaseUrl, appCheckToken, cipsApiKey }` into every action's ctx.

**Gotcha (commit `8953e2d`):** When a hook switches from raw `fetch` to a server action, the test mocks MUST switch too. The smart-routing page test mocks `getRoutingReportAction`, NOT global `fetch` — old fetch mocks silently no-op against actions.

**Error wire format:** actions throw `Error("ROUTING_ERROR:<code>")` where `<code>` is one of `invalid_hours | auth_setup | report_disabled | upstream_auth | upstream | forbidden`. The hooks parse the prefix and surface a typed `RoutingErrorCode` to the UI. `TERMINAL_ERROR_CODES` (`forbidden | auth_setup | report_disabled | invalid_hours`) skip retry.

## 4. CIPS legacy bridge (NOT through the gateway)

Legacy CIPS endpoints (`/v1c86ea3/order/refund`, `/order/standalone-refund`, `/order/void`, `/task`) live on a **separate external host** with its own auth (`?key=` + `x-device-uuid`). The portal routes these directly, bypassing the cluster IngressGateway.

**Three commits to know:**

| Commit    | What it did                                                                              |
|-----------|------------------------------------------------------------------------------------------|
| `eaee053` | Added a `SecretProviderClass` (`cips-api-key` in GCP Secret Manager) to the eqp-portal deployment |
| `57d9c1c` | Switched portal to read the key from the MOUNTED FILE (GKE CSI has no sync-controller)   |
| `758452d` | Added `cipsBaseUrl` (defaults to `gatewayUrl`, override with `CIPS_API_BASE_URL` env) to action ctx |

**Critical GKE gotcha:** GKE's managed Secret Store CSI add-on does NOT include the sync-controller that turns SPC `secretObjects` into k8s Secrets. **Code MUST read the key from the mounted file (`/var/secrets/cips-api-key`), NOT from `process.env.CIPS_API_KEY`.** The `loadCipsApiKey()` helper in `apps/eqp-portal/lib/safe-action.ts` tries env first (local dev) then the mounted file (GKE), caches the result, and throws if neither is present.

**Prod base URL:** `https://cips-test-api-gateway.equiti-pay.com`. **Dev:** `http://localhost:4000` — the local proxy's `EXTERNAL_ROUTES` table forwards `/v1c86ea3/*` to the same external host.

## 5. dev-proxy routing

`scripts/local-proxy.ts` (the `bun run dev:proxy` server on :4000):
- `/smart-routing/*` → `localhost:3003` (orchestration-service) — added in commit `9421fbe`
- Commit `b3af03d` removed a duplicate route entry — keep ONE block under orchestration-service
- `/v1c86ea3/*` → external `https://cips-test-api-gateway.equiti-pay.com` via the `EXTERNAL_ROUTES` table

## 6. Cache invalidation

Each admin write DELs the matching snapshot key (NOT a wildcard pattern). Keys live in `packages/redis/src/redis-keys.helper.ts`:

| Key                                 | Read TTL constant                  | Invalidated by                  |
|-------------------------------------|------------------------------------|---------------------------------|
| `smartRoutingEngineHealth()`        | `SMART_ROUTING_ENGINE_HEALTH`      | (never — read-only)             |
| `smartRoutingReport(hours)`         | (in ReportService)                 | (TTL only)                      |
| `smartRoutingHealthSnapshot()`      | `SMART_ROUTING_HEALTH_SNAPSHOT`    | `record`, `reset`               |
| `smartRoutingVolumeSnapshot()`      | `SMART_ROUTING_VOLUME_SNAPSHOT`    | `record`, `reset`               |
| `smartRoutingCapabilities()`        | `SMART_ROUTING_CAPABILITIES`       | (TTL only — engine restart)     |
| `smartRoutingDecisionCache()`       | `SMART_ROUTING_DECISION_CACHE`     | `reset`                         |

`/route` and `/decision/:id` are explicitly NOT cached. See the leading comment block on `RoutingEngineService` for the why.

## 7. i18n

Strings live in `packages/i18n/src/locales/{en,fr,pt,ar}/smart-routing.json` (full namespace, registered in `packages/i18n/src/I18nContext.tsx` as `smartRouting`). Use `useI18n()` and reference `t.smartRouting.*`. EN is the seed; the other three locales were committed alongside but currently mirror EN until translation. Run `i18n-completeness-checker` when adding new keys.

## 8. Permissions

Every endpoint uses `@RequirePermission('retailer', X)`:
- `'R'` (read): all 6 snapshot endpoints + `engine-health` + `report` + `decision/:id`
- `'C'` (create): `POST /route`
- `'U'` (update): `record` (health + volume)
- `'D'` (delete): `reset` (health + volume + decision-cache)

No `@Public()` anywhere — the whole module is behind the standard 7-guard chain. The page is a `(dashboard)/` route so SSR auth + Firebase JWT cookie is implicit.
