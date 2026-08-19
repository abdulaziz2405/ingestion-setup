# AgroBank seed migration

A single container that populates AgroBank's two platform databases and uploads
the platform's objects to AgroBank's S3 bucket. Everything it writes is baked
into the image — there is no bundle to copy, no export step to run, and no
network path back to Amudario.

This is **Stage 4** of the AmudarIO platform delivery — see the deployment and
handover guide for how the stage fits into the wider rollout:

📄 **[DEPLOYMENT_AGROBANK.pdf](docs/DEPLOYMENT_AGROBANK.pdf)** — AmudarIO / AgroBank
on-site deployment & handover guide.

```
                       ┌──────────────────────────────┐
  agrobank-seed  ──────┤ oxus-backend-db  (PostGIS)   │  reference data + the
   (payload inside)    │ oxus-models-db   (PostGIS)   │  tenant's own rows
                       └──────────────────────────────┘
         │
         └───────────▶  S3 bucket   knowledge_base/ · models/ · device_resources/
```

Every write is an **upsert**. Nothing is ever deleted, and a second run
converges rather than duplicating. Database and S3 credentials come from
**Vault**.

---

## What it writes

### oxus-backend-db

| Group | Tables |
|-------|--------|
| Platform reference data (full copy) | `plants`, `diseases`, `disease_models`, `disease_stages`, `pheromones`, `plant_disease`, `photo_blobs` |
| The tenant's own rows, in FK order | `orgs`, `forecasts`, `forecast_resources`, `device_groups`, `users`, `devices`, `device_resources`, `device_disease_models`, `device_plant_settings`, `device_irrigation_settings`, `device_notification_settings`, `device_messages`, `device_statuses`, `alerts`, `licenses`, `seasons`, `user_devices`, `user_model_scopes`, `pheromone_trap_seasons`, `pheromone_trap_readings` |
| Never touched | `cache*`, `jobs`, `failed_jobs`, `sessions`, `password_reset*`, `personal_access_tokens`, `migrations`, `spatial_ref_sys` |

### oxus-models-db

| Group | Tables |
|-------|--------|
| Shipped | `station_devices`, `historical_station_stats`, `station_stats` |
| Asserted, not shipped | `pests`, `plants`, `plant_pest`, `disease_images`, the irrigation reference tables and the rest of the models reference set — these arrive through `alembic upgrade head` + `python -m scripts.deploy_seed`, so AgroBank stays on the same upgrade path as every other environment |
| Never shipped | `pest_model_predictions` (recomputed nightly), `alembic_version` |

`historical_station_stats` is the reason this stage exists: its source is a
rate-limited weather API and the models need roughly a year of daily history per
station before they produce anything, so it cannot be re-fetched on the target in
reasonable time.

### S3 objects

| Group | What | Selected by |
|-------|------|-------------|
| `knowledge_base/` | pest and disease content YAML plus their photos | whole prefix — platform content, tenant independent |
| `models/` | model weights and the forecast-adjust artifact pointer | whole prefix |
| `device_resources/` | the tenant's station chart images | **derived from the tenant's own `device_resources` rows**, not copied wholesale, so no other tenant's images travel |

Inspect the exact contents of the image you hold, without contacting anything:

```bash
docker run --rm registry.gitlab.com/amudario/development/agrobank-seed:latest plan
```

---

## Order of operations

The seeder writes **data**; migrations own **schema**. It refuses to run against
a schema it was not built for, so the sequence matters:

1. `php artisan migrate --force` — oxus-backend (creates the backend schema)
2. `alembic upgrade head` — oxus-models (creates the models schema)
3. `python -m scripts.deploy_seed` — oxus-models (models reference data)
4. **`agrobank-seed seed`** ← this container
5. `python -m scripts.import_from_backend` — oxus-models, once the backend rows
   exist, to mirror device/disease links across

Steps 1–3 and 5 run inside their own service containers, not this one.
`preflight` checks all of it and names the missing step if something is out of
order.

---

## Requirements

- Docker on a host that can reach **Vault**, both **PostgreSQL** instances and
  the **S3 endpoint**.
- A Vault token (or AppRole) that can read the secret below.
- Database roles that can `INSERT`/`UPDATE` on both databases and `setval` on
  their sequences. `session_replication_role = replica` is set per table, which
  needs a **superuser or `pg_write_all_data`-equivalent** role; without it,
  grant the role explicitly or expect FK ordering to be enforced strictly.
- An S3 key that can `PutObject` (and `HeadObject`, used to skip existing keys).

---

## Vault secret layout

One flat KV secret. Default path `secret/amudario/agrobank/seed` (mount `secret`,
path `amudario/agrobank/seed`), override with `VAULT_SECRET_PATH`. Both KV v2 and KV v1 mounts work.

