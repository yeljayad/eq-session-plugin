---
name: Kafka Domain-Event Spine
description: GCP Managed Kafka spine for EQP inter-service domain events. mTLS (cert-manager), CloudEvents 1.0 JSON envelope + Zod-validated bodies, kafkajs wrapped by @repo/kafka. Kafka for EQP domain events; Pub/Sub for GCP integration glue.
type: reference
---

The EQP platform's inter-service domain-event spine. Every NestJS service is a producer for its own domain topic; targeted services consume topics they care about. At-least-once delivery with consumer-side idempotency. Cluster lives on GCP Managed Service for Apache Kafka behind PSC inside `vpc-be`.

Spec of record: `docs/superpowers/specs/2026-05-04-kafka-design.md`. Package: `packages/kafka/`. Event schemas: `packages/common/src/events/`. TF: `infra/terraform/modules/managed-kafka/` + `infra/terraform/modules/cert-manager/`.

## 1. Why Kafka and Pub/Sub coexist

The platform uses both messaging primitives, in non-overlapping lanes. Don't blur them.

- **Kafka** — EQP domain events: transactions, auth, fraud, audit-drain, cross-service replay, future stream-processing. Per-key ordering at scale, configurable retention window, cloud-portable, team familiarity. Internal-only — no partner-facing Kafka access.
- **Pub/Sub** — GCP-native integration glue: Eventarc, Cloud Storage notifications, Cloud Scheduler triggers, future BigQuery sinks. Stays scaffolded in `@repo/gcp` even though it has zero current consumers; that's intentional.

Choose Kafka when the event is part of EQP's domain model. Choose Pub/Sub when GCP itself emits the trigger.

## 2. Locked decisions (don't re-litigate)

| Decision | Choice | Why |
|---|---|---|
| Auth | **mTLS with X.509 client certs** (NOT SASL/OAUTHBEARER) | Strongest identity, defence-in-depth with transport TLS, supported by Managed Kafka |
| CA | **cert-manager + self-signed root in K8s Secret** (NOT GCP CAS) | No recurring CA cost, automatic 75-day rotation, migration path to CAS preserved |
| Client library | **kafkajs 2.2.4 directly** wrapped by `@repo/kafka` (NOT `@nestjs/microservices` Kafka transport) | Required behaviours not exposed by the transport: manual per-message offset commit, DLQ pre-handler routing on schema-fail, `maxInFlightRequests=5` for idempotent producer, cert-watcher driven `disconnect/connect` |
| Wire format | **CloudEvents 1.0 envelope + JSON `data` validated by Zod** (NOT Avro/Protobuf, NOT Schema Registry) | Eventarc-compatible, runtime-typed, fits Zod-everywhere pattern, `dataschema` field reserves Schema-Registry pointer for later |
| Delivery semantics | **At-least-once + consumer-side idempotency** keyed on `event.id` via Redis | Exactly-once across topics deferred indefinitely; transactional producer not used |
| Ownership rule | **One topic per service-domain; only the owning service writes** | No promiscuous global write; consumers subscribe to whichever topics they need |

## 3. Cluster

GCP Managed Service for Apache Kafka, one cluster per environment named `eqp-<env>-kafka`, region `us-central1`, ZONAL availability. PSC-attached to `vpc-be` via the existing env-named regional subnet (e.g. `eqp-dev-us-central1`). No new `google_service_networking_connection` — Managed Kafka reconciles its own PSC service attachment. No collision with the Cloud SQL + Cloud Build worker pool peering ranges.

Capacity floor: **3 vCPU minimum**. The spec was originally sized at 1 vCPU based on docs that no longer reflect the API surface; GCP rejects sub-3 vCPU clusters at apply time. Memory is 1 GiB per vCPU minimum.

CMEK: Google-managed encryption on dev (`kms_key_id = null`). Staging/prod MUST pass `module.kms_pci.kafka_key_id` — enforced by a `check "cmek_required_on_staging_prod"` block in the module that fails the plan if null on non-dev (PCI 3.4/3.5, same pattern as `cloud-sql` and `artifact-registry` modules).

TF resource shape: `google_managed_kafka_cluster` with `gcp_config.access_config.network_configs[].subnet` set to the subnet's **relative path** (use `.id`, NOT `.self_link` — provider rejects the full URL). KMS key is a scalar `gcp_config.kms_key` attribute in provider `~> 7.x` (not a sub-block). `rebalance_config.mode = "AUTO_REBALANCE_ON_SCALE_UP"`.

