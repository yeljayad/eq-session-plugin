---
name: Guard Chain and Security Architecture
description: Six-guard pipeline, request enrichment flow, auth decorators, DI token indirection, error propagation, scope validation with Redis caching
type: reference
---

## Six-Guard Pipeline

Every request passes through 6 guards in order. Each enriches the `AuthRequest` object for downstream guards.

### Guard 1: TokenGuard
- Reads `x-endpoint-api-userinfo` header (base64 JSON from ESPv2 gateway)
- `Buffer.from(header, 'base64')` → JSON parse → `DecodedToken`
- Calls `FirebaseAuthService.getUserById(uid)` to check token revocation
- Compares `auth_time * 1000 < tokensValidAfterTime` → `UnauthorizedException(auth_000002)`
- **Attaches:** `req.decodedToken = { user_id, email, auth_time, iat, exp }`

### Guard 2: DeviceGuard (disabled)
- Full implementation is commented out
- Would read `x-device-uuid` header → attach `req.deviceId`
- Currently always returns `true`

### Guard 3: UserGuard
- Extracts `ownerId` from `req.query['ownerId']` → `UnauthorizedException(auth_000004)` if missing
- `UserRepository.findByResourceId(decodedToken.user_id, ownerId)` → Firestore `owner/{oid}/user/{uid}`
- Validates `status_name === 'Active'` → `ForbiddenException(auth_000006)`
- Validates `device_id === req.deviceId` (no-op since DeviceGuard disabled)
- Looks up `USER_PROFILES[user.user_profile_name]` → `ForbiddenException(auth_000008)` if role unknown
- **Attaches:** `req.currentUser = User`, `req.userProfile = ProfilePermissions`

### Guard 4: PermissionGuard
- Reads `@RequirePermission(resource, permission)` metadata from Reflector
- If no metadata → returns `true` (routes without permission decorator pass through)
- Checks `req.userProfile[resource][permission]` → `ForbiddenException(auth_000009)` if false
- Pure in-memory check — no DB calls

### Guard 5: RoleScopeGuard
- Reads `@ValidateScope()` and `@ValidateOptionalScope()` metadata
- If neither present → returns `true`
- Filters `ROLE_REQUIRED_SCOPE[role]` to only levels the endpoint accepts
- Ensures all required scope query params are present → `BadRequestException(scope_000002)` if missing
- Example: RetailerAdmin on endpoint accepting `[owner, workspace, space, retailer]` must provide all 4 IDs

### Guard 6: ScopeGuard
- Holds `Firestore` and `RedisService` directly (not through repository)
- For each scope level (owner, workspace, space, retailer):
  1. Check Redis cache `scope:{level}:{ids}:u:{userId}` → return on hit
  2. Read Firestore doc via `FirestorePathHelper` → `NotFoundException` if not exists
  3. If `!skipMembership` (non-SuperAdmin): `checkMembership()` — reads `user` subcollection doc
  4. Cache result in Redis (TTL 600s)
- `SCOPE_BYPASS_ROLES = Set(['SuperAdmin'])` — skips membership check

### Membership check (`checkMembership`)
```
scope:member:{level}:{scopeId}:u:{userId} → Redis check
  miss → FirestorePathHelper.{level}UserRef(db, ...ids, userId).get()
    not exists → ForbiddenException(scope_000001: ACCESS_DENIED)
    exists → cache true, TTL 600s
```

## Auth Decorators

| Decorator | Metadata Key | Read By | Purpose |
|-----------|-------------|---------|---------|
| `@Public()` | `IS_PUBLIC` | All 6 guards | Bypass entire chain |
| `@RequirePermission(resource, perm)` | `REQUIRE_PERMISSION` | PermissionGuard | CRUD permission check |
| `@ValidateScope(...levels)` | `SCOPE_LEVELS` | RoleScopeGuard + ScopeGuard | Mandatory scope validation |
| `@ValidateOptionalScope(...levels)` | `OPTIONAL_SCOPE_LEVELS` | RoleScopeGuard + ScopeGuard | Optional scope validation |
| `@CurrentUser()` | param decorator | Controller | Injects `req.currentUser` |
| `@AuthToken()` | param decorator | Controller | Injects `req.decodedToken` |
| `@CurrentUserProfile()` | param decorator | Controller | Injects `req.userProfile` |
| `@Idempotent(ttl?)` | `IDEMPOTENCY_METADATA` | IdempotencyInterceptor | Request dedup |

## DI Token Indirection Pattern

Guards in `@repo/api` cannot import `@repo/firebase` (would create circular deps). Solution:

```
@repo/api defines:  AUTH_REPOSITORY_TOKENS.USER = 'AUTH_USER_REPOSITORY' (string token)
@repo/api exports:  UserRepository class (uses @Inject(FIREBASE_APP_TOKEN))
Each service:       { provide: AUTH_REPOSITORY_TOKENS.USER, useClass: UserRepository }
UserGuard:          constructor(@Inject(AUTH_REPOSITORY_TOKENS.USER) userRepo)
```

This decouples the guard package from specific Firebase implementations.

## Request Enrichment Flow

```
Request → TokenGuard → req.decodedToken
        → DeviceGuard → req.deviceId (disabled)
        → UserGuard → req.currentUser + req.userProfile
        → PermissionGuard → validates (no enrichment)
        → RoleScopeGuard → validates (no enrichment)
        → ScopeGuard → validates + caches (no enrichment)
        → Controller: @CurrentUser(), @AuthToken() read from req
```

## Error Propagation

```
ZodValidationPipe → BadRequestException(ApiValidationError) → 400
Guards → UnauthorizedException (401), ForbiddenException (403), BadRequestException (400), NotFoundException (404)
Service → NotFoundException, ConflictException (409)
ApiExceptionFilter catches all:
  1. Validation errors (has .errors array) → pass through
  2. HttpException with error code → getErrorMessage(code) → { error_code, error_message }
  3. Unhandled → 500 with generic message
```

## Scope Cache Keys

```
scope:owner:{ownerId}:u:{userId}
scope:workspace:{ownerId}:{workspaceId}:u:{userId}
scope:space:{ownerId}:{workspaceId}:{spaceId}:u:{userId}
scope:retailer:{ownerId}:{workspaceId}:{spaceId}:{retailerId}:u:{userId}
scope:member:{level}:{scopeId}:u:{userId}
```

All use `CACHE_TTL.ITEM = 600s`. No active invalidation — TTL-based expiry only.

## Routes Without Guards

To bypass specific guards without `@Public()` (which bypasses ALL):
- Omit `@RequirePermission` → PermissionGuard returns `true`
- Omit `@ValidateScope` + `@ValidateOptionalScope` → RoleScopeGuard + ScopeGuard return `true`
- TokenGuard + UserGuard always run (except `@Public()`) — they need `ownerId` query param

This pattern used for `GET /auth/me` — self-lookup endpoints that need authentication but not permission/scope checks.
