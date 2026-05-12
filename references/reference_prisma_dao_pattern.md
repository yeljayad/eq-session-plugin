---
name: Prisma DAO/Helper pattern (mirrors @repo/firebase)
description: Every @repo/database-* package follows the same internal layout as @repo/firebase — a PrismaBaseRepository abstract class (CRUD primitives), PrismaPathHelper-style static helpers (composable query builders), a NestJS DI module, and concrete repositories per domain entity that extend the base.
type: reference
---

**Decision (2026-05-11, user-confirmed):** Each `@repo/database-<svc>` package mirrors the structure of `@repo/firebase` for Firestore — a base repository for CRUD primitives, static helper(s) for path/query composition, and a NestJS module that wires the Prisma client. Concrete domain repos (e.g. `RetailerRepository` in `@repo/database-organization`) extend the base and implement only the parts that vary.

**Why:** The current Firestore DAO pattern in `@repo/firebase` is well-established across all services. Keeping the new Prisma packages structurally identical means: (a) zero learning curve for the team during migration, (b) consumers swap an import path and most of their repo code stays the same shape, (c) reviewers (`eqp-db-reviewer`) get one consistent target to validate.

## Existing Firestore pattern (reference)

`@repo/firebase` provides:

1. **`FirestoreBaseRepository<T extends BaseEntity>`** (abstract) — primitives: `findByResourceId`, `findAll`, `findById`, `findByName`, `create`, `update`, `delete`. Subclasses implement only `getCollectionRef(...pathArgs)`.
2. **`FirestorePathHelper`** (static methods) — builds `CollectionReference`/`DocumentReference` for every collection path in the hierarchy (`ownerRef`, `workspaceRef`, `spaceRef`, `retailersRef`, `customersByWorkspaceRef`, `transactionsByAccountRef`, etc.).
3. **`FirebaseModule` + `FIREBASE_APP_TOKEN`** — DI wiring: services inject the Firebase app, the repo derives `this.db = app.firestore()`.
4. **Concrete repository example** (`apps/business-service/src/retailer/retailer.repository.ts`): extends `FirestoreBaseRepository<Retailer>`, overrides `getCollectionRef(...)` to delegate to `FirestorePathHelper.retailersRef(...)`, adds `findBySpace/findByWorkspace/findByOwner` + a shared `applyFilters` private method.

## Target Prisma pattern per `@repo/database-<svc>` package

### Package internal layout

```
packages/database-<svc>/
├── package.json                  # @repo/database-<svc>
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── src/
│   ├── index.ts                  # re-exports
│   ├── prisma.module.ts          # NestJS module
│   ├── prisma.provider.ts        # PrismaClient factory + lifecycle
│   ├── prisma.token.ts           # DI token (PRISMA_<SVC>_TOKEN)
│   ├── prisma.types.ts           # type re-exports from generated client
│   ├── base/
│   │   ├── prisma-base.repository.ts     # abstract CRUD primitives
│   │   ├── repository-config.ts          # config (e.g. softDelete on/off)
│   │   └── index.ts
│   ├── helpers/
│   │   ├── prisma-tenant.helper.ts       # SET LOCAL app.current_owner_id for RLS
│   │   ├── prisma-lock.helper.ts         # SELECT ... FOR UPDATE via $queryRawUnsafe
│   │   ├── prisma-snapshot.helper.ts     # snapshot denormalization helpers
│   │   └── index.ts
│   └── repositories/                     # OPTIONAL — shared cross-domain repos
│       └── (most concrete repos live in apps/<svc>/src/<domain>/)
└── tsconfig.json
```

### `PrismaBaseRepository<TDelegate, TModel>` (abstract)

Mirrors `FirestoreBaseRepository<T>` but parametrized on the Prisma model delegate (so each subclass binds to a specific `prisma.retailer`, `prisma.customer`, …). The constructor signature mirrors Firestore's: inject the Prisma client, the subclass picks the delegate.

