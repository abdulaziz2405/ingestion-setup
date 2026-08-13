# Daily Database Backups

Cronjob here are meant to perform database backups daily in 02:00 AM and store only 7 latest copies of it.

Here's what the mechanism expects to have:
- **PostGIS, PostgreSQL, InfluxDB + Node-RED** instances
- Data directories of the applications **mounted** at `/mnt/data/`, where the data disk is mounted on
- Backup **R2 bucket** ready in Cloudflare

In the current infrastructure – it's three servers in total.

Once done, the job syncronizes the directory with Cloudflare R2 bucket to store it there safely.

## 1. Setting up directories

Here's the expected layout:

/mnt/data/:
    application-directory/
        aplication-data
    backups/
        <application>/
            backups.tar.gz
            logs/
                logs.log

There are exact expected directory names at (`/mnt/data/backups/<application>`) on each server.

On the **DB** server:
- `backend/`
- `models/`

On the **Ingestion** server:
- `influxdb/`
- `nodered/`

On the **Tool** server:
- **none** (backups are saved directly into `/mnt/data/backups`)

**Create** the corresponding directories listed above into `/mnt/data/backups`:
```bash
# on the DB server:
mkdir -p /mnt/data/backups/backend/logs
mkdir -p /mnt/data/backups/models/logs

# on the Ingestion server:
mkdir -p /mnt/data/backups/influxdb/logs
mkdir -p /mnt/data/backups/nodered/logs

# on the Tool server:
mkdir -p /mnt/data/backups/logs
```

## 2. Create the scripts

There's total of three separate servers that require backups, as mentioned above.
Each requires their own **backup script**.

Copy and paste them all into `/usr/local/bin/amudario-backup.sh` on separate servers.

Create the backup script on the **DB** server:
```bash
#!/usr/bin/env bash
set -euo pipefail
umask 077

KEEP=7
MIN_FREE_MB=1024
DATE="$(date +%d-%m-%y)"

log()  { echo "[INFO] $(date -Is) $*"; }
fail() { echo "[FAIL] $(date -Is) $*"; exit 1; }

mountpoint -q /mnt/data || fail "/mnt/data is not mounted"

FREE_MB="$(df -Pm /mnt/data | awk 'NR==2 {print $4}')"
[ "$FREE_MB" -ge "$MIN_FREE_MB" ] \
  || fail "/mnt/data has ${FREE_MB}MB free, need ${MIN_FREE_MB}MB"
log "free space on /mnt/data: ${FREE_MB}MB"

WORK_DIR="$(mktemp -d -p /mnt/data)"
trap 'rm -rf "${WORK_DIR:?}"' EXIT

backup_instance() {
  local container="$1" name="$2" dir="$3" prefix="$4"

  [ -d "$dir" ] || fail "$dir is missing"
  mkdir -p "$dir/logs"

  {
    log "$name backup started"

    docker exec "$container" sh -c \
      'PGPASSWORD="$POSTGRES_PASSWORD" pg_dumpall -U postgres -h 127.0.0.1' \
      > "$WORK_DIR/$name.sql" \
      || fail "$name pg_dumpall failed"
    [ -s "$WORK_DIR/$name.sql" ] || fail "$name dump is empty"
    log "$name dump complete"

    tar -czf "$dir/$name-backup-$DATE.tar.gz" -C "$WORK_DIR" "$name.sql" \
      || fail "$name archive creation failed"
    rm -f "$WORK_DIR/$name.sql"

    tar -tzf "$dir/$name-backup-$DATE.tar.gz" > /dev/null \
      || fail "$name archive is corrupt"
    log "$name archive verified: $name-backup-$DATE.tar.gz"

    ls -1t "$dir"/*.tar.gz    | tail -n +$((KEEP + 1)) | xargs -r rm -f
    ls -1t "$dir"/logs/*.log  | tail -n +$((KEEP + 1)) | xargs -r rm -f
    log "$name pruned to $KEEP archives"

    rclone sync "$dir" "r2:amudario-backups/$prefix" \
      --exclude "logs/**" \
      --delete-after \
      --max-delete 2 \
      --stats-one-line \
      || fail "$name sync to r2:amudario-backups/$prefix failed"
    log "$name synced to r2:amudario-backups/$prefix"

    log "$name backup finished"
  } 2>&1 | tee -a "$dir/logs/$name-$DATE.log"
}

backup_instance postgis-backend postgis-backend /mnt/data/backups/backend backend
backup_instance postgis-models  postgis-models  /mnt/data/backups/models  models
```

