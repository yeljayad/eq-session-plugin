---
name: Backend Service Development Blueprint
description: Complete NestJS service architecture blueprint based on retailer/business-crud-service — DI wiring, controller patterns, service caching, repository layer, DTO/schema, error handling, idempotency
type: reference
---

## Canonical Template: `apps/business-crud-service` (retailer domain)

Follow this exact layered architecture for any new NestJS feature.

---

## 1. Module Wiring (`feature.module.ts`)

```typescript
@Module({
    controllers: [FeatureController],
    providers: [FeatureService, FeatureRepository],
    exports: [FeatureService],  // only if consumed by other modules (e.g., gRPC controller)
})
export class FeatureModule {}
```

- **No imports needed** — `FirebaseModule` and `RedisModule` are `@Global()` at AppModule level
- **No factory for repository** — use `@Inject(FIREBASE_APP_TOKEN)` directly in the repository constructor
- Register in `AppModule.imports` array

## 2. AppModule Guard Stack

Seven global guards registered as `APP_GUARD` via `useFactory` (NOT `useClass` — gives explicit DI control). ApiKeyGuard is FIRST (per-org X-API-Key auth path); the rest are the JWT path.

```
1. ApiKeyGuard     → inject: [Reflector, API_KEY_VALIDATOR, RedisService, ApiKeyRateLimitService]
2. TokenGuard      → inject: [Reflector, FirebaseAuthService]
3. DeviceGuard     → inject: [Reflector]  (stub, returns true)
4. UserGuard       → inject: [Reflector, AUTH_REPOSITORY_TOKENS.USER]
5. PermissionGuard → inject: [Reflector]
6. RoleScopeGuard  → inject: [Reflector]
7. ScopeGuard      → inject: [Reflector, FIREBASE_APP_TOKEN, RedisService]
```

Also registered:
- `IdempotencyInterceptor` as `APP_INTERCEPTOR` → inject: `[Reflector, IdempotencyService]`
- `ApiExceptionFilter` as `APP_FILTER` → `useClass`
- `{ provide: AUTH_REPOSITORY_TOKENS.USER, useClass: UserRepository }` — string token DI for guard decoupling
- `{ provide: API_KEY_VALIDATOR, useClass: ApiKeyValidatorInProcess }` (iam-service ONLY) OR `{ provide: API_KEY_VALIDATOR, useClass: ApiKeyValidatorClient }` (other 10 services — gRPC client to iam-service via `IAM_API_KEY_VALIDATOR` in `GrpcClientModule.forServices()`)

**DO NOT** use `@Idempotent` on API-key issuance/rotate endpoints (`POST /api-keys`, `POST /api-keys/:id/rotate`) — the `IdempotencyInterceptor` caches the response body in Redis, which would re-serve the raw secret on replay.

## 3. Bootstrap (`main.ts`)

```typescript
import '@repo/observability/instrumentation';  // MUST be first import (side-effect)

const app = await NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter());
app.useLogger(ObservabilityModule.getLogger());
app.enableCors();
app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.GRPC,
    options: { package: 'feature_package', protoPath: join(__dirname, '.../@repo/protos/src/feature.proto'), url: `0.0.0.0:${GRPC_PORT}` },
});
await app.startAllMicroservices();  // gRPC before HTTP
await app.listen(HTTP_PORT, '0.0.0.0');
```

## 4. Controller Layer (`feature.controller.ts`)

### Route table pattern

| Method | Path | Decorators | HttpCode |
|--------|------|-----------|----------|
| GET | `/features` | `@RequirePermission('feature', 'R')` `@ValidateScope('owner')` `@ValidateOptionalScope('workspace', 'space', 'retailer')` | 200 |
| GET | `/features/:id` | `@RequirePermission('feature', 'R')` `@ValidateScope('owner', 'workspace', 'space')` | 200 |
| POST | `/features` | `@Idempotent()` `@RequirePermission('feature', 'C')` `@ValidateScope('owner', 'workspace', 'space')` `@HttpCode(201)` | 201 |
| PATCH | `/features/:id` | `@Idempotent(CACHE_TTL.LONG)` `@RequirePermission('feature', 'U')` `@ValidateScope('owner', 'workspace', 'space')` | 200 |
| DELETE | `/features/:id` | `@Idempotent(CACHE_TTL.LONG)` `@RequirePermission('feature', 'D')` `@ValidateScope('owner', 'workspace', 'space')` `@HttpCode(204)` | 204 |

### Validation pattern — ZodValidationPipe per-parameter

```typescript
@Post()
@Idempotent()
@HttpCode(HttpStatus.CREATED)
@RequirePermission('feature', 'C')
@ValidateScope('owner', 'workspace', 'space')
async create(
    @Query(new ZodValidationPipe(ScopeQuerySchema)) scope: ScopeQuery,
    @Body(new ZodValidationPipe(CreateFeatureSchema)) dto: CreateFeatureDto,
    @CurrentUser() _currentUser: User,
): Promise<FeatureResponseDto> { ... }
```

### Key patterns
- `@CurrentUser() _currentUser` — prefix `_` when only needed for guard validation
- `findAll` uses separate `@Query()` extractions for optional scope params (not ScopeQuerySchema)
- Response: always map through `toFeatureResponseDto()` / `toFeatureResponseDtoList()` — never return raw Firestore docs
- `:id` URL param = business ID (not resource_id) for Retailer/Customer

## 5. Service Layer (`feature.service.ts`)

### Constructor

```typescript
constructor(
    private readonly repository: FeatureRepository,
    private readonly redis: RedisService,
) {}
```

### Three-tier scope resolution for `findAll`

