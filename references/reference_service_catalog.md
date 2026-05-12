---
name: Confirmed Service Catalog & Data Ownership (2026-04-28)
description: Confirmed backend service architecture — 11 services, 40 PG tables, Firestore for config-service only. Source of truth for service boundaries, table ownership, comms patterns, SLAs.
type: reference
---

**Source:** Tech Spec "EquitiPay Backend Services Data Ownership", presenter Teddy, 2026-04-28. Status: Draft, awaiting Architecture Council + Payment Domain Owner sign-off — but confirmed by user as the architectural direction.

**Why:** This is a major restructuring of the current service catalog (in CLAUDE.md). Service renames, consolidations, and a new database boundary model. All future backend work should target this layout.

**How to apply:** Treat this as the canonical target architecture. Flag any code/PR that contradicts these boundaries. Update CLAUDE.md/AGENTS.md/refs in a coordinated cleanup PR. Do not start large refactors without an approved migration plan — the codebase still uses the old names.

---

## Confirmed services (11)

| Service                      | Domain                                                   | DB tables                                                                                                                                                                                                                                                                                                                     |
| ---------------------------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `iam-service`                | Identity, RBAC, scoped access                            | role, role_permission, user, user_access_scope, permission (5)                                                                                                                                                                                                                                                                |
| `orchestration-service`      | Pure flow coordinator, zero domain logic                 | — (stateless)                                                                                                                                                                                                                                                                                                                 |
| `organization-service`       | All entity lifecycle + cards + HSM + approvals (LARGEST) | owner, workspace, space, retailer, retailer_supported_currency, customer, service, approval_policy, approval_request, approval_decision, bin, card_program, card, card_sequence, saved_card, virtual_account, source_bank_account, apple_merchant_identity_certificate, apple_processing_certificate, service_definition (20) |
| `payment-processing-service` | Financial engine, atomic BEGIN/COMMIT writes             | account, transaction, transaction_fee, transaction_card_detail, transaction_bank_transfer_detail, transaction_crypto_detail, transaction_wallet_detail, transaction_cash_detail, transaction_digital_wallet_detail, ledger_entry, limit_rule, routing_target, routing_attempt (13)                                            |
| `payment-gateway-service`    | PSP translation layer                                    | — (stateless)                                                                                                                                                                                                                                                                                                                 |
| `config-service`             | Routing rules, integration configs                       | — (Firestore only)                                                                                                                                                                                                                                                                                                            |
| `session-service`            | Acquiring ecommerce session lifecycle                    | session (1)                                                                                                                                                                                                                                                                                                                   |
| `risk-service`               | Fraud, velocity, blacklists                              | — (TBD)                                                                                                                                                                                                                                                                                                                       |
| `notification-service`       | SMS/email/push/webhook delivery                          | — (TBD)                                                                                                                                                                                                                                                                                                                       |
| `callback-service`           | Inbound PSP webhooks → normalize → orchestration         | — (TBD)                                                                                                                                                                                                                                                                                                                       |
| `audit-service`              | Async audit trail consumer                               | audit_log (1)                                                                                                                                                                                                                                                                                                                 |

**Total: 40 PostgreSQL tables.**

## Renames & consolidations vs. current CLAUDE.md

| Current name                        | New name                     | Notes                                                                                                     |
| ----------------------------------- | ---------------------------- | --------------------------------------------------------------------------------------------------------- |
| `transaction-service`               | `payment-processing-service` | Renamed; owns financial atomic writes (13 tables)                                                         |
| `business-service` + `card-service` | `organization-service`       | Consolidated into one service (20 tables incl. cards, BINs, approvals, Apple Pay certs, virtual accounts) |
| `fraud-service`                     | `risk-service`               | Renamed                                                                                                   |
| (none)                              | `audit-service`              | NEW — async consumer of audit events                                                                      |

## Architectural rules confirmed

- **Single PostgreSQL** (one DB). Table ownership enforced at app layer, NOT DB level. No cross-service SQL.
- **Inter-service data access via HTTP/gRPC only** — never direct DB queries.
- **config-service is the only Firestore consumer**. All other Firestore usage in the current codebase is legacy / pre-cutover.
- **ISO & MOP reference data** (country, currency, language, method_of_payment) live as static TS constants in `@equitipay/reference-data` (not yet existing) — plain string columns, no FK constraints, app-layer validation.
- **Atomic financial writes** live exclusively in `payment-processing-service` — BEGIN/COMMIT spans transaction + ledger_entry + balance updates. Hierarchical `SELECT FOR UPDATE` on accounts.
- **orchestration-service contains ZERO domain logic** — no CVV2, no auth-code gen, no balance, no business rules. Only routing, parallelism, error mapping.
- **callback-service exists separately from orchestration** specifically to handle PSP-facing concerns: HMAC/RSA signature verify, payload normalization, deduplication, PSP-timeout-window ack, dead-letter retry.

## Comms patterns

- **Sync (HTTP/gRPC)** — orchestration → org/processing/risk/gateway/session; any service → config-service (config read), iam-service (auth/permission).
- **Async (message broker)** — payment-processing → notification + audit; organization → audit; iam → audit.
- **Inbound webhooks** — External PSP → callback-service → orchestration.

## SLA tiers (measured at platform boundary, PSP latency excluded)

| Tier                                   | P50    | P99    | Availability |
| -------------------------------------- | ------ | ------ | ------------ |
| CRUD                                   | <50ms  | <200ms | 99.9%        |
| Session creation                       | <80ms  | <300ms | 99.99%       |
| Issuer mode (card auth from network)   | <200ms | <500ms | 99.99%       |
| Acquirer mode (platform overhead only) | <300ms | <800ms | 99.99%       |

**Enforcement:** P99 latency HPA trigger (scale at 80% of budget for 60s), PgBouncer pool isolation per SLA category (issuer mode has dedicated pool so CRUD spike can't exhaust connections).

## organization-service scaling

- Read-heavy (entity validation on every payment) + write-heavy (issuance, approvals) in one deployment.
- HPA: P99 > 160ms for 60s; min 3 / max 20 replicas.
- PgBouncer: transaction pooling, 20 conns/pod.
- Redis cache: 5-min TTL for retailer/card_program/service config; evict on write.
- Read replica can be added later transparently via PgBouncer routing.

## Gaps vs. current code (TODO when migration starts)

- Code still has `transaction-service`, `business-service`, `card-service`, `fraud-service` as separate apps.
- **`audit-service` is in flight as PR #68 (`feat/audit-log`, author ttchimou, opened 2026-04-28, +1379/-37 across 45 files)** — do NOT duplicate work. Any audit_log table / Prisma model / message-broker consumer code belongs in that PR.
- Firestore is still primary for retailer/customer/workspace/space (per current CLAUDE.md "Gotchas") — needs cutover to Postgres in `organization-service`.
- `@equitipay/reference-data` package does not exist yet.
- `audit-service` does not exist yet.
- Message-broker async path is partial — Kafka spine exists (`reference_kafka_spine.md`) but not all event flows are wired.
- CLAUDE.md service table (ports 3001–3011) lists the OLD names; will need update once renames begin.
