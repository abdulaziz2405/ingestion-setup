# Verifying ingestion (agrobank-ingestion-test)

Repository: https://gitlab.com/amudario/development/agrobank-ingestion-test

Variables that are used here and require change:
- `<TOOL_SERVER>`: IP of the tool server that runs Vault, see [Vault](../tools/vault/vault.md).
- `<INGESTION_SERVER>`: IP of the ingestion server that runs Mosquitto and InfluxDB.
- `<VAULT_TOKEN>`: a Vault token that can read the secret below.

Prerequisites:
- [InfluxDB](./influxdb.md), [Mosquitto](./mosquitto.md) and [Node-RED](./node-red.md) are up.
- [Vault](../tools/vault/vault.md) is up.

A one-off container that checks the ingestion chain end to end: it publishes test station
readings to Mosquitto over MQTT, then queries InfluxDB and confirms every value arrived
unchanged. Test points are removed from InfluxDB afterwards. Run it from any host that can
reach Vault, Mosquitto and InfluxDB.

The full reference (all cases, options, how to read a failure) is in the repository README.

## 1. Secret store

In Vault UI, at path `secret/amudario/agrobank/ingestion-test`, add the variables:

```bash
MQTT_HOST=<INGESTION_SERVER>
MQTT_PORT=1883
MQTT_USER=<MOSQUITTO USER>
MQTT_PASSWORD=<MOSQUITTO PASSWORD>
MQTT_TOPIC=meteodb
INFLUX_URL=http://<INGESTION_SERVER>:8086
INFLUX_TOKEN=<INFLUX TOKEN, READ + WRITE>
INFLUX_ORG=<INFLUX ORG NAME>
INFLUX_BUCKET=<INFLUX BUCKET NAME>
```

## 2. Pull the image

Root has to be logged in to the registry, see [GitLab Runner](../tools/gitlab-runner.md), step 3:

```bash
docker pull registry.gitlab.com/amudario/development/agrobank-ingestion-test:latest
```

## 3. Run

Check connectivity first, it publishes nothing:

```bash
docker run --rm \
  -e VAULT_ADDR=http://<TOOL_SERVER>:8200 \
  -e VAULT_TOKEN=<VAULT_TOKEN> \
  registry.gitlab.com/amudario/development/agrobank-ingestion-test:latest preflight
```

Then the full test:

```bash
docker run --rm \
  -e VAULT_ADDR=http://<TOOL_SERVER>:8200 \
  -e VAULT_TOKEN=<VAULT_TOKEN> \
  registry.gitlab.com/amudario/development/agrobank-ingestion-test:latest run
```

Expected ending: `PASS  6/6 cases passed — ingestion pipeline verified end to end`.
Exit code `0` means success, anything else is a failure.
