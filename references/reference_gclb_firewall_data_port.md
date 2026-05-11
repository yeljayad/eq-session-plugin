---
name: GCLB → istio-ingressgateway firewall — open BOTH 15021 AND 80
description: A GCLB→ingressgateway firewall that opens only the health-check port (15021) makes the backend show HEALTHY but every actual client request times out 503. Both ports must be in the allow list.
type: reference
---

Plan 3a's `module.gclb_wiring.google_compute_firewall.gclb_hc_ingressgateway` originally opened TCP **15021** only — the Istio status port that the GCLB health check probes. That made the backend health check pass and the backend show HEALTHY, but every actual client request timed out at the GCLB → NEG → pod hop with `503 upstream connect timeout` after ~9 seconds.

**Why the bug stayed hidden through Plan 3a smoke:** the smoke test was an in-cluster nginx port-forward, which never exercised the public GCLB → pod path. The bug surfaced only when Plan 3b's actual workloads tried to receive traffic from the public IP.

**Fix (`infra/terraform/modules/gclb-wiring/main.tf:14`):**

```hcl
allow {
  protocol = "tcp"
  ports    = ["15021", "80"]   # health check + data plane
}
```

Source ranges stay `["130.211.0.0/22", "35.191.0.0/16"]` (Google's documented health-checker / GCLB IPs).

The auto-created GKE rule `gke-csm-thc-<hash>` only opens TCP 7877 (CSM telemetry), not the data plane — don't rely on it.

## Diagnostic chain that finally cracked this (2026-05-01)

1. **Backend health: HEALTHY ×2 endpoints** (so health-check path works on 15021)
2. **From inside ingress pod**: `curl -H 'Host: …' http://localhost:80/auth/me` → 401 (so Envoy + routes + AuthZ healthy on port 80)
3. **Pod-to-pod**: `kubectl exec` into iam-service pod, `curl http://<ingress-pod-ip>:80/auth/me -H 'Host: …'` → 401 (so the ingress pod accepts traffic from any in-cluster IP on port 80)
4. **Public**: `curl https://<host>/auth/me` → 503 timeout (so the only thing different is firewall scope on the GCLB → pod hop)

Step 3 is the tell — pod-to-pod works because intra-cluster traffic isn't filtered by the firewall, but GCLB-routed traffic is.

## How to apply on a new env

Merge the TF fix (PR #121 on cips-test). On a fresh env, `terraform apply` creates the firewall with both ports from day one — no manual step.

## Signature: when to suspect this

If you ever see a `503 upstream connect timeout` at exactly ~9-10 seconds with the backend showing HEALTHY, this is the first thing to check. Same signature: ~9 s GCLB connect timeout despite a HEALTHY backend (because the health probe and the data plane are on different ports).
