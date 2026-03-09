# Register and Publish Streaming Applications

After the portal is deployed, register NVCF streaming functions so users can launch them.

> **Reference:** https://docs.omniverse.nvidia.com/omniverse-dgxc/latest/portal-sample/verification-and-publishing.html

---

## Prerequisites

You need the **Function ID** and **Function Version ID** from NVCF. Get them by:
- Running `scripts/create_function.sh` (see below), or
- Looking them up in the [NGC portal](https://nvcf.ngc.nvidia.com) under your deployed functions

---

## Create an NVCF Streaming Function

### Basic Function

```bash
export NVCF_TOKEN="<your NGC API key>"
export STREAMING_CONTAINER_IMAGE="<your Kit app container image>"
export NUCLEUS_SERVER="omniverse://<your-nucleus-host>"
export STREAMING_FUNCTION_NAME="my-kit-app"          # optional, defaults to usd-composer
export STREAMING_START_ENDPOINT="/sign_in"            # optional
export STREAMING_SERVER_PORT=49100                    # optional
export CONTROL_SERVER_PORT=8111                       # optional

bash scripts/create_function.sh
```

### Function with DDCS and Content Cache

```bash
bash scripts/create_function_with_caches.sh
```

Both scripts output:
```
Function ID: <uuid>
Function Version ID: <uuid>
```

Note these — you'll need them to register the app in the portal.

---

## Publish via Swagger UI

1. Open `https://<YOUR_FQDN>/api/` in your browser (must be logged in as admin)
2. Find the `PUT /apps/` endpoint
3. Submit with your app details

> **Important:** Only members of the `adminGroup` can publish applications. Do NOT access Swagger directly via the backend URL (e.g. `http://127.0.0.1:8000/api/`) as this bypasses authentication.

### Required Fields for PUT /apps/

| Field | Format | Notes |
|-------|--------|-------|
| `id` | `[a-zA-Z\d-_]+` | Unique app identifier slug |
| `name` | string | Display name shown to users |
| `function_id` | UUID | From NVCF function creation |
| `function_version_id` | UUID | From NVCF function creation |
| `version` | semver | Must be semantic versioning (e.g. `1.0.0`) |
| `icon` | URL | URL to app bitmap icon |
| `authentication_type` | `NUCLEUS` or empty | Set `NUCLEUS` if Nucleus mounting required |
| `media_server` | IP or empty | Private endpoint IP for Azure PE/PLS streaming; empty for public |
| `page` | string (optional) | Sidebar page group name |

---

## Publish via API (using API key)

```bash
curl -X PUT https://<YOUR_FQDN>/api/apps/ \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <YOUR_API_KEY>" \
  -d '[
    {
      "id": "usd-composer",
      "name": "USD Composer",
      "function_id": "<NVCF_FUNCTION_ID>",
      "function_version_id": "<NVCF_FUNCTION_VERSION_ID>",
      "version": "2025.1.0",
      "icon": "https://example.com/icon.png",
      "authentication_type": "NUCLEUS",
      "media_server": "",
      "page": "Design Tools"
    }
  ]'
```

> **Note:** `PUT /apps/` replaces the entire app list. Always include all apps you want visible.

---

## Check Registered Apps

```bash
# Via API key
curl https://<YOUR_FQDN>/api/apps/ -H "X-API-Key: <YOUR_API_KEY>"

# Via browser (logged in as admin) — Swagger GET /apps/
```

## Check App Pages (sidebar groupings)

```bash
curl https://<YOUR_FQDN>/api/pages/ -H "X-API-Key: <YOUR_API_KEY>"
```

---

## Enable API Keys

To enable API key authentication for CI/service-to-service use:

**Standalone:** add to `backend/settings.toml`:
```toml
[api_keys]
enabled = true

[[api_keys.keys]]
name = "ci-pipeline"
value = "your-secret-key-here"
```

**Kubernetes:** add to `values.local.yaml`:
```yaml
apiKeys:
  enabled: true
  keys:
    - name: "ci-pipeline"
      value: "your-secret-key-here"
```

---

## NVCF Function States

| State | Meaning | Portal Behavior |
|-------|---------|----------------|
| `ACTIVE` | Ready to accept sessions | Users can launch |
| `DEPLOYING` | Still initializing | Shows as unavailable |
| `DEGRADING` | Partially available | Portal shows warning |
| `DEGRADED` | Function unhealthy | Shows degraded status |
| `ERROR` | Failed | Check NGC portal logs |

Only `ACTIVE` functions accept new streaming sessions.

---

## Session Management

The portal tracks sessions in four states:

| State | Meaning |
|-------|---------|
| `Active` | User currently connected |
| `Connecting` | Establishing connection |
| `Idle` | Temporarily disconnected but resumable |
| `Stopped` | Terminated |

- **Regular users** can view and terminate only their own sessions
- **Admin users** can view and forcefully terminate any session
- Idle sessions auto-terminate after `sessionIdleTimeout` seconds (default: 300s, max 300s — must match NVCF function config)
- Sessions auto-terminate after `sessionTtl` seconds (default: 28800s = 8 hours)
