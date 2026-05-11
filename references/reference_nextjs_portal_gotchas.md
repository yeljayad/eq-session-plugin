---
name: Next.js Portal Build Gotchas
description: Critical Next.js 16 build issues and workarounds — proxy.ts, SSR cookie guards, NODE_ENV conflict, prerender errors
type: reference
---

## Next.js 16: proxy.ts (not middleware.ts)

Next.js 16 renamed `middleware.ts` → `proxy.ts`. The exported function must be named `proxy` (not `middleware`). Runtime is Node.js only (Edge not supported). Same API: `NextRequest`, `NextResponse`, `config.matcher`.

File location: `apps/eqp-portal/proxy.ts` (same level as `app/`).

## SSR-Safe Cookie Access

Client components (`'use client'`) ARE rendered on the server during prerendering. `useEffect` doesn't run during SSR, but `useState` initializers DO.

**Pattern:** Guard all `document.cookie` access:
```typescript
function getCookie(name: string): string | undefined {
    if (typeof document === 'undefined') return undefined;
    // ... read document.cookie
}
```

This was required in `workspace-context.tsx` where `getCookie()` is called in `useState` initializers.

## useSearchParams Requires Suspense

Components using `useSearchParams()` must be wrapped in `<Suspense>` for static prerendering. Without it, `next build` fails. Example: `reset-password/page.tsx` wraps `ResetPasswordPage` in `<Suspense>`.

## NODE_ENV in .env (DO NOT SET)

`apps/eqp-portal/.env` must NOT contain `NODE_ENV`. Next.js manages it automatically:
- `next dev` → `NODE_ENV=development`
- `next build` → `NODE_ENV=production`

If `.env` has `NODE_ENV=development`, it overrides `next build`'s `NODE_ENV=production`, causing `_global-error` prerender failure (vercel/next.js#87719).

The `.env` file is in `.gitignore` — only exists locally. CI gets env vars from GitHub environment secrets.

## .next Cache After Changes

After deleting/moving pages or changing layout structure, stale `.next` cache can cause TS2307 errors or stale route artifacts. Run `rm -rf apps/eqp-portal/.next` before `check-types` or `build` when layout structure changes.

## Pre-commit Hook and .env Files

The `.husky/pre-commit` hook blocks ALL `.env` file commits (new and modified, `diff-filter=ACM`). This is intentional. The portal's `.env` contains Firebase `NEXT_PUBLIC_*` keys which are public, but the hook treats all `.env` files as potential secrets. Do NOT weaken this check.

## GKE CSI: read secrets from the MOUNTED FILE, not process.env

GKE's managed Secret Store CSI add-on does NOT include the sync-controller that turns `SecretProviderClass.secretObjects` into k8s Secrets. **Code MUST read secret values from the mounted file path (`/var/secrets/<name>`), NOT from `process.env.<NAME>`.**

This bit the portal when it tried to inject `CIPS_API_KEY` as an env var via `valueFrom.secretKeyRef` referencing the SPC-synced k8s Secret — the Secret never materialised because the sync-controller isn't installed (commits `eaee053`, `57d9c1c`).

**Pattern in `apps/eqp-portal/lib/safe-action.ts` (`loadCipsApiKey()`):**
```ts
function loadCipsApiKey(): string {
    if (process.env.CIPS_API_KEY) return process.env.CIPS_API_KEY;          // local dev
    try {
        return readFileSync('/var/secrets/cips-api-key', 'utf8').trim();    // GKE prod/staging
    } catch { throw new Error('CIPS_API_KEY not configured'); }
}
```

For backend services, the same pattern applies — `iam-service` reads `api-key-pepper` from `/var/secrets/api-key-pepper`, `firebase-admin-credentials` from the mounted file, etc.

**Alternative (for env vars that can be optional at first boot):** `optional: true` on `valueFrom.secretKeyRef`. Pod admits with the env unset, SPC volume mounts, the (eventually-synced) Secret materialises, next pod restart picks up the value. The portal's earlier `API_GATEWAY_KEY` env used this pattern before per-org API keys removed the env entirely.

## Portal SSR auth headers to ASM IngressGateway

The portal calls back to the IngressGateway for SSR-side data fetching:
- `API_GATEWAY_URL=http://istio-ingressgateway.eqp-system.svc.cluster.local:80/api` (cluster-internal)
- `Host: <public-hostname>` header MUST be set — Istio Gateway's listener filters by `hosts:`, and cluster-internal calls otherwise carry the service FQDN as Host (commit `52ed0cd`)
- `x-endpoint-api-userinfo` header carries the base64-encoded Firebase JWT for `TokenGuard` (commit `73b22ef`)
- `x-jwt-payload` header (asm mode) is sent alongside (commit `3c5d097`) for compatibility with the ASM `RequestAuthentication` decoder

All backend API paths are routed under `/api/<svc>/*` at the gateway (rewrite to `/<svc>/*` inside, commit `07b5bf8`). The portal SSR catch-all is at `/` — see `reference_gke_asm_architecture.md` §15.