```ts
import { Inject } from '@nestjs/common';
import { PrismaClient } from '../../generated/client'; // or @prisma-clients/<svc>
import { PRISMA_TOKEN } from '../prisma.token';
import { DEFAULT_REPOSITORY_CONFIG, type RepositoryConfig } from './repository-config';

export abstract class PrismaBaseRepository<
    TDelegate extends {
        findUnique: (...args: any[]) => Promise<any>;
        findMany: (...args: any[]) => Promise<any[]>;
        create: (...args: any[]) => Promise<any>;
        update: (...args: any[]) => Promise<any>;
        delete: (...args: any[]) => Promise<any>;
    },
    TModel extends { id: string },
> {
    private readonly config: Required<RepositoryConfig>;

    protected constructor(
        @Inject(PRISMA_TOKEN) protected readonly prisma: PrismaClient,
        config: RepositoryConfig = {},
    ) {
        this.config = { ...DEFAULT_REPOSITORY_CONFIG, ...config };
    }

    /** Subclass returns its Prisma model delegate (e.g. this.prisma.retailer). */
    protected abstract getDelegate(): TDelegate;

    /** Subclass returns the WHERE clause that filters out archived rows when softDelete is on. */
    protected activeWhere(): Record<string, unknown> {
        return this.config.softDelete ? { archived: false } : {};
    }

    async findById(id: string): Promise<TModel | null> {
        return this.getDelegate().findUnique({
            where: { id, ...this.activeWhere() },
        }) as Promise<TModel | null>;
    }

    async findAll(where: Record<string, unknown> = {}): Promise<TModel[]> {
        return this.getDelegate().findMany({
            where: { ...where, ...this.activeWhere() },
        }) as Promise<TModel[]>;
    }

    async create(data: Omit<TModel, 'id' | 'created_at' | 'updated_at'>): Promise<TModel> {
        return this.getDelegate().create({ data }) as Promise<TModel>;
    }

    async update(id: string, data: Partial<Omit<TModel, 'id' | 'created_at'>>): Promise<TModel> {
        return this.getDelegate().update({ where: { id }, data }) as Promise<TModel>;
    }

    /** Soft delete (archived = true) when config.softDelete; otherwise hard delete. */
    async delete(id: string): Promise<void> {
        if (this.config.softDelete) {
            await this.getDelegate().update({ where: { id }, data: { archived: true } });
            return;
        }
        await this.getDelegate().delete({ where: { id } });
    }
}
```

`repository-config.ts` mirrors `@repo/firebase`:

```ts
export interface RepositoryConfig {
    softDelete?: boolean; // default true — schema convention: archived BOOLEAN
}

export const DEFAULT_REPOSITORY_CONFIG: Required<RepositoryConfig> = {
    softDelete: true,
};
```

### Helpers (mirror `FirestorePathHelper` — static, pure, composable)

Where Firestore needed _path_ helpers (because collection refs are nested), Prisma needs _cross-cutting_ helpers since the model graph is flat. The replacements:

#### `PrismaTenantHelper` — RLS scope setter

```ts
import { PrismaClient } from '../../generated/client';

export class PrismaTenantHelper {
    /** Set the per-request RLS scope. Must run inside a $transaction for SET LOCAL to apply. */
    static async setOwner(
        tx: PrismaClient | { $executeRawUnsafe: PrismaClient['$executeRawUnsafe'] },
        ownerId: string,
    ): Promise<void> {
        await tx.$executeRawUnsafe(`SET LOCAL app.current_owner_id = '${ownerId}'::uuid`);
    }
}
```

Used by services that touch RLS-enabled tables (`workspace`, `role`, `user`, `space`). Aligned with `project_postgres_schema_spec.md` §16.

#### `PrismaLockHelper` — `SELECT FOR UPDATE`

