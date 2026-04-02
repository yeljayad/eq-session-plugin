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

## Guard Chain (all services)

1. **TokenGuard** — decodes `x-endpoint-api-userinfo` header
2. **DeviceGuard** — stub (returns true)
3. **UserGuard** — loads user from Firestore, looks up `USER_PROFILES[user.user_profile_name]` (NO Firestore profile read)
4. **PermissionGuard** — checks `userProfile[resource][permission]` (flat access, no `menu-1`/`menu-2`)
5. **ScopeGuard** — validates Firestore scope entities exist

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
