---
name: ASM deny-no-jwt — adding/removing exempt paths
description: The Plan 2 deny-no-jwt AuthorizationPolicy at the IngressGateway. PR #130 INVERTED it to a positive `paths:` list (deny only backend prefixes). PR #44db756 updated the list to `/api/<svc>/*` for the 2026-05-08 routing migration. To add a new public auth/webhook endpoint, REMOVE it from the list; to add a new backend, ADD it.
type: reference
---

Source: `infra/terraform/modules/asm-platform/main.tf` resource `kubernetes_manifest.authz_policy_deny_default` (TF source of truth). Mirrored at `infra/k8s/base/platform/deny-no-jwt.yaml` for kubectl-direct apply when TF connectivity to the cluster is blocked (commit `f30f176`).

## Current shape (inverted positive list)

```yaml
action: DENY
selector:
  matchLabels:
    istio: ingressgateway
rules:
  - from:
    - source:
        notRequestPrincipals: ["*"]   # no JWT principal
    to:
    - operation:
        paths:                         # deny ONLY these prefixes
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

Reads as: "deny any request to the IngressGateway where there is NO JWT principal AND the path matches a backend prefix. All other paths fall through unauthenticated (handled by the portal catch-all VirtualService)."

## How to apply

| Change | Action |
|---|---|
| Adding a new backend service | ADD its prefix to `paths:` (and create the matching VirtualService at `infra/k8s/base/<svc>/virtualservice.yaml`) |
| Adding a public auth-flow endpoint (e.g. `/api/auth/me`) | LEAVE it out of `paths:` — it falls through deny-no-jwt and reaches the backend, which returns 401 itself if no JWT |
| Adding a public webhook (e.g. PSP callback) | LEAVE it out of `paths:` — the webhook receiver validates by signature/HMAC, not JWT |
| Removing a backend | REMOVE its prefix |

**Apply:** `terraform apply -target=module.asm_platform.kubernetes_manifest.authz_policy_deny_default` from the env's terraform dir. Verify with:
```bash
kubectl get authorizationpolicy -n eqp-system deny-no-jwt -o yaml
```

If the TF apply times out connecting to the cluster (private cluster + worker pool reachability), apply the mirrored YAML directly:
```bash
kubectl apply -f infra/k8s/base/platform/deny-no-jwt.yaml
```

## Why inverted (PR #130, 2026-05-02)

The original Plan 2 shape was a deny-everything-except policy:
```yaml
to:
- operation:
    notPaths:
      - /health
      - /healthz
      - /auth/*
      - /iam/auth/*
```
That **broke the portal**. Every browser route (`/`, `/login`, `/dashboard`, `/_next/*`, `/favicon.ico`) was denied with `403 RBAC: access denied` because the unauthenticated browser had no JWT principal. PR #118 tried to add more `notPaths` exemptions and kept missing static assets; PR #130 inverted the rule entirely — much smaller surface area, easier to reason about.

## Why `/api/*` prefixes (PR #44db756, 2026-05-08)

PR #07b5bf8 moved all backend APIs under `/api/<svc>/*` at the gateway. The deny-no-jwt `paths:` list was updated to match — `/iam/*` → `/api/iam/*`, etc. The IngressGateway VirtualService rewrites `/api/<svc>/...` → `/<svc>/...` before forwarding, so backend NestJS controllers stay at their original prefixes.

Note: `/api/auth/me` is intentionally NOT in the deny list — iam-service serves its own public auth-flow endpoints and returns 401 itself if no JWT. Backends still independently enforce auth via `TokenGuard`; the gateway DENY is only the first line.

## Diagnostic

From inside any ingressgateway pod:
```bash
curl -H 'Host: <public-host>' http://localhost:80/api/<svc>/test
```
Returns `403 RBAC: access denied` if deny-no-jwt is firing for that path. That's the cleanest way to confirm the policy is actually applied vs. some other layer denying.

If you see 403 on a path you expected to pass through, check:
1. The path is in the `paths:` list (it shouldn't be, if you want it public).
2. The TF apply landed (use `kubectl get` to see the live policy YAML).
3. The mirrored kubectl YAML wasn't drifted from the TF apply.
