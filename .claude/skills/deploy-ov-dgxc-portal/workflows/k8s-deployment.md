# Kubernetes Deployment (AKS + Helm)

Deploy the OV-DGXC Portal on Azure Kubernetes Service with managed NGINX ingress, cert-manager, and Let's Encrypt.

> **Reference:** https://docs.omniverse.nvidia.com/omniverse-dgxc/latest/portal-sample/k8s-deployment/k8s-deployment.html
>
> **Warning:** The nip.io / Let's Encrypt configuration below is for demonstration/testing only. Production deployments must use your organization's DNS and certificate management infrastructure.

---

## Prerequisites

- Azure subscription with permissions to create AKS resources
- Linux workstation with Docker installed
- NGC Account with Personal API Key (NVCF + Private Registry scopes)
- OIDC identity provider configured
- Nucleus server FQDN (if using Omniverse assets)

---

## Step 1: Create an AKS Cluster

If you already have an AKS cluster, skip to Step 2.

1. Go to: https://portal.azure.com/#browse/Microsoft.ContainerService%2FmanagedClusters
2. Click **Kubernetes Cluster**
3. Configure a node pool — the portal has minimal requirements: **2-4 vCPUs, 4-8 GB RAM** is sufficient
4. Accept defaults or customize per your org's requirements
5. Create and wait for deployment to complete

---

## Step 2: Prepare the AKS Cluster

### Install Azure CLI

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

### Log in to Azure

```bash
az login --use-device-code
```

Open https://microsoft.com/devicelogin and enter the displayed code. If you have multiple tenants, select the right one.

### Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm -rf kubectl
kubectl version
```

### Get kubeconfig

> **Warning:** The command below deletes existing Kubernetes configurations. Back up `~/.kube` first if needed.

```bash
rm -rf .kube
az aks list --output table   # note Name and ResourceGroup
az aks get-credentials --admin --name <cluster-name> --resource-group <resource-group>
kubectl get nodes             # verify connection
```

### Install Helm

```bash
wget https://get.helm.sh/helm-v3.17.1-linux-amd64.tar.gz
tar -zxvf helm-v3.17.1-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
rm -rf linux-amd64 helm-v3.17.1-linux-amd64.tar.gz
helm version
```

### Install Managed AKS Ingress

Skip if you already have a custom Ingress or the managed AKS Ingress is installed.

```bash
kubectl get ingressClasses -A  # should return nothing if not installed
az aks approuting enable --resource-group <resource-group> --name <cluster-name>
# Takes ~5 minutes
kubectl get ingressClasses -A  # should now show webapprouting.kubernetes.azure.com
kubectl get svc nginx -n app-routing-system  # note the EXTERNAL-IP
```

Note the `EXTERNAL-IP` — you'll use it for DNS.

---

## Step 3: DNS Configuration

### Option A: Production (Recommended)
Configure a proper DNS A record pointing your FQDN to the EXTERNAL-IP. Consult your IT/DevOps team.

### Option B: Testing with nip.io (no DNS config required)

Construct your FQDN using the pattern: `ov-portal.<EXTERNAL-IP>.nip.io`

Example: if EXTERNAL-IP is `4.255.101.179`, your FQDN is `ov-portal.4.255.101.179.nip.io`

Verify it resolves:
```bash
curl http://ov-portal.<EXTERNAL-IP>.nip.io
# Expected: 404 from nginx (that's correct at this stage)
```

---

## Step 4: TLS Certificates

### Option A: Import Existing Certificates
Create a Kubernetes secret from your cert files and reference it in the Helm values. No cert-manager needed.

```bash
kubectl create secret tls web-streaming-example \
  --cert=cert.pem \
  --key=key.pem \
  -n ov-portal
```

Then in your Helm deploy command, omit the cert-manager issuer annotation and set `certificate.create=false` and `clusterIssuer.create=false`.

### Option B: Let's Encrypt via cert-manager (testing only)

Install cert-manager:
```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.0 \
  --set crds.enabled=true