Create the backup script on the **Tool** server:
```bash
#!/usr/bin/env bash
set -euo pipefail
umask 077

BACKUP_DIR="/mnt/data/backups"
KEEP=7
MIN_FREE_MB=1024
NAME="postgresql"
DATE="$(date +%d-%m-%y)"

log()  { echo "[INFO] $(date -Is) $*"; }
fail() { echo "[FAIL] $(date -Is) $*"; exit 1; }

mountpoint -q /mnt/data || fail "/mnt/data is not mounted"
[ -d "$BACKUP_DIR" ] || fail "$BACKUP_DIR is missing"
mkdir -p "$BACKUP_DIR/logs"

FREE_MB="$(df -Pm /mnt/data | awk 'NR==2 {print $4}')"
[ "$FREE_MB" -ge "$MIN_FREE_MB" ] \
  || fail "/mnt/data has ${FREE_MB}MB free, need ${MIN_FREE_MB}MB"
log "free space on /mnt/data: ${FREE_MB}MB"

WORK_DIR="$(mktemp -d -p /mnt/data)"
trap 'rm -rf "${WORK_DIR:?}"' EXIT

{
  log "$NAME backup started"

  docker exec postgres sh -c \
    'PGPASSWORD="$POSTGRES_PASSWORD" pg_dumpall -U postgres -h 127.0.0.1' \
    > "$WORK_DIR/$NAME.sql" \
    || fail "$NAME pg_dumpall failed"
  [ -s "$WORK_DIR/$NAME.sql" ] || fail "$NAME dump is empty"
  log "$NAME dump complete"

  tar -czf "$BACKUP_DIR/$NAME-backup-$DATE.tar.gz" -C "$WORK_DIR" "$NAME.sql" \
    || fail "$NAME archive creation failed"

  tar -tzf "$BACKUP_DIR/$NAME-backup-$DATE.tar.gz" > /dev/null \
    || fail "$NAME archive is corrupt"
  log "$NAME archive verified: $NAME-backup-$DATE.tar.gz"

  ls -1t "$BACKUP_DIR"/*.tar.gz   | tail -n +$((KEEP + 1)) | xargs -r rm -f
  ls -1t "$BACKUP_DIR"/logs/*.log | tail -n +$((KEEP + 1)) | xargs -r rm -f
  log "$NAME pruned to $KEEP archives"

  rclone sync "$BACKUP_DIR" r2:amudario-backups/postgresql \
    --exclude "logs/**" \
    --delete-after \
    --max-delete 2 \
    --stats-one-line \
    || fail "$NAME sync to r2:amudario-backups/postgresql failed"
  log "$NAME synced to r2:amudario-backups/postgresql"

  log "$NAME backup finished"
} 2>&1 | tee -a "$BACKUP_DIR/logs/$NAME-$DATE.log"
```

