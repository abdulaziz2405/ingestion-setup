# AgroBank ingestion smoke test

A single container that verifies the AgroBank ingestion pipeline end to end: it
publishes **station measurement data as InfluxDB line protocol to Mosquitto over
MQTT**, then queries **InfluxDB** to confirm every value arrived unchanged.

This is **Stage 2** of the AmudarIO platform delivery — see the deployment and
handover guide for how the stage fits into the wider rollout:

📄 **[DEPLOYMENT_AGROBANK.pdf](docs/DEPLOYMENT_AGROBANK.pdf)** — AmudarIO / AgroBank
on-site deployment & handover guide.

Everything the test needs is baked into the image: the payloads, the CLI, and the
verification logic. The only thing it reads from the outside is **Vault**, which
holds the broker and InfluxDB settings.

```
  ingestion-test  ──publish line protocol──▶  Mosquitto  ──▶  (ingestion host consumer)
        │                topic: meteodb                              │
        │                                                            ▼
        └────────────────── Flux query: did it land? ────────────  InfluxDB
```

The test treats the pipeline as a black box. It knows nothing about how the
ingestion host moves data from the broker into InfluxDB — only that a
measurement published to the topic must become a point in the bucket.

---

## What it checks

| Case | What it proves |
|------|----------------|
| `single-point` | A normal station reading (air, wind, soil, battery) is stored with the right values and timestamp. |
| `multi-point-history` | Three consecutive readings are stored as three points — a backlog flush does not overwrite itself. |
| `batch-single-message` | Several newline-separated lines in **one** MQTT message are all ingested. |
| `full-sensor-set` | Every field the platform reads downstream, including the 5-depth soil-moisture profile, soil EC, leaf wetness and solar radiation. |
| `nan-field` | A real malformed reading (`Balance= NAN`) does not poison the batch: the valid fields land and the NaN field is absent. |
| `edge-values` | Sub-zero temperatures, exact zeros, full-scale wind direction and high-precision decimals survive the round trip. |

Each case is a JSON file under [`src/ingestion_test/data/`](src/ingestion_test/data);
the tool renders line protocol from it at run time. Add or edit a file to change
what is published — no code change needed.

All points are published as station **`001010001`** — override per run with
`--station-id`, or permanently via `TEST_STATION_ID` in Vault. Each case writes
into its own 15-minute timestamp band so the cases cannot overwrite one another.

See exactly what will be published, without connecting to anything:

```bash
docker run --rm agrobank-ingestion-test:latest cases --show-payloads
```

```
single-point
  One typical weather-station reading: the field set a station reports every cycle.
  1 point(s), one message each  [01-single-point.json]
    msg 1> meteometric,stationID=001010001 AirT=34.82,AirH=30.93,AirP=955.34,WindD=270.0,WindS=0.82,WindMax=1.99,Rain=0.0,SoilT=27.84,SoilVWC=29.04,Batt=13.24 1786979190029536000
```

---

## Requirements

- Docker on a host that can reach **Vault**, **Mosquitto** and **InfluxDB**.
- A Vault token (or AppRole) that can read the secret described below.
- An InfluxDB token with **read + write** access to the target bucket. Write is
  needed only for the automatic cleanup — run with `--keep` if the token is
  read-only, and purge the points later with `ingestion-test cleanup`.

---

## Vault secret layout

The test reads one flat KV secret. Default path
`secret/amudario/agrobank/ingestion-test`, override with `VAULT_SECRET_PATH`.
Both KV v2 and KV v1 mounts work.

