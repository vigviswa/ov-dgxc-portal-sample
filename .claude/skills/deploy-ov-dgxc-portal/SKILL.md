---
name: deploy-ov-dgxc-portal
description: Deploy the Omniverse on DGX Cloud (OV-DGXC) web streaming portal. Guides through Docker image builds, Helm chart configuration, and Kubernetes deployment. Use when deploying, upgrading, or configuring the ov-dgxc-portal-sample.
---

<!--
Progressive Disclosure:
- Level 1 (YAML front matter): Skill metadata
- Level 2 (This file): Overview, quick start, full deployment walkthrough
- Level 3: workflows/ for detailed sub-procedures

Related skills:
- None currently
-->

# Deploy OV-DGXC Portal

Deploy the [Omniverse on DGX Cloud Portal Sample](https://github.com/NVIDIA-Omniverse/ov-dgxc-portal-sample) — a FastAPI backend + React frontend that lets users launch Kit-based streaming apps via NVCF.

## Current Version
**1.4.1** (Jan 2026) — source of truth: `CHANGELOG.md` in the repo.

## Prerequisites

| Tool | Purpose |
|------|---------|
| `docker` | Build and push container images |
| `kubectl` | Interact with Kubernetes cluster |
| `helm` | Deploy the chart |
| `jq` | Parse JSON (used in NVCF scripts) |
| `~/.npmrc` | NGC npm registry credentials for web build |

Verify tools:
```bash
docker info && kubectl version --client && helm version && jq --version
```

---

## Deployment Overview

```
1. Clone / pull repo          → get latest source
2. Configure values.local.yaml → set secrets and site-specific values
3. Build & push images         → docker build + push
4. Deploy Helm chart           → helm install / upgrade
5. Verify                      → check pods, open URL
```

---

## Step 1: Get the Repo

```bash
git clone https://github.com/NVIDIA-Omniverse/ov-dgxc-portal-sample.git
cd ov-dgxc-portal-sample
# OR if already cloned:
git pull origin main
```

---

## Step 2: Create `values.local.yaml`

Never edit `values.yaml` directly — create a `values.local.yaml` override alongside it in `helm/web-streaming-example/`:

```yaml
# helm/web-streaming-example/values.local.yaml

images:
  backend:
    repository: "<YOUR_REGISTRY>/web-streaming-example/backend"
    tag: "latest"
  web:
    repository: "<YOUR_REGISTRY>/web-streaming-example/web"
    tag: "latest"

nvcf:
  key: "<YOUR_NGC_API_KEY>"
  ngcOrg: "<YOUR_NGC_ORG_ID>"   # Cloud Account Number from NGC portal

config:
  auth:
    clientId: "<IDP_CLIENT_ID>"
    redirectUri: "https://<YOUR_HOSTNAME>/openid"
    authority: "<IDP_AUTHORITY_URL>"
    metadataUri: "<IDP_OPENID_CONNECT_DISCOVERY_URL>"
    adminGroup: "admin"
    groupsClaim: "groups"

  endpoints:
    backend: "https://<YOUR_HOSTNAME>/api"
    nucleus: "<YOUR_NUCLEUS_SERVER_URL>"    # e.g. omniverse://nucleus.example.com

  userInterface:
    title: "My Omniverse Portal"

ingress:
  hosts:
    - host: <YOUR_HOSTNAME>
  tls:
    - secretName: web-streaming-example
      hosts:
        - <YOUR_HOSTNAME>
```

### Key fields explained

| Field | Where to get it |
|-------|----------------|
| `nvcf.key` | NGC portal → Organization → API Keys |
| `nvcf.ngcOrg` | NGC portal → Organization → Cloud Account Number |
| `config.auth.clientId` | Your IdP (Keycloak / Entra ID / Okta) application registration |
| `config.auth.metadataUri` | IdP's `/.well-known/openid-configuration` URL |
| `config.auth.authority` | IdP base URL (e.g. `https://login.microsoftonline.com/<tenant-id>`) |
| `config.endpoints.nucleus` | Nucleus server URL in your environment |

---

## Step 3: Build & Push Docker Images

```bash
cd ov-dgxc-portal-sample
./build.sh <YOUR_REGISTRY>
```

The script builds `backend` and `web` images and pushes both to your registry. Requires:
- Docker daemon running
- `~/.npmrc` with NGC npm credentials for the web build (`//npm.pkg.github.com/:_authToken=...` or NGC-specific entry)

```bash
# Check ~/.npmrc exists
cat ~/.npmrc
```

---

## Step 4: Deploy with Helm

```bash
cd helm/web-streaming-example

# Fresh install
helm uninstall web-streaming-example --wait --ignore-not-found
helm install web-streaming-example . \
  -f values.yaml \
  -f values.local.yaml \
  --wait

# OR upgrade existing deployment
helm upgrade web-streaming-example . \
  -f values.yaml \
  -f values.local.yaml \
  --wait
```

---

## Step 5: Verify

```bash
# Check pods are running
kubectl get pods -l app.kubernetes.io/name=web-streaming-example

# Check ingress
kubectl get ingress

# Tail backend logs
kubectl logs -l app.kubernetes.io/component=backend -f

# Tail web/nginx logs
kubectl logs -l app.kubernetes.io/component=web -f
```

Portal should be live at `https://<YOUR_HOSTNAME>`.

---

## Common Configuration Scenarios

### Add API Keys (for CI/service-to-service access)
```yaml
# values.local.yaml
apiKeys:
  enabled: true
  keys:
    - name: "ci-pipeline"
      value: "your-secret-key-here"
```

### Enable OpenTelemetry metrics
```yaml
config:
  extraVars:
    OTEL_EXPORTER_OTLP_ENDPOINT: "http://jaeger:4317"
    OTEL_EXPORTER_OTLP_PROTOCOL: "grpc"
    OTEL_RESOURCE_ATTRIBUTES: "service.name=ov-dgxc-portal,service.version=1.4.1"
    OTEL_TRACES_EXPORTER: "otlp"
    OTEL_METRICS_EXPORTER: "otlp"
    OTEL_LOGS_EXPORTER: "otlp"
```

### Large header support (e.g. Keycloak with many groups)
```yaml
config:
  websocketsMaxLineLength: 65536
  websocketsMaxNumHeaders: 512
  h11MaxIncompleteEventSize: 65536
```

### Scale replicas
```yaml
replicaCount: 2
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70
```

---

## Create an NVCF Streaming Function

Before users can stream, you need at least one NVCF function registered. Use the provided script:

```bash
export NVCF_TOKEN="<YOUR_NGC_API_KEY>"
export STREAMING_CONTAINER_IMAGE="<YOUR_KIT_APP_IMAGE>"
export NUCLEUS_SERVER="omniverse://<YOUR_NUCLEUS_HOST>"
export STREAMING_FUNCTION_NAME="my-kit-app"

# Basic function
bash scripts/create_function.sh

# Function with DDCS and content cache
bash scripts/create_function_with_caches.sh
```

After creation, note the **Function ID** and **Function Version ID** — you'll register the app in the portal UI under Admin → Applications.

---

## Upgrade Workflow

```bash
# 1. Pull latest source
git pull origin main

# 2. Rebuild images with new source
./build.sh <YOUR_REGISTRY>

# 3. Upgrade the chart (values.local.yaml stays the same)
cd helm/web-streaming-example
helm upgrade web-streaming-example . -f values.yaml -f values.local.yaml --wait
```

See `CHANGELOG.md` for breaking changes between versions.

---

## Troubleshoot

| Symptom | Check |
|---------|-------|
| Backend `CrashLoopBackOff` | `kubectl logs <backend-pod>` — usually missing env vars (NVCF key, auth config) |
| `401 Unauthorized` on login | Verify `config.auth.metadataUri` points to correct IdP discovery URL |
| WebSocket disconnects | Increase `websocketsMaxLineLength` / `websocketsMaxNumHeaders` in values |
| Can't pull images | Check `imagePullSecrets` and registry credentials |
| NVCF functions not showing | Verify `nvcf.key` and `nvcf.ngcOrg` are correct; check backend logs for API errors |
| Session not starting | Ensure NVCF function is in `ACTIVE` state in NGC portal |

For detailed sub-procedures see:
- [workflows/configure-idp.md](workflows/configure-idp.md) — IdP setup (Keycloak, Entra ID)
- [workflows/configure-apps.md](workflows/configure-apps.md) — registering streaming apps post-deploy
