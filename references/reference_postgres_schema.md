---
name: Confirmed PostgreSQL Schema v1.0 (2026-04-27)
description: Confirmed PostgreSQL schema for EquitiPay platform — 44 tables, PG15+, design conventions (UUID PKs, NUMERIC(18,4) money, TIMESTAMPTZ, soft delete, RLS, polymorphic FKs, immutability triggers).
type: reference
---

**Source:** Tech Spec "EquitiPay PostgreSQL Database Schema v1.0", presenter Teddy, dated 2026-04-27 (presented 2026-04-28 alongside the service-ownership spec). Status: Draft awaiting Architecture Council + Payment Domain Owner sign-off.

**Why:** This is the canonical schema for the Firestore → Postgres migration finalization. All Prisma schema work, new repositories, and migrations must conform to this layout.

**How to apply:** Treat as the target schema. When adding new tables/columns, follow the conventions below. When writing repositories, follow the polymorphic + locking patterns. Flag any code that diverges. Do NOT generate migrations unilaterally — the schema is still draft.

---

## ⚠️ Direct contradiction with service-ownership spec

The 2026-04-28 service-ownership spec said: _"ISO & MOP reference data (`country`, `currency`, `language`, `method_of_payment`) are NOT database tables. They are stored as static TypeScript constants in `@equitipay/reference-data`. Columns are plain strings — no FK constraints."_

This 2026-04-27 schema spec defines all four AS actual tables with natural-key PKs AND requires FK constraints (`country_code CHAR(2) REFERENCES country(alpha2_code)`, etc.). The two specs are presented as the same architecture milestone but disagree on this point.

**Action:** Flag to Teddy before any migration starts. Pending resolution, the schema doc (this file) is the more recent of the two on table count (44 vs 40 — delta = 4 reference tables), suggesting the table-based version is the intended final.

---

## Core conventions

| Convention     | Rule                                                                                                                                                                            |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Primary keys   | `UUID PRIMARY KEY DEFAULT gen_random_uuid()` — except reference tables (natural keys: `alpha2_code`, `alphabetic_code`, ISO 639-1 `id`, MOP `name`)                             |
| Money          | `NUMERIC(18,4)` — never float, never BIGINT-as-minor-units                                                                                                                      |
| Timestamps     | `TIMESTAMPTZ NOT NULL DEFAULT now()` — never BIGINT or VARCHAR                                                                                                                  |
| Naming         | snake_case columns and tables (matches current Prisma `@map`/`@@map`)                                                                                                           |
| Soft delete    | `status entity_status` + `archived BOOLEAN` — no hard deletes, FKs are safety net                                                                                               |
| External IDs   | `reference VARCHAR(64) UNIQUE` (nullable) on customer/retailer/account for user-supplied IDs                                                                                    |
| Enums vs CHECK | ENUM only when shared across tables (`entity_status`). CHECK for single-table value sets (account_type, card_type, gender, ...). Rationale: ENUM values can't be removed in PG. |

## Enum

`entity_status` = `('active', 'inactive', 'cancelled')` — used by workspace, space, retailer, customer, account, card, card_program, user, service, limit_rule, approval_policy, apple_processing_certificate.

## 44 tables organized by domain

### Reference (4) — natural keys

- `country` (alpha2_code PK, alpha3, numeric, name)
- `currency` (alphabetic_code PK, numeric, minor_unit, name) — `minor_unit` drives amount formatting (USD=2, JPY=0, KWD=3)
- `language` (id CHAR(2) PK = ISO 639-1)
- `method_of_payment` (name PK, domestic_solution BOOLEAN) — ~130 values, table not ENUM because adds are inserts not migrations + carries metadata
- `service_definition` (code VARCHAR(10) PK, name, category, requires_configuration) — 50 service codes; `requires_configuration=true` means `service` row needed for entity capability
- `permission` (id VARCHAR(100) PK in `entity:action` format, name, category)

### Organizational hierarchy (3)