| Key | Required | Default | Meaning |
|-----|:--------:|---------|---------|
| `MQTT_HOST` | ✅ | — | Mosquitto host on the ingestion server. |
| `MQTT_USER` | ✅ | — | Mosquitto username. |
| `MQTT_PASSWORD` | ✅ | — | Mosquitto password. |
| `MQTT_PORT` | | `1883` | `8883` when publishing over TLS. |
| `MQTT_TOPIC` | | `meteodb` | Topic the stations publish measurements to. |
| `MQTT_TLS` | | `false` | `true` to connect over TLS. |
| `MQTT_TLS_INSECURE` | | `false` | `true` skips broker-certificate verification. |
| `MQTT_QOS` | | `2` | Publish QoS (`0`, `1` or `2`). |
| `INFLUX_URL` | ✅ | — | e.g. `http://10.0.0.4:8086`. |
| `INFLUX_TOKEN` | ✅ | — | InfluxDB API token. |
| `INFLUX_ORG` | ✅ | — | InfluxDB organisation, e.g. `amudario`. |
| `INFLUX_BUCKET` | ✅ | — | Target bucket, e.g. `oxus2`. |
| `INFLUX_MEASUREMENT` | | `meteometric` | Measurement the station data is written to. |
| `INFLUX_TLS_VERIFY` | | `true` | `false` to accept a self-signed InfluxDB certificate. |
| `TEST_STATION_ID` | | `001010001` | Station ID the test publishes as. |

Create it with the Vault CLI:

```bash
vault kv put secret/amudario/agrobank/ingestion-test MQTT_HOST=10.0.0.5 MQTT_PORT=1883 MQTT_USER=nodered MQTT_PASSWORD='***' MQTT_TOPIC=meteodb INFLUX_URL=http://10.0.0.5:8086 INFLUX_TOKEN='***' INFLUX_ORG=amudario INFLUX_BUCKET=oxus2
```

A token used only for this test needs nothing more than:

```hcl
path "secret/data/amudario/agrobank/ingestion-test" {
  capabilities = ["read"]
}
```

### Vault connection (the only process environment the container reads)

| Variable | Required | Meaning |
|----------|:--------:|---------|
| `VAULT_ADDR` | ✅ | Vault address, e.g. `https://vault.internal:8200`. |
| `VAULT_TOKEN` | ✅¹ | Vault token. |
| `VAULT_ROLE_ID` / `VAULT_SECRET_ID` | ✅¹ | AppRole credentials, used when no token is given. |
| `VAULT_SECRET_PATH` | | Full path including the KV mount. Default `secret/amudario/agrobank/ingestion-test`. |
| `VAULT_NAMESPACE` | | Vault Enterprise namespace. |
| `VAULT_SKIP_VERIFY` | | `true` to accept a self-signed Vault certificate. |

¹ Either `VAULT_TOKEN` **or** the AppRole pair.

No broker or InfluxDB credentials are ever passed on the command line or in the
container environment — they exist only in Vault and in the process's memory.

---

## Getting the image

The container is delivered through the **GitLab Container Registry**, the same
channel as the platform services. Log in with the read-only deploy token issued
for the AgroBank project, then pull:

```bash
docker login registry.gitlab.com
```

```bash
docker pull registry.gitlab.com/amudario/development/agrobank-ingestion-test:latest
```

Every build pushes two tags: the immutable **short commit SHA** — record this
with the stage sign-off — and the moving **`latest`**. Pin the SHA to reproduce
an earlier run exactly:

```bash
docker pull registry.gitlab.com/amudario/development/agrobank-ingestion-test:<short-sha>
```

The examples below use the short name `agrobank-ingestion-test:latest`; tag the
pulled image locally for the same brevity:

```bash
docker tag registry.gitlab.com/amudario/development/agrobank-ingestion-test:latest agrobank-ingestion-test:latest
```

<details>
<summary>Building from source instead</summary>

```bash
git clone git@gitlab.com:amudario/development/agrobank-ingestion-test.git
```

```bash
docker build -t agrobank-ingestion-test:latest agrobank-ingestion-test
```

</details>

---

## Running the test

### Check connectivity first (writes nothing)

```bash
docker run --rm -e VAULT_ADDR=https://vault.internal:8200 -e VAULT_TOKEN=hvs.example agrobank-ingestion-test:latest preflight
```

```
PASS  vault: configuration loaded (13 settings)
PASS  mosquitto: connected to 10.0.0.5:1883 as 'nodered' (client ingestion-test-9f2c1a4b73)
PASS  influxdb: http://10.0.0.5:8086 reachable (v2.7.11), bucket 'oxus2' present in org 'amudario'

PASS  preflight complete — ready to run the smoke test
```

### Run the full smoke test

```bash
docker run --rm -e VAULT_ADDR=https://vault.internal:8200 -e VAULT_TOKEN=hvs.example agrobank-ingestion-test:latest run
```