```ts
import { PrismaClient } from '../../generated/client';

export class PrismaLockHelper {
    /** Acquire a row-level lock inside a transaction. Returns the locked row or null. */
    static async lockRow<T>(
        tx: PrismaClient | { $queryRawUnsafe: PrismaClient['$queryRawUnsafe'] },
        schema: string,
        table: string,
        id: string,
    ): Promise<T | null> {
        const rows = await tx.$queryRawUnsafe<T[]>(
            `SELECT * FROM "${schema}"."${table}" WHERE id = $1 FOR UPDATE`,
            id,
        );
        return rows[0] ?? null;
    }
}
```

Used by `payment-processing` (balance writes) and `organization` (approval concurrency). Aligned with `project_postgres_schema_spec.md` §12.

#### `PrismaSnapshotHelper` — denormalization

```ts
export class PrismaSnapshotHelper {
    /** Pluck snapshot fields from a customer entity onto a transaction.create payload. */
    static customerSnapshot(customer: {
        id: string;
        name: string;
        email?: string | null; /* … */
    }): Record<string, unknown> {
        return {
            customer_id: customer.id,
            customer_name: customer.name,
            customer_email: customer.email ?? null,
            // … all snapshot columns per schema spec
        };
    }

    static retailerSnapshot(retailer: {
        id: string;
        name: string;
        location?: string | null;
        mcc?: string | null;
    }): Record<string, unknown> {
        return {
            retailer_id: retailer.id,
            retailer_name: retailer.name,
            retailer_location: retailer.location ?? null,
            mcc: retailer.mcc ?? null,
        };
    }
}
```

Used in `payment-processing` when writing `transaction` rows — encapsulates the "freeze entity state at txn time" requirement (`project_postgres_schema_spec.md` §7).

### NestJS module (`prisma.module.ts`)

Mirrors `firebase.module.ts` — provides the `PrismaClient` as a singleton via the `PRISMA_TOKEN` DI token, with `onModuleInit/onModuleDestroy` lifecycle for `$connect/$disconnect`.

```ts
import { Global, Module } from '@nestjs/common';
import { prismaProvider } from './prisma.provider';

@Global()
@Module({
    providers: [prismaProvider],
    exports: [prismaProvider],
})
export class PrismaModule {}
```

```ts
// prisma.provider.ts
import { type FactoryProvider } from '@nestjs/common';
import { PrismaClient } from '../generated/client';
import { PRISMA_TOKEN } from './prisma.token';

export const prismaProvider: FactoryProvider<PrismaClient> = {
    provide: PRISMA_TOKEN,
    useFactory: () => {
        const client = new PrismaClient({
            log:
                process.env['LOG_LEVEL'] === 'debug'
                    ? ['query', 'warn', 'error']
                    : ['warn', 'error'],
        });
        return client;
    },
};
```

### Concrete repository (consumer pattern)

```ts
// apps/organization-service/src/retailer/retailer.repository.ts
import { Inject, Injectable } from '@nestjs/common';
import {
    PrismaBaseRepository,
    PRISMA_TOKEN,
    type PrismaClient,
    type Retailer,
} from '@repo/database-organization';

@Injectable()
export class RetailerRepository extends PrismaBaseRepository<
    typeof PrismaClient.prototype.retailer,
    Retailer
> {
    constructor(@Inject(PRISMA_TOKEN) prisma: PrismaClient) {
        super(prisma, { softDelete: true });
    }

    protected getDelegate() {
        return this.prisma.retailer;
    }

    async findBySpace(spaceId: string, filters: FilterRetailerDto): Promise<Retailer[]> {
        return this.findAll({
            space_id: spaceId,
            ...(filters.status ? { status: filters.status } : {}),
            ...(filters.id ? { id: { startsWith: filters.id.toLowerCase() } } : {}),
        });
    }

    async findByWorkspace(workspaceId: string, filters: FilterRetailerDto): Promise<Retailer[]> {
        return this.findAll({
            space: { workspace_id: workspaceId },
            ...(filters.status ? { status: filters.status } : {}),
        });
    }
}
```

Compared to the Firestore version, the consumer code shrinks because Prisma handles paging/filtering natively — no `applyFilters` helper, no `__name__` cursor gymnastics.

### `index.ts` (package exports)