| Key | Required | Default | Meaning |
|-----|:--------:|---------|---------|
| `BACKEND_DB_HOST` | ✅ | — | oxus-backend-db host. |
| `BACKEND_DB_NAME` | ✅ | — | e.g. `amudario`. |
| `BACKEND_DB_USER` | ✅ | — | Role used for the load. |
| `BACKEND_DB_PASSWORD` | ✅ | — | |
| `BACKEND_DB_PORT` | | `5432` | |
| `MODELS_DB_HOST` | ✅ | — | oxus-models-db host. |
| `MODELS_DB_NAME` | ✅ | — | e.g. `oxus_models`. |
| `MODELS_DB_USER` | ✅ | — | |
| `MODELS_DB_PASSWORD` | ✅ | — | |
| `MODELS_DB_PORT` | | `5433` | |
| `AWS_ENDPOINT` | ✅ | — | AgroBank's S3 endpoint. |
| `AWS_BUCKET` | ✅ | — | Target bucket. |
| `AWS_ACCESS_KEY_ID` | ✅ | — | |
| `AWS_SECRET_ACCESS_KEY` | ✅ | — | |
| `AWS_DEFAULT_REGION` | | `auto` | |
| `AWS_USE_PATH_STYLE_ENDPOINT` | | `false` | `true` for MinIO/Ceph. |
| `S3_STORAGE_CLASS` | | — | Set only if the endpoint requires it. |

```bash
vault kv put secret/amudario/agrobank/seed BACKEND_DB_HOST=10.0.0.6 BACKEND_DB_NAME=amudario BACKEND_DB_USER=admin BACKEND_DB_PASSWORD='***' MODELS_DB_HOST=10.0.0.6 MODELS_DB_NAME=oxus_models MODELS_DB_USER=oxus_models MODELS_DB_PASSWORD='***' AWS_ENDPOINT=https://s3.agrobank.internal AWS_BUCKET=amudario-images AWS_ACCESS_KEY_ID='***' AWS_SECRET_ACCESS_KEY='***' AWS_USE_PATH_STYLE_ENDPOINT=true
```

The seeder needs elevated database rights that the applications do not.
Give it **its own AppRole**, and revoke it once the one-off run is done:

```hcl
path "secret/data/amudario/agrobank/seed" {
  capabilities = ["read"]
}
```

### Vault connection (the only process environment the container reads)

| Variable | Required | Meaning |
|----------|:--------:|---------|
| `VAULT_ADDR` | ✅ | e.g. `https://vault.internal:8200`. |
| `VAULT_TOKEN` | ✅¹ | Vault token. |
| `VAULT_ROLE_ID` / `VAULT_SECRET_ID` | ✅¹ | AppRole credentials, used when no token is given. |
| `VAULT_SECRET_PATH` | | Full path including the KV mount. Default `secret/amudario/agrobank/seed`. |
| `VAULT_NAMESPACE` | | Vault Enterprise namespace. |
| `VAULT_SKIP_VERIFY` | | `true` to accept a self-signed Vault certificate. |
| `AGROBANK_SEED_CONFIRM` | ✅ | Must equal the target **backend database name**. A safety interlock so the seeder cannot be pointed at the wrong system by accident. |

No database password or S3 key is ever passed on a command line or set in the
container environment — they exist only in Vault and in the process's memory.

---

## Getting the image

```bash
docker login registry.gitlab.com
```

```bash
docker pull registry.gitlab.com/amudario/development/agrobank-seed:latest
```

Every build pushes two tags: the immutable **short commit SHA** — record this
with the stage sign-off — and the moving **`latest`**.

---

## Running it

### 1. Look at what you have (no connections)

```bash
docker run --rm registry.gitlab.com/amudario/development/agrobank-seed:latest plan
```

### 2. Check the target (writes nothing)

```bash
docker run --rm -e VAULT_ADDR=https://vault.internal:8200 -e VAULT_TOKEN=hvs.example registry.gitlab.com/amudario/development/agrobank-seed:latest preflight
```

```
OK    payload scope version 1
OK    payload complete: 104486 rows, 1643 objects
OK    backend db reachable (16.4)
OK    models db reachable (16.4)
OK    s3 bucket 'amudario-images' reachable and writable target
OK    backend schema at or beyond '2026_08_08_110000_add_logo_key_to_orgs_table'
OK    models schema at alembic revision '2026_08_06_soil_moisture_vs_vwc'
OK    models reference data present (populated by migrations + deploy_seed)

OK    preflight complete — ready to seed
```

### 3. Rehearse, then seed

