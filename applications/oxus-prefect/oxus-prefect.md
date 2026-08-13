# Deploying Oxus-prefect

Variables that are used here and require change:
- `<APPLICATION_SERVER>`: IP of the server this app runs on. Server and worker both live here.
- `<TOOL_SERVER>`: IP of the tool server that runs the plain PostgreSQL, see [PostgreSQL](../../tools/postgres.md).
- `<PREFECT_DB_PASSWORD>`: password you pick for the `prefect` PostgreSQL role.
- `<PIPELINE_IID>`: IID of the last successful pipeline, taken from the GitLab UI. It is the image tag suffix.

Prerequisites:
- Vault Agent is installed and rendering secrets into `/opt/app/shared/secrets/`, see [Vault Agent](../../tools/vault/vault-agent.md).
- The runner has git access to GitLab, see [GitLab Runner](../../tools/gitlab-runner.md), step 3.

## 1. Preparing the database

Prefect server keeps its state in PostgreSQL. Create a dedicated database and role for it.
Run this against the shared instance from [PostgreSQL](../../tools/postgres.md), the same one
Vault uses, as `postgres`:

```bash
docker exec -i postgres psql -U postgres
```

```sql
CREATE ROLE prefect WITH LOGIN PASSWORD '<PREFECT_DB_PASSWORD>';

CREATE DATABASE prefect OWNER prefect;
```

The role has to own the database: the server creates its tables there on first start.

Confirm the role can log in:
```bash
docker exec -i postgres psql -U prefect -d prefect -c '\conninfo'
```

The same role, password and host go into `PREFECT_API_DATABASE_CONNECTION_URL` in the next step.

## 2. Secret store

Vault is used here.
In Vault UI, at paths:
- `secret/amudario/applications/oxus-prefect-server`
- `secret/amudario/applications/oxus-prefect-worker`
... add the necessary variables.
You can see the example ones without values, at example.*.env.

Vault Agent renders these two paths into two separate files on the application server:
- `/opt/app/shared/secrets/oxus-prefect-server-agro.env`
- `/opt/app/shared/secrets/oxus-prefect-worker-agro.env`

Both units below read those files, so Vault Agent must be running before you start them.

## 3. Code checkout on disk

The flows and the sibling helpers they import are mounted from disk, so three repositories
have to be cloned onto the server itself.

All of `/apps` belongs to `gitlab-runner`, which is the user that reaches GitLab from this
server: the deploy pipeline checks out `oxus-prefect` as it, and the other two are pulled
through it. Its SSH key and `known_hosts` are set up in
[GitLab Runner](../../tools/gitlab-runner.md), step 3, and have to be in place first.

```bash
mkdir -p /apps
chown gitlab-runner: /apps

sudo -u gitlab-runner -H git clone -b agrobank git@gitlab.com:amudario/development/oxus-prefect.git  /apps/oxus-prefect
sudo -u gitlab-runner -H git clone -b agrobank git@gitlab.com:amudario/development/oxus-models.git  /apps/oxus-models
sudo -u gitlab-runner -H git clone -b agrobank git@gitlab.com:amudario/development/backend2.git  /apps/oxus-backend
```

`/apps/oxus-prefect` is kept current by the deploy pipeline, which checks out the commit it
deploys and restarts the worker. It stays on a detached HEAD, so do not `git pull` it by hand.

`/apps/oxus-models` and `/apps/oxus-backend` are **not** touched by any pipeline.
When the helpers the flows import change, refresh those two yourself and restart the worker:

```bash
sudo -u gitlab-runner -H git -C /apps/oxus-models pull
sudo -u gitlab-runner -H git -C /apps/oxus-backend pull
systemctl restart agrobank-app-oxus-prefect-worker
```

## 4. Image tag files: one per unit

Server and worker run the same image, so both tag files carry the same `PIPELINE_IID`.
They are kept separate so the two units can be rolled forward independently.

```bash
mkdir -p /etc/oxus-prefect

cat > /etc/oxus-prefect/server.env <<'EOF'
IMAGE=registry.gitlab.com/amudario/development/oxus-prefect
TAG=agrobank-<PIPELINE_IID>
EOF

cat > /etc/oxus-prefect/worker.env <<'EOF'
IMAGE=registry.gitlab.com/amudario/development/oxus-prefect
TAG=agrobank-<PIPELINE_IID>
EOF

chmod 600 /etc/oxus-prefect/server.env /etc/oxus-prefect/worker.env
```

Pull the initial image. Root has to be logged in to the registry, see
[GitLab Runner](../../tools/gitlab-runner.md), step 4. That deploy token is a separate
credential from the runner's SSH key: the key reaches the git repositories, the token reaches
the registry.

```bash
docker pull registry.gitlab.com/amudario/development/oxus-prefect:agrobank-<PIPELINE_IID>
```

## 5. Systemd units

Server - `/etc/systemd/system/agrobank-app-oxus-prefect-server.service`:

```ini
[Unit]
Description=Oxus Prefect Server
Wants=dev-secretstore-vault-agent.service
Requires=docker.service network-online.target
After=docker.service dev-secretstore-vault-agent.service network-online.target

[Service]
Restart=on-failure
RestartSec=5
EnvironmentFile=/etc/oxus-prefect/server.env

ExecStartPre=/usr/bin/docker pull ${IMAGE}:${TAG}

ExecStartPre=/usr/bin/test -s /opt/app/shared/secrets/oxus-prefect-server-agro.env

ExecStartPre=-/usr/bin/docker rm -f prefect-server

ExecStart=/usr/bin/docker run --rm \
        --name prefect-server \
        --network host \
        --env-file /opt/app/shared/secrets/oxus-prefect-server-agro.env \
        --cpus 2 \
        --memory 1500m --memory-swap 1500m \
        --memory-reservation 768m \
        --cpu-shares 2048 \
        --oom-score-adj -500 \
        --pids-limit 512 \
        --health-cmd 'curl -fsS http://<APPLICATION_SERVER>:4200/api/health || exit 1' \
        --health-interval 30s \
        --health-timeout 5s \
        --health-retries 3 \
        --health-start-period 60s \
        ${IMAGE}:${TAG} \
        prefect server start --host <APPLICATION_SERVER> --port 4200

ExecStop=/usr/bin/docker stop prefect-server

[Install]
WantedBy=multi-user.target
```

