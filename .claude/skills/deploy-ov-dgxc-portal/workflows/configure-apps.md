# Register Streaming Applications

After the portal is deployed, you register Kit streaming apps through the Admin UI or API.

## Via Admin UI

1. Log in with an account in the `adminGroup`
2. Go to **Admin → Applications**
3. Click **Add Application**
4. Fill in:
   - **Name**: Display name shown to users
   - **NVCF Function ID**: From `scripts/create_function.sh` output or NGC UI
   - **NVCF Version ID**: From the same output
   - **Page** (optional): Group apps on a sidebar page

## Via API (using API key)

```bash
curl -X PUT https://<YOUR_HOSTNAME>/api/apps/ \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <YOUR_API_KEY>" \
  -d '[
    {
      "id": "<NVCF_FUNCTION_ID>",
      "versionId": "<NVCF_FUNCTION_VERSION_ID>",
      "name": "USD Composer",
      "page": "Design Tools"
    }
  ]'
```

The PUT replaces the full app list, so always include all apps you want visible.

## Check Registered Apps

```bash
curl https://<YOUR_HOSTNAME>/api/apps/ \
  -H "X-API-Key: <YOUR_API_KEY>"
```

## Check App Pages (sidebar)

```bash
curl https://<YOUR_HOSTNAME>/api/pages/ \
  -H "X-API-Key: <YOUR_API_KEY>"
```

## NVCF Function States

| State | Meaning |
|-------|---------|
| `ACTIVE` | Ready to accept sessions |
| `DEPLOYING` | Still initializing — users will see "unavailable" |
| `DEGRADING` | Partially available — portal shows warning |
| `DEGRADED` | Function unhealthy |
| `ERROR` | Failed — check NGC portal logs |

Only `ACTIVE` functions can accept new streaming sessions.
