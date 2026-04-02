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
