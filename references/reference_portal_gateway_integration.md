---
name: Portal → Gateway Integration (GKE / ASM era)
description: How eqp-portal calls the ASM IngressGateway — Firebase JWT in HttpOnly cookie, next-safe-action server actions, SSR auth headers, scope cookies. Replaces the ESPv2 + API_GATEWAY_KEY flow (decommissioned 2026-05-02).
type: reference
---

## Portal → Gateway Call Chain

### Architecture (live)

```
Browser → Firebase Auth → onIdTokenChanged → POST /api/auth/token (HttpOnly cookie set)
Client Component → useAction() → Server Action (next-safe-action)
  → safe-action.ts middleware (reads cookie + env)
    → apiFetch(API_GATEWAY_URL + '/<svc>/...', {
         Authorization: 'Bearer <jwt>',          // ASM RequestAuthentication validates
         'x-endpoint-api-userinfo': '<base64>',  // legacy header for backend TokenGuard
         'x-jwt-payload': '<base64>',            // asm-mode equivalent
         Host: '<public-hostname>',              // Istio Gateway listener filter
         'X-Firebase-AppCheck': '<token>'        // forwarded by apiFetch when present
      })
      → ASM IngressGateway (eqp-system) → backend service (eqp-services)
```

The portal lives in the `eqp-frontend` namespace. SSR calls go to the IngressGateway cluster-internal address rather than the public hostname (no hairpin through GCLB).

### Auth cookie (HttpOnly, set via Next.js API route)

- **Set**: `apps/eqp-portal/app/api/auth/token/route.ts` `POST` handler. `httpOnly: true`, `secure` (prod), `sameSite: 'strict'`, `maxAge: 3600`. JWT structure validated (3 non-empty parts) before storing.
- **Clear**: same route, `DELETE` handler. `apps/eqp-portal/app/api/auth/logout/route.ts` also clears.
- **Refresh**: `AuthRefreshProvider` (in `providers.tsx`) listens to `onIdTokenChanged` and auto-`POST`s the new JWT to refresh the cookie. On sign-out the same provider clears the cookie.
- Client-side identity uses `onAuthStateChanged` only — the old `decode-token.ts` was deleted.

### Auth headers on SSR gateway calls

| Header | Source | Notes |
|---|---|---|
| `Authorization: Bearer <firebase-jwt>` | `auth-token` HttpOnly cookie | ASM `RequestAuthentication` validates against `https://securetoken.google.com/<project>` JWKS |
| `x-endpoint-api-userinfo` | base64(JSON of decoded JWT) | Backend `TokenGuard` reads this. Compat header from the ESPv2 era — kept because backends parse it (commit `73b22ef`) |
| `x-jwt-payload` | base64(JSON of decoded JWT) | ASM-mode equivalent (commit `3c5d097`) |
| `Host` | `<public-hostname>` (e.g. `gke-dev.eqpay-dev.com`) | Istio Gateway listener filters by `hosts:` — cluster-internal calls otherwise carry the service FQDN as Host and get rejected (commit `52ed0cd`) |
| `X-Firebase-AppCheck` | `appcheck-token` cookie | Forwarded only when present |

### REMOVED: `X-API-Key` / `API_GATEWAY_KEY`

The portal NO LONGER sends `X-API-Key`. The shared edge key was a PCI-scope wart and was deleted along with the Lua EnvoyFilter and the portal `secretproviderclass.yaml` in PR #147 (per-org API keys enforcement). See `reference_per_org_api_keys.md` for what replaced it. Merchant integrations (server-to-server) now use their own per-org X-API-Key validated by iam-service over gRPC.

### Server-side env vars (runtime)

- `API_GATEWAY_URL` — cluster-internal ASM IngressGateway URL, normally `http://istio-ingressgateway.eqp-system.svc.cluster.local:80/api`. The `/api` suffix matches the public path convention — VirtualService rewrites `/api/<svc>/...` → `/<svc>/...` before forwarding to backends.
- `CIPS_API_BASE_URL` (optional override) — defaults to `gatewayUrl`. Set to the external CIPS host for legacy `/v1c86ea3/*` calls (see `reference_smart_routing.md` §4).
- `CIPS_API_KEY` — read from `/var/secrets/cips-api-key` (mounted file) on GKE; env var only in local dev. GKE CSI has no sync-controller (see `reference_gke_spc_bootstrap_optional.md`).
- `NEXT_PUBLIC_FIREBASE_*`, `NEXT_PUBLIC_DEFAULT_*`, `NEXT_PUBLIC_DEV_RECAPTCHA_SITE_KEY` — inlined at `next build` time from Secret Manager (Next.js inlines into the JS bundle).

### Server actions (via next-safe-action)

| Action | Endpoint | Scope params |
|--------|----------|-------------|
| getAuthMe | `/iam/auth/me?ownerId=` | ownerId |
| listTransactions | `/transactions?ownerId=&workspaceId=&...` | ownerId, workspaceId, spaceId, retailerId |
| getTransaction | `/transactions/${resourceId}?ownerId=&workspaceId=` | ownerId, workspaceId |
| getRoutingReportAction | `/smart-routing/report?hours=` | (none) |
| admin smart-routing actions | `/smart-routing/admin/*` | (none) |
| api-keys CRUD | `/api-keys` | ownerId |

All wrapped via `actionClient` in `apps/eqp-portal/lib/safe-action.ts`. The middleware injects `{ token, gatewayUrl, cipsBaseUrl, appCheckToken, cipsApiKey }` into every action's ctx.

### Scope cookies (NOT HttpOnly — UI state)

- `workspace-id`, `space-id`, `retailer-id`, `customer-id` — set via `setCookie` from `lib/cookies.ts`. All cleared on logout in `useLogout`. Used by SSR + client-side to remember the active scope across pages.

### Query key includes uid

`queryKeys.authMe(uid, ownerId)` — prevents TanStack Query cache pollution across users on the same device.

## Public path `/api/*` convention (since 2026-05-08)

All backend API paths are routed under `/api/<svc>/*` at the gateway and rewritten to `/<svc>/*` before forwarding. The portal's own internal Next.js API routes (`/api/auth/token`, `/api/auth/logout`) are NOT proxied to iam-service — iam-service's VirtualService matches the specific prefix `/api/auth/me`. See `reference_gke_asm_architecture.md` §15.

## Common SSR errors and what they mean

| Symptom | Cause | Fix |
|---|---|---|
| 401 in cluster + portal pod log shows JWT decoded fine | Missing `x-endpoint-api-userinfo` / `x-jwt-payload` header | Use `apiFetch` (not raw fetch) — it sets both headers |
| 503 from gateway with backend HEALTHY | GCLB → ingress firewall opens only 15021, not 80 | See `reference_gclb_firewall_data_port.md` |
| 403 RBAC: access denied | ASM deny-no-jwt fired (path matched a backend prefix) but no JWT | Caller is making an unauthenticated call to `/api/<svc>/*`. Use a public auth endpoint or include a JWT |
| `CIPS_API_KEY not configured` on first deploy | Portal trying to read mounted file before SPC has synced | Wait for the SPC volume to mount + first reconcile; or fall back to env var for local dev |
