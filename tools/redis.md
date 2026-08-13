# Redis instance

Variables that are used here and require change:
- `<PASSWORD>`: the password Redis will require, the same one the applications put in `REDIS_PASSWORD`.
- `<DB_SERVER>`: IP of the db server Redis runs on, shared with the two PostGIS instances.

Set up directory that will be mounted into the container:
```bash
mkdir -p /mnt/data/redis/data
```

Put the password inside the file:
```bash
mkdir -p /etc/redis

cat > /etc/redis/redis.env <<'EOF'
REDIS_PASSWORD=<PASSWORD>
EOF

chmod 600 /etc/redis/redis.env
```

Create the systemd unit `/etc/systemd/system/agrobank-db-redis-01.service`:
```ini
[Unit]
Description=Redis in Docker
Requires=docker.service
After=docker.service network-online.target
Wants=network-online.target

[Service]
Restart=always
RestartSec=5
TimeoutStartSec=120
TimeoutStopSec=60

ExecStartPre=-/usr/bin/docker stop -t 30 redis-cache
ExecStartPre=-/usr/bin/docker rm redis-cache

ExecStart=/usr/bin/docker run --rm \
        --name redis-cache \
        -p <DB_SERVER>:6379:6379 \
        -v /mnt/data/redis/data:/data \
        --env-file /etc/redis/redis.env \
        redis:7-alpine \
        sh -c 'exec redis-server --appendonly yes --dir /data --requirepass "$$REDIS_PASSWORD"'

ExecStop=/usr/bin/docker stop -t 30 redis-cache

[Install]
WantedBy=multi-user.target
```

Start and check the unit:

```bash
systemctl daemon-reload
systemctl enable --now agrobank-db-redis-01
systemctl status agrobank-db-redis-01
```

Confirm the password took effect.
The first command must fail with `NOAUTH`, the second must answer `PONG`:

```bash
docker exec redis-cache redis-cli ping
docker exec redis-cache redis-cli -a '<PASSWORD>' ping
```