```ts
export * from './base/prisma-base.repository';
export * from './base/repository-config';
export * from './helpers/prisma-tenant.helper';
export * from './helpers/prisma-lock.helper';
export * from './helpers/prisma-snapshot.helper';
export * from './prisma.module';
export * from './prisma.token';
export * from './prisma.types';
export type * from '../generated/client'; // re-export generated model + enum types
export { PrismaClient } from '../generated/client';
```

## Payment-grade helpers (`@repo/database-payment-processing` only)

Beyond the cross-cutting helpers above, `payment-processing-service` is the ONLY service that writes to financial tables (account, transaction, ledger_entry, transaction_fee, routing_target, routing_attempt, limit_rule). Its package ships extra helpers that encode the atomic-write invariants from the service-ownership spec §3.4 + the schema spec §7/§9/§10/§15.

```
packages/database-payment-processing/src/helpers/
├── prisma-tenant.helper.ts          # (shared)
├── prisma-lock.helper.ts            # (shared)
├── prisma-snapshot.helper.ts        # (shared)
├── prisma-money.helper.ts           # Decimal-safe arithmetic + minor-unit awareness
├── prisma-ledger.helper.ts          # double-entry pair builder + validation
├── prisma-financial-tx.helper.ts    # canonical atomic write: lock → limits → tx → ledger → fees
├── prisma-limit.helper.ts           # evaluate limit_rule against a candidate write (inside the tx)
├── prisma-routing.helper.ts         # routing_target + routing_attempt management
├── prisma-rrn.helper.ts             # RRN generation + uniqueness check
├── prisma-auth-code.helper.ts       # authorization-code generation
├── prisma-approval.helper.ts        # maker-checker concurrency (SELECT FOR UPDATE + threshold)
└── prisma-outbox.helper.ts          # transactional outbox for Kafka audit/notification emission
```

### `PrismaMoneyHelper` — Decimal-safe arithmetic

Uses `Prisma.Decimal` (not JS `number`). Pulls `currency.minor_unit` to format/parse correctly.

```ts
import { Prisma } from '../../generated/client';
type Decimal = Prisma.Decimal;

export class PrismaMoneyHelper {
    static decimal(value: string | number | Decimal): Decimal {
        return new Prisma.Decimal(value);
    }

    /** amount × conversion_rate, rounded HALF_EVEN to 4 dp (matches NUMERIC(18,4)). */
    static convert(amount: Decimal, rate: Decimal): Decimal {
        return amount.mul(rate).toDecimalPlaces(4, Prisma.Decimal.ROUND_HALF_EVEN);
    }

    /** Format an internal NUMERIC(18,4) amount for a given currency's minor unit. */
    static format(amount: Decimal, minorUnit: number): string {
        return amount.toDecimalPlaces(minorUnit, Prisma.Decimal.ROUND_HALF_EVEN).toFixed(minorUnit);
    }

    /** Sum a list of amounts; throws on currency mismatch (caller passes a single-currency list). */
    static sum(amounts: Decimal[]): Decimal {
        return amounts.reduce((a, b) => a.add(b), new Prisma.Decimal(0));
    }
}
```

**Rule:** every money column in/out of Prisma is `Decimal`. Never `as number`. Never `parseFloat`. The helper is the only place arithmetic happens.

### `PrismaLedgerHelper` — double-entry pair

Builds the matching debit/credit pair for a transaction. Validates that the pair balances and that `amount > 0` (enforced by DB CHECK too but fail fast at app layer).

