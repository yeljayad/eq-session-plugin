---
name: NestJS DI and Injection Patterns
description: How Firebase, Redis, guards, repositories are wired — useFactory vs useClass, string tokens, @Inject decorator, global modules
type: reference
---

## Global Modules (no imports needed in feature modules)

- `FirebaseModule` — `@Global()`, provides `FIREBASE_APP_TOKEN` (FirebaseApp.App) and `FirebaseAuthService`
- `RedisModule` — provides `RedisService` and `IdempotencyService`
- `ConfigModule.forRoot({ isGlobal: true })` — env vars

## Firebase Injection

### In repositories (standard pattern)
```typescript
@Injectable()
export class FeatureRepository extends FirestoreBaseRepository<Feature> {
    constructor(@Inject(FIREBASE_APP_TOKEN) protected readonly app: FirebaseApp.App) {
        super(app);
    }
}
```
- `FIREBASE_APP_TOKEN = 'FIREBASE_APP'` — string constant from `@repo/firebase`
- `@Inject()` required because it's a string token (NestJS can't auto-inject by string)
- `FirestoreBaseRepository.constructor(app)` stores `this.db = app.firestore()`

### In non-base-repository classes
```typescript
@Injectable()
export class AuthRepository {
    private readonly db: ReturnType<FirebaseApp.App['firestore']>;
    constructor(@Inject(FIREBASE_APP_TOKEN) app: FirebaseApp.App) {
        this.db = app.firestore();
    }
}
```
- Same `@Inject(FIREBASE_APP_TOKEN)` pattern
- Use `ReturnType<FirebaseApp.App['firestore']>` for Firestore type (avoids importing `firebase-admin/firestore` directly — type resolution issues in services that don't have firebase-admin as direct dependency)

### In modules
```typescript
@Module({
    providers: [FeatureService, FeatureRepository],  // plain provider, NOT factory
})
```
- No `useFactory` needed — `@Inject(FIREBASE_APP_TOKEN)` on the repository constructor handles DI
- This is the convention used by business-crud-service (retailer, workspace, space, customer)

## Guard Wiring (useFactory pattern)

Guards use `useFactory` in AppModule (NOT `useClass`) for explicit DI control:

```typescript
{
    provide: APP_GUARD,
    useFactory: (reflector: Reflector, userRepo: UserRepository) =>
        new UserGuard(reflector, userRepo),
    inject: [Reflector, AUTH_REPOSITORY_TOKENS.USER],
}
```

**Why useFactory over useClass:** Allows injecting by string token (`AUTH_REPOSITORY_TOKENS.USER`). With `useClass`, NestJS would try to resolve `UserRepository` by class reference, which won't find the string-token registration.

## String Token Indirection

```
@repo/api defines:   AUTH_REPOSITORY_TOKENS.USER = 'AUTH_USER_REPOSITORY'
Each service wires:  { provide: AUTH_REPOSITORY_TOKENS.USER, useClass: UserRepository }
Guards inject:       @Inject(AUTH_REPOSITORY_TOKENS.USER) userRepo
```

This breaks the circular dep: `@repo/api` (guards) ← `@repo/firebase` (UserRepository). Guards don't import firebase directly — they depend on the abstract token.

## Redis Injection

```typescript
constructor(
    private readonly redis: RedisService,
) {}
```
- Class token — NestJS auto-resolves (no `@Inject` needed)
- `RedisService` is a VALUE import (NOT type-only) — required for DI metadata

## Important: Value vs Type Imports for DI

NestJS constructor-injected classes MUST use value imports:
```typescript
import { RedisService } from '@repo/redis';           // ✅ value import
import { type RedisService } from '@repo/redis';      // ❌ SWC strips this, DI breaks
import { AuthRepository } from './auth.repository';    // ✅ value import
```

The `consistent-type-imports` ESLint rule is disabled in the codebase for this reason.

## ScopeGuard — Direct Firestore Access

ScopeGuard is the one guard that accesses Firestore directly (not through a repository):
```typescript
constructor(
    private readonly reflector: Reflector,
    @Inject(FIREBASE_APP_TOKEN) app: FirebaseApp.App,
    private readonly redis: RedisService,
) {
    this.db = app.firestore();
}
```
It uses `FirestorePathHelper` static methods for path construction. This is intentional — the guard validates scope entity existence across all entity types, so a single-entity repository pattern doesn't fit.
