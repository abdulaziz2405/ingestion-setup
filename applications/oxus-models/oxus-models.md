# Deploying Oxus-models

Variables that are used here and require change:
- `<APPLICATION_SERVER>`: IP of the server this app runs on.
- `<PIPELINE_IID>`: IID of the last successful pipeline, taken from the GitLab UI. It is the image tag suffix.

## 1. Secret store

Vault is used here.
In Vault UI, at path `secret/amudario/applications/oxus-models`, add the necessary variables.
You can see the example ones without values, at example.env.

To pull the credentials, application relies on Vault Agent functioning on the same server.
You can see the instructions on how to deploy one in [Vault Agent](../../tools/vault/vault-agent.md).

Before proceeding, make sure Vault Agent is already installed and configured.

## 2. Setting up environment

Create directories:

```bash
mkdir -p /etc/oxus-models-api
```

Create a docker network for celery/api communication. Both units share it, so ignore the error
if it already exists:
```bash
docker network create oxus-models-agrobank
```

Pull the initial image. Root has to be logged in to the registry, see
[GitLab Runner](../../tools/gitlab-runner.md), step 3:

```bash
docker pull registry.gitlab.com/amudario/development/oxus-models:agrobank-<PIPELINE_IID>
```

Create the .env file for image tagging (both celery and api units will read this file):

```bash
cat > /etc/oxus-models-api/agrobank.env <<'EOF'
IMAGE=registry.gitlab.com/amudario/development/oxus-models
TAG=agrobank-<PIPELINE_IID>
EOF

chmod 600 /etc/oxus-models-api/agrobank.env
```

## 3. Systemd units

Create `/etc/systemd/system/agrobank-app-oxus-models-api.service` and paste the following code:

```ini
[Unit]
Description=Oxus Models
Wants=dev-secretstore-vault-agent.service
Requires=docker.service network-online.target
After=docker.service dev-secretstore-vault-agent.service network-online.target

[Service]
Type=simple
EnvironmentFile=/etc/oxus-models-api/agrobank.env

ExecStartPre=/usr/bin/docker pull ${IMAGE}:${TAG}

ExecStartPre=/usr/bin/test -s /opt/app/shared/secrets/oxus-models-agro.env

ExecStartPre=-/usr/bin/docker rm -f oxus-models-api-agrobank

ExecStartPre=/usr/bin/docker run --rm \
  --name oxus-models-migration-1 \
  --network oxus-models-agrobank \
  --env-file /opt/app/shared/secrets/oxus-models-agro.env \
  --entrypoint sh \
  ${IMAGE}:${TAG} \
  -c 'alembic upgrade head'

ExecStartPre=/usr/bin/docker run --rm \
  --name oxus-models-migration-2 \
  --network oxus-models-agrobank \
  --env-file /opt/app/shared/secrets/oxus-models-agro.env \
  --entrypoint sh \
  ${IMAGE}:${TAG} \
  -c 'python -m scripts.deploy_seed'

ExecStart=/usr/bin/docker run --rm \
  --name oxus-models-api-agrobank \
  --network oxus-models-agrobank \
  -p <APPLICATION_SERVER>:8011:8000 \
  --env-file /opt/app/shared/secrets/oxus-models-agro.env \
  --health-cmd '[ "$(curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8000/health)" -eq 200 ] || exit 1' \
  --health-interval 30s \
  --health-timeout 5s \
  --health-retries 3 \
  --health-start-period 15s \
  ${IMAGE}:${TAG}

ExecStop=/usr/bin/docker stop oxus-models-api-agrobank
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

Create `/etc/systemd/system/agrobank-app-oxus-models-celery.service` and paste the following code: 

```ini
[Unit]
Description=Oxus Models Celery worker
Wants=dev-secretstore-vault-agent.service
Requires=docker.service network-online.target
After=network-online.target docker.service dev-secretstore-vault-agent.service agrobank-app-oxus-models-api.service

PartOf=agrobank-app-oxus-models-api.service

[Service]
EnvironmentFile=/etc/oxus-models-api/agrobank.env

ExecStartPre=/usr/bin/test -s /opt/app/shared/secrets/oxus-models-agro.env

ExecStartPre=-/usr/bin/docker rm -f oxus-models-celery-agrobank

ExecStartPre=/usr/bin/docker pull ${IMAGE}:${TAG}

ExecStart=/usr/bin/docker run --rm \
  --name oxus-models-celery-agrobank \
  --network oxus-models-agrobank \
  --env-file /opt/app/shared/secrets/oxus-models-agro.env \
  --workdir /app \
  --health-cmd 'celery -A celery_app inspect ping -d celery@$$(hostname) -t 5 >/dev/null 2>&1 || exit 1' \
  --health-interval 60s \
  --health-timeout 10s \
  --health-retries 3 \
  --health-start-period 40s \
  ${IMAGE}:${TAG} celery -A celery_app worker --loglevel=info --concurrency=2

ExecStop=/usr/bin/docker stop oxus-models-celery-agrobank
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

`PartOf` connection works only for stop/restart.
This means, when the API gets stopped/restarted - Celery gets the same action as well.

Enable and start both units explicitly:

```bash
systemctl daemon-reload
systemctl enable --now agrobank-app-oxus-models-api agrobank-app-oxus-models-celery
systemctl status agrobank-app-oxus-models-api

docker logs oxus-models-api-agrobank --tail=50
```