```ts
import { Prisma } from '../../generated/client';

export interface LedgerPair {
    debit: { account_id: string; amount: Prisma.Decimal; currency_code: string };
    credit: { account_id: string; amount: Prisma.Decimal; currency_code: string };
}

export class PrismaLedgerHelper {
    static pair(input: {
        debitAccountId: string;
        creditAccountId: string;
        amount: Prisma.Decimal;
        currencyCode: string;
    }): LedgerPair {
        if (input.amount.lte(0)) throw new Error('ledger amount must be > 0');
        if (input.debitAccountId === input.creditAccountId)
            throw new Error('debit and credit accounts must differ');
        return {
            debit: {
                account_id: input.debitAccountId,
                amount: input.amount,
                currency_code: input.currencyCode,
            },
            credit: {
                account_id: input.creditAccountId,
                amount: input.amount,
                currency_code: input.currencyCode,
            },
        };
    }

    /** Build the raw entries array for prisma.ledger_entry.createMany inside a $transaction. */
    static toEntries(txnId: string, pair: LedgerPair) {
        return [
            { transaction_id: txnId, entry_type: 'debit', ...pair.debit },
            { transaction_id: txnId, entry_type: 'credit', ...pair.credit },
        ];
    }
}
```

Note: `account.ledger_balance` is updated automatically by the `trg_ledger_entry_balance` trigger (§15) — the helper does NOT update balances directly.

### `PrismaFinancialTxHelper` — canonical atomic write

The single function the payment-processing service uses to write any transaction. Encapsulates the BEGIN/COMMIT, the hierarchical lock, the limit check, and the ledger entries. Aligns with service-ownership spec §3.4 ("Atomic write: transaction + ledger entries + balance updates in one BEGIN/COMMIT").

```ts
import { PrismaClient, Prisma } from '../../generated/client';
import { PrismaLockHelper } from './prisma-lock.helper';
import { PrismaLimitHelper } from './prisma-limit.helper';
import { PrismaLedgerHelper, type LedgerPair } from './prisma-ledger.helper';

export interface AtomicWriteInput {
    accountIds: string[]; // accounts to lock (FOR UPDATE) — order matters: lock in sorted order to avoid deadlock
    transaction: Prisma.transactionCreateInput;
    ledger: LedgerPair;
    fees?: Prisma.transaction_feeCreateManyInput[];
    limitContext: {
        // for PrismaLimitHelper
        entityType: 'workspace' | 'space' | 'retailer' | 'customer' | 'card';
        entityId: string;
        serviceCode: string;
    };
}

export class PrismaFinancialTxHelper {
    static async write(prisma: PrismaClient, input: AtomicWriteInput) {
        return prisma.$transaction(
            async (tx) => {
                // 1. Hierarchical lock — sort to avoid deadlock
                for (const id of [...input.accountIds].sort()) {
                    await PrismaLockHelper.lockRow(tx, 'payment_processing', 'account', id);
                }

                // 2. Limit enforcement (inside the same tx — consistent with the lock)
                await PrismaLimitHelper.enforce(tx, input.limitContext, input.transaction.amount);

                // 3. Insert transaction (status: 'pending') — trg_transaction_available_balance fires here
                const txn = await tx.transaction.create({ data: input.transaction });

                // 4. Insert ledger entries — trg_ledger_entry_balance fires here
                await tx.ledger_entry.createMany({
                    data: PrismaLedgerHelper.toEntries(txn.id, input.ledger),
                });

                // 5. Insert fees (if any)
                if (input.fees?.length) {
                    await tx.transaction_fee.createMany({
                        data: input.fees.map((f) => ({ ...f, transaction_id: txn.id })),
                    });
                }

                return txn;
            },
            {
                isolationLevel: Prisma.TransactionIsolationLevel.ReadCommitted,
                timeout: 5_000, // P99 < 500ms (issuer mode) leaves ample headroom
            },
        );
    }
}
```

**Invariants enforced here, not by callers:**

- Account locks are sorted (deadlock-free).
- Limit check happens AFTER the lock, BEFORE any insert.
- Transaction inserted before ledger (FK constraint + trigger ordering).
- All inserts in one BEGIN/COMMIT — partial failures roll back fully.

### `PrismaLimitHelper`

