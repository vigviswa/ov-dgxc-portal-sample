# Configure Identity Provider (IdP)

The portal uses OpenID Connect for authentication. Here's how to configure common IdPs.

## Microsoft Entra ID (Azure AD)

1. Go to Azure Portal → Entra ID → App Registrations → New Registration
2. Set redirect URI to: `https://<YOUR_HOSTNAME>/openid`
3. Under **Certificates & secrets**, create a client secret
4. Under **Token configuration**, add a groups claim
5. Note your **Application (client) ID** and **Tenant ID**

```yaml
config:
  auth:
    clientId: "<APPLICATION_CLIENT_ID>"
    redirectUri: "https://<YOUR_HOSTNAME>/openid"
    authority: "https://login.microsoftonline.com/<TENANT_ID>"
    metadataUri: "https://login.microsoftonline.com/<TENANT_ID>/v2.0/.well-known/openid-configuration"
    jwksAlg: "RS256"
    adminGroup: "<ENTRA_GROUP_OBJECT_ID>"
    groupsClaim: "groups"
```

## Keycloak

1. Create a new Realm (or use existing)
2. Create a Client with `openid-connect` protocol
3. Set Valid Redirect URIs to `https://<YOUR_HOSTNAME>/openid`
4. Under Client Scopes → add `groups` mapper

```yaml
config:
  auth:
    clientId: "<KEYCLOAK_CLIENT_ID>"
    redirectUri: "https://<YOUR_HOSTNAME>/openid"
    authority: "https://<KEYCLOAK_HOST>/realms/<REALM>"
    metadataUri: "https://<KEYCLOAK_HOST>/realms/<REALM>/.well-known/openid-configuration"
    jwksAlg: "RS256"
    adminGroup: "admin"
    groupsClaim: "groups"
```

## Okta

```yaml
config:
  auth:
    clientId: "<OKTA_CLIENT_ID>"
    redirectUri: "https://<YOUR_HOSTNAME>/openid"
    authority: "https://<YOUR_OKTA_DOMAIN>/oauth2/default"
    metadataUri: "https://<YOUR_OKTA_DOMAIN>/oauth2/default/.well-known/openid-configuration"
    jwksAlg: "RS256"
    adminGroup: "admin"
    groupsClaim: "groups"
```

## Testing Auth Config

Once deployed, verify the metadata URI is reachable from the backend pod:
```bash
kubectl exec -it <backend-pod> -- curl <YOUR_METADATA_URI>
```