```

Create namespace for the portal:
```bash
kubectl create namespace ov-portal
```

Create Issuers (use staging first to avoid production rate limits):
```bash
kubectl apply -n ov-portal -f - << EOF
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: <YOUR_VALID_EMAIL>
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
      - http01:
          ingress:
            ingressClassName: webapprouting.kubernetes.azure.com
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <YOUR_VALID_EMAIL>
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            ingressClassName: webapprouting.kubernetes.azure.com
EOF
```

> **Note:** Use a real email — Let's Encrypt uses it only to notify you 30 days before expiry if auto-renewal fails.

Verify issuers:
```bash
kubectl get issuer -n ov-portal
```

---

## Step 5: Build and Push Container Images

```bash
git clone https://github.com/NVIDIA-Omniverse/ov-dgxc-portal-sample
cd ov-dgxc-portal-sample

# Authenticate to NGC registry
docker login nvcr.io
# Username: $oauthtoken
# Password: <your NGC Personal API Key>

# Get your NGC Cloud Account ID (NCA_ID) from: https://org.ngc.nvidia.com/profile
export NCA_ID=<your-nca-id>
export DOCKER_REGISTRY=nvcr.io/$NCA_ID
export DOCKER_BUILDKIT=1

# Build and push backend
docker build -t $DOCKER_REGISTRY/ov-portal-backend:latest -f backend/Dockerfile backend
docker push $DOCKER_REGISTRY/ov-portal-backend:latest

# Build and push frontend (requires ~/.npmrc with NGC npm credentials)
docker build -t $DOCKER_REGISTRY/ov-portal-frontend:latest -f web/Dockerfile --secret id=npmrc,src=./web/.npmrc web
docker push $DOCKER_REGISTRY/ov-portal-frontend:latest
```

---

## Step 6: Deploy with Helm

### Set Variables

```bash
export NVCF_TOKEN=<your-ngc-api-key>
export MY_OV_PORTAL_FQDN=<your-portal-fqdn>          # e.g. ov-portal.4.255.101.179.nip.io
export MY_OV_NUCLEUS_FQDN=<your-nucleus-fqdn>
export DOCKER_REGISTRY=nvcr.io/<your-nca-id>
```

### Create Image Pull Secret

```bash
kubectl create secret docker-registry ngc \
  -n ov-portal \
  --docker-server=nvcr.io \
  --docker-username='$oauthtoken' \
  --docker-password=$NVCF_TOKEN
```

### Deploy

```bash
helm upgrade --install web-streaming-example ./helm/web-streaming-example/. \
  -f ./helm/web-streaming-example/values.yaml \
  -n ov-portal --create-namespace \
  --set images.backend.repository="$DOCKER_REGISTRY/ov-portal-backend" \
  --set images.backend.tag=latest \
  --set images.web.repository="$DOCKER_REGISTRY/ov-portal-frontend" \
  --set images.web.tag=latest \
  --set imagePullSecrets[0].name=ngc \
  --set nvcf.key=$NVCF_TOKEN \
  --set ingress.className=webapprouting.kubernetes.azure.com \
  --set ingress.hosts[0].host=$MY_OV_PORTAL_FQDN \
  --set ingress.tls[0].hosts[0]=$MY_OV_PORTAL_FQDN \
  --set "ingress.annotations.cert-manager\.io\/issuer=letsencrypt-prod" \
  --set config.auth.clientId="<your OIDC client ID>" \
  --set config.auth.redirectUri="https://$MY_OV_PORTAL_FQDN/openid" \
  --set config.auth.authority="<your OIDC authority URL>" \
  --set config.auth.metadataUri="<your OIDC .well-known URL>" \
  --set config.auth.jwksAlg="RS256" \
  --set config.auth.jwksTtl=1000 \
  --set config.auth.userinfoTtl=1000 \
  --set config.auth.scope="openid profile email" \
  --set config.auth.adminGroup="<your admin group name or ID>" \
  --set config.endpoints.backend="https://$MY_OV_PORTAL_FQDN/api" \
  --set config.endpoints.nucleus="$MY_OV_NUCLEUS_FQDN" \
  --set config.maxAppInstancesCount=2 \
  --set certificate.create=false \
  --set clusterIssuer.create=false
