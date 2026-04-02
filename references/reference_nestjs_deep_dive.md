---
name: NestJS Architecture Deep Dive
description: Comprehensive NestJS service architecture — bootstrap, app module, guard chain, feature modules, controller/service/repository patterns, validation, error handling, idempotency, DI, testing. Source of truth: packages/docs/Tech-Stack/nestjs.md
type: reference
---

## Source of Truth

`packages/docs/Tech-Stack/nestjs.md` — 1300+ lines, all code snippets from actual files. Read it before building any NestJS feature.

## Key Architecture Decisions

### Bootstrap (`main.ts`)
- `import '@repo/observability/instrumentation'` MUST be first import (side-effect)
- Fastify adapter (NOT Express): `NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter())`
- Hybrid HTTP + gRPC: `app.connectMicroservice()` with `Transport.GRPC`
- Listen on `0.0.0.0` (required for Cloud Run)
- Port from `process.env.PORT` with dev fallback

### App Module — 6 Global Guards (useFactory, NOT useClass)
```
1. TokenGuard      → [Reflector, FirebaseAuthService]
2. DeviceGuard     → [Reflector]
3. UserGuard       → [Reflector, AUTH_REPOSITORY_TOKENS.USER]
4. PermissionGuard → [Reflector]
5. RoleScopeGuard  → [Reflector]
6. ScopeGuard      → [Reflector, FIREBASE_APP_TOKEN, RedisService]
```
Plus: `IdempotencyInterceptor` (APP_INTERCEPTOR), `ApiExceptionFilter` (APP_FILTER), `AUTH_REPOSITORY_TOKENS.USER → UserRepository` (string token DI).

### Feature Module Structure
```
<feature>/
├── <feature>.module.ts
├── <feature>.controller.ts + .spec.ts
├── <feature>.service.ts + .spec.ts
├── <feature>.repository.ts
└── dto/                    # Re-exports from @repo/common
    ├── create-<feature>.dto.ts
    └── update-<feature>.dto.ts
```
DTOs (Zod schemas) live in `@repo/common` — feature `dto/` dirs re-export for discoverability.

### Controller Decorators
- `@RequirePermission('resource', 'C'|'R'|'U'|'D')` — RBAC check
- `@ValidateScope('owner', 'workspace', 'space')` — mandatory scope
- `@ValidateOptionalScope(...)` — optional scope filtering
- `@Idempotent()` / `@Idempotent(CACHE_TTL.LONG)` — on POST/PATCH/DELETE
- `@CurrentUser()` — injects `req.currentUser`
- `@Public()` — bypasses entire guard chain
- `@HttpCode(HttpStatus.CREATED)` — on POST (201)
- `new ZodValidationPipe(Schema)` — per-parameter validation

### Service Pattern
- Constructor: `repository` + `RedisService`
- Three-tier scope resolution: `spaceId → findBySpace`, `workspaceId → findByWorkspace`, else `findByOwner`
- Dual-entry caching for dual-ID entities (by `resource_id` + by business `id`)
- Pattern invalidation on writes: `redis.invalidatePattern()`

### Repository Pattern
- Extends `FirestoreBaseRepository<T>`
- `@Inject(FIREBASE_APP_TOKEN)` in constructor (string token DI)
- `getCollectionRef(...pathArgs)` → `FirestorePathHelper`
- Spread order: `{ ...rest, resource_id: doc.id }` (destructure first to prevent overwrite)

### Validation
- Zod schemas (NOT class-validator) in `@repo/common`
- `ZodValidationPipe` uses structural typing (`ZodLikeSchema`) — version-decoupled
- Query string booleans: `z.enum(['true','false']).transform(v => v === 'true')`
- Numbers: `z.coerce.number()`

### Error Handling
- Error codes: `{domain}_{6_digit_number}` (e.g., `retailer_000003`)
- Separate files: `feature.errors.ts` (codes) + `feature.messages.ts` (messages)
- `ApiExceptionFilter`: validation errors → passthrough, HTTP errors → `getErrorMessage()`, unhandled → 500

### Testing
- Vitest with `unplugin-swc` (decorator metadata support)
- Controller tests: `Test.createTestingModule` with mock service
- Service tests: `new Service(mockRepo, mockRedis)` direct instantiation
- Mock Redis: `{ get: vi.fn(), set: vi.fn(), del: vi.fn(), invalidatePattern: vi.fn() }`
