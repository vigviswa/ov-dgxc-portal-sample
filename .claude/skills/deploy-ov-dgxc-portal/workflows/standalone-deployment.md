# Standalone Deployment (NGINX on VM)

Deploy the OV-DGXC Portal on a single Linux machine using NGINX as the reverse proxy.

> **Reference:** https://docs.omniverse.nvidia.com/omniverse-dgxc/latest/portal-sample/standalone-deployment/standalone-deployment.html

## Architecture

```
Browser → HTTPS → NGINX (port 443)
                    ├── /      → React frontend (static files at /usr/share/nginx/html)
                    └── /api   → FastAPI backend (http://127.0.0.1:8000)
```

---

## Step 1: Install NGINX

```bash
sudo apt update -y
sudo apt install -y nginx
sudo systemctl stop nginx
```

> Validated with nginx 1.18.0-6ubuntu14.6

---

## Step 2: Install TLS Certificates

```bash
sudo mkdir -p /etc/nginx/certs
sudo cp <cert_file>.pem /etc/nginx/certs/cert.pem
sudo cp <key_file>.pem /etc/nginx/certs/key.pem
sudo chown -R www-data:www-data /etc/nginx
```

Verify placement:
```bash
ls -al /etc/nginx/certs/
```

Validate the certificate chain:
```bash
openssl x509 -in /etc/nginx/certs/cert.pem -text -noout
```

---

## Step 3: Configure NGINX

Create `/etc/nginx/nginx.conf` with the following content:

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
    worker_connections 16384;
}

http {
    include /etc/nginx/mime.types;
    disable_symlinks off;

    client_max_body_size 32m;
    client_body_buffer_size 128k;
    client_header_buffer_size 5120k;
    large_client_header_buffers 16 5120k;

    server {
        listen 443 ssl;

        ssl_certificate /etc/nginx/certs/cert.pem;
        ssl_certificate_key /etc/nginx/certs/key.pem;

        server_name _;

        location = /api {
            return 302 /api/;
        }
        location /api/ {
            proxy_pass http://127.0.0.1:8000/api/;
            proxy_http_version 1.1;
            proxy_read_timeout 60s;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $http_host:8000;
        }

        location / {
            root /usr/share/nginx/html;
            index index.html;
            try_files $uri $uri/ /index.html;
        }
    }

    log_format upstreamlog '[$time_local] $remote_addr - $remote_user - $server_name $host to: $upstream_addr: $request $status upstream_response_time $upstream_response_time msec $msec request_time $request_time';
    access_log /var/log/nginx/access.log upstreamlog;
    error_log /var/log/nginx/error.log warn;
}
```

Start NGINX:
```bash
sudo systemctl start nginx
sudo systemctl status nginx
```

---

## Step 4: Build and Deploy the Frontend

### Install Node.js 20 and npm

```bash
sudo apt install -y ca-certificates curl gnupg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --yes --dearmor -o /etc/apt/keyrings/nodesource.gpg
NODE_MAJOR=20
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list
sudo apt -y update
sudo apt -y install nodejs npm
```

> Validated with node 20.18.1 and npm 10.8.2

### Install Dependencies

```bash
cd web
npm install
npm list
```

### Create Frontend Config

```bash
mkdir -p ./public/config
touch ./public/config/main.json
```

Populate `web/public/config/main.json`:

```json
{
  "auth": {
    "authority": "<authority domain name>",
    "clientId": "<client id>",
    "redirectUri": "https://<your-fqdn>/openid",
    "metadataUri": "https://<issuer>/.well-known/openid-configuration",
    "scope": "openid profile email <groups-scope>"
  },
  "endpoints": {
    "backend": "https://<your-fqdn>/api",
    "nucleus": "<nucleus-server-address>"
  },
  "sessions": {
    "maxTtl": 28800,
    "sessionEndNotificationTime": 600,
    "sessionEndNotificationDuration": 30
  }
}
```

> **Note:** `authority` is NOT the OIDC authorization endpoint — it's the base URL that, when suffixed with `/.well-known/openid-configuration`, returns the OIDC discovery document.
>
> `nucleus` should be the server address **without** the `https://` prefix.

### Build and Copy to NGINX

```bash
npm run build
sudo cp -r ./dist/* /usr/share/nginx/html/
```

Restart NGINX to pick up the new files:
```bash
sudo systemctl restart nginx
```

---

## Step 5: Set Up the Backend

### Install Python 3.12+

Check current version:
```bash
python3 --version
```

Upgrade if below 3.12:
```bash
sudo apt update -y && sudo apt upgrade -y
sudo apt install software-properties-common -y
sudo add-apt-repository -y ppa:deadsnakes/ppa
sudo apt update -y
sudo apt install -y python3.12
sudo update-alternatives --install /usr/bin/python3 python /usr/bin/python3.12 2
```

### Install pip

```bash
curl -sS https://bootstrap.pypa.io/get-pip.py | python3.12
# Or: sudo apt install python3-pip
pip3 -V
```

### Install Poetry

```bash
curl -sSL https://install.python-poetry.org | python3 -
export PATH="/home/<your-username>/.local/bin:$PATH"
poetry --version
```

### Install Backend Dependencies

```bash
cd backend/
poetry install
```

### Create Backend Config

```bash
touch ./settings.toml
```

Populate `backend/settings.toml`:

```toml
root_path = "/api"
client_id = "<your OIDC client ID>"
nvcf_api_key = "<your NGC API key, e.g. nvapi-XXXX>"
metadata_uri = "https://<issuer>/.well-known/openid-configuration"
jwks_alg = "<algorithm, e.g. RS256>"
jwks_ttl = 1000
userinfo_ttl = 1000
admin_group = "<admin group name or ID>"
max_app_instances_count = 2
```

Verify:
```bash
cat settings.toml
```

### Start the Backend

```bash
poetry run api
```

> Keep the terminal open — the Python process runs in the foreground. Use a process manager (systemd, tmux, screen) for persistent deployments.

---

## Step 6: Verify

1. Navigate to `https://<YOUR_FQDN>` — you should be redirected to your IdP login page
2. After login: empty portal is expected ("no applications configured")
3. Visit `https://<YOUR_FQDN>/api/` — Swagger UI should load
4. Run `GET /apps/` from Swagger — should return `[]`

> **Note:** The FQDN in the browser must match the CN in your TLS certificate.

---

## Next: Register Streaming Apps

See [configure-apps.md](configure-apps.md) to publish NVCF functions to the portal.