```
        vault    https://vault.internal:8200 (amudario/agrobank/ingestion-test)
        mqtt     10.0.0.5:1883 topic 'meteodb'
        influx   http://10.0.0.5:8086 bucket 'oxus2'
        station  001010001

PASS  single-point — One typical weather-station reading...  (2.3s)
PASS  multi-point-history — Three consecutive readings...  (2.1s)
PASS  batch-single-message — Two readings in a single MQTT message...  (2.2s)
PASS  full-sensor-set — Every measurement field the platform reads...  (2.4s)
PASS  nan-field — A real-world malformed reading...  (2.2s)
PASS  edge-values — Value ranges that expose rounding...  (2.1s)

PASS  6/6 cases passed — ingestion pipeline verified end to end
        test points tagged stationID="001010001"
        test points removed from InfluxDB
```

### Keep a machine-readable record for the stage sign-off

```bash
docker run --rm -v "$PWD/reports:/reports" -e VAULT_ADDR=https://vault.internal:8200 -e VAULT_TOKEN=hvs.example agrobank-ingestion-test:latest run --json-report /reports/stage2.json
```

---

## CLI reference

| Command | Purpose |
|---------|---------|
| `cases` | List the bundled payloads; add `--show-payloads` to print the exact line protocol. Contacts nothing. |
| `preflight` | Verify Vault, Mosquitto and InfluxDB are reachable and the bucket exists. Publishes nothing. |
| `run` | Publish every payload and verify it lands in InfluxDB. |
| `cleanup` | Delete leftover points from an earlier `run --keep` (`--since 2h` by default, never all of history). |

`run` options:

| Option | Default | Purpose |
|--------|---------|---------|
| `--case NAME` | all | Run only the named case. Repeatable. |
| `--timeout SECONDS` | `60` | How long to wait for a case's points to appear. |
| `--poll-interval SECONDS` | `2` | Gap between InfluxDB checks while waiting. |
| `--cleanup` / `--keep` | `--cleanup` | Delete this run's points afterwards, or keep them for inspection. |
| `--station-id ID` | `001010001` | Publish as a different station ID. |
| `--json-report PATH` | — | Write the full result as JSON. |
| `--payload-dir DIR` | bundled | Use payload files from a mounted directory instead. |
| `--no-color` | — | Plain output for log capture. |

Exit codes: `0` all cases passed · `1` a case failed · `2` configuration or
payload problem · `3` Vault, Mosquitto or InfluxDB unreachable.

---

## Reading a failure

| Symptom | Most likely cause |
|---------|-------------------|
| `missing` for every field of every case | Nothing consumes the topic, or it writes to a different bucket/measurement. Check the ingestion host's consumer and its InfluxDB target. |
| `missing` only for the `batch-single-message` case | The consumer handles one line per message and drops the rest of a multi-line body. |
| `stored at a different time (+N ms)` | The consumer is re-stamping points server-side instead of honouring the published nanosecond timestamp. Historical backfill will land in the wrong place. |
| `value differs` on decimals | Values are being coerced (e.g. to integer or float32) somewhere in the path. |
| `unexpected` on `Balance` | Malformed `NAN` fields are being written verbatim instead of dropped. |
| `mosquitto: Broker refused the connection: bad username or password` | The Vault secret's MQTT credentials no longer match the broker's password file. |
| `influxdb: Bucket '…' does not exist` | Wrong `INFLUX_ORG`/`INFLUX_BUCKET`, or the token is scoped to another org. |

Add `--keep` to leave the points in InfluxDB and inspect them directly:

```bash
influx query 'from(bucket:"oxus2") |> range(start:-2h) |> filter(fn:(r) => r.stationID == "001010001")'
```

---

## Repository layout

```
src/ingestion_test/
  cli.py         CLI: cases / preflight / run / cleanup
  config.py      configuration model — Vault is the only source
  vault.py       KV v2 (and v1) secret read, token or AppRole auth
  payloads.py    case-file parsing and line-protocol rendering
  publisher.py   MQTT publish with delivery confirmation
  verifier.py    Flux query, value comparison, test-data cleanup
  report.py      terminal output and the JSON report
  data/*.json    the test payloads
docs/            deployment & handover guide (PDF)
```