Bootstrap address: **not exported** by `google_managed_kafka_cluster` in the current provider. Fetch post-apply via `gcloud managed-kafka clusters describe eqp-<env>-kafka --location=us-central1 --format='value(bootstrapAddress)'` and feed it into `platform-env` ConfigMap as `KAFKA_BOOTSTRAP_SERVERS`. There is no Terraform output for it — the `outputs.tf` field tries to read `google_managed_kafka_cluster.eqp.bootstrap_address` but that field is empty until the broker reconciles.

## 4. mTLS via cert-manager and self-signed CA

The Managed Kafka broker presents a Google-managed server cert that validates against Node's system trust. Client mTLS uses cert-manager-issued certs whose root we install in the cluster's truststore.

Bootstrap order is fragile because of CRD chicken-and-egg:

1. Install cert-manager via Helm (`helm_release` resource pointing at `https://charts.jetstack.io`, chart `cert-manager`).
2. Generate the self-signed root (`tls_private_key` ECDSA P-256 + `tls_self_signed_cert`, 5-year validity).
3. Store root key+cert in K8s Secret `eqp-kafka-ca-root` (type `kubernetes.io/tls`) in the `cert-manager` namespace.
4. Apply the `ClusterIssuer` CR named `eqp-kafka-ca` (kind `ca`) referencing the Secret. **Must be a `null_resource` with `local-exec kubectl apply`, NOT `kubernetes_manifest`.** The latter validates the CRD schema at plan time, before the CRDs exist on the cluster, causing `no matches for kind "ClusterIssuer" in group "cert-manager.io"`. Trade-off: weaker drift detection on the ClusterIssuer — acceptable for a CA controller resource.
5. Wire the cluster truststore via `tls_config.trust_config.cas_configs[].ca_pool` pointing at a GCP CAS CA pool seeded with the cert-manager root (`google_managed_kafka_cluster.tls_config` is a `dynamic` block that activates when `var.trust_anchor_ca_pool_id` is non-null).

Without the truststore wiring step, every client TLS handshake fails with `UNKNOWN_CA` (Phase 1 ships the cluster + cert-manager but truststore landed in Phase 2). The cluster's `tls_config.trust_config.cas_configs[]` takes references to GCP CAS CA pools, not raw PEM — the workaround is a CAS CA pool that proxies/imports the cert-manager root.

Root cert rotation cadence ~ once every 4 years; the spec calls for a 6-month-lead-time alert in `monitoring-baseline`.

## 5. Per-service client certs

Each producing/consuming service ships an `infra/k8s/base/<svc>/kafka-cert.yaml` containing a cert-manager `Certificate` CR:

- `secretName: <svc>-kafka-client-tls`
- `duration: 2160h` (90 days), `renewBefore: 360h` (75 days)
- `privateKey: { algorithm: ECDSA, size: 256 }`
- `commonName: <svc>.eqp-services.svc.cluster.local`
- **Bare CN — no `subject.organizations`** so the DN reduces to `CN=<host>` and Kafka's default principal builder yields `User:CN=<host>`. ACL principals match this exact string.
- `usages: [client auth]`, `issuerRef: { name: eqp-kafka-ca, kind: ClusterIssuer }`

The Secret is mounted as a volume in the service's `Deployment`. Mount with `secret.optional: true` so pods admit before the Certificate has issued — the kafka client config-loader retries until the cert lands rather than CrashLoopBackOff (same Plan-3b SPC bootstrap pattern as `reference_gke_spc_bootstrap_optional.md`).

IAM: each service GSA gets `roles/managedkafka.client` at the project level (the cluster-scoped IAM resource `google_managed_kafka_cluster_iam_member` is not yet in the provider; use `google_project_iam_member` for now). This grants network connect — per-topic permission is then mTLS ACLs, NOT project IAM.

A `KafkaCertWatcher` provider in `@repo/kafka` watches the cert file mtime and triggers `producer.disconnect()` + `producer.connect()` on change. Connection blip <= 5 seconds; in-flight messages retried by the producer's at-least-once semantics.

## 6. Topics and ACLs

**Naming convention:** `<service-domain>.events.v<version>` (per-topic), with the version embedded in the topic name itself. Event-type strings use `eqp.<domain>.<event-name>.v<version>` (e.g. `eqp.business.workspace_created.v1`, `eqp.transactions.created.v1`).

**Catalog (v1):** 11 production topics (one per service), each with a paired DLQ topic `<topic>.dlq.v1`.

