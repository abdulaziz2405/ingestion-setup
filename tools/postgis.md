# PostGIS instances

Variables that are used here and require change:
- `<PASSWORD>`: password for the `postgres` role
- `<BACKEND_PASSWORD>`: password for the `admin` role, for oxus-backend
- `<MODELS_PASSWORD>`: password for the `oxus_models` role, for oxus-models
- `<DB_SERVER>`: IP of the db server both instances run on, shared with Redis

Two instances run side by side on the same server.
- oxus-backend on `:5432`
- oxus-models on `:5433`
The applications expect exactly this setup.

## 1. PostGIS for Backend

Set up directory that will be mounted into the container:
```bash
mkdir -p /mnt/data/postgresql-backend/data
```

Put the `postgres` user password inside the file:
```bash
mkdir -p /etc/postgres

cat > /etc/postgres/backend.env <<'EOF'
POSTGRES_PASSWORD=<PASSWORD>
EOF

chmod 600 /etc/postgres/backend.env
```

Create the systemd unit `/etc/systemd/system/agrobank-db-postgis-backend.service`:
```ini
[Unit]
Description=PostGIS for oxus-backend
Requires=docker.service
After=docker.service network-online.target
Wants=network-online.target

[Service]
Restart=always
RestartSec=5
TimeoutStartSec=120
TimeoutStopSec=60

ExecStartPre=-/usr/bin/docker stop -t 55 postgis-backend
ExecStartPre=-/usr/bin/docker rm postgis-backend

ExecStart=/usr/bin/docker run --rm --name postgis-backend \
  -p <DB_SERVER>:5432:5432 \
  -e PGDATA=/var/lib/postgresql/data/pgdata \
  --env-file /etc/postgres/backend.env \
  -v /mnt/data/postgresql-backend/data:/var/lib/postgresql/data \
  --shm-size=256m \
  --health-cmd="pg_isready -U postgres" \
  --health-interval=10s \
  --health-timeout=5s \
  --health-retries=5 \
  postgis/postgis:16-3.4

ExecStop=/usr/bin/docker stop -t 55 postgis-backend

[Install]
WantedBy=multi-user.target
```

Start and check the unit:
```bash
systemctl daemon-reload
systemctl enable --now agrobank-db-postgis-backend
systemctl status agrobank-db-postgis-backend
```

Create the `amudario` application database under `admin` user.
Then, enable the **PostGIS extension** inside it.

```bash
docker exec -i postgis-backend psql -U postgres
```

```sql
CREATE USER admin WITH LOGIN PASSWORD '<BACKEND_PASSWORD>';

CREATE DATABASE amudario OWNER admin;

\c amudario

CREATE EXTENSION IF NOT EXISTS postgis;
```

Confirm it is there:
```sql
\c amudario

SELECT postgis_version();
```

## 2. PostGIS for Models

Set up directory that will be mounted into the container:
```bash
mkdir -p /mnt/data/postgresql-models/data
```

Put the `postgres` user password inside the file:
```bash
mkdir -p /etc/postgres

cat > /etc/postgres/models.env <<'EOF'
POSTGRES_PASSWORD=<PASSWORD>
EOF

chmod 600 /etc/postgres/models.env
```

Create the systemd unit `/etc/systemd/system/agrobank-db-postgis-models.service`:
```ini
[Unit]
Description=PostGIS for oxus-models
Requires=docker.service
After=docker.service network-online.target
Wants=network-online.target

[Service]
Restart=always
RestartSec=5
TimeoutStartSec=120
TimeoutStopSec=60

ExecStartPre=-/usr/bin/docker stop -t 55 postgis-models
ExecStartPre=-/usr/bin/docker rm postgis-models

ExecStart=/usr/bin/docker run --rm --name postgis-models \
  -p <DB_SERVER>:5433:5432 \
  -e PGDATA=/var/lib/postgresql/data/pgdata \
  --env-file /etc/postgres/models.env \
  -v /mnt/data/postgresql-models/data:/var/lib/postgresql/data \
  --shm-size=256m \
  --health-cmd="pg_isready -U postgres" \
  --health-interval=10s \
  --health-timeout=5s \
  --health-retries=5 \
  postgis/postgis:16-3.4

ExecStop=/usr/bin/docker stop -t 55 postgis-models

[Install]
WantedBy=multi-user.target
```

Start and check the unit:
```bash
systemctl daemon-reload
systemctl enable --now agrobank-db-postgis-models
systemctl status agrobank-db-postgis-models
```

Create the application database and enable the extension in it too.

```bash
docker exec -i postgis-models psql -U postgres
```

```sql
CREATE USER oxus_models WITH LOGIN PASSWORD '<MODELS_PASSWORD>';

CREATE DATABASE oxus_models OWNER oxus_models;

\c oxus_models

CREATE EXTENSION IF NOT EXISTS postgis;
```

Confirm it is there:
```sql
\c oxus_models

SELECT postgis_version();
```