---
name: Static Permissions Architecture
description: How USER_PROFILES replaced Firestore user_profile lookups — UserGuard, PermissionGuard, GET /iam/users/:uid
type: reference
---

## Overview

Permissions are static in-memory maps, NOT Firestore documents. The `user_profile` Firestore collection is no longer read by the backend.

## USER_PROFILES Map

Location: `packages/common/src/config/user-profiles.ts`

9 roles (PascalCase keys matching `USER_ROLE` constants) × 9 resources. Each resource maps to `{ C, R, U, D }` booleans.

Helpers: `CRUD`, `R_ONLY`, `NONE` constants for readability.

Type: `Record<UserRole, ProfilePermissions>` where `ProfilePermissions = Record<string, ResourcePermissions>`.

## Guard Chain (all services — 7 guards since per-org API keys)

1. **ApiKeyGuard** — validates `X-API-Key` (prefix `eqp_`) via gRPC to iam-service; on hit, attaches `req.apiKey` and short-circuits JWT guards
2. **TokenGuard** — decodes `x-endpoint-api-userinfo` header (Firebase JWT path)
3. **DeviceGuard** — stub (returns true)
4. **UserGuard** — loads user from Firestore, looks up `USER_PROFILES[user.user_profile_name]` (NO Firestore profile read)
5. **PermissionGuard** — checks `userProfile[resource][permission]` (JWT path) or `req.apiKey.scopes[resource][permission]` (API-key path). Flat access, no `menu-1`/`menu-2`.
6. **RoleScopeGuard** — validates required scope query params are present for the role
7. **ScopeGuard** — validates Firestore scope entities exist + Redis caching

### Key changes from Firestore-based permissions

- `UserGuard` constructor takes 2 args (Reflector + UserRepository), NOT 3
- `AuthRequest.USER_PROFILE` type is `ProfilePermissions`, NOT `UserProfile`
- `@CurrentUserProfile()` decorator returns `ProfilePermissions`
- `PermissionGuard` reads `userProfile[resource]` directly (was `userProfile['menu-2'][resource] ?? userProfile['menu-1'][resource]`)

### Dead code removed

- `packages/api/src/repositories/user-profile.repository.ts` — deleted
- `AUTH_REPOSITORY_TOKENS.USER_PROFILE` — removed from tokens
- `UserProfileRepository` — no longer imported in any AppModule

## GET /iam/users/:uid Endpoint

- `@RequirePermission('user', 'R')` + `@ValidateScope('owner')`
- Returns all user fields + `permissions: ProfilePermissions`
- Strictly self-lookup: `:uid` must match authenticated user's `resource_id` (403 on mismatch)
- No caching — guard already fetches user, permissions are in-memory

## Portal Integration

- `UserProvider` fetches user via server action `GET /iam/users/:uid?ownerId=...`
- Permissions come from the API response (backend is source of truth)
- `buildMenuGroups()` filters `MENU_ITEMS` by `permissions[resource]?.R`
- `MENU_ITEMS` config in `packages/common/src/config/menu-items.ts` (string icon names, mapped to React components in portal)