`--dry-run` does every read, every checksum and every conflict check, then rolls
back instead of committing:

```bash
docker run --rm -e VAULT_ADDR=https://vault.internal:8200 -e VAULT_TOKEN=hvs.example registry.gitlab.com/amudario/development/agrobank-seed:latest seed --dry-run
```

Then for real. Mount a volume at `/state` so a re-run can skip finished phases,
and keep the JSON report for the stage sign-off:

```bash
docker run --rm -v agrobank-seed-state:/state -v "$PWD/reports:/reports" -e VAULT_ADDR=https://vault.internal:8200 -e VAULT_TOKEN=hvs.example -e AGROBANK_SEED_CONFIRM=amudario registry.gitlab.com/amudario/development/agrobank-seed:latest seed --json-report /reports/stage4.json
```

### 4. Afterwards

Run `python -m scripts.import_from_backend` in oxus-models, then re-check any
time:

```bash
docker run --rm -e VAULT_ADDR=https://vault.internal:8200 -e VAULT_TOKEN=hvs.example registry.gitlab.com/amudario/development/agrobank-seed:latest verify
```

---

## Command reference

| Command | Purpose |
|---------|---------|
| `plan` | List the payload's tables, row counts and object groups. Contacts nothing. |
| `preflight` | Check Vault, both databases, S3, the payload checksums and the schema. Writes nothing. |
| `seed` | Load both databases, reset sequences, upload objects, verify. |
| `verify` | Re-run the post-seed checks only. |

`seed` options:

| Option | Default | Purpose |
|--------|---------|---------|
| `--dry-run` | off | Do everything except commit. |
| `--phase NAME` | all | Run only these phases: `backend`, `models`, `objects`, `sequences`, `verify`. Repeatable. |
| `--force` | off | Re-run completed phases and re-upload objects that already exist. |
| `--allow-schema-drift` | off | Downgrade schema mismatches from refusal to warning. |
| `--state PATH` | `/state/agrobank-seed.json` | Where completed phases are recorded. |
| `--json-report PATH` | — | Write the full result as JSON. |
| `--payload-dir DIR` | `/payload` | Use a payload from elsewhere (testing). |

Exit codes: `0` success · `1` a check or load failed · `2` configuration or
payload problem · `3` Vault, a database or S3 unreachable.

---

## What it checks after loading

- Every shipped table has at least as many rows as the payload.
- **Scope containment:** no `orgs`, `devices` or `users` rows belong to another
  tenant. If a payload ever leaked another customer's data, this catches it at
  the destination, which is the only place it matters.
- **Sequences** are past the highest seeded id, so the application's first
  `INSERT` does not collide. Ids at or above `900000000` are excluded, leaving
  that range free for platform-created rows such as a bootstrap admin.
- **The licence gate.** Pest and disease predictions are gated on a currently
  valid `diseases` licence. Without one the platform installs perfectly and
  produces zero predictions, with no error anywhere — so an expired date range
  is reported as a warning by name rather than left to be discovered later.
- Every station has ≥ 360 rows of history, the models' minimum.
- A sample of uploaded objects exists in the bucket at the expected size.

---

## Notes on the data

- **Credentials are destroyed at export, not at load.** `users.password` is set
  to `!` — an unusable bcrypt hash — and `remember_token` and `devices.token` are
  nulled. Every account on the target must go through a password reset. Token
  and session tables are not exported at all.
- **The payload is a point-in-time freeze**, taken when the image was built
  (`plan` prints the timestamp). If the handover happens months later, the
  tenant's devices, licences and forecasts will have moved on; rebuild the image
  from a fresh export at cutover, or accept the drift knowingly.
- **`station_stats` is included** even though it is normally derived from
  InfluxDB, because InfluxDB starts empty on AgroBank's side. It gives the
  dashboards something to show before the stations are re-pointed (Stage 5);
  from then on it is maintained by the scheduled fills.
- **`AWS_ENDPOINT` is validated.** The seeder refuses to run against an Amudario
  object-store hostname, so a half-configured environment fails loudly instead of
  writing a third party's data into our bucket.

---

## Rebuilding the payload

`tools/export.py` regenerates `payload/` from live Amudario sources. It runs on
**our** side only — it needs read access to both production databases and the
source bucket, which AgroBank neither has nor needs:

```bash
python tools/export.py --out payload
```

`src/agrobank_seed/scope.py` is the single source of truth for what travels.
Both the exporter and the seeder import it, and it **fails if either database
contains a table it does not classify** — so a table added upstream cannot
silently ride along, and cannot silently be left behind either. Adding a table
is a one-line `TableSpec`; bump `SCOPE_VERSION` when the shape changes, and the
seeder will refuse a payload built by a different version.