```ts
import { type PrismaClient, Prisma } from '../../generated/client';

export class PrismaLimitHelper {
    static async enforce(
        tx: PrismaClient | Prisma.TransactionClient,
        ctx: { entityType: string; entityId: string; serviceCode: string },
        amount: Prisma.Decimal,
    ): Promise<void> {
        const rules = await tx.limit_rule.findMany({
            where: {
                entity_type: ctx.entityType,
                entity_id: ctx.entityId,
                status: 'active',
                OR: [{ service_id: null }, { service: { service_code: ctx.serviceCode } }],
            },
        });

        for (const rule of rules) {
            // single-tx amount check
            if (
                rule.limit_type === 'transaction_amount' &&
                rule.max_amount &&
                amount.gt(rule.max_amount)
            ) {
                throw new LimitExceededError(
                    rule.id,
                    'transaction_amount',
                    amount,
                    rule.max_amount,
                );
            }
            // cumulative window aggregation (daily/weekly/monthly)
            if (rule.limit_type === 'cumulative_amount' && rule.max_amount) {
                const since = startOfPeriod(rule.period!);
                const agg = await tx.transaction.aggregate({
                    where: {
                        // resolve the entity → account hierarchy at this point; out of scope of this snippet
                        date: { gte: since },
                        status: { in: ['confirmed', 'pending'] }, // include pending — they hold balance
                    },
                    _sum: { amount: true },
                });
                const total = (agg._sum.amount ?? new Prisma.Decimal(0)).add(amount);
                if (total.gt(rule.max_amount))
                    throw new LimitExceededError(
                        rule.id,
                        'cumulative_amount',
                        total,
                        rule.max_amount,
                    );
            }
            // transaction_count window
            if (rule.limit_type === 'transaction_count' && rule.max_count) {
                const since = startOfPeriod(rule.period!);
                const count = await tx.transaction.count({
                    where: { date: { gte: since }, status: { in: ['confirmed', 'pending'] } },
                });
                if (count + 1 > rule.max_count)
                    throw new LimitExceededError(
                        rule.id,
                        'transaction_count',
                        count + 1,
                        rule.max_count,
                    );
            }
        }
    }
}
```

### `PrismaRoutingHelper`

```ts
export class PrismaRoutingHelper {
    static async createTargets(
        tx: Prisma.TransactionClient,
        txnId: string,
        plan: Array<{ integrationId: string; settlementPartnerId: string; priority: number }>,
    ) {
        await tx.routing_target.createMany({
            data: plan.map((p) => ({
                transaction_id: txnId,
                integration_id: p.integrationId,
                settlement_partner_id: p.settlementPartnerId,
                priority: p.priority,
            })),
        });
    }

    static async recordAttempt(
        tx: Prisma.TransactionClient,
        attempt: {
            transaction_id: string;
            routing_target_id: string;
            channel: 'payment' | 'wallet_validation';
            gateway_code?: string;
            response_code?: string;
            error_message?: string;
            attempted_at: Date;
            duration_ms: number;
        },
    ) {
        await tx.routing_attempt.create({ data: attempt });
    }

    static async nextTarget(
        prisma: PrismaClient,
        txnId: string,
    ): Promise<{ id: string; integration_id: string; settlement_partner_id: string } | null> {
        const attempted = await prisma.routing_attempt.findMany({
            where: { transaction_id: txnId },
            select: { routing_target_id: true },
        });
        const excluded = new Set(attempted.map((a) => a.routing_target_id));
        const next = await prisma.routing_target.findMany({
            where: { transaction_id: txnId, id: { notIn: [...excluded] } },
            orderBy: { priority: 'asc' },
            take: 1,
        });
        return next[0] ?? null;
    }
}
```

### `PrismaRrnHelper` + `PrismaAuthCodeHelper`

```ts
export class PrismaRrnHelper {
    /** Generate a candidate RRN; caller MUST INSERT with the column UNIQUE and retry on conflict. */
    static generate(): string {
        // 12-char numeric per ISO 8583 spec: YDDDhhmmssNN (year-day-time-sequence)
        return /* … */;
    }
}

export class PrismaAuthCodeHelper {
    /** 6-char alphanumeric authorization code. Non-cryptographic — uniqueness within the issuer is enough. */
    static generate(): string {
        return /* … */;
    }
}
```

