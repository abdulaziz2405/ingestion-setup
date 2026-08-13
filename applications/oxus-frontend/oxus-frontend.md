# Deploying Oxus-frontend

Variables that are used here and require change:
- `<APPLICATION_SERVER>`: IP of the server this app runs on
- `<PIPELINE_IID>`: IID of the last successful pipeline, taken from the GitLab UI

Frontend does not need Vault Agent.

## 1. Setting up environment

Create directory:

```bash
mkdir -p /etc/oxus-frontend
```

Pull the initial image. Root has to be logged in to the registry, see
[GitLab Runner](../../tools/gitlab-runner.md), step 4:

```bash
docker pull registry.gitlab.com/amudario/development/frontend:agrobank-<PIPELINE_IID>
```

Create the .env file for image tagging:

```bash
cat > /etc/oxus-frontend/agrobank.env <<'EOF'
IMAGE=registry.gitlab.com/amudario/development/frontend
TAG=agrobank-<PIPELINE_IID>
EOF

chmod 600 /etc/oxus-frontend/agrobank.env
```

## 2. Systemd unit

Create `/etc/systemd/system/agrobank-app-oxus-frontend.service` and paste the following code:

```ini
[Unit]
Description=Oxus Frontend
Wants=network-online.target
Requires=docker.service network-online.target
After=docker.service network-online.target

[Service]
Type=simple
EnvironmentFile=/etc/oxus-frontend/agrobank.env

ExecStartPre=/usr/bin/docker pull ${IMAGE}:${TAG}

ExecStartPre=-/usr/bin/docker rm -f oxus-frontend-app-agrobank

ExecStart=/usr/bin/docker run --rm \
    --name oxus-frontend-app-agrobank \
    -p <APPLICATION_SERVER>:9090:80 \
    --health-cmd 'wget -qO- http://127.0.0.1:80/healthz >/dev/null 2>&1 || exit 1' \
    --health-interval 30s \
    --health-timeout 5s \
    --health-retries 3 \
    --health-start-period 10s \
    ${IMAGE}:${TAG}

ExecStop=/usr/bin/docker stop oxus-frontend-app-agrobank
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

Start and check the unit:

```bash
systemctl daemon-reload
systemctl enable --now agrobank-app-oxus-frontend
systemctl status agrobank-app-oxus-frontend

docker logs oxus-frontend-app-agrobank --tail=50
```

## 3. Gateway

The containers listen on the internal NIC only. The gateway NGINX is the single entry point and
routes by path: `/api/` to the backend, `/models/` to oxus-models, `/prefect` to the Prefect
server UI, everything else to this frontend.

The upstreams go at the `http` level of the NGINX config:

```nginx
upstream amudario-agrobank-client-frontend {
    server <APPLICATION_SERVER>:9090;
}

upstream amudario-agrobank-client-backend {
    server <APPLICATION_SERVER>:8080;
}

upstream amudario-agrobank-client-models {
    server <APPLICATION_SERVER>:8011;
}

upstream amudario-agrobank-client-prefect {
    server <APPLICATION_SERVER>:4200;
}
```

The locations go inside the `server` block that terminates `<DOMAIN>`.
No `proxy_pass` target ends with a slash, so the path prefix is passed through untouched. That
is what the backend, oxus-models (`API_ROOT_PATH=/models/api`) and the Prefect server
(`PREFECT_UI_SERVE_BASE=/prefect`) expect:

```nginx
    location /api/ {
        proxy_pass http://amudario-agrobank-client-backend;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
    }

    location /models/ {
        proxy_pass http://amudario-agrobank-client-models;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
    }

    location /prefect {
        proxy_pass http://amudario-agrobank-client-prefect;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # the Prefect UI holds its event stream open over websockets
        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 120s;
    }

    location / {
        proxy_pass http://amudario-agrobank-client-frontend;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
```

The `<DOMAIN>` served here is the one the Prefect env files point at through
`PREFECT_UI_URL` and `PREFECT_UI_API_URL`.