```

> **Tip:** While testing, use `letsencrypt-staging` instead of `letsencrypt-prod` to avoid hitting rate limits. Switch to `letsencrypt-prod` once you confirm everything works.

### Verify Certificate Issuance

```bash
kubectl get certificates -n ov-portal
# NAME                    READY   SECRET                  AGE
# web-streaming-example   True    web-streaming-example   2m13s
```

Certificate issuance can take a few minutes. If it stays `False`, check cert-manager logs.

---

## Step 7: Verify Deployment

```bash
# Check pods
kubectl get pods -n ov-portal -l app.kubernetes.io/name=web-streaming-example

# Check ingress
kubectl get ingress -n ov-portal

# Tail backend logs
kubectl logs -n ov-portal -l app.kubernetes.io/component=backend -f

# Tail frontend/nginx logs
kubectl logs -n ov-portal -l app.kubernetes.io/component=web -f
```

Open `https://<YOUR_FQDN>` — should redirect to IdP login.

---

## Using values.local.yaml Instead of CLI Flags

For repeated deployments, create `helm/web-streaming-example/values.local.yaml` instead of passing all `--set` flags:

```yaml
images:
  backend:
    repository: "nvcr.io/<NCA_ID>/ov-portal-backend"
    tag: "latest"
  web:
    repository: "nvcr.io/<NCA_ID>/ov-portal-frontend"
    tag: "latest"

imagePullSecrets:
  - name: ngc

nvcf:
  key: "<YOUR_NGC_API_KEY>"
  ngcOrg: "<YOUR_NCA_ID>"

config:
  auth:
    clientId: "<OIDC_CLIENT_ID>"
    redirectUri: "https://<YOUR_FQDN>/openid"
    authority: "<OIDC_AUTHORITY_URL>"
    metadataUri: "<OIDC_METADATA_URL>"
    jwksAlg: "RS256"
    jwksTtl: 1000
    userinfoTtl: 1000
    adminGroup: "<ADMIN_GROUP>"
    scope: "openid profile email"
  endpoints:
    backend: "https://<YOUR_FQDN>/api"
    nucleus: "<NUCLEUS_FQDN>"
  maxAppInstancesCount: 2

ingress:
  className: webapprouting.kubernetes.azure.com
  annotations:
    cert-manager.io/issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-buffer-size: "128k"
    nginx.ingress.kubernetes.io/proxy-buffers-number: "8"
    nginx.ingress.kubernetes.io/client-header-buffer-size: "128k"
    nginx.ingress.kubernetes.io/large-client-header-buffers: "8 128k"
  hosts:
    - host: <YOUR_FQDN>
  tls:
    - secretName: web-streaming-example
      hosts:
        - <YOUR_FQDN>

certificate:
  create: false
clusterIssuer:
  create: false
```

Then deploy with:
```bash
helm upgrade --install web-streaming-example ./helm/web-streaming-example/. \
  -f ./helm/web-streaming-example/values.yaml \
  -f ./helm/web-streaming-example/values.local.yaml \
  -n ov-portal --create-namespace
```

---

## Troubleshoot

| Symptom | Check |
|---------|-------|
| Backend `CrashLoopBackOff` | `kubectl logs <backend-pod> -n ov-portal` — usually misconfigured auth/nvcf settings |
| Certificate stays `False` | `kubectl describe certificate web-streaming-example -n ov-portal` — check cert-manager events |
| `401 Unauthorized` | Verify `metadataUri` is reachable: `kubectl exec -it <backend-pod> -n ov-portal -- curl <metadataUri>` |
| Images not pulling | Check `imagePullSecrets` and `docker login nvcr.io` succeeded |
| WebSocket disconnects | Increase `websocketsMaxLineLength` / `websocketsMaxNumHeaders` in values |
| NVCF functions not showing | Verify `nvcf.key` and `nvcf.ngcOrg`; check backend logs for API errors |

---

## Next: Register Streaming Apps

See [configure-apps.md](configure-apps.md) to publish NVCF functions to the portal.
