---
name: Guard Chain and Security Architecture
description: Seven-guard pipeline (ApiKeyGuard added first for per-org API keys), request enrichment flow, auth decorators, DI token indirection, error propagation, scope validation with Redis caching. Two parallel auth paths — Firebase JWT or per-org X-API-Key — mutually exclusive per request.
type: reference
---

## Seven-Guard Pipeline

Every request passes through 7 global guards (registered as `APP_GUARD` via `useFactory`) in this exact order. Each enriches the `AuthRequest` object for downstream guards.

```
1. ApiKeyGuard      → [Reflector, API_KEY_VALIDATOR, RedisService, ApiKeyRateLimitService]
2. TokenGuard       → [Reflector, FirebaseAuthService]
3. DeviceGuard      → [Reflector]   (stub, returns true)
4. UserGuard        → [Reflector, AUTH_REPOSITORY_TOKENS.USER]
5. PermissionGuard  → [Reflector]
6. RoleScopeGuard   → [Reflector]
7. ScopeGuard       → [Reflector, FIREBASE_APP_TOKEN, RedisService]
```

### Two parallel auth paths

| Path | Header | Validated by | Downstream behaviour |
|------|--------|--------------|----------------------|
| Firebase JWT | `Authorization: Bearer <jwt>` (or `x-endpoint-api-userinfo` from ASM) | TokenGuard → UserGuard | PermissionGuard/ScopeGuard use `req.currentUser` + `req.userProfile` |
| Per-org API key | `X-API-Key: eqp_<...>` | ApiKeyGuard (gRPC → iam-service) | PermissionGuard/ScopeGuard use `req.apiKey.scopes` + `req.apiKey.firestoreOwnerId`; JWT guards short-circuit |

Mutually exclusive — both headers present → `400 Bad Request` (`AUTH_ERRORS.MIXED_AUTH`). Code: `packages/api/src/auth/api-key/api-key.guard.ts`.

### Guard 1: ApiKeyGuard (NEW, runs first)

- Reads `X-API-Key` header. Must start with prefix `eqp_` — else defers to TokenGuard.
- gRPC contexts (`context.getType() !== 'http'`) short-circuit — inter-service trust is enforced by mTLS at the mesh layer (`PeerAuthentication STRICT`).
- Cache key: `apikey:v1:<sha256(secret)>` (Redis). Positive TTL 300s, negative TTL 30s.
- Validator is injected via string token `API_KEY_VALIDATOR`:
  - **iam-service**: `useClass: ApiKeyValidatorInProcess` (direct Prisma query + argon2id verify + pepper)
  - **Other 10 services**: `useClass: ApiKeyValidatorClient` (gRPC `ApiKeyValidatorService.Validate(secret) → ApiKeyContext` to iam-service over mTLS — registered via `IAM_API_KEY_VALIDATOR` in `GrpcClientModule.forServices()`)
- 503 fail-CLOSED on `UNAVAILABLE`/`DEADLINE_EXCEEDED` (negative cache NOT poisoned on transient outage)
- Rate-limit check AFTER auth succeeds — don't charge invalid keys
- **Attaches:** `req.apiKey = { keyId, orgId, scopes, firestoreOwnerId, rateLimitRpm }`

### Guard 2: TokenGuard
- Reads `x-endpoint-api-userinfo` header (base64 JSON from the edge — was ESPv2, now ASM `RequestAuthentication`).
- Portal SSR also sends `x-jwt-payload` (asm mode) for compatibility.
- `Buffer.from(header, 'base64')` → JSON parse → `DecodedToken`.
- Calls `FirebaseAuthService.getUserById(uid)` to check token revocation.
- Compares `auth_time * 1000 < tokensValidAfterTime` → `UnauthorizedException(auth_000002)`.
- **Attaches:** `req.decodedToken = { user_id, email, auth_time, iat, exp }`.
- Skipped when `req.apiKey` is already set.

### Guard 3: DeviceGuard (disabled)
- Full implementation commented out — would read `x-device-uuid` header.
- Currently always returns `true`.

### Guard 4: UserGuard
- Extracts `ownerId` from `req.query['ownerId']` → `UnauthorizedException(auth_000004)` if missing.
- `UserRepository.findByResourceId(decodedToken.user_id, ownerId)` → Firestore `owner/{oid}/user/{uid}`.
- Validates `status_name === 'Active'` → `ForbiddenException(auth_000006)`.
- Looks up `USER_PROFILES[user.user_profile_name]` → `ForbiddenException(auth_000008)` if role unknown.
- **Attaches:** `req.currentUser = User`, `req.userProfile = ProfilePermissions`.
- Skipped when `req.apiKey` is already set.

### Guard 5: PermissionGuard
- Reads `@RequirePermission(resource, permission)` metadata from Reflector.
- If no metadata → returns `true`.
- Source of truth:
  - JWT path: `req.userProfile[resource][permission]`
  - API key path: `req.apiKey.scopes[resource]?.[permission]`
- `ForbiddenException(auth_000009)` if denied. No DB calls.

