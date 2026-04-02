---
name: Portal to gateway integration
description: How eqp-portal calls the API gateway — auth flow, HttpOnly cookies, server actions, env vars
type: reference
---

## Portal → Gateway Call Chain

### Architecture
```
Browser → Firebase Auth → onIdTokenChanged → POST /api/auth/token (HttpOnly cookie)
Client Component → useAction() → Server Action (next-safe-action)
  → safe-action.ts middleware (reads cookie + env vars)
    → apiFetch(gatewayUrl + path, { Authorization, X-API-Key, X-Firebase-AppCheck })
      → ESPv2 Gateway → Backend Service
```

### Auth cookie (HttpOnly)
- Set via API route handler: `app/api/auth/token/route.ts` (POST sets, DELETE clears)
- `httpOnly: true`, `secure` (prod), `sameSite: 'strict'`, `maxAge: 3600`
- JWT structure validated (3 non-empty parts) before storing
- `AuthRefreshProvider` (in providers.tsx) listens to `onIdTokenChanged` — auto-refreshes cookie
- On sign-out: `else` branch in AuthRefreshProvider clears the cookie
- Client-side identity from `onAuthStateChanged` (not cookie decoding) — `decode-token.ts` deleted

### Auth headers (all REQUIRED — AND logic in ESPv2 security)
- `Authorization: Bearer <firebase-jwt>` — from `auth-token` HttpOnly cookie
- `X-API-Key: <api-key>` — from `API_GATEWAY_KEY` env var
- `X-Firebase-AppCheck: <token>` — from `appcheck-token` cookie (forwarded by apiFetch)

### Server-side env vars (runtime)
- `API_GATEWAY_URL` — gateway Cloud Run URL (required)
- `API_GATEWAY_KEY` — GCP API key (required)

### Server actions (via next-safe-action)
| Action | Endpoint | Scope params |
|--------|----------|-------------|
| getAuthMe | `/iam/auth/me?ownerId=` | ownerId |
| listTransactions | `/transactions?ownerId=&workspaceId=&...` | ownerId, workspaceId, spaceId, retailerId |
| getTransaction | `/transactions/${resourceId}?ownerId=&workspaceId=` | ownerId, workspaceId |

### Scope cookies (NOT HttpOnly — UI state)
- `workspace-id`, `space-id`, `retailer-id`, `customer-id` — set via `setCookie` from `lib/cookies.ts`
- All cleared on logout in `useLogout` hook

### Query key includes uid
`queryKeys.authMe(uid, ownerId)` — prevents cache pollution across users.