- `owner` (id, company_name, email UNIQUE, application_name, addresses, currency_code, country_code)
- `workspace` (id, owner*id FK, name, status, archived, currency_code, validation_max_level, auto_generate*\*\_id flags) — UNIQUE(owner_id, name)
- `space` (id, workspace_id FK, name, status, archived) — UNIQUE(workspace_id, name)

### Retailer & Customer (3)

- `retailer` (id, reference UNIQUE, space_id FK, mcc, tcc, MCC/TCC categories, soft_pos_activated, cart_activated, google/apple_merchant_id, google_public_key, max_number_of_users, etc.)
- `retailer_supported_currency` (id, retailer_id FK, currency_code FK) — UNIQUE(retailer_id, currency_code)
- `customer` (id, reference UNIQUE, space_id FK, retailer_id FK NULLABLE, location, currency_code) — `retailer_id NULL = issuing mode`, non-null = acquiring

### Financial accounts (1) — polymorphic

- `account` (id, reference UNIQUE, owner_type, workspace_id/space_id/retailer_id/customer_id, account_type CHECK in 4 values, ledger_balance NUMERIC(18,4), available_balance NUMERIC(18,4), currency_code)
- `chk_account_owner`: exactly one of the 4 FKs must be set matching `owner_type`
- Dual balance model: `available = ledger - holds + pending_credits`

### Card system (5)

- `bin` (id, workspace_id FK, bin_number, issuer_name, country/currency, HSM key indexes: cvk/pvk/zek/pbk/mk_ac/mk_enc/mk_mac) — UNIQUE(workspace_id, bin_number)
- `card_program` (id, workspace_id FK, bin_id FK, scheme, bin_range_min/max, card_type CHECK in [charge,debit,revolving,prepaid], card_length, card_life_cycle months, pin_length, pin_consecutive_attempts, replacement_min_expiry_days)
- `card` (id, customer_id FK, card_program_id FK, encrypted_pan, name_on_card, expiry mm/yy, current_card_sequence_id) — **circular FK to card_sequence added via ALTER TABLE after both exist**
- `card_sequence` (id, card_id FK, reason CHECK in [initial,expiry,lost,stolen,damaged,compromised,customer_request], personalization_status_name, start/end dates)
- `saved_card` (id, customer_id FK, encrypted_pan, masked_pan, token, gateway_merchant_id, wallet_provider, is_payout_eligible, payout_type CHECK in [fast_funds,standard], not_tp_confirmed)

### Transactions & Ledger (10)

- `transaction` (id, account*id FK, amount, currency, status CHECK in 10 values, original_amount/conversion_rate, settlement_amount/rate/partner, clearing_date, \*\*customer*_ snapshot fields**, **retailer\__ snapshot fields\*_, gateway\__, acquirer_response_code, authorization_code, rrn, stan, integration_id, order_id, reconciliation_status, ai_risk_score/status/review_decision, provider_data JSONB)
- `transaction_fee` (per-transaction, fee_provider, amount, multi-currency conversion rates, fee_percentage/fixed/min/max)
- `ledger_entry` (id, transaction_id FK, account_id FK, entry_type CHECK [debit|credit], amount > 0 CHECK) — **IMMUTABLE via trigger**
- 1:1 detail tables (UNIQUE transaction_id):
    - `transaction_card_detail` (is*3ds, masked_pan/with_bin, scheme_type, bin*\* snapshot)
    - `transaction_bank_transfer_detail` (virtual_account_id FK, source_bank_account_id FK, bank_reference, remittance_info)
    - `transaction_crypto_detail` (crypto_currency_code, network, wallet_address, destination_tag, tx_hash, travel_rule JSONB)
    - `transaction_wallet_detail` (wallet_provider, device_id, token_requestor_id, cryptogram, eci_indicator)
    - `transaction_cash_detail` (agent_id/ref/name, city, transfer_number, receipt_number, collected_at, payment_method_code)
    - `transaction_digital_wallet_detail` (phone_number, account_identifier, account_holder_name) — covers mobile money + e-wallets + UPI