```
if (spaceId)     → repo.findBySpace(ownerId, workspaceId, spaceId, filters)
else if (wsId)   → repo.findByWorkspace(ownerId, workspaceId, filters)
else             → repo.findByOwner(ownerId, filters)
```

### Caching strategy (dual-entry for dual-ID entities)

**List cache:**
- Key: `features:owner:{ownerId}:ws:{wsId}:sp:{spId}:f:{md5_8chars}` — filter hash appended
- TTL: `CACHE_TTL.LIST = 120s`
- Invalidation: `redis.invalidatePattern(RedisKeys.ownerFeaturePattern(ownerId))` — wipes all scope variants

**Item cache (two entries per entity):**
- `feature:{resource_id}` — Firestore doc ID
- `feature:business:{id}` — business-provided ID
- TTL: `CACHE_TTL.ITEM = 600s`
- Both populated on read via `Promise.all`
- Both deleted on write via `Promise.all` + list invalidation

### CRUD patterns

**Create:** Check for conflicts first (`repo.findById(dto.id)`) → `ConflictException` if exists. Enrich DTO with computed fields (lowercase names, resolved country/currency, status defaults). Then `repo.create(...)`. Invalidate lists.

**Update:** Resolve entity via `findOneByBusinessId` → get `resource_id`. Call `repo.update(resource_id, dto, ...)`. Invalidate both item caches + lists. Re-fetch fresh entity.

**Delete:** Resolve entity via `findOneByBusinessId` → get `resource_id`. Call `repo.delete(resource_id, ...)`. Invalidate both item caches + lists.

### Error handling

```typescript
throw new NotFoundException(FEATURE_ERRORS.DOESNT_EXIST);     // 'feature_000003'
throw new ConflictException(FEATURE_ERRORS.ID_SHOULD_BE_VALID); // 'feature_000002'
```

`ApiExceptionFilter` translates error codes to messages via `getErrorMessage()`.

## 6. Repository Layer (`feature.repository.ts`)

### Injection pattern (matches business-crud convention)

```typescript
@Injectable()
export class FeatureRepository extends FirestoreBaseRepository<Feature> {
    constructor(@Inject(FIREBASE_APP_TOKEN) protected readonly app: FirebaseApp.App) {
        super(app);
    }

    protected getCollectionRef(ownerId: string, workspaceId: string, spaceId: string): CollectionReference {
        return FirestorePathHelper.retailersRef(this.db, ownerId, workspaceId, spaceId);
    }
}
```

**Key:** `@Inject(FIREBASE_APP_TOKEN)` on constructor — string token DI. NOT a factory in the module.

### FirestoreBaseRepository provides
- `findByResourceId(id, ...pathArgs)` — by Firestore doc ID
- `findById(id, ...pathArgs)` — by business `id` field query
- `create(data, ...pathArgs)` — auto-generates doc ID, sets `resource_id` + `resource_path`
- `update(resourceId, data, ...pathArgs)` — does NOT return updated doc
- `delete(resourceId, ...pathArgs)`

### Filter pattern in `applyFilters(query, filters)`

1. Always `where('archived', '==', false)`
2. Optional equality filters: `status_name`, `validation_status_name`
3. Prefix range queries: `>= start` and `<= start\uf8ff`
4. Name filter on `name_lower_case` (requires `name_lower_case = name.toLowerCase()` on writes)
5. Cursor pagination: `collectionGroup('feature').where('__name__', '==', startAfterId)` then `startAfter(snap)`
6. `.limit(filters.pageSize)` always

### Three scope methods
- `findBySpace(ownerId, wsId, spId, filters)` — canonical path
- `findByWorkspace(ownerId, wsId, filters)` — denormalized workspace-level collection
- `findByOwner(ownerId, filters)` — denormalized owner-level collection

## 7. DTO/Schema Layer (`@repo/common`)

### Zod schemas (NOT class-validator)
- `CreateFeatureSchema` — full validation with `z.string().min().max()`, enums, defaults
- `UpdateFeatureSchema` — `CreateFeatureSchema.partial()`
- `FilterFeatureSchema` — extends `PaginationSchema` + domain filters + sortBy + order
- `ScopeQuerySchema` — `{ ownerId, workspaceId, spaceId }` all `z.string().min(1)`
- `OwnerScopeQuerySchema` — `{ ownerId }` only

### Response DTO
Plain TypeScript interface + pure mapper function `toFeatureResponseDto()`. No class-transformer. Explicit field-by-field projection strips internal fields.

### Query string transforms
- Booleans: `z.enum(['true','false']).transform(v => v === 'true').default(false)`
- Numbers: `z.coerce.number()`

## 8. Error Codes (`@repo/common/src/errors/`)

Pattern: `{domain}_{6_digit_number}`. Separate files for codes (`feature.errors.ts`) and messages (`feature.messages.ts`). All merged into `ALL_MESSAGES` map. `getErrorMessage(code)` resolves with fallback.

## 9. Idempotency

`@Idempotent()` — default 24h TTL. `@Idempotent(CACHE_TTL.LONG)` — 1h for updates/deletes.

Key extraction: `idempotency-key` header if present, else `md5({method, url, body})`.
State machine: `check(key)` → new/replay/locked → `lock` → execute → `store` or `unlock` on error.

## 10. Test Patterns

- `describe('ClassName')` → `it('should verb noun')`
- `Test.createTestingModule` with mock providers: `{ provide: Service, useValue: mockService }`
- Service tests: instantiate directly with mock deps (`new Service(mockRepo, mockRedis)`)
- Mock Redis: `{ get: vi.fn(), set: vi.fn(), del: vi.fn(), invalidatePattern: vi.fn() }`
- Test relaxations in ESLint config: `max-lines-per-function: 100`, `no-magic-numbers: off`, `no-console: off`