Create the backup script on the **Ingestion** server:
```bash
#!/usr/bin/env bash
set -euo pipefail
umask 077

source /etc/amudario/backup.env
export INFLUX_TOKEN

KEEP=7
MIN_FREE_MB=1024
DATE="$(date +%d-%m-%y)"

log()  { echo "[INFO] $(date -Is) $*"; }
fail() { echo "[FAIL] $(date -Is) $*"; exit 1; }

mountpoint -q /mnt/data || fail "/mnt/data is not mounted"

FREE_MB="$(df -Pm /mnt/data | awk 'NR==2 {print $4}')"
[ "$FREE_MB" -ge "$MIN_FREE_MB" ] \
  || fail "/mnt/data has ${FREE_MB}MB free, need ${MIN_FREE_MB}MB"
log "free space on /mnt/data: ${FREE_MB}MB"

WORK_DIR="$(mktemp -d -p /mnt/data)"
trap 'rm -rf "${WORK_DIR:?}"' EXIT

rotate() {
  local name="$1" dir="$2" prefix="$3"

  tar -tzf "$dir/$name-backup-$DATE.tar.gz" > /dev/null \
    || fail "$name archive is corrupt"
  log "$name archive verified: $name-backup-$DATE.tar.gz"

  ls -1t "$dir"/*.tar.gz   | tail -n +$((KEEP + 1)) | xargs -r rm -f
  ls -1t "$dir"/logs/*.log | tail -n +$((KEEP + 1)) | xargs -r rm -f
  log "$name pruned to $KEEP archives"

  rclone sync "$dir" "r2:amudario-backups/$prefix" \
    --exclude "logs/**" \
    --delete-after \
    --max-delete 2 \
    --stats-one-line \
    || fail "$name sync to r2:amudario-backups/$prefix failed"
  log "$name synced to r2:amudario-backups/$prefix"
}

# InfluxDB
INFLUX_DIR="/mnt/data/backups/influxdb"
[ -d "$INFLUX_DIR" ] || fail "$INFLUX_DIR is missing"
mkdir -p "$INFLUX_DIR/logs"

{
  log "influxdb backup started"

  influx backup "$WORK_DIR/influxdb" --host http://localhost:8086 \
    || fail "influxdb backup command failed"
  log "influxdb dump complete"

  tar -czf "$INFLUX_DIR/influxdb-backup-$DATE.tar.gz" -C "$WORK_DIR" influxdb \
    || fail "influxdb archive creation failed"
  rm -rf "$WORK_DIR/influxdb"

  rotate influxdb "$INFLUX_DIR" influxdb

  log "influxdb backup finished"
} 2>&1 | tee -a "$INFLUX_DIR/logs/influxdb-$DATE.log"

# Node-RED
NODERED_DIR="/mnt/data/backups/nodered"
[ -d "$NODERED_DIR" ] || fail "$NODERED_DIR is missing"
mkdir -p "$NODERED_DIR/logs"

{
  log "node-red backup started"

  tar -czf "$NODERED_DIR/node-red-backup-$DATE.tar.gz" -C /mnt/data node-red \
    || fail "node-red archive creation failed"
  log "node-red archive created"

  rotate node-red "$NODERED_DIR" nodered

  log "node-red backup finished"
} 2>&1 | tee -a "$NODERED_DIR/logs/node-red-$DATE.log"
```

## 3. Configuring rclone

In the root home directory, create an `rclone` **config** with provided credentials:
```ini
[r2]
type = s3
provider = Cloudflare
access_key_id = <PASTE>
secret_access_key = <PASTE>
endpoint = https://<PASTE>.r2.cloudflarestorage.com
region = auto
acl = private
no_check_bucket = true
```

These credentials will be used by `rclone` utility for syncing the backup directory with the bucket.

## 4. Create the daily cronjob

On **each server** at `/etc/cron.d`, create a job file named `amudario-backup`:
```bash
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

30 2 * * * root /usr/bin/flock -n /var/lock/amudario-backup.lock /usr/local/bin/amudario-backup.sh >> /var/log/amudario-backup.log 2>&1
```

Grant the correct rights:
```bash
chmod 644 /etc/cron.d/amudario-backup
```

Now the cronjob will automatically run **every day at 02:00 AM** and run these scripts.

## 5. Restoring a backup

Restoring **overwrites** the current data of the application.

### Getting the archive

Take the archive from `/mnt/data/backups/<application>` if it's still there.

Otherwise, download it from the bucket:
```bash
rclone ls r2:amudario-backups/<prefix>
rclone copy r2:amudario-backups/<prefix>/<name>-backup-<DD-MM-YY>.tar.gz /mnt/data/restore
```

The `<prefix>` is one of `backend`, `models`, `postgresql`, `influxdb`, `nodered`.

### Restoring PostGIS and PostgreSQL

The archive holds a single `pg_dumpall` **SQL** file:
```bash
tar -xzf /mnt/data/restore/<name>-backup-<DD-MM-YY>.tar.gz -C /mnt/data/restore

docker exec -i <db-container> sh -c \
  'PGPASSWORD="$POSTGRES_PASSWORD" psql -U postgres -h 127.0.0.1 -d postgres' \
  < /mnt/data/restore/<name>.sql
```

The containers are `postgis-backend` and `postgis-models` on the **DB** server, `postgres` on the **Tool** server.

### Restoring InfluxDB

The archive holds an `influx backup` **directory**:
```bash
tar -xzf /mnt/data/restore/influxdb-backup-<DD-MM-YY>.tar.gz -C /mnt/data/restore

influx restore /mnt/data/restore/influxdb --host http://localhost:8086 --full
```

### Restoring Node-RED

The archive holds the `node-red` **data directory**:
```bash
docker stop node-red
tar -xzf /mnt/data/restore/node-red-backup-<DD-MM-YY>.tar.gz -C /mnt/data
docker start node-red
```

### Checking the restore

Check that the databases are in place:
```bash
docker exec -it <db-container> psql -U postgres -c '\l'
influx bucket list --host http://localhost:8086
```

For Node-RED, open the editor and check that the flows are there.