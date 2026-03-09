---
name: deploy-ov-dgxc-portal
description: Deploy the Omniverse on DGX Cloud (OV-DGXC) web streaming portal. Guides through standalone (NGINX) and Kubernetes (AKS/Helm) deployments, Docker image builds, IdP/OIDC configuration, and NVCF app publishing. Use when deploying, upgrading, or configuring the ov-dgxc-portal-sample.
---

<!--
Progressive Disclosure:
- Level 1 (YAML front matter): Skill metadata
- Level 2 (This file): Prerequisites, deployment path selection, quick reference
- Level 3: workflows/ for detailed procedures

Official docs:
- Standalone: https://docs.omniverse.nvidia.com/omniverse-dgxc/latest/portal-sample/standalone-deployment/standalone-deployment.html
- Kubernetes: https://docs.omniverse.nvidia.com/omniverse-dgxc/latest/portal-sample/k8s-deployment/k8s-deployment.html
-->

# Deploy OV-DGXC Portal

Deploy the [Omniverse on DGX Cloud Portal Sample](https://github.com/NVIDIA-Omniverse/ov-dgxc-portal-sample) — a FastAPI backend + React frontend that lets users launch Kit-based streaming apps via NVCF.

**Current version:** 1.4.1 (Jan 2026) — source of truth: `CHANGELOG.md` in the repo.

---

## Choose Your Deployment Path

| | Standalone | Kubernetes (AKS) |
|---|---|---|
| **Best for** | Simple, single-VM setups | Scalable, production-grade |
| **Reverse proxy** | NGINX on bare metal/VM | Managed AKS Ingress (NGINX) |
| **TLS** | Manual cert files | cert-manager + Let's Encrypt |
| **Guide** | [workflows/standalone-deployment.md](workflows/standalone-deployment.md) | [workflows/k8s-deployment.md](workflows/k8s-deployment.md) |

---

## Prerequisites (Both Paths)

### Required Accounts & Access
- **NGC Account** with a Personal API Key — scopes: `NVIDIA Cloud Functions` + `Private Registry`
- **Identity Provider (IdP)** — must support OpenID Connect (OIDC); you need admin access to create app registrations and manage group memberships
- **Nucleus server** FQDN (if connecting to Omniverse assets)

### Infrastructure Requirements
- Linux machine running **Ubuntu 22.04+** with admin access and internet connectivity
- A **fully qualified domain name (FQDN)** for the portal — e.g., `https://myovc.com`
- **TLS certificate + private key** in PEM format matching the FQDN (wildcard certs OK; self-signed OK for testing)

### Important Rules
- **HTTPS is mandatory** — the portal will not function correctly over plain HTTP
- **Use the FQDN, not an IP address** — WebRTC streaming requires it
- **Frontend and backend must share the same domain** — different ports under the same FQDN are treated as separate origins by browsers; use a reverse proxy

### Certificates Required
```
cert.pem   — Full-chain TLS certificate (CN must exactly match your FQDN)
key.pem    — Private key in PEM format
```

### IdP Information to Collect

Get these from your IdP admin before starting:

| Field | Example | Notes |
|-------|---------|-------|
| `authority` | `https://auth.keycloak.com/realms/ovc-auth/protocol/openid-connect/auth` | Base OIDC URL — append `/.well-known/openid-configuration` to get metadata |
| `clientId` | `portal-sample-auth` | Public app identifier from IdP app registration |
| `redirectUri` | `https://myovc.com/openid` | Must end with `/openid`; register this in your IdP |
| `metadataUri` | `https://<issuer>/.well-known/openid-configuration` | OpenID Connect Discovery URL |
| `scope` | `openid profile email groups` | Scopes requested from IdP |
| `jwksAlg` | `RS256` | Token signing algorithm |
| `jwksTtl` | `1000` | How long to cache public keys (seconds) |
| `userinfo_ttl` | `1000` | How long to cache userinfo (seconds) |
| `adminGroup` | `admin` | Group name/ID required for admin API operations |
| `nvcf_api_key` | `nvapi-XXXX` | NGC Personal API Key |
| `nucleus` | `nucleus-server.com` | Nucleus server address (no `https://` prefix) |

For IdP-specific setup (Keycloak, Entra ID, Okta) see [workflows/configure-idp.md](workflows/configure-idp.md).

---

## Get the Source

```bash
git clone https://github.com/NVIDIA-Omniverse/ov-dgxc-portal-sample.git
cd ov-dgxc-portal-sample
```

Or pull latest if already cloned:
```bash
git pull origin main
```

---

## Deployment Guides

- **Standalone (NGINX on VM):** [workflows/standalone-deployment.md](workflows/standalone-deployment.md)
- **Kubernetes (AKS + Helm):** [workflows/k8s-deployment.md](workflows/k8s-deployment.md)
- **Configure IdP (Keycloak / Entra ID / Okta):** [workflows/configure-idp.md](workflows/configure-idp.md)
- **Register streaming apps post-deploy:** [workflows/configure-apps.md](workflows/configure-apps.md)

---

## After Deployment — Verify

1. Open `https://<YOUR_FQDN>` in a browser — you should be redirected to your IdP login
2. After login: "no Omniverse on DGX Cloud applications have been configured" is expected on first run
3. Check the API: `https://<YOUR_FQDN>/api/` — Swagger UI should load
4. Run `GET /apps/` — returns `[]` if no apps published yet

---

## Quick Upgrade

```bash
git pull origin main          # get latest source
./build.sh <YOUR_REGISTRY>    # rebuild images
# For standalone: restart backend + copy new web dist
# For k8s: helm upgrade (see k8s-deployment.md)
```

Check `CHANGELOG.md` for breaking changes between versions.