### Guard 6: RoleScopeGuard
- Reads `@ValidateScope()` and `@ValidateOptionalScope()` metadata.
- Filters `ROLE_REQUIRED_SCOPE[role]` to levels the endpoint accepts.
- Ensures all required scope query params are present → `BadRequestException(scope_000002)` if missing.
- Example: `RetailerAdmin` on endpoint accepting `[owner, workspace, space, retailer]` must provide all 4 IDs.

### Guard 7: ScopeGuard
- Direct Firestore + Redis access (not via repository).
- For each scope level (owner, workspace, space, retailer):
  1. Check Redis cache `scope:{level}:{ids}:u:{userId}` → return on hit.
  2. Read Firestore doc via `FirestorePathHelper` → `NotFoundException` if missing.
  3. If `!skipMembership` (non-SuperAdmin): `checkMembership()` — reads `user` subcollection doc.
  4. Cache result in Redis (TTL 600s).
- `SCOPE_BYPASS_ROLES = Set(['SuperAdmin'])` — skips membership.
- API key path: validates against `req.apiKey.firestoreOwnerId` (single-tenant scope).

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
| `@Public()` | `IS_PUBLIC` | All 7 guards | Bypass entire chain |
| `@RequirePermission(resource, perm)` | `REQUIRE_PERMISSION` | PermissionGuard | CRUD permission check |
| `@ValidateScope(...levels)` | `SCOPE_LEVELS` | RoleScopeGuard + ScopeGuard | Mandatory scope validation |
| `@ValidateOptionalScope(...levels)` | `OPTIONAL_SCOPE_LEVELS` | RoleScopeGuard + ScopeGuard | Optional scope validation |
| `@CurrentUser()` | param | Controller | Injects `req.currentUser` |
| `@CurrentApiKey()` | param | Controller | Injects `req.apiKey` (API-key path) |
| `@AuthToken()` | param | Controller | Injects `req.decodedToken` |
| `@CurrentUserProfile()` | param | Controller | Injects `req.userProfile` |
| `@Idempotent(ttl?)` | `IDEMPOTENCY_METADATA` | IdempotencyInterceptor | Request dedup |

**DO NOT** use `@Idempotent` on API-key issuance/rotate endpoints — it would cache the raw secret in Redis (security regression).

## DI Token Indirection Pattern

Guards in `@repo/api` cannot import service-specific implementations (would create circular deps). Solution: string-token DI.

```
@repo/api defines:  AUTH_REPOSITORY_TOKENS.USER     = 'AUTH_USER_REPOSITORY'
@repo/api defines:  API_KEY_VALIDATOR               = 'API_KEY_VALIDATOR'
Each service wires: { provide: AUTH_REPOSITORY_TOKENS.USER, useClass: UserRepository }
                    { provide: API_KEY_VALIDATOR, useClass: ApiKeyValidator{InProcess|Client} }
Guards inject:      @Inject(AUTH_REPOSITORY_TOKENS.USER) userRepo
                    @Inject(API_KEY_VALIDATOR) validator
```

This decouples the guard package from specific Firebase / Prisma / gRPC implementations.

## Request Enrichment Flow

```
Request → ApiKeyGuard → req.apiKey  (if X-API-Key path, else passthrough)
        → TokenGuard → req.decodedToken (JWT path)
        → DeviceGuard → req.deviceId (disabled)
        → UserGuard → req.currentUser + req.userProfile (JWT path)
        → PermissionGuard → validates against userProfile OR apiKey.scopes
        → RoleScopeGuard → validates scope params
        → ScopeGuard → validates entities + caches
        → Controller: @CurrentUser() / @CurrentApiKey() / @AuthToken() read from req
```

## Error Propagation

```
ZodValidationPipe → BadRequestException(ApiValidationError) → 400
Guards → UnauthorizedException (401), ForbiddenException (403),
         BadRequestException (400 — incl MIXED_AUTH), NotFoundException (404),
         ServiceUnavailableException (503 — ApiKeyGuard on validator UNAVAILABLE),
         TOO_MANY_REQUESTS (429 — ApiKeyGuard rate limit)
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
apikey:v1:{sha256(secret)}   ← ApiKeyGuard (300s positive, 30s negative)
```

All scope keys use `CACHE_TTL.ITEM = 600s`. No active invalidation — TTL-based expiry only.

## Routes Without Guards

To bypass specific guards without `@Public()` (which bypasses ALL):
- Omit `@RequirePermission` → PermissionGuard returns `true`.
- Omit `@ValidateScope` + `@ValidateOptionalScope` → RoleScopeGuard + ScopeGuard return `true`.
- TokenGuard + UserGuard always run on JWT path (need `ownerId` query param).

This pattern used for `GET /iam/auth/me` — self-lookup endpoints that need authentication but not permission/scope checks.

## gRPC Trust Boundary

`ApiKeyGuard` short-circuits on `context.getType() !== 'http'`. Inter-service gRPC trust is enforced one layer down:
- **Transport**: ASM `PeerAuthentication` STRICT — only pods with mesh certs can connect.
- **Application**: Controllers wired with `@GrpcMethod` may still apply `@RequirePermission` for fine-grained checks, but they don't re-validate API keys.

Cross-service idempotency-key propagation in gRPC metadata is wired via `GrpcMetadataInterceptor` (in `@repo/grpc`).
