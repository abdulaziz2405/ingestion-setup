# Amudario

Deployment documentation for AgroBank.

## Server layout

Before proceeding, the stack expects exactly **four** separate hosts to operate on.
You'll find these corresponding host variables in the docs:

- **`<APPLICATION_SERVER>`**: Oxus-Backend, Oxus-Frontend, Oxus-Models
  (API + Celery), Oxus-Prefect (server + worker) and Vault Agent.
- **`<DB_SERVER>`**: 2 x PostGIS instances (backend `:5432`, models
  `:5433`) and Redis (`:6379`).
- **`<TOOL_SERVER>`**: Vault and PostgreSQL (Vault and Prefect).
- **`<INGESTION_SERVER>`**: InfluxDB, Mosquitto and Node-RED.

---

## Contents

### Ingestion

- [InfluxDB](./ingestion/influxdb.md)
- [Mosquitto](./ingestion/mosquitto.md)
- [Node-RED](./ingestion/node-red.md)

### Tools

- [PostGIS](./tools/postgis.md)
- [PostgreSQL](./tools/postgres.md)
- [Redis](./tools/redis.md)
- [Vault](./tools/vault/vault.md)
- [Vault Agent](./tools/vault/vault-agent.md)
- [GitLab Runner](./tools/gitlab-runner.md)

### Applications

- [Oxus-Backend](./applications/oxus-backend/oxus-backend.md)
- [Oxus-Models](./applications/oxus-models/oxus-models.md)
- [Oxus-Prefect](./applications/oxus-prefect/oxus-prefect.md)
- [Oxus-Frontend](./applications/oxus-frontend/oxus-frontend.md)

---

## Deployment order

| # | Component | Group |
|:--|:----------|:------|
| 1 | [InfluxDB](./ingestion/influxdb.md) | Ingestion |
| 2 | [Mosquitto](./ingestion/mosquitto.md) | Ingestion |
| 3 | [Node-RED](./ingestion/node-red.md) | Ingestion |
| 4 | [PostGIS](./tools/postgis.md) | Tools |
| 5 | [PostgreSQL](./tools/postgres.md) | Tools |
| 6 | [Redis](./tools/redis.md) | Tools |
| 7 | [Vault](./tools/vault/vault.md) | Tools |
| 8 | [Vault Agent](./tools/vault/vault-agent.md) | Tools |
| 9 | [GitLab Runner](./tools/gitlab-runner.md) | Tools |
| 10 | [Oxus-Backend](./applications/oxus-backend/oxus-backend.md) | Applications |
| 11 | [Oxus-Models](./applications/oxus-models/oxus-models.md) | Applications |
| 12 | [Oxus-Prefect](./applications/oxus-prefect/oxus-prefect.md) | Applications |
| 13 | [Oxus-Frontend](./applications/oxus-frontend/oxus-frontend.md) | Applications |