### `PrismaApprovalHelper` — maker-checker concurrency

Encapsulates the §12 pattern verbatim — `SELECT FOR UPDATE` on `approval_request`, check status + maker, insert decision (UNIQUE handles double-vote), increment counter, auto-resolve at threshold.

```ts
export class PrismaApprovalHelper {
    static async approve(
        prisma: PrismaClient,
        requestId: string,
        checkerUserId: string,
        comment?: string,
    ) {
        return prisma.$transaction(async (tx) => {
            const [req] = await tx.$queryRawUnsafe<ApprovalRequest[]>(
                `SELECT * FROM "organization"."approval_request" WHERE id = $1::uuid FOR UPDATE`,
                requestId,
            );
            if (!req) throw new Error('approval request not found');
            if (req.status !== 'pending') throw new Error('already resolved');
            if (req.maker_user_id === checkerUserId)
                throw new Error('maker cannot approve own request');

            await tx.approval_decision.create({
                data: {
                    approval_request_id: requestId,
                    checker_user_id: checkerUserId,
                    decision: 'approved',
                    comment,
                },
            });
            const newCount = req.current_approvals + 1;
            const resolved = newCount >= req.required_approvals;

            return tx.approval_request.update({
                where: { id: requestId },
                data: {
                    current_approvals: newCount,
                    status: resolved ? 'approved' : 'pending',
                    resolved_at: resolved ? new Date() : null,
                },
            });
        });
    }

    static async reject(
        prisma: PrismaClient,
        requestId: string,
        checkerUserId: string,
        comment?: string,
    ) {
        return prisma.$transaction(async (tx) => {
            const [req] = await tx.$queryRawUnsafe<ApprovalRequest[]>(
                `SELECT * FROM "organization"."approval_request" WHERE id = $1::uuid FOR UPDATE`,
                requestId,
            );
            if (!req) throw new Error('approval request not found');
            if (req.status !== 'pending') throw new Error('already resolved');
            if (req.maker_user_id === checkerUserId)
                throw new Error('maker cannot reject own request');

            await tx.approval_decision.create({
                data: {
                    approval_request_id: requestId,
                    checker_user_id: checkerUserId,
                    decision: 'rejected',
                    comment,
                },
            });
            return tx.approval_request.update({
                where: { id: requestId },
                data: { status: 'rejected', resolved_at: new Date() },
            });
        });
    }
}
```

Note: `PrismaApprovalHelper` is shipped in `@repo/database-organization` (where `approval_*` tables live), not `payment-processing`. Listed here only because the pattern is identical to financial locking.

### `PrismaOutboxHelper` — transactional outbox for Kafka

Emitting events to the Kafka spine (notification/audit) from inside the same transaction as the write — without dual-write inconsistency. Two-phase pattern:

1. Insert event into `outbox_event` table inside the same `$transaction` as the financial write.
2. A separate poller (or `pgmq`/`pg_logical_emitter`) drains `outbox_event` → Kafka topic → marks emitted.

```ts
export class PrismaOutboxHelper {
    static async emit(
        tx: Prisma.TransactionClient,
        event: {
            topic: string;
            key: string;
            payload: Record<string, unknown>;
            headers?: Record<string, string>;
        },
    ) {
        await tx.outbox_event.create({
            data: {
                topic: event.topic,
                key: event.key,
                payload: event.payload as Prisma.JsonObject,
                headers: event.headers ?? {},
            },
        });
    }
}
```

Note: `outbox_event` is not yet in the 44-table spec — Teddy's schema does not show it. This is an open design point — the spec says "fire-and-forget via message broker" but doesn't specify the producer-side delivery guarantee. **Flag to Teddy.**

## DI tokens convention

Per-package DI tokens prevent collision when a service imports its own package + a shared helper consumes a different one (rare, but possible during migration):

