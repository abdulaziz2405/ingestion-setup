# PostgreSQL instance

One plain PostgreSQL, shared by Vault and Prefect as their storage backend.

Variables that are used here and require change:
- `<PASSWORD>`: password for the `postgres` role
- `<TOOL_SERVER>`: IP of the tool server this instance runs on, shared with Vault

Set up directory that will be mounted into the container:
```bash
mkdir -p /mnt/data/postgresql/data
```

Put the `postgres` user password inside the file:
```bash
mkdir -p /etc/postgres

cat > /etc/postgres/.env <<'EOF'
POSTGRES_PASSWORD=<PASSWORD>
EOF

chmod 600 /etc/postgres/.env
```

Create the systemd unit `/etc/systemd/system/agrobank-db-postgres-01.service`:
```ini
[Unit]
Description=PostgreSQL in Docker
Requires=docker.service
After=docker.service network-online.target
Wants=network-online.target

[Service]
Restart=always
RestartSec=5
TimeoutStartSec=120
TimeoutStopSec=60

ExecStartPre=-/usr/bin/docker stop -t 55 postgres
ExecStartPre=-/usr/bin/docker rm postgres

ExecStart=/usr/bin/docker run --rm --name postgres \
  -p <TOOL_SERVER>:5432:5432 \
  --env-file /etc/postgres/.env \
  -v /mnt/data/postgresql/data:/var/lib/postgresql \
  --health-cmd="pg_isready -U postgres" \
  --health-interval=10s \
  docker.io/library/postgres:18

ExecStop=/usr/bin/docker stop -t 55 postgres

[Install]
WantedBy=multi-user.target
```

Start and check the unit:
```bash
systemctl daemon-reload
systemctl enable --now agrobank-db-postgres-01
systemctl status agrobank-db-postgres-01

docker exec -i postgres psql -U postgres -c 'SELECT version();'
```

The databases and roles themselves are created later, each in its own document:
`vault` in [Vault](./vault/vault.md), step 1, and `prefect` in
[Oxus-Prefect](../applications/oxus-prefect/oxus-prefect.md), step 1.
