---
name: GKE SecretProviderClass bootstrap deadlock — read the mounted file or use optional secretKeyRef
description: GKE's managed Secret Store CSI has NO sync-controller. SPC `secretObjects` (which materialize a k8s Secret) only sync AFTER a pod successfully mounts the SPC volume. Kubelet validates `secretKeyRef` BEFORE the volume mount step — first deploy deadlocks. Two valid fixes.
type: reference
---

Pattern observed in PR #115 for `eqp-portal` and repeated for the CIPS bridge in commit `57d9c1c`. Use whenever a workload would read a value backed by a `SecretProviderClass` `secretObjects` block.

## The deadlock

1. SecretProviderClass declares `secretObjects: [- secretName: portal-server-secrets, ...]`.
2. **GKE's managed CSI driver does NOT include the sync-controller** — `secretObjects` only materialise into a k8s `Secret` AFTER a pod successfully mounts the SPC volume.
3. Kubelet validates `valueFrom.secretKeyRef` BEFORE the volume mount step.
4. First deploy: pod stays in `CreateContainerConfigError` forever — `Error: secret "portal-server-secrets" not found`.

(Upstream secrets-store-csi-driver has a `--enable-secret-rotation` sync-controller flag; the GKE-managed flavour doesn't expose it and doesn't run one. Confirmed by behaviour, not by docs.)

## Fix 1 — read the MOUNTED FILE (preferred)

The SPC volume mount itself ALWAYS works. The chicken-and-egg is only on the `Secret`-materialisation step. Code can read the secret directly from `/var/secrets/<name>` and skip the env-var indirection entirely.

```ts
// apps/eqp-portal/lib/safe-action.ts
function loadCipsApiKey(): string {
  if (process.env.CIPS_API_KEY) return process.env.CIPS_API_KEY;        // local dev
  try {
    return readFileSync('/var/secrets/cips-api-key', 'utf8').trim();    // GKE prod/staging
  } catch { throw new Error('CIPS_API_KEY not configured'); }
}
```

iam-service follows the same pattern for `api-key-pepper` and `firebase-admin-credentials`.

## Fix 2 — `optional: true` on `secretKeyRef`

When env-var access is unavoidable (e.g. NestJS `ConfigService.get('FOO')`):

```yaml
- name: API_GATEWAY_KEY
  valueFrom:
    secretKeyRef:
      name: portal-server-secrets
      key: api-gateway-key
      optional: true   # break the bootstrap deadlock
```

`optional: true` lets kubelet admit the pod with the env var unset. The pod admits → SPC volume mounts → CSI driver creates the K8s Secret. On the next reconcile (or pod restart) kubelet picks up the now-existing Secret and the env var resolves.

**Trade-off:** On first deploy the env var is undefined. Code that uses it must either (a) tolerate undef + degrade gracefully, or (b) the readiness probe must not exercise the secret-using code path. eqp-portal's `/` readiness returns static Next.js HTML before SSR action calls happen, so the pod is Ready even with API_GATEWAY_KEY unset. SSR action calls failed with `API_GATEWAY_KEY is not configured` until the SPC sync completed and a pod restart picked up the materialised Secret.

(eqp-portal has since moved off this pattern — per-org API keys removed `API_GATEWAY_KEY` entirely, and CIPS moved to Fix 1. Fix 2 is still valid for any future env-var-bound secret.)

## Anti-pattern (rejected)

**Pre-create the K8s Secret out-of-band as a one-time bootstrap.** Cleaner runtime semantics but adds a manual step on every new env, defeating the SPC sync's per-env automation. Not used in this repo.

## SPC driver name

The pod-spec `csi.driver:` field uses the GKE-specific name `secrets-store-gke.csi.k8s.io`, NOT the upstream `secrets-store.csi.k8s.io` (PR #114). The CRD `apiVersion` stays `secrets-store.csi.x-k8s.io/v1`. Mismatching the driver name results in `FailedMount: driver secrets-store.csi.k8s.io not registered` on every pod.

## When NOT to use SPC

- **Env values that vary per env but aren't secret**: use the per-env ConfigMap pattern. See `reference_gke_runtime_env_pattern.md`.
- **Build-time inlined values** (Next.js `NEXT_PUBLIC_*`): wire via Cloud Build `--build-arg` from Secret Manager, not via SPC at runtime. The portal Dockerfile inlines them into the JS bundle.