| Topic | Partitions | Retention | Cleanup | Partition key |
|---|---|---|---|---|
| `iam.events.v1` | 8 | 1d | delete | `user_id` |
| `session.events.v1` | 6 | 1d | delete | `user_id` |
| `orchestration.events.v1` | 8 | 1d | delete | `workflow_run_id` |
| `transactions.events.v1` | 20 | 1d | delete | `transaction_id` |
| `payment-gateway.events.v1` | 12 | 1d | delete | `transaction_id` |
| `notification.events.v1` | 6 | 1d | delete | `user_id` |
| `callbacks.events.v1` | 12 | 7d | delete | `psp_provider:reference_id` |
| `config.events.v1` | 4 | 7d | **compact** | `config_key` |
| `fraud.events.v1` | 10 | 1d | delete | `transaction_id` |
| `cards.events.v1` | 8 | 1d | delete | `card_id` |
| `business.events.v1` | 6 | 7d | delete | `workspace_id` |

DLQ topics: 2 partitions each, 3-day retention. Production topics use `replication_factor = 3` (Managed Kafka minimum) and `min.insync.replicas = 2` to survive one broker failure with `acks=all`.

**ACLs — one TF resource per `acl_id`:** `google_managed_kafka_acl` is identified by `acl_id` and represents one live ACL on the broker. The provider replaces the full `acl_entries` list on update — there is no merge semantics. If two TF resources share the same `acl_id` they race on every apply and either end up inconsistent or get auto-deleted. **Consolidate every entry per `acl_id` into a single TF resource** (PR #172 hotfix). Each topic's WRITE + DESCRIBE for the producer and READ + DESCRIBE for the consumer go in one resource block with multiple `acl_entries {}` blocks.

`acl_id` format encodes both resource type and pattern type (no separate `pattern_type` field):
- Exact: `topic/<name>`, `consumerGroup/<name>`, `cluster`, `allTopics`
- Prefixed: `topicPrefixed/<prefix>`, `consumerGroupPrefixed/<prefix>` — the value after the slash is the LITERAL prefix string with NO trailing wildcard

Principal format: `User:CN=<service>.eqp-services.svc.cluster.local` — matches the bare-CN client certs.

Consumer-group ACLs use `consumerGroupPrefixed/<service>-` so a service can spin up new groups without a TF apply per group (consumer group naming convention is `<service>-<topic>-v1`).

## 7. `@repo/kafka` package

Lives at `packages/kafka/`. NestJS `@Global` dynamic module — apps register once in `AppModule.imports` via `KafkaModule.forRoot({ source: '/eqp/<service-name>' })`, then every feature module injects providers directly.

Layout:

```
packages/kafka/src/
├── kafka.module.ts            — @Global dynamic module
├── kafka.config.ts            — buildKafkaConfig() env-loader
├── kafka.tokens.ts            — KAFKA_CLIENT, KAFKA_PRODUCER, KAFKA_SOURCE DI tokens
├── envelope/cloud-event.ts    — cloudEventEnvelope() generic Zod schema + CloudEvent<T> type
├── envelope/builder.ts        — buildEnvelope() helper (id, time, source defaults)
├── producer/producer.service.ts — KafkaProducerService.emit() + emitTombstone()
├── consumer/kafka-handler.decorator.ts — @KafkaHandler(topic, schema, {groupId?})
├── consumer/consumer.service.ts — DiscoveryService scan + kafkajs consumer.run() loop
├── consumer/message-handler.ts  — parse + Zod + idempotency + retry/backoff + DLQ
├── cert-watcher/cert-watcher.service.ts — mtime-driven reconnect (producer + consumers)
├── dead-letter/dead-letter.service.ts — typed DLQ routing with header metadata
├── tracing/traceparent.ts     — W3C traceparent inject/extract
├── health/kafka.health.ts     — Terminus HealthIndicator
├── constants/topics.ts        — TOPICS map + dlqTopicFor() + isProductionTopic()
└── cli/replay-dlq.ts          — operator CLI for DLQ inspection + replay
```

Producer config (kafkajs `Kafka.producer()`): `idempotent: true`, `maxInFlightRequests: 5` (max allowed by idempotent producer; 1 would serialize and cripple throughput), `allowAutoTopicCreation: false`, `acks: -1` on every `send()`, gzip compression via `compression.type = "producer"` on the topic.

Consumer config: manual offset commit (autoCommit=false), commit AFTER handler returns, 30s session timeout, 3s heartbeat, `maxBytesPerPartition: 1 MiB`, `fromBeginning: false`.

The TLS config is read at boot from `KAFKA_CLIENT_CERT_PATH` and `KAFKA_CLIENT_KEY_PATH` env vars. `ssl.ca` is intentionally **omitted** (`undefined`) — Node falls back to the system trust store to validate the broker's Google-managed server cert. Setting `ssl.ca` would override the system trust and break the server-cert handshake. Client mTLS is unaffected (the broker uses its truststore to validate the client cert).

Local-dev plaintext path: when both cert paths are empty strings, SSL is skipped entirely (`brokers` + `clientId` only). Production / staging always set both.

`forbidPii` is exported from `@repo/common/src/security/forbid-pii.ts` (NOT from `@repo/kafka`) — it was moved to break a circular dep with `@repo/common`'s `events/` directory. `@repo/kafka` still re-exports it from its barrel for backwards compatibility.

## 8. CloudEvents envelope

```ts
{
  specversion: '1.0',
  id: <uuid>,                    // uuidv7 from local impl, partition-stable for ordered events
  source: '/eqp/<service>',
  type: 'eqp.<domain>.<event-name>.v<version>',
  subject?: '<partition-key>',   // e.g. "transaction:txn_abc"
  time: '<ISO 8601 UTC>',
  datacontenttype: 'application/json',
  dataschema?: '<URL>',          // reserved for future Schema Registry pointer
  data: <T>                      // body validated by Zod schema for the event type
}
```

Validated end-to-end by `cloudEventEnvelope<T>(dataSchema: T)` which is generic over the Zod body schema. Producers and consumers fail closed on validation error. Built with Zod 4 helpers — `z.uuid()`, `z.iso.datetime()`, `z.url()` (top-level format helpers; NOT the old `z.string().uuid()`).

**Per-event schemas (36 total in v1) live in `packages/common/src/events/topics/`** — one file per topic, each exporting:
- `<TOPIC>_EVENT_TYPES` const map of event-type literals
- One `<EventName>DataSchema` Zod schema per event type (every schema is `z.strictObject({...}).superRefine(forbidPii)`)
- `<EventName>Data` inferred TypeScript type
- `<TOPIC>_EVENT_SCHEMAS` lookup map keyed by event-type literal

A combined `ALL_EVENT_SCHEMAS` in `all-events.ts` covers cross-topic tooling (audit-drain, DLQ replay). Per-topic consumers should import the specific topic's schemas map for tighter typing.

**Shared primitives** in `packages/common/src/events/shared/primitives.ts`:
- `Money = z.string().regex(/^-?\d+\.\d{4}$/)` — Decimal(18,4) string, NEVER float
- `RiskScore = z.string().regex(/^(0\.\d{4}|1\.0000)$/)` — bounded [0,1]
- `IsoDatetime = z.iso.datetime()`
- `HashedString = z.string().regex(/^[a-fA-F0-9]{64}$/)` — sha256 hex, case-insensitive
- `CurrencyCode = z.string().length(3).regex(/^[A-Z]{3}$/)` — ISO 4217 alpha-3 uppercase
- `Uuid = z.uuid()`

**Sensitive-data redaction policy enforced by `forbidPii` Zod refinement:** never include full PAN, CVV, full expiry, full email, full phone, raw PSP response with sensitive fields, JWT tokens, password hashes. Use `last_four` for PAN, `email_hashed`/`phone_hashed`/`ip_hashed` for identifiers, `*_redacted` field stripping known-sensitive PSP keys. Money as `Decimal(18,4)` strings only.

## 9. Producer pattern

The canonical order is: **persist first, emit second**. Don't gate the response on Kafka success.

```ts
constructor(
  private readonly repo: WorkspaceRepository,
  private readonly kafkaProducer: KafkaProducerService,
) {}

async create(input: CreateWorkspaceInput): Promise<Workspace> {
  const workspace = await this.repo.create(input);   // Firestore = source of truth

  try {
    await this.kafkaProducer.emit({
      topic: TOPICS.BUSINESS,
      type: BUSINESS_EVENT_TYPES.WORKSPACE_CREATED,
      subject: `workspace:${workspace.resource_id}`,
      schema: WorkspaceCreatedDataSchema,
      data: { workspace_id: workspace.resource_id, /* ... */ },
    });
  } catch (err) {
    this.logger.error('kafka emit failed (workspace_created)', err);
    // Don't rethrow — domain write succeeded. Phase 7 audit-drain reconciles missed emits.
  }

  return workspace;
}
```

If the emit fails, the consumer-side state is missing one event but Firestore is consistent. The Phase 7 audit-drain reconciliation catches missed emits by cross-checking against the domain of record. The per-topic feature flag `KAFKA_PUBLISH_ENABLED_<topic>=true|false` in `platform-env` ConfigMap is the operator-flippable rollback (no redeploy required).

Inside `emit()`: builds envelope (`id` = uuidv7, `time` = now UTC, `source` = `/eqp/<service>`), validates `data` against the Zod schema (fails closed on invalid), JSON-serializes, sets kafkajs message `key = subject ?? envelope.id` for partition-stable routing, and sends with `acks=-1`. Headers: `eqp-event-id`, `eqp-event-type`, `eqp-event-source`, `content-type: application/cloudevents+json`, W3C `traceparent` for trace continuity.

For compacted topics (currently only `config.events.v1`) use `KafkaProducerService.emitTombstone(topic, key)` to delete a key. Cannot use `emit()` because it always JSON-serializes a non-null envelope, which keeps the key alive in the compacted log.

## 10. Consumer pattern

`@nestjs/microservices` is NOT used — `KafkaConsumerService` uses NestJS `DiscoveryService` + `MetadataScanner` to find `@KafkaHandler`-decorated methods at module init and wires kafkajs consumers directly.

```ts
@Controller()
export class TransactionEventConsumer {
  constructor(private readonly notify: NotificationService) {}

  @KafkaHandler(TOPICS.TRANSACTIONS, TransactionCreatedDataSchema)
  async onTransactionCreated(event: CloudEvent<TransactionCreatedData>): Promise<void> {
    if (event.type !== TRANSACTION_EVENT_TYPES.CREATED) return;
    await this.notify.sendReceiptPending(event.data.customer_id, event.data);
  }
}
```

The `MessageHandler` runs for every kafkajs message:

1. Parse JSON, validate envelope via `cloudEventEnvelope(handlerMetadata.schema)` — malformed JSON → DLQ with `failureType=envelope-error`, Zod fail on `data` → DLQ with `failureType=zod-validation-error`.
2. Idempotency check: `IdempotencyService.check('kafka:idempotent:<event.id>')` (24h Redis TTL). On `replay` status, skip handler and commit offset.
3. Invoke handler. On throw, retry with exponential backoff (1s / 5s / 30s, 3 attempts).
4. After 3 failures → DLQ with `failureType=handler-error`.
5. Store the idempotency key on success, then commit offset.

Consumer group naming: `<service>-<topic>-v1` (e.g. `notification-service-transactions.events.v1-v1`). Override per handler via `@KafkaHandler(topic, schema, { groupId: '...' })`.

DLQ payload preserves the original message + adds headers: `x-dlq-original-topic`, `x-dlq-failure-type`, `x-dlq-failure-reason` (<=1KB), `x-dlq-attempt-count`, `x-dlq-failed-at`, `x-dlq-consumer-service`.

For business-level idempotency (e.g. `transactions.settled.v1` triggering an external bank transfer), use a database-backed idempotency key in the handler — the Redis check only protects against replay within the 24h TTL.

## 11. Deployment manifest gotcha

For every new producer or consumer service, `infra/k8s/base/<svc>/deployment.yaml` MUST include BOTH:

1. **Env block** — the Kafka client config:
   - `KAFKA_BOOTSTRAP_SERVERS` from `platform-env` ConfigMap (env-specific)
   - `KAFKA_CLIENT_ID` (typically the service name)
   - `KAFKA_CLIENT_CERT_PATH=/etc/kafka/tls/tls.crt`
   - `KAFKA_CLIENT_KEY_PATH=/etc/kafka/tls/tls.key`
2. **Volume mount** — the cert-manager-issued client cert Secret mounted at `/etc/kafka/tls` with `secret.optional: true`

Plus `infra/k8s/base/<svc>/kafka-cert.yaml` registering the `Certificate` CR, and `kustomization.yaml` including it.

**Missing either block = pod boots but cannot connect.** This is the most common manifest mistake when adding a service to the spine — PR #162 + #165 caught card-service mid-rollout (Cohort 2 producer added but deployment.yaml not updated). The `KafkaConfig` loader fails closed with a clear "is required" error when env vars are missing; missing volume mount surfaces as `ENOENT` on the cert path during boot.

The `verify-deploy.yaml` smoke Job runs `kcat -L` against every topic — keep its topic list in sync as cohorts ship.

## 12. GKE Autopilot specifics

cert-manager has two Autopilot-specific knobs:

1. **`global.leaderElection.namespace=cert-manager`** — must be the `cert-manager` namespace, NOT the default `kube-system`. Autopilot's Warden admission controller blocks writes to `kube-system` for non-Google SAs (`denied by managed-namespaces-limitation`). Default leader election fails, which prevents CA injection into the `ValidatingWebhookConfiguration` caBundle, which makes every `Certificate` create fail with `tls: failed to verify certificate: x509: certificate signed by unknown authority`.
2. **`startupapicheck.enabled=false`** — the post-install validation Job (which issues a test cert through the webhook) can take longer than its own ~120s timeout on Autopilot, causing the Helm post-install to fail even when cert-manager itself is healthy. Disable it and validate cert-issuance via an operator-runbook smoke (`kafka-smoke-cert` Certificate + `kcat`).

Helm release `timeout = 600` — the first install on a fresh cluster pulls images (~30-60s) before pods reach Running.

Namespace labels: `pod-security.kubernetes.io/enforce: baseline`, `audit: restricted`, `warn: restricted`. NOT `enforce: restricted` (cert-manager's webhook needs `baseline`).

Resource floor: `requests.cpu=10m`, `requests.memory=32Mi`, `limits.memory=128Mi`. cert-manager is control-plane, not data-path.

## 13. Binary Authorization

The cert-manager Helm chart images ship from `quay.io/jetstack/cert-manager-*`. Add to `infra/terraform/modules/binauthz/main.tf` `admission_whitelist_patterns`:

```
quay.io/jetstack/cert-manager-*
```

Without this, Binary Authorization denies the controller/webhook/cainjector pods on staging/prod (dev runs without BinAuthz enforcement). Same pattern for any future Kafka-adjacent images (Kafka Connect, schema registry, etc.).

## 14. Cost

Phase 1 onwards: **~$330/mo** at the 3 vCPU minimum on dev. The original spec sized it at 1 vCPU ($192.85/mo with 1-yr CUD); GCP rejected sub-3 vCPU clusters at apply time, pushing the floor up. Partition cost (~$11/mo for the 22 DLQ partitions on top of the 100 production partitions) and DLQs are within the cluster-level pricing.

The 1-yr CUD commits the cluster regardless of whether topics carry traffic — there's no incremental cluster cost from adding topics, only from the per-partition line items. Retention bumps to 7-30d roughly double the storage line.

Staging/prod also incur the CAS pool for the truststore (DEVOPS tier ~$20/mo if subordinate-import works; ENTERPRISE ~$200/mo as fallback).

## 15. Operational

**Fetch bootstrap address (not exported by TF):**

```bash
gcloud managed-kafka clusters describe eqp-<env>-kafka \
  --location=us-central1 --project=<project> \
  --format='value(bootstrapAddress)'
```

**Smoke against the cluster with `kcat`:** need a client cert (issue one via cert-manager `Certificate` CR) and the bootstrap:

```bash
kcat -b $BOOTSTRAP -X security.protocol=ssl \
     -X ssl.certificate.location=/etc/kafka/tls/tls.crt \
     -X ssl.key.location=/etc/kafka/tls/tls.key \
     -L
```

Expected: metadata listing for all topics. If TLS handshake fails with `UNKNOWN_CA`, the cluster truststore isn't wired. If a `TopicAuthorizationFailedError` appears for specific operations, that's expected for topics the principal doesn't have an ACL on.

**Inspect a DLQ:** same `kcat` invocation against `<topic>.dlq.v1` with `-C -o beginning`.

**Replay from a DLQ:** `bun run --cwd packages/kafka replay-dlq --source <dlq-topic> --target <production-topic> --offset <N> --partition <P>` (or `--all` to drain a DLQ wholesale after fixing a producer bug).

**Per-topic feature flag rollback:** flip `KAFKA_PUBLISH_ENABLED_<topic>=false` in the env's `platform-env` ConfigMap. Producers respect the flag and skip the emit — domain writes still succeed, no event published. Operator-flippable in <1 minute, no redeploy.

**Verify cluster is healthy:**

```bash
gcloud managed-kafka clusters describe eqp-<env>-kafka --location=us-central1 --format='value(state)'
# expect: ACTIVE

kubectl get pods -n cert-manager
# expect: 3 pods Running (controller, cainjector, webhook)

kubectl get clusterissuer eqp-kafka-ca \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
# expect: True
```