| Package                             | Token export                      | Symbol value                  |
| ----------------------------------- | --------------------------------- | ----------------------------- |
| `@repo/database-iam`                | `PRISMA_IAM_TOKEN`                | `'PRISMA_IAM'`                |
| `@repo/database-organization`       | `PRISMA_ORGANIZATION_TOKEN`       | `'PRISMA_ORGANIZATION'`       |
| `@repo/database-payment-processing` | `PRISMA_PAYMENT_PROCESSING_TOKEN` | `'PRISMA_PAYMENT_PROCESSING'` |
| `@repo/database-session`            | `PRISMA_SESSION_TOKEN`            | `'PRISMA_SESSION'`            |
| `@repo/database-audit`              | `PRISMA_AUDIT_TOKEN`              | `'PRISMA_AUDIT'`              |

Each package's `PrismaModule` provides ONLY its own token. A service that imports two packages would get two separate `PrismaClient` singletons (one per generated client) — which is also why importing two is a violation.

## Mapping table — Firestore → Prisma pattern

| `@repo/firebase`                        | `@repo/database-<svc>`                                             | Notes                                                                                        |
| --------------------------------------- | ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| `FirestoreBaseRepository<T>`            | `PrismaBaseRepository<TDelegate, TModel>`                          | Both abstract; both expose `findById/findAll/create/update/delete`. Subclass picks delegate. |
| `getCollectionRef(...pathArgs)`         | `getDelegate()`                                                    | Firestore needed path args because of subcollections; Prisma has a flat model graph.         |
| `FirestorePathHelper.*Ref(...)`         | `PrismaTenantHelper` / `PrismaLockHelper` / `PrismaSnapshotHelper` | Different need shape: PG doesn't need path composition, but needs RLS + locking + snapshots. |
| `FIREBASE_APP_TOKEN`                    | `PRISMA_TOKEN` (or `PRISMA_<SVC>_TOKEN` if collision)              | String DI token per package.                                                                 |
| `FirebaseModule`                        | `PrismaModule` (one per package, `@Global()`)                      | Each package's module is global within its consuming service.                                |
| `RepositoryConfig.withValidationStatus` | `RepositoryConfig.softDelete`                                      | Firestore had a validation-status filter; Postgres uses `archived BOOLEAN` per schema spec.  |
| `BaseEntity { resource_id }`            | Prisma-generated model (`{ id, created_at, updated_at, ... }`)     | The "resource_id is doc.id" gymnastic disappears — PG uses real UUID PKs.                    |

## Apps consume one and only one `@repo/database-*`

- `iam-service` → `@repo/database-iam`
- `organization-service` → `@repo/database-organization`
- `payment-processing-service` → `@repo/database-payment-processing`
- `session-service` → `@repo/database-session`
- `audit-service` → `@repo/database-audit` — **in flight in PR #68 (`feat/audit-log` by ttchimou). The base+helper pattern above applies, but any audit-package scaffolding belongs to that PR — do not duplicate.**
- Importing two packages from a service = data-ownership violation; the `eqp-db-reviewer` agent must flag it.

## Migration sequencing

When scaffolding actual packages later:

1. Start with **one pilot package** end-to-end (recommend `@repo/database-iam` — smallest at 5 tables) to validate the pattern lands cleanly, generator output, build pipeline, NestJS DI works.
2. Once green: scaffold the other 4 packages with identical structure.
3. Migrate `transaction-service` → `payment-processing-service` (rename + swap `@repo/database` import for `@repo/database-payment-processing`).
4. Consolidate `business-service` + `card-service` → `organization-service` (swap to `@repo/database-organization`).
5. Delete `packages/database` once nothing depends on it (or shrink it to host reference data if that decision lands on "tables").

## Sign-off blockers

- Pattern itself: confirm with user before scaffolding any package
- Cross-package types: if a service repository accepts a parameter typed as another service's model (e.g. `payment-processing` accepting a `Customer` ref for snapshotting), the type comes from `@repo/common`, NEVER by importing the foreign Prisma package. The `PrismaSnapshotHelper` parameter types are plain interfaces, not Prisma types.
- Prisma version: confirm 5+ (multi-schema GA, typed delegates)