- `limit_rule` (polymorphic entity_type/id, limit_type CHECK [transaction_amount|cumulative_amount|transaction_count], period CHECK [per_transaction|daily|weekly|monthly], service_id NULL=all, min/max_amount, max_count)

### Virtual accounts & bank transfer (2)

- `virtual_account` (id, customer_id FK, integration_id, iban, currency_code, status) — UNIQUE(customer, integration, currency)
- `source_bank_account` (id, customer_id FK, virtual_account_id FK, iban, bank_name, account_holder_name, first_seen_at) — UNIQUE(virtual_account_id, iban)

### Service config (1) — polymorphic

- `service` (id, entity_type CHECK in 6 values [owner|workspace|space|retailer|customer|card], entity_id, service_code FK→service_definition, account_id FK NULLABLE, currency_code) — UNIQUE(entity_type, entity_id, service_code). No FK on entity_id — polymorphic, app-layer integrity.

### Session & routing (3)

- `session` — huge table (~50 columns), JSONB blobs for card/card*details/authentication/device, mop, retailer_id, customer*_, crypto\__, integration*id, routing_rule*_, gateway\__ (session/order/charge/url IDs), transaction_id FK, status CHECK [active|pending|paid|failed|expired], **self-referencing last_session_id for retry**
- `routing_target` (transaction_id FK, integration_id, settlement_partner_id, priority)
- `routing_attempt` (transaction_id FK, routing_target_id FK, channel CHECK [payment|wallet_validation], gateway_code, response_code, duration_ms)

### RBAC (4)

- `role` (id, owner_id FK, name, description, is_default) — UNIQUE(owner_id, name). 9 default roles per owner: Owner, Workspace Admin/User, Space Admin/User, Retailer Admin/User, Customer Admin/User.
- `role_permission` (id, role_id FK, permission_id FK) — UNIQUE(role_id, permission_id)
- `"user"` (id, firebase_auth_id, owner_id, role_id, name fields, email, mobile, status, gender CHECK [female|male], screening_status CHECK [verified|not_verified], identity_type CHECK [national_id|passport|driver_licence], MFA mobile) — UNIQUE per owner: (email), (user_name), (firebase_auth_id)
- `user_access_scope` (user_id FK, scope_type CHECK [workspace|space|retailer|customer], scope_entity_id) — UNIQUE(user_id, scope_type, scope_entity_id). **Hierarchical**: workspace scope grants all spaces/retailers/customers below.

### Maker-Checker (3)

- `approval_policy` (id, workspace_id FK, entity_type, service_code FK→service_definition, required_approvals) — UNIQUE(workspace, entity_type, service_code)
- `approval_request` (id, approval_policy_id FK, status CHECK [pending|approved|rejected], service_code FK, target_entity_type/id, payload JSONB, maker_user_id FK, required_approvals, current_approvals, resolved_at)
- `approval_decision` (id, approval_request_id FK, checker_user_id FK, decision CHECK [approved|rejected], comment) — UNIQUE(request, checker). **IMMUTABLE via trigger**

### Audit (1)

- `audit_log` (workspace_id FK, date, activity_status_name, activity_source_name, service_id/name, entity_type/id, user_id FK, user_email snapshot, space/customer/retailer FK, transaction_amount/currency, masked_pan, approval_request_id FK, diff JSONB) — **IMMUTABLE via trigger**

### Apple Pay (2)

- `apple_merchant_identity_certificate` (retailer_id FK, apple_merchant_id, secret_manager_uri, secret_id, valid_from/until)
- `apple_processing_certificate` (retailer_id FK, apple_merchant_id, public_key_hash, private_key_id, protocol_version, valid_from/until, status)

## Triggers

### Balance management (auto)

- `trg_ledger_entry_balance` AFTER INSERT on ledger_entry → updates `account.ledger_balance` (debit -=, credit +=)
- `trg_transaction_available_balance` AFTER INSERT OR UPDATE OF status on transaction:
    - INSERT status=pending → hold (available -= amount)
    - pending → voided/cancelled/rejected → release (available += OLD.amount)
    - pending → confirmed → no change (already held)
    - confirmed → refunded → release (available += OLD.amount)

