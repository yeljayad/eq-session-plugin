---
name: GKE + ASM Platform Architecture
description: Live architecture — GCLB → ASM IngressGateway → NestJS services (HTTP) → gRPC (mTLS). Workload Identity, BinAuthz, Cloud Build/Deploy. Replaced Cloud Run+ESPv2 (decommissioned 2026-05-02). Use when working on infra, deploy, or auth at the edge.
type: reference
---

## 1. Request flow (live)

The platform runs on GKE Autopilot behind Anthos Service Mesh. Cloud Run + ESPv2 were decommissioned on 2026-05-02.

**Public API request:**
```
Browser
  -> GCLB :443  (HTTPS, GCP-managed cert, Cloud Armor CRS v4.22 + WAF, TLS 1.2 RESTRICTED)
  -> google_compute_backend_service "<name-prefix>-ingressgateway-backend"
  -> Standalone NEG `asm-ingressgateway-neg` (zonal, per-zone)
  -> istio-ingressgateway pod :80 in eqp-system  (Envoy)
       - ASM RequestAuthentication validates Firebase JWT (jwks for securetoken@system)
       - ASM AuthorizationPolicy "deny-no-jwt" denies if no JWT principal AND path matches /api/<svc>/*
       - EnvoyFilter "max-request-bytes" caps body at 1 MiB
       - VirtualService rewrite /api/<svc>/... -> /<svc>/...
  -> NestJS service ClusterIP :3001-3011 (Fastify, HTTP)
  -> Inter-service gRPC :50051-50061 (mTLS via PeerAuthentication STRICT, mesh-wide)
```

**Portal SSR path** (browser navigates to `/`, `/login`, `/dashboard`, `/transactions`, etc.):
```
Browser
  -> GCLB :443
  -> istio-ingressgateway   (path does NOT match /api/<svc>/* -> deny-no-jwt passes)
  -> VirtualService eqp-portal catch-all (prefix /)
  -> eqp-portal pod :3000 in eqp-frontend  (Next.js standalone)
  -> server actions / fetchers call back to the IngressGateway over cluster-internal HTTP:
       API_GATEWAY_URL=http://istio-ingressgateway.eqp-system.svc.cluster.local:80/api
       Host: gke-dev.eqpay-dev.com   (must match the gateway listener's `hosts:` filter)
  -> backend service (same chain as above, /api/<svc>/... -> /<svc>/...)
```

**Two parallel auth paths** (mutually exclusive per request): Firebase JWT for browser/portal; per-org `X-API-Key` validated by `iam-service.ApiKeyValidatorService` over mTLS gRPC for server-to-server merchant integrations. `ApiKeyGuard` is the first global guard on every NestJS service.

## 2. Edge — GCLB + firewall

**Module**: `infra/terraform/modules/gclb-edge` (IP, cert, Cloud Armor) + `infra/terraform/modules/gclb-wiring` (backend, NEG, URL maps, forwarding rules).

Cert is `google_compute_managed_ssl_certificate` with `create_before_destroy`. The HTTPS forwarding rule is on the reserved global IP at port 443; a second forwarding rule on port 80 issues `301 -> https://` via a dedicated URL map with `default_url_redirect.https_redirect = true`. The port-80 rule is **required** for the managed cert to provision — Google's HTTP-01 prober talks to port 80, not 443. Without it the cert sticks in `PROVISIONING` forever.

