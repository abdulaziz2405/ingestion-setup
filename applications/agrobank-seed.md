# Seeding the databases (agrobank-seed)

Repository: https://gitlab.com/amudario/development/agrobank-seed

Variables that are used here and require change:
- `<TOOL_SERVER>`: IP of the tool server that runs Vault, see [Vault](../tools/vault/vault.md).
- `<DB_SERVER>`: IP of the server with both PostGIS instances, see [PostGIS](../tools/postgis.md).
- `<VAULT_TOKEN>`: a Vault token that can read the secret below.

Prerequisites:
- [Oxus-Backend](./oxus-backend/oxus-backend.md) and [Oxus-Models](./oxus-models/oxus-models.md) are up.
  Their units already ran the migrations (`php artisan migrate`, `alembic upgrade head`) and
  `python -m scripts.deploy_seed`. The seeder refuses to run against a schema without them.

A one-off container that fills both platform databases (backend `:5432`, models `:5433`)
with the initial data and uploads the platform objects to the S3 bucket. All data is baked
into the image. Every write is an upsert: nothing is deleted, a second run does not duplicate.
Run it once, before starting [Oxus-Prefect](./oxus-prefect/oxus-prefect.md), from any host
that can reach Vault, both databases and S3.

The full reference (what it writes, options, post-seed checks) is in the repository README.

## 1. Secret store

In Vault UI, at path `secret/amudario/agrobank/seed`, add the variables:

```bash
BACKEND_DB_HOST=<DB_SERVER>
BACKEND_DB_PORT=5432
BACKEND_DB_NAME=amudario
BACKEND_DB_USER=
BACKEND_DB_PASSWORD=
MODELS_DB_HOST=<DB_SERVER>
MODELS_DB_PORT=5433
MODELS_DB_NAME=oxus_models
MODELS_DB_USER=
MODELS_DB_PASSWORD=
AWS_ENDPOINT=<LINK>
AWS_BUCKET=<IMAGES BUCKET NAME>
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=auto
```

The database roles need `INSERT`/`UPDATE` on both databases and `setval` on their sequences.
The seeder sets `session_replication_role = replica`, so a superuser is the simplest choice.
Give the seeder its own token and revoke it after the run.

## 2. Pull the image

Root has to be logged in to the registry, see [GitLab Runner](../tools/gitlab-runner.md), step 3:

```bash
docker pull registry.gitlab.com/amudario/development/agrobank-seed:latest
```

## 3. Run

Check the target first, it writes nothing:

```bash
docker run --rm \
  -e VAULT_ADDR=http://<TOOL_SERVER>:8200 \
  -e VAULT_TOKEN=<VAULT_TOKEN> \
  registry.gitlab.com/amudario/development/agrobank-seed:latest preflight
```

It has to end with `OK    preflight complete — ready to seed`. If not, it names the missing step.

Rehearse: every upsert runs and is rolled back:

```bash
docker run --rm \
  -e VAULT_ADDR=http://<TOOL_SERVER>:8200 \
  -e VAULT_TOKEN=<VAULT_TOKEN> \
  registry.gitlab.com/amudario/development/agrobank-seed:latest seed --dry-run
```

Seed for real. `AGROBANK_SEED_CONFIRM` must equal the backend database name:

```bash
docker run --rm \
  -v agrobank-seed-state:/state \
  -e VAULT_ADDR=http://<TOOL_SERVER>:8200 \
  -e VAULT_TOKEN=<VAULT_TOKEN> \
  -e AGROBANK_SEED_CONFIRM=amudario \
  registry.gitlab.com/amudario/development/agrobank-seed:latest seed
```

## 4. After seeding

Mirror the device/disease links into oxus-models:

```bash
docker exec -w /app oxus-models-api-agrobank python -m scripts.import_from_backend
```

Re-check at any time:

```bash
docker run --rm \
  -e VAULT_ADDR=http://<TOOL_SERVER>:8200 \
  -e VAULT_TOKEN=<VAULT_TOKEN> \
  registry.gitlab.com/amudario/development/agrobank-seed:latest verify
```

User passwords are not carried over: every account has to go through a password reset.