Worker - `/etc/systemd/system/agrobank-app-oxus-prefect-worker.service`:

```ini
[Unit]
Description=Oxus Prefect Worker
Wants=dev-secretstore-vault-agent.service
Requires=docker.service network-online.target agrobank-app-oxus-prefect-server.service
After=docker.service network-online.target dev-secretstore-vault-agent.service agrobank-app-oxus-prefect-server.service

[Service]
Restart=always
RestartSec=5
EnvironmentFile=/etc/oxus-prefect/worker.env

ExecStartPre=/usr/bin/docker pull ${IMAGE}:${TAG}

ExecStartPre=/usr/bin/test -s /opt/app/shared/secrets/oxus-prefect-worker-agro.env

ExecStartPre=-/usr/bin/docker rm -f prefect-worker
ExecStartPre=-/usr/bin/docker rm -f prefect-init

ExecStartPre=/bin/sh -c 'for i in $(seq 1 120); do curl -fsS http://<APPLICATION_SERVER>:4200/api/health >/dev/null 2>&1 && exit 0; sleep 1; done; echo "server API not ready" >&2; exit 1'

# deploy dev manifest
ExecStartPre=/usr/bin/docker run --rm \
    --name prefect-init \
    --network host \
    --env-file /opt/app/shared/secrets/oxus-prefect-worker-agro.env \
    --cpus 1 --memory 768m --memory-swap 768m \
    -e PREFECT_DEPLOY_FILE=prefect.dev.yaml \
    -v /apps/oxus-prefect/flows:/opt/prefect/flows \
    -v /apps/oxus-prefect/scripts:/opt/prefect/scripts:ro \
    -v /apps/oxus-prefect/prefect.prod.yaml:/opt/prefect/prefect.prod.yaml:ro \
    -v /apps/oxus-prefect/prefect.dev.yaml:/opt/prefect/prefect.dev.yaml:ro \
    -v /apps/oxus-prefect/oxus_models:/opt/prefect/oxus_models:ro \
    -v /apps/oxus-backend/scrapers:/opt/prefect/oxus_backend_scrapers:ro \
    -v /apps/oxus-models/knowledge_base:/opt/prefect/knowledge_base:ro \
    -w /opt/prefect \
    ${IMAGE}:${TAG} \
    bash /opt/prefect/scripts/init.sh

# start the worker
ExecStart=/usr/bin/docker run --rm \
    --name prefect-worker \
    --network host \
    --env-file /opt/app/shared/secrets/oxus-prefect-worker-agro.env \
    --cpus 2 \
    --memory 2g --memory-swap 2g \
    --memory-reservation 256m \
    --cpu-shares 256 \
    --oom-score-adj 500 \
    --pids-limit 512 \
    -v /apps/oxus-prefect/flows:/opt/prefect/flows \
    -v /apps/oxus-prefect/prefect.prod.yaml:/opt/prefect/prefect.prod.yaml:ro \
    -v /apps/oxus-prefect/prefect.dev.yaml:/opt/prefect/prefect.dev.yaml:ro \
    -v /apps/oxus-prefect/oxus_models:/opt/prefect/oxus_models:ro \
    -v /apps/oxus-backend/scrapers:/opt/prefect/oxus_backend_scrapers:ro \
    -v /apps/oxus-models/knowledge_base:/opt/prefect/knowledge_base:ro \
    -w /opt/prefect \
    --health-cmd 'curl -fsS http://<APPLICATION_SERVER>:4200/api/health || exit 1' \
    --health-interval 30s \
    --health-timeout 5s \
    --health-retries 3 \
    --health-start-period 30s \
    ${IMAGE}:${TAG} \
    prefect worker start --pool dev --type process

TimeoutStopSec=150
ExecStop=/usr/bin/docker stop -t 120 prefect-worker

[Install]
WantedBy=multi-user.target
```

Prefect deployments init script needed for worker at `/opt/prefect/scripts/init.sh`:
```shell
#!/usr/bin/env bash
set -euo pipefail

cd /opt/prefect

DEPLOY_FILE="${PREFECT_DEPLOY_FILE:-prefect.dev.yaml}"

echo "[init] creating work pools (idempotent)…"
for pool in dev test prod; do
if prefect work-pool inspect "${pool}" >/dev/null 2>&1; then
echo "[init] work pool '${pool}' already exists — skipping"
else
prefect work-pool create "${pool}" --type process
echo "[init] created work pool '${pool}'"
fi
done

echo "[init] applying deployments from ${DEPLOY_FILE}…"
prefect deploy --all --prefect-file "/opt/prefect/${DEPLOY_FILE}"

echo "[init] done."
```

Start the units in the correct order:

```bash
systemctl daemon-reload
systemctl enable --now agrobank-app-oxus-prefect-server
systemctl enable --now agrobank-app-oxus-prefect-worker

systemctl status agrobank-app-oxus-prefect-server --no-pager
journalctl -u agrobank-app-oxus-prefect-worker -f    # 'Worker ... started!'
```

Verify pools and deployments:

```bash
docker exec prefect-server prefect work-pool ls
docker exec prefect-server prefect deployment ls
```