### Immutability protection (raises exception on UPDATE/DELETE)

- `ledger_entry`
- `audit_log`
- `approval_decision`

## Row-Level Security

RLS enabled only on top of the hierarchy where the subquery cost is bounded:

- `workspace` (direct `owner_id = current_setting('app.current_owner_id')`)
- `role` (direct owner_id)
- `"user"` (direct owner_id)
- `space` (via workspace IN subquery)

**Deeper tables** (retailer, customer, account, transaction, card, ...) intentionally do NOT use RLS — too expensive. Tenant isolation must be enforced at app layer using `workspace_id` / `owner_id` resolved from JWT.

Application MUST set `SET LOCAL app.current_owner_id = '<uuid>'` per request.

## Indexing strategy

- Hierarchy FK columns indexed for parent lookups (workspace.owner_id, space.workspace_id, retailer.space_id, customer.space_id+retailer_id)
- Polymorphic discriminators indexed as composite (e.g. `service(entity_type, entity_id)`, `audit_log(entity_type, entity_id)`)
- Transaction (high-volume) has 7 single-column indexes: account_id, date, status, customer_id, retailer_id, gateway_transaction_id, order_id
- All 1:1 transaction detail tables index their query columns (masked_pan, bin_number, wallet_provider, tx_hash, wallet_address, agent_id, transfer_number, phone_number, account_identifier)
- Text search: `lower(name)` indexes on retailer, customer, user

## Critical patterns for repositories

### Polymorphic FKs (no DB constraint on entity_id)

- `account.owner_type` + 4 nullable FKs (CHECK constraint enforces exactly one)
- `service.entity_type` + `entity_id` (6 entity types, no FK, app-layer integrity)
- `limit_rule.entity_type` + `entity_id` (5 entity types)
- `user_access_scope.scope_type` + `scope_entity_id` (4 scope types)
- `audit_log.entity_type` + `entity_id` (any entity)

### SELECT FOR UPDATE pattern (Prisma)

Used for approval concurrency and (per service-ownership spec) for `payment-processing-service` balance writes. Prisma does NOT support `FOR UPDATE` natively — use `tx.$queryRawUnsafe` inside `$transaction`. Lock holds until txn commits.

### Circular FK

`card.current_card_sequence_id → card_sequence.id` and `card_sequence.card_id → card.id` — schema creates `card_sequence` after `card`, then `ALTER TABLE card ADD CONSTRAINT ...`. Prisma needs `@relation(... onDelete: NoAction)` or the migration needs to be split.

### Snapshot denormalization

`transaction.customer_*` and `transaction.retailer_*` fields are intentional snapshots — frozen at txn time, decoupled from live entity. Card/BIN snapshots live in `transaction_card_detail`. Repositories must populate snapshots on insert, never JOIN through.

### `provider_data JSONB`

Raw PSP response stored unstructured. MOP-specific structured data goes in the matching `transaction_*_detail` 1:1 table.

## Gaps vs current Prisma schema (`packages/database/prisma/schema.prisma`)

To verify on next migration sprint:

- Current schema has partial coverage (transaction/account/ledger families per CLAUDE.md hint of `Decimal(18,4)`)
- 44 tables target — current model count needs auditing
- Reference tables (country/currency/language/MOP) need a decision (conflict above)
- Triggers + RLS + immutability — Prisma does not manage these natively; must be raw SQL in migrations (`$queryRaw`-style) or applied out-of-band

## Sign-off blockers

Before this becomes the canonical Prisma schema:

1. Resolve reference-data conflict (tables vs `@equitipay/reference-data` TS constants)
2. Architecture Council + Payment Domain Owner sign-off
3. Migration plan from current Firestore + partial PG hybrid → full 44-table PG
4. Decide trigger/RLS management strategy (in-migration raw SQL vs out-of-band setup)