**Firewall — open BOTH 15021 AND 80 (PR #121 fix).** `infra/terraform/modules/gclb-wiring/main.tf:14` defines `google_compute_firewall.gclb_hc_ingressgateway` with `ports = ["15021", "80"]` from source ranges `130.211.0.0/22, 35.191.0.0/16`. Opening only 15021 makes the GCLB health-check pass (the backend shows `HEALTHY`) but every actual request times out at the GCLB -> NEG -> pod hop, surfacing to clients as `503 upstream connect timeout` ~9 s in. Discovered live on cips-test 2026-04-30. The auto-created GKE rule `gke-csm-thc-<hash>` only opens TCP 7877 (CSM telemetry), not the data plane.

Backend service `port_name` is intentionally omitted — `GCE_VM_IP_PORT` (zonal NEG) backends encode port routing inside the NEG endpoint. NEGs are referenced by full self-link string (not `data` source) to defer validation to apply time, since the NEG is created asynchronously by GKE from a Service annotation in `eqp-system`.

## 3. ASM Authorization — `deny-no-jwt` policy (inverted)

**Module**: `infra/terraform/modules/asm-platform/main.tf` resource `kubernetes_manifest.authz_policy_deny_default` (TF source of truth); mirrored as plain YAML at `infra/k8s/base/platform/deny-no-jwt.yaml` for kubectl-direct apply when TF connectivity to the cluster is blocked.

**Current shape (PR #130 + post-2026-05-08 routing migration):** the policy DENIES at the IngressGateway only when both conditions hold: the request has no JWT principal AND the path matches a backend prefix under `/api/<svc>/*`:

```yaml
action: DENY
rules:
  - from: [{ source: { notRequestPrincipals: ["*"] } }]
    to:
      - operation:
          paths:
            - /api/iam/*
            - /api/api-keys/*
            - /api/sessions/*
            - /api/orchestration/*
            - /api/smart-routing/*
            - /api/transactions/*
            - /api/payments/*
            - /api/notifications/*
            - /api/callbacks/*
            - /api/config/*
            - /api/fraud/*
            - /api/cards/*
            - /api/business/*
            - /api/customers/*
            - /api/retailers/*
            - /api/spaces/*
            - /api/workspaces/*
```

Adding a new backend service prefix REQUIRES editing this list (and the matching `infra/k8s/base/<svc>/virtualservice.yaml`).

**Deprecated shape (DO NOT USE — for context only):** PR #118 first tried `notPaths` exemptions like `/auth/*` and `/iam/auth/*` under a deny-everything-else policy, which broke every portal SSR route (`/`, `/login`, `/dashboard`, `/_next/*`, `/favicon.ico`, ...). PR #130 inverted to the positive `paths` list above so all browser routes fall through unauthenticated to the portal catch-all VS. PR #44db756 then updated `paths` from `/iam/*`-style to `/api/iam/*`-style for the 2026-05-08 `/api/*` routing convention. `/api/auth/*` is intentionally absent — iam-service serves its own public auth-flow endpoints and returns 401 on its own.

Backends still independently enforce auth via `TokenGuard`; the gateway DENY is only the first line.

## 4. Identity — Workload Identity

**Module**: `infra/terraform/modules/workload-identity/main.tf`. One GSA per workload (12 total: 11 backends + portal). For each GSA:

- `google_service_account` named `<workload>` in the project (e.g., `iam-service@<project>.iam.gserviceaccount.com`).
- Project-level role grants from `var.workloads[].roles` (e.g., `roles/datastore.user`, `roles/firebaseauth.admin`).
- WI binding: `google_service_account_iam_member` with `role = "roles/iam.workloadIdentityUser"` and `member = "serviceAccount:<project>.svc.id.goog[<namespace>/<workload>-sa]"`.

The matching Kubernetes ServiceAccount lives at `infra/k8s/base/<svc>/serviceaccount.yaml` with annotation:
```yaml
annotations:
  iam.gke.io/gcp-service-account: <workload>@<gcp-project-id>.iam.gserviceaccount.com
```
The `<gcp-project-id>` placeholder is substituted at build time by Cloud Build's `resolve-env` step.

Firebase Admin SDK uses `admin.credential.applicationDefault()` (`packages/firebase/src/firebase.provider.ts`) which on GKE resolves to the WI-bound GSA token automatically. No JSON service-account keys exist anywhere. This is why Plan 3b skips `firebase-admin-credentials` secret provisioning.

Cluster has `workload_identity_config.workload_pool = "${project_id}.svc.id.goog"` (set in `infra/terraform/modules/gke-autopilot/main.tf`).

## 5. Secrets — SecretProviderClass + CSI driver

GKE Autopilot ships the Secret Manager add-on (`secret_manager_config.enabled = true` on the cluster, `gke-autopilot/main.tf`). It installs the Secret Store CSI driver, the GCP provider plugin, and the `SecretProviderClass` CRDs.

**Driver name on GKE is `secrets-store-gke.csi.k8s.io`, NOT the upstream `secrets-store.csi.k8s.io`** (PR #114). The CRD `apiVersion` stays `secrets-store.csi.x-k8s.io/v1`; only the pod-spec `csi.driver:` field uses the GKE-specific name. Every base deployment under `infra/k8s/base/<svc>/deployment.yaml` references:
```yaml
volumes:
  - name: secrets
    csi:
      driver: secrets-store-gke.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: <svc>-secrets
```

**SPC bootstrap deadlock (PR #115).** GKE's managed CSI add-on does NOT include the sync-controller. `secretObjects` (the SPC stanza that materializes a k8s `Secret`) only sync AFTER the volume mounts successfully. Kubelet validates `valueFrom.secretKeyRef` BEFORE the volume mount step. Net effect: any env var that references a k8s Secret created by SPC blocks first-deploy admission forever.

**Two valid patterns:**

1. **Mounted file (preferred)** — read the secret directly from the mounted path. Used by iam-service (`/var/secrets/api-key-pepper`, `/var/secrets/firebase-admin-credentials.json`) and the portal (`/var/secrets/cips-api-key`, read by `lib/safe-action.ts`). The portal switched from env var to file mount in commit `57d9c1c` for CIPS_API_KEY.
2. **`optional: true` on `secretKeyRef`** — pod admits, volume mounts, SPC syncs the Secret, a pod restart picks up the value. First-boot the env var is undefined; SSR calls that need it return clean errors instead of blocking admission.

SPC manifests at `infra/k8s/base/<svc>/secretproviderclass.yaml` carry `<gcp-project-id>` placeholders (substituted at deploy time) and list each secret by full resource name + path:
```yaml
spec:
  provider: gke
  parameters:
    secrets: |
      - resourceName: "projects/<gcp-project-id>/secrets/<name>/versions/latest"
        path: "<filename>"
```

## 6. Runtime env — per-env ConfigMap pattern

Every backend container in `infra/k8s/base/<svc>/deployment.yaml` carries:
```yaml
envFrom:
  - configMapRef:
      name: platform-env
      optional: true
```
The actual values live in `infra/k8s/overlays/<env>/platform-env-configmap.yaml`, which declares **two** ConfigMaps (ConfigMaps are namespace-scoped):

- `name: platform-env, namespace: eqp-services` — consumed by all 11 backends
- `name: platform-env, namespace: eqp-frontend` — consumed by eqp-portal

Both carry the same keys (`REDIS_URL`, `GCP_PROJECT_ID`, `KAFKA_BOOTSTRAP_SERVERS`, plus per-env extras like `ROUTING_ENGINE_URL`). `optional: true` lets pods start even if the overlay forgets the ConfigMap — they fall back to in-app defaults (the app surfaces a clean error rather than blocking pod admission).

`GCP_PROJECT_ID` was previously read from a pod annotation (`metadata.annotations['iam.gke.io/gcp-project-id']`), but that annotation isn't auto-populated on Autopilot+ASM — surfaced empty in `@repo/observability` (Cloud Trace exporter) and `@repo/firebase`. Per-env ConfigMap is the correct source (PR #125).

For secrets, use SPC (Section 5) — never put secret material in a ConfigMap.

## 7. Binary Authorization

**Module**: `infra/terraform/modules/binauthz/main.tf` (policy + attestor + note + signer IAM). The `infra/terraform/modules/binauthz-cv` module is separate — it wires the Continuous Validation platform policy and Pub/Sub findings sink.

Cluster has `binary_authorization.evaluation_mode = "PROJECT_SINGLETON_POLICY_ENFORCE"`. The project-singleton policy:
- **Default rule**: `ALWAYS_ALLOW` with `enforcement_mode = DRYRUN_AUDIT_LOG_ONLY` (so Cloud Run and other clusters in the project stay unbroken).
- **Per-cluster rule** for `<region>.<cluster_name>` (e.g., `us-central1.gke-eqp-dev`): `REQUIRE_ATTESTATION`, `ENFORCED_BLOCK_AND_AUDIT_LOG`, signer = the EQP attestor. Format must be exactly `<region>.<cluster_name>` — typo silently falls through to default ALWAYS_ALLOW.

**Whitelisted image patterns** (`admission_whitelist_patterns`):
- `gke.gcr.io/*`, `gcr.io/gke-release/**`, `gcr.io/gke-release-staging/**`, `registry.k8s.io/**`, `k8s.gcr.io/**`, `gcr.io/google-containers/*`, `gcr.io/stackdriver-agents/*` — Google-managed
- `gcr.io/google.com/cloudsdktool/*` — Cloud Deploy verify Jobs (PR #87)
- `gcr.io/cloud-sql-connectors/*` — Cloud SQL Auth Proxy sidecar (PR #114)
- `quay.io/jetstack/cert-manager-*` — cert-manager (Kafka Phase 1)
- `gcr.io/cloud-builders/*` — Cloud Build builder images
- `index.docker.io/edenhill/kcat:*` + `docker.io/edenhill/kcat:*` — Kafka smoke verify Job (both Docker Hub host forms because admission resolves either)

Whitelisting is preferred over signing a third-party image with our attestor (signing would mis-attest provenance — "this image was built by our pipeline").

**Cloud Build signer IAM (PR #76 + PR #78).** `sign-and-create --validate` needs four permissions; the least-privilege role that grants both `binaryauthorization.attestors.get` AND `binaryauthorization.attestors.verifyImageAttested` AND can be bound at attestor scope is **`roles/binaryauthorization.attestorsVerifier`** — NOT `attestorsViewer` (which covers only `get`) and NOT `attestorsAttacher` / Editor (which only bind at project scope). Codified in `binauthz/main.tf` resource `google_binary_authorization_attestor_iam_member.signer_verifier`.

**KMS algorithm mapping.** KMS exposes `EC_SIGN_*` / `RSA_SIGN_*`; BinAuthz expects `ECDSA_*` / `RSA_*`. `binauthz/main.tf` carries `local.signature_algorithm_map` with a precondition that fails the apply with a clear error if a rotated key uses an unsupported algorithm.

## 8. CI/CD pipeline

Tag-build pipeline lives in Google Cloud, not GitHub Actions. See `reference_cicd_gke_pipeline.md` for the full pipeline. Headline:

- **Trigger**: `<name-prefix>-tag-push` Cloud Build trigger watches `^v.*$` tag pushes (release-please owns the namespace).
- **Build**: `infra/cloudbuild-images.yaml` on the private worker pool (`eqp-<env>-cb-pool`, `e2-highcpu-32`). BuildKit/`docker buildx` (since PR #171, replacing Kaniko). Cache lives in a separate mutable AR repo `eqp-platform-cache`; image repo `eqp-platform` is immutable.
- **Sign**: BinAuthz attestation per digest.
- **Roll out**: Cloud Deploy `gcloud deploy releases create` with digest-pinned `--images=`. `skaffold render` over `infra/k8s/overlays/<env>/`.
- **DELETED**: `.github/workflows/deploy.yml` (Cloud Run path) — Plan 5 Task 9, 2026-05-02.

## 9. Cloud Deploy -> private GKE control plane connectivity

Six-PR saga (2026-04-29 -> 2026-04-30):

| PR | Attempt | Result |
|---|---|---|
| #94 | Operator-only `cidr_blocks`, public endpoint | Worker pool egress IP not in allowlist -> timeout |
| #96 | `internal_ip = true` (private endpoint via VPC peering) | VPC peering is non-transitive; would require Cloud VPN bridge |
| #98 | Public endpoint + `gcp_public_cidrs_access_enabled = true` | Default `private_endpoint_enforcement_enabled = true` blackholes the public endpoint at network layer regardless |
| #100 | Above + `private_endpoint_enforcement_enabled = false` | GKE rejects RFC1918 ranges in `cidr_blocks` once enforcement is off |
| #102 | Above + drop the worker-pool peered range from `cluster.authorized_networks` | Cloud Build PRIVATE pool egress isn't classified as "GCE Public IPs" by `gcp_public_cidrs_access_enabled` |
| **#106** | **DNS-based control plane endpoint** | ✅ Canonical fix |

**Canonical config** (per GCP docs — "DNS endpoint is the simplest way to connect to a private cluster"):
```hcl
# infra/terraform/modules/gke-autopilot/main.tf
control_plane_endpoints_config {
  dns_endpoint_config { allow_external_traffic = true }
}
master_authorized_networks_config {
  dynamic "cidr_blocks" { ... }                  # operator IPs only — defense in depth for manual kubectl
  gcp_public_cidrs_access_enabled       = true
  private_endpoint_enforcement_enabled  = false   # default is true; must explicitly disable
}

# infra/terraform/modules/cicd-pipeline/main.tf
gke {
  cluster      = var.gke_dev_cluster_id
  dns_endpoint = true
}

# infra/terraform/envs/dev/main.tf
authorized_networks = var.authorized_networks    # DO NOT concat the peered RFC1918 range
```

GKE auto-provisions a regional FQDN like `gke-<hash>-<project-num>.<region>.gke.goog` resolving to a Google-managed control-plane endpoint. Auth via gcloud creds (TLS + service account). Bypasses VPC peering, master_authorized_networks, Cloud VPN, and PSC entirely.

**Important**: Cloud Deploy snapshots target config into each release. Releases created before `dns_endpoint = true` will keep using the IP endpoint on retry — need a fresh tag to pick up the change.

## 10. Cluster authorized networks

`master_authorized_networks_config.cidr_blocks` is for **admin laptops and operator-IP allowlists only** — it does not need to contain the worker-pool peered range. The worker pool reaches the control plane via the DNS endpoint (Section 9), not via authorized_networks. PR #102 dropped the peered RFC1918 range from `authorized_networks` because GKE rejects RFC1918 entries once `private_endpoint_enforcement_enabled = false`. The earlier `concat(var.authorized_networks, [cloudbuild peered range])` pattern (from Plan 2 Task 4 Step 4.5) was the original error this whole saga inherited.

## 11. Networking

- **VPC**: `vpc-be` (pre-existing on cips-test). New envs should reference an existing VPC name via `var.vpc_name`.
- **Cloud NAT**: an `ALL_SUBNETWORKS_ALL_IP_RANGES` NAT already exists on `vpc-be` (egress IP `34.31.215.31`). GCP rejects a second NAT of that scope per router. `infra/terraform/modules/network` Cloud NAT resources are **OPTIONAL — skip on envs with an existing NAT**. Staging/prod with their own clean VPCs should apply the module normally. Partners already allowlist the existing IP on cips-test.
- **Service Networking peering**: the `cloudbuild-worker-pool` module reserves a `/24` (`google_compute_global_address` with `purpose = "VPC_PEERING"`) and creates a `google_service_networking_connection`. Cloud SQL, Memorystore, etc. hand their own `global_address` names to this module via `var.additional_peering_ranges` and ride the same single connection (GCP allows only one `(network, service)` pair).
- **Subnet self-link form**: `peered_network` on `google_cloudbuild_worker_pool.network_config` rejects full https self-links; use the relative form `projects/<id>/global/networks/<name>` (the module strips the prefix automatically).
- **Namespaces**: `eqp-system` (gateway + ASM, NO `pod-security` enforce label — gateway needs caps), `eqp-services` (11 backends, PSS `restricted` enforced), `eqp-frontend` (eqp-portal, PSS `restricted`), `eqp-batch` (empty post-PR-57). All workload namespaces carry `istio.io/rev = "asm-managed"` for sidecar auto-injection. `istio-system` is managed via `ignore_changes` on labels/annotations so ASM can mutate freely.

## 12. Tag-based deploys

Release-please owns the entire `v*` tag namespace. **Never `git tag v*` manually** — merge release-please's open PR (it auto-creates the tag on merge).

| Branch event | Tag format | Target environment | GCP project |
|---|---|---|---|
| Push to `dev` (release-please autobetabump) | `v1.2.0-beta.3` | dev | currently `cips-test`; moving to `eq-envs` |
| PR merge to `staging` | `v1.2.0-rc.1` | staging | `eq-envs-stg` |
| Release-please merge to `main` | `v1.2.0` (clean semver) | prod | `eq-envs-prod` |

Cloud Build trigger filename: `^v.*$` regex. The trigger detects which env to deploy by parsing the tag suffix (`-beta.*`, `-rc.*`, clean) in the `resolve-env` step of `infra/cloudbuild-images.yaml`. Tag-build pipeline name: `<name-prefix>-tag-push`.

## 13. Observability

**Module**: `infra/terraform/modules/monitoring-baseline/main.tf`. Slack webhook notification channel + Cloud Monitoring alert policies (`pod_restart_loop`, `mesh_error_rate`, etc.). Plan 4 soak guardrails reference these.

Cluster-side `monitoring_config.enable_components` covers SYSTEM_COMPONENTS, APISERVER, SCHEDULER, CONTROLLER_MANAGER, STORAGE, POD, DEPLOYMENT, STATEFULSET, DAEMONSET, HPA, CADVISOR, KUBELET (PCI 10.x audit trail). `managed_prometheus.enabled = true`. `logging_config.enable_components` covers SYSTEM_COMPONENTS, WORKLOADS, APISERVER, SCHEDULER, CONTROLLER_MANAGER.

Cloud Trace export uses `@repo/observability` reading `GCP_PROJECT_ID` from the `platform-env` ConfigMap (NOT from the pod annotation — see Section 6).

ASM mesh telemetry correlates by `mesh_id = "proj-${projectNumber}"` (the resource label on the cluster). Using `project_id` (slug) instead produces a label ASM doesn't recognize and mesh dashboards stay empty (`gke-autopilot/main.tf:164`).

## 14. Service namespaces

| Namespace | Workloads | PSS enforce |
|---|---|---|
| `eqp-system` | istio-ingressgateway, Gateway/RequestAuthentication/AuthorizationPolicy/EnvoyFilter CRDs | NO (gateway needs `NET_BIND_SERVICE`-equivalent context) |
| `eqp-services` | iam, session, orchestration, transaction, payment-gateway, notification, callback, config, fraud, card, business (11 services) | `restricted` (PSS version `v1.31`) |
| `eqp-frontend` | eqp-portal | `restricted` (PSS version `v1.31`) |
| `eqp-batch` | empty post-PR-57 | `restricted` |
| `istio-system` | ASM root namespace (PeerAuthentication STRICT lives here mesh-wide) | NO (ASM-managed, `ignore_changes` on labels) |
| `cert-manager` | cert-manager controller/cainjector/webhook (Kafka mTLS) | per Helm chart |

PSS `restricted` requires Pod-level + container-level `securityContext` (runAsNonRoot, allowPrivilegeEscalation=false, readOnlyRootFilesystem, capabilities.drop=[ALL], seccompProfile.type=RuntimeDefault) on every workload. Pod-level seccomp is inherited by the ASM-injected istio-proxy sidecar. eqp-portal omits `readOnlyRootFilesystem` because Next.js standalone writes to `.next/cache` at runtime (PSS `restricted` doesn't mandate it).

## 15. Public path `/api/*` convention (since 2026-05-08)

Backend APIs are served under `/api/*` at the public hostname. The IngressGateway VirtualService rewrites `/api/<prefix>/...` -> `/<prefix>/...` before forwarding so backend NestJS controllers stay unchanged (no `setGlobalPrefix('api')` in code).

**Routing rules** (`infra/k8s/base/<svc>/virtualservice.yaml`):
```yaml
http:
  - match:
      - uri: { prefix: /api/iam }
      - uri: { prefix: /api/auth/me }     # specific path, not /api/auth
      - uri: { prefix: /api/api-keys }
    rewrite:
      uriRegexRewrite: { match: ^/api/(.*)$, rewrite: /\1 }
    route:
      - destination: { host: iam-service.eqp-services.svc.cluster.local, port: { number: 3001 } }
```

The portal owns root-relative paths: `/`, `/login`, `/dashboard`, `/transactions`, `/spaces/<id>`, `/workspaces/<id>`, `/customers/<id>`, `/retailers/<id>`, `/cards`, `/smart-routing`, etc. — all fall through to the eqp-portal catch-all VirtualService (prefix `/` in `eqp-frontend`).

Portal's own internal Next.js API routes (`/api/auth/token`, `/api/auth/logout` in `apps/eqp-portal/app/api/auth/`) live under `/api/auth/*` and are intentionally NOT routed at iam-service — iam-service's VS uses the specific prefix `/api/auth/me` (PR #07b5bf8).

The `deny-no-jwt` AuthorizationPolicy (Section 3) was updated to match (`/api/<svc>/*` paths) in PR #44db756 — keep its `paths` list in sync with the VirtualService prefixes when adding a new service. Portal SSR fetchers point at `API_GATEWAY_URL=http://istio-ingressgateway.eqp-system.svc.cluster.local:80/api` and set `Host: <public-hostname>` so the Gateway listener's `hosts:` filter accepts the request.
