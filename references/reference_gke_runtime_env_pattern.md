---
name: GKE runtime env pattern (per-env ConfigMap)
description: How runtime environment variables (REDIS_URL, KAFKA_BOOTSTRAP_SERVERS, GCP_PROJECT_ID, etc.) flow into pods on GKE — base deployment.yaml uses envFrom with optional ConfigMap, overlay/<env>/ creates the namespace-scoped ConfigMap with concrete values.
type: reference
---

Pattern shipped in PR #115 (REDIS_URL) and extended in PR #125 (GCP_PROJECT_ID). Used for any runtime env value that varies per environment (dev/staging/prod) but isn't a secret.

## Base (`infra/k8s/base/<svc>/deployment.yaml`)

Every `app` container has:

```yaml
envFrom:
  - configMapRef:
      name: platform-env
      optional: true   # pods start even on envs without the ConfigMap
```

`optional: true` is critical: it lets pods start before the overlay's ConfigMap is created (e.g., during initial bootstrap on a new env). The pod runs with whatever in-app defaults exist; the next reconcile picks up the values.

## Overlay (`infra/k8s/overlays/<env>/platform-env-configmap.yaml`)

Two ConfigMaps because workloads span two namespaces (ConfigMaps are namespace-scoped):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: platform-env
  namespace: eqp-services
data:
  REDIS_URL: "redis://10.255.137.203:6379"
  GCP_PROJECT_ID: "cips-test"
  KAFKA_BOOTSTRAP_SERVERS: "bootstrap.eqp-dev-kafka.us-central1.managedkafka.cips-test.cloud.goog:9092"
  ROUTING_ENGINE_URL: "https://routing-engine-...run.app"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: platform-env
  namespace: eqp-frontend
data:
  REDIS_URL: "redis://10.255.137.203:6379"
  GCP_PROJECT_ID: "cips-test"
  # … same keys, used by eqp-portal
```

Both ConfigMaps carry the same keys (with possibly different per-namespace tweaks). One in `eqp-services` for the 11 backends, one in `eqp-frontend` for the portal.

## For staging/prod

Replicate the overlay file with that env's Memorystore IP, Kafka bootstrap, project id, routing-engine URL, etc. Until a new env's ConfigMap exists, the `optional: true` flag means pods start with the in-app defaults (e.g. `@repo/redis` falls back to `redis://localhost:6379`) — they fail-fast on connect, surfacing a clean error rather than blocking pod admission.

## For new env keys

Just add to the per-env ConfigMap data block. No base manifest edit needed — `envFrom: [configMapRef]` exposes ALL keys.

## When NOT to use this pattern

- **Per-service env that ISN'T env-varying** (e.g. `HTTP_PORT`, `GRPC_PORT`, `JWT_HEADER_SOURCE`): use the explicit `env: [- name: ..., value: ...]` block in the base deployment.yaml. Keeps the per-service config grep-able.
- **Secrets**: use SecretProviderClass + Workload Identity (mounted from Secret Manager). NEVER put secret material in a ConfigMap. See `reference_gke_spc_bootstrap_optional.md`.

## Why GCP_PROJECT_ID can't be auto-derived

Previously some services tried to read `GCP_PROJECT_ID` from a pod annotation (`metadata.annotations['iam.gke.io/gcp-project-id']`), but that annotation isn't auto-populated on Autopilot + ASM. Result: `@repo/observability` Cloud Trace exporter ran without a project id, and `@repo/firebase` couldn't initialise Firestore. The fix (PR #125) was to wire it via this ConfigMap pattern.
