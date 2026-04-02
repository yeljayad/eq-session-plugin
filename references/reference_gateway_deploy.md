---
name: Gateway deployment architecture
description: ESPv2 gateway deploy flow, service names, API key setup, and common pitfalls
type: reference
---

## ESPv2 Gateway Deployment

### Two Cloud Endpoints services exist
- `api-gateway-v2-857138853031.us-central1.run.app` — the ACTIVE one (Cloud Run URL)
- `api-gateway-v2.endpoints.cips-test.cloud.goog` — OLD, stopped at March 10 2026 config

ESPv2 must use `ENDPOINTS_SERVICE_NAME=api-gateway-v2-857138853031.us-central1.run.app`

### Deploy flow (CI: deploy-gateway job in deploy.yml)
1. `redocly bundle` → single-file spec
2. `substitute-env.sh` → replaces SERVICE_IDENTIFIER, PROJECT_ID, BASE_PATH, GATEWAY_CLOUD_RUN_HOSTNAME
3. `gcloud endpoints services deploy` → uploads config to Cloud Endpoints
4. `gcloud_build_image.sh` → builds ESPv2 image with baked-in config
5. `gcloud run deploy` → deploys ESPv2 to Cloud Run with `ENDPOINTS_SERVICE_NAME` env var

### Critical GitHub variables (repo Settings → Environments)
| Variable | Value |
|----------|-------|
| `SERVICE_IDENTIFIER` | `857138853031.us-central1` |
| `GCP_PROJECT_ID` | `cips-test` |
| `GATEWAY_CLOUD_RUN_HOSTNAME` | `api-gateway-v2-857138853031.us-central1.run.app` |

### API Key for gateway
- Must be a GCP API key created in the `cips-test` project
- The Cloud Endpoints service must be enabled for the key's project
- Key from a different project → 404 (Service Control rejects it)
- Current key: `AIzaSyAJQFfq-ndf23MIQckHcAZC31pvU9ZWymM` (display name: "EQP Gateway Key")

### Common pitfalls
- **Empty SERVICE_IDENTIFIER** → backend addresses become `https://iam-service-.run.app` → 404 after auth passes
- **Wrong ENDPOINTS_SERVICE_NAME** → ESPv2 loads old config from different service
- **API key from different project** → Google 404 (not the usual 403)
- **`gcloud` commands in CI sandbox return exit 120** → use `script -q -c "..." /dev/null` wrapper
