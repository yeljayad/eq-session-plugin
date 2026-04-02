---
name: Frontend Page Development Blueprint
description: How to build new dashboard pages — file structure, hooks, services, API integration, next-safe-action pattern
type: reference
---

## Page Structure (follow for every new page)

```
app/(dashboard)/page-name/page.tsx            — Minimal, delegates to feature
features/page-name/
  components/                                  — UI components + orchestrator
  hooks/                                       — State management + data fetching
  types/                                       — TypeScript interfaces
  utils/                                       — Helpers, formatters
  actions/                                     — next-safe-action server actions
```

## Key Patterns

1. **Page entry is minimal** — just imports and renders the feature orchestrator
2. **Feature module** under `features/` not in `app/` routes
3. **Custom hooks** extract ALL state logic (data fetching, filters, pagination)
4. **Server actions** via `next-safe-action` for all backend calls (not raw fetch)
5. **Error/loading states** in hooks, rendered by orchestrator
6. **i18n** — all strings via `useI18n()`, nested keys like `t.transactions.filters.status`
7. **Auth token** from cookie (`auth-token`), passed as `x-endpoint-api-userinfo` header

## next-safe-action Pattern

All new server-side functions MUST use next-safe-action:
```typescript
// features/transactions/actions/list-transactions.ts
'use server';
import { actionClient } from '@/lib/safe-action';
import { z } from 'zod';

export const listTransactions = actionClient
  .schema(FilterTransactionSchema)
  .action(async ({ parsedInput }) => {
    const res = await fetch(`http://localhost:3004/transactions/list?...`, {
      headers: { 'x-endpoint-api-userinfo': getToken() },
    });
    return res.json();
  });
```

## Data Flow

```
Server Action (next-safe-action) → NestJS backend → Firestore
         ↓
Custom Hook (useTransactions) → manages state, loading, error
         ↓
Orchestrator Component → renders table, filters, pagination
```

## Transaction Page Specifics

- Backend: `GET /transactions/list?ownerId=...&workspaceId=...` (17 filters)
- Backend: `GET /transactions/list/:id?ownerId=...&workspaceId=...` (full detail)
- Auth: `x-endpoint-api-userinfo` header with base64 Firebase JWT
- Scope: ownerId + workspaceId required, spaceId/retailerId/accountId optional
