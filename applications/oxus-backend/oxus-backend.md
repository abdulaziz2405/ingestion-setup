# Deploying Oxus-backend

Variables that are used here and require change:
- `<APPLICATION_SERVER>`: IP of the server this app runs on
- `<PIPELINE_IID>`: IID of the last successful pipeline, taken from the GitLab UI

## 1. Secret store

Vault and Vault Agent are used here. Make sure they're up before proceeding.

In Vault UI, at path `secret/amudario/applications/oxus-backend`, add the necessary variables.
You can see the example ones without values, at example.env.

To pull the credentials, application relies on Vault Agent functioning on the same server.

## 2. Setting up environment

Create directory:

```bash
mkdir -p /etc/oxus-backend
```

Pull the initial image. Root has to be logged in to the registry, see
[GitLab Runner](../../tools/gitlab-runner.md), step 3:

```bash
docker pull registry.gitlab.com/amudario/development/backend2:agrobank-<PIPELINE_IID>
```

Create the .env file for image tagging:
```bash
cat > /etc/oxus-backend/agrobank.env <<'EOF'
IMAGE=registry.gitlab.com/amudario/development/backend2
TAG=agrobank-<PIPELINE_IID>
EOF
chmod 600 /etc/oxus-backend/agrobank.env
```

## 3. Systemd unit

Create `/etc/systemd/system/agrobank-app-oxus-backend.service` and paste the following code:

```ini
[Unit]
Description=Oxus Backend API
Wants=dev-secretstore-vault-agent.service
Requires=docker.service network-online.target
After=docker.service dev-secretstore-vault-agent.service network-online.target

[Service]
Type=simple
EnvironmentFile=/etc/oxus-backend/agrobank.env

ExecStartPre=/usr/bin/docker pull ${IMAGE}:${TAG}

ExecStartPre=/usr/bin/test -s /opt/app/shared/secrets/oxus-backend-agro.env

ExecStartPre=-/usr/bin/docker rm -f oxus-backend-app-agrobank

ExecStartPre=/usr/bin/docker run --rm \
    --name oxus-backend-app-migration-1 \
    -e WWWUSER=1000 \
    -v /opt/app/shared/secrets/oxus-backend-agro.env:/var/www/html/.env:ro \
    --entrypoint sh \
    ${IMAGE}:${TAG} \
    -c 'php artisan migrate --force'

ExecStartPre=/usr/bin/docker run --rm \
    --name oxus-backend-app-migration-2 \
    -e WWWUSER=1000 \
    -v /opt/app/shared/secrets/oxus-backend-agro.env:/var/www/html/.env:ro \
    --entrypoint sh \
    ${IMAGE}:${TAG} \
    -c 'php artisan config:cache'

ExecStart=/usr/bin/docker run --rm \
    --name oxus-backend-app-agrobank \
    -e WWWUSER=1000 \
    -p <APPLICATION_SERVER>:8080:80 \
    -v /opt/app/shared/secrets/oxus-backend-agro.env:/var/www/html/.env:ro \
    --health-cmd '[ "$(curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:80/api/health)" -eq 200 ] || exit 1' \
    --health-interval 30s \
    --health-timeout 5s \
    --health-retries 3 \
    --health-start-period 15s \
    ${IMAGE}:${TAG}

ExecStop=/usr/bin/docker stop oxus-backend-app-agrobank
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

Start and check the unit:

```bash
systemctl daemon-reload
systemctl enable --now agrobank-app-oxus-backend
systemctl status agrobank-app-oxus-backend

docker logs oxus-backend-app-agrobank --tail=50
```