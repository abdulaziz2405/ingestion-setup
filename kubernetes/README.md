# Amudario on Kubernetes

Kubernetes equivalents of the systemd units in [applications/](../applications/). Same images,
same env variables, same start-up steps (migrations, seed, Prefect init), same routing as the
gateway NGINX in [Oxus-Frontend](../applications/oxus-frontend/oxus-frontend.md#3-gateway).

The databases, Redis, InfluxDB and Vault stay **outside** the cluster, on the servers from the
main [README](../README.md). Only the applications move.

```
kubernetes/
├── common/
│   ├── namespace.yaml          # namespace `agrobank`
│   ├── registry-secret.yaml    # Secret gitlab-registry: image pull credentials
│   ├── certificate.yaml        # OPTIONAL: cert-manager Certificate -> Secret agrobank-tls
│   ├── tls-secret.yaml         # OPTIONAL: Secret agrobank-tls by hand, when there is no cert-manager
│   └── gateway.yaml            # OPTIONAL: Gateway, only if you use HTTPRoutes and none exists
├── oxus-backend/               # deployment, service, ingress | httproute, secret
├── oxus-models/                # deployment-api, deployment-celery, service, ingress | httproute, secret
├── oxus-prefect/               # deployment-server, deployment-worker, service, ingress | httproute, secret-server, secret-worker
├── oxus-frontend/              # deployment, service, ingress | httproute (no secret)
└── agrobank-seed/              # A: job + secret (Vault token) | B: job-local-vault + secret-local-vault (no Vault)
```

### Secrets

| Secret manifest | Secret name | Used by | How the app gets it |
|:----------------|:------------|:--------|:--------------------|
| `common/registry-secret.yaml` | `gitlab-registry` | every pod | `imagePullSecrets` |
| `common/tls-secret.yaml` *or* `common/certificate.yaml` | `agrobank-tls` | ingresses / gateway | TLS termination |
| `oxus-backend/secret.yaml` | `oxus-backend-env` | backend (+ migrate init) | **file**, key `.env` mounted at `/var/www/html/.env` (read-only), as Vault Agent's file was on the VM |
| `oxus-models/secret.yaml` | `oxus-models-env` | models API (+ inits), Celery | env vars (`envFrom`), as `docker --env-file` was |
| `oxus-prefect/secret-server.yaml` | `oxus-prefect-server-env` | Prefect server | env vars (`envFrom`) |
| `oxus-prefect/secret-worker.yaml` | `oxus-prefect-worker-env` | Prefect worker (+ inits) | env vars (`envFrom`) |
| `agrobank-seed/secret.yaml` (option A) | `agrobank-seed-vault` | seed Job | env vars (`envFrom`): Vault address + token |
| `agrobank-seed/secret-local-vault.yaml` (option B) | `agrobank-seed-env` | seed Job, local-Vault variant | **files** at `/seed-secret/<KEY>` in the Job's own Vault sidecar, which the seeder reads (see step 6) |

Models and Prefect read their settings from environment variables, not from a file. On the VM
the file was only docker's `--env-file` input, so injecting the keys as env vars is the exact
equivalent. Frontend needs no secret.

**The `secret*.yaml` files hold placeholders. Fill them on the deploy machine and do not commit
real values.** After applying, `git checkout -- kubernetes/` puts the placeholders back.

| App | Service (in-cluster) | Container port | Public path |
|:----|:---------------------|:---------------|:------------|
| oxus-backend | `oxus-backend:8080` | 80 | `/api` |
| oxus-models (API) | `oxus-models-api:8011` | 8000 | `/models` |
| oxus-prefect (server) | `oxus-prefect-server:4200` | 4200 | `/prefect` |
| oxus-frontend | `oxus-frontend:9090` | 80 | `/` (everything else) |

Service ports equal the old host ports, so the only change in the secret values is
`<APPLICATION_SERVER>` → the Service name. Those lines are marked `[K8S]` in the `secret*.yaml`.

---

## 0. Look at the cluster first

Run these before touching anything. The answers decide which values you fill in below.

```bash
kubectl version                         # client works, server reachable
kubectl get nodes -o wide               # node IPs: the DB/Redis/Influx/Vault firewalls must allow them

kubectl get ingressclass                # -> <INGRESS_CLASS>, if you go the Ingress way
kubectl get gatewayclass                # Gateway API installed? -> <GATEWAY_CLASS>
kubectl get gateway -A                  # existing Gateway -> <GATEWAY_NAME> / <GATEWAY_NAMESPACE>
kubectl get clusterissuer               # cert-manager present? -> <CLUSTER_ISSUER>

kubectl get ns agrobank                 # namespace name already taken?
```

**Routing: Ingress or HTTPRoute?** Pick one for all four apps.
- `kubectl get gateway -A` lists a Gateway → use `httproute.yaml`.
- Otherwise, `kubectl get ingressclass` lists something → use `ingress.yaml`.
- Neither → the cluster has no way in; that has to be installed first.

**Can pods reach the outside servers?** Test from inside the cluster, before deploying:

```bash
kubectl run nettest --rm -it --restart=Never --image=busybox:1.36 -- sh -c '
  nc -zvw3 <DB_SERVER> 5432; nc -zvw3 <DB_SERVER> 5433; nc -zvw3 <DB_SERVER> 6379;
  nc -zvw3 <TOOL_SERVER> 5432; nc -zvw3 <TOOL_SERVER> 8200; nc -zvw3 <INGESTION_SERVER> 8086'
# <TOOL_SERVER> 8200 (Vault) only matters for seed option A, see step 6.
```

Every line must say `open`. If not: firewall / `pg_hba.conf` / Redis `bind` on those servers
must allow the cluster's node IPs (or pod CIDR, depending on whether the CNI SNATs egress).

---

## 1. Values to change

Find every leftover placeholder at any time with:

```bash
grep -rnE '<[A-Z_ ]+>' kubernetes/ --include='*.yaml'
```

### In the YAML manifests

| Placeholder | Where | What |
|:------------|:------|:-----|
| `<DOMAIN>` | every `ingress.yaml` / `httproute.yaml`, `common/certificate.yaml`, `common/gateway.yaml` | public domain, same one as `PREFECT_UI_URL` |
| `<BACKEND_PIPELINE_IID>` | `oxus-backend/deployment.yaml` (2×) | image tag suffix, from GitLab pipelines of `backend2` |
| `<MODELS_PIPELINE_IID>` | `oxus-models/deployment-api.yaml` (3×), `deployment-celery.yaml` (1×) | same tag in both files |
| `<PREFECT_PIPELINE_IID>` | `oxus-prefect/deployment-server.yaml` (1×), `deployment-worker.yaml` (3×) | same tag in both files |
| `<FRONTEND_PIPELINE_IID>` | `oxus-frontend/deployment.yaml` | |
| `<INGRESS_CLASS>` | every `ingress.yaml` | from `kubectl get ingressclass` |
| `<GATEWAY_NAME>`, `<GATEWAY_NAMESPACE>` | every `httproute.yaml` | the Gateway to attach to (`agrobank-gateway` / `agrobank` if you apply `common/gateway.yaml`) |
| `<GATEWAY_CLASS>` | `common/gateway.yaml` | only if you create your own Gateway |
| `<CLUSTER_ISSUER>` | `common/certificate.yaml` | only with cert-manager |

Quick replace (GNU sed on Linux; on macOS use `sed -i ''`):

```bash
cd kubernetes
grep -rl '<DOMAIN>' . | xargs sed -i 's|<DOMAIN>|agro.example.uz|g'
sed -i 's|<BACKEND_PIPELINE_IID>|1234|g'  oxus-backend/deployment.yaml
sed -i 's|<MODELS_PIPELINE_IID>|1234|g'   oxus-models/deployment-*.yaml
sed -i 's|<PREFECT_PIPELINE_IID>|1234|g'  oxus-prefect/deployment-*.yaml
sed -i 's|<FRONTEND_PIPELINE_IID>|1234|g' oxus-frontend/deployment.yaml
sed -i 's|<INGRESS_CLASS>|nginx|g' */ingress.yaml
# or, for HTTPRoutes:
sed -i 's|<GATEWAY_NAME>|my-gateway|g; s|<GATEWAY_NAMESPACE>|gateway|g' */httproute.yaml
```

### Things you may want to change, not placeholders

| What | Where | Default |
|:-----|:------|:--------|
| Namespace | `namespace:` in every file, `common/namespace.yaml`, `-n` in commands | `agrobank` |
| Replicas | `replicas:` in deployments | 1 everywhere, frontend 2. See [Scaling](#scaling) |
| CPU / memory | `resources:` | requests only; Prefect limits copied from the docker flags |
| Upload size (413 errors) | `oxus-backend/ingress.yaml` annotation | controller default (1m on ingress-nginx) |
| Proxy timeout | ingress annotations / `timeouts.request` in httproutes | 120s, as in the NGINX gateway |
| TLS Secret name | `secretName: agrobank-tls` in ingresses, `common/*.yaml` | `agrobank-tls` |

Namespace rename in one go (only exact `agrobank` values change, not names like `agrobank-tls`):

```bash
grep -rl 'agrobank$' kubernetes --include='*.yaml' \
  | xargs sed -i 's/namespace: agrobank$/namespace: NEWNS/; s/^  name: agrobank$/  name: NEWNS/'
```

### In the secret manifests

Fill every `''` and `<...>` in the `secret*.yaml` files (table in [Secrets](#secrets)).

Format rules:
- `oxus-backend/secret.yaml`: the body under `.env: |` is a plain Laravel `.env`, indented
  4 spaces. Laravel syntax applies (quote a value that has spaces or `#`).
- All other `secret*.yaml`: `KEY: 'value'`. Keep the single quotes (so `0123`, `true`, `:`, `#`
  stay literal strings); a `'` inside a value is written `''`.

**Fastest:** print the current values from the running Vault, already in the right format,
and paste them over the matching block:

```bash
export VAULT_ADDR=http://<TOOL_SERVER>:8200 VAULT_TOKEN=<TOKEN>
dotenv() { vault kv get -format=json "secret/amudario/applications/$1" \
  | jq -r '.data.data | to_entries[] | "    \(.key)=\(.value)"'; }
yamlkv() { vault kv get -format=json "secret/amudario/applications/$1" \
  | jq -r --arg q "'" '.data.data | to_entries[] | "  \(.key): \($q)\(.value | gsub($q; $q+$q))\($q)"'; }

dotenv oxus-backend           # paste under `.env: |`   in oxus-backend/secret.yaml
yamlkv oxus-models            # paste under `stringData:` in oxus-models/secret.yaml
yamlkv oxus-prefect-server    # paste under `stringData:` in oxus-prefect/secret-server.yaml
yamlkv oxus-prefect-worker    # paste under `stringData:` in oxus-prefect/secret-worker.yaml
```

Then fix the lines that pointed at the application server:

| File | Variable | Value |
|:-----|:---------|:------|
| `oxus-backend/secret.yaml` | `MODELS_API_URL` | `http://oxus-models-api:8011/models/api` |
| `oxus-models/secret.yaml` | `OXUS_BACKEND_URL` | `http://oxus-backend:8080` |
| `oxus-prefect/secret-worker.yaml` | `PREFECT_API_URL` | `http://oxus-prefect-server:4200/api` |
| `oxus-prefect/secret-worker.yaml` | `OXUS_BACKEND_URL` | `http://oxus-backend:8080` |
| `oxus-prefect/secret-worker.yaml` | `OXUS_MODELS_URL` | `http://oxus-models-api:8011/models/api` |
| `oxus-prefect/secret-server.yaml` | `PREFECT_UI_URL`, `PREFECT_UI_API_URL` | only if `<DOMAIN>` changes |

`grep -nE ':8080|:8011|:4200' kubernetes/*/secret*.yaml` shows them. If the namespace is not
`agrobank` the short Service names still work (same namespace).

---

## 2. Namespace and registry access

```bash
cd kubernetes
kubectl apply -f common/namespace.yaml

# Fill <DEPLOY_TOKEN_USERNAME> / <DEPLOY_TOKEN> first
# (Vault UI -> secrets -> amudario -> tools -> gitlab -> agrobank-deploy-token)
kubectl apply -f common/registry-secret.yaml
```

## 3. TLS certificate

The ingresses/gateway read the certificate from Secret `agrobank-tls`. Pick one:

```bash
# a) cert-manager present (fill <DOMAIN>, <CLUSTER_ISSUER> first):
kubectl apply -f common/certificate.yaml
kubectl -n agrobank get certificate agrobank-tls -w     # wait for READY=True

# b) no cert-manager, you have the cert + key: paste them into common/tls-secret.yaml, then
kubectl apply -f common/tls-secret.yaml
```

TLS is terminated somewhere else (a load balancer in front)? Delete the `tls:` block from the
four `ingress.yaml` files and skip this step.

With HTTPRoutes and an **existing shared** Gateway, the certificate belongs to that Gateway's
listener (usually in its own namespace), not to `agrobank`. Ask whoever runs it to add
`<DOMAIN>`, or create the Secret/Certificate in the Gateway's namespace and reference it there.

## 4. Application secrets

```bash
grep -nE "<[A-Z_ ]+>" oxus-*/secret*.yaml        # must print nothing (except comments)
kubectl apply -f oxus-backend/secret.yaml \
              -f oxus-models/secret.yaml \
              -f oxus-prefect/secret-server.yaml \
              -f oxus-prefect/secret-worker.yaml
kubectl -n agrobank get secrets
```

Check a value landed as intended:

```bash
kubectl -n agrobank get secret oxus-models-env -o jsonpath='{.data.OXUS_BACKEND_URL}' | base64 -d; echo
kubectl -n agrobank get secret oxus-backend-env -o jsonpath='{.data.\.env}' | base64 -d | head
```

## 5. Deploy, in the original order

Replace `ingress.yaml` with `httproute.yaml` everywhere if you went the Gateway way
(and apply `common/gateway.yaml` first if you need your own Gateway).

```bash
# Backend (initContainer runs `php artisan migrate --force`)
kubectl apply -f oxus-backend/deployment.yaml -f oxus-backend/service.yaml -f oxus-backend/ingress.yaml
kubectl -n agrobank rollout status deploy/oxus-backend --timeout=5m

# Models (initContainers: `alembic upgrade head`, then `python -m scripts.deploy_seed`)
kubectl apply -f oxus-models/deployment-api.yaml -f oxus-models/service.yaml -f oxus-models/ingress.yaml
kubectl -n agrobank rollout status deploy/oxus-models-api --timeout=5m
kubectl apply -f oxus-models/deployment-celery.yaml
kubectl -n agrobank rollout status deploy/oxus-models-celery --timeout=5m
```

## 6. Seed (once), then the rest

Two variants, pick **one**:
- **A, `job.yaml` + `secret.yaml`:** the seeder reads its credentials from the existing Vault
  (`<TOOL_SERVER>:8200`, secret already at `secret/amudario/agrobank/seed`). The default.
- **B, `job-local-vault.yaml` + `secret-local-vault.yaml`:** no Vault reachable. You put the
  DB/S3 credentials in the Secret and the Job brings its own temporary Vault.

### Option A: existing Vault

```bash
# Fill <TOOL_SERVER> and a Vault token that can read secret/amudario/agrobank/seed
kubectl apply -f agrobank-seed/secret.yaml

# Run the Job; change `args:` in agrobank-seed/job.yaml between runs:
#   ["preflight"] -> ["seed", "--dry-run"] -> ["seed"]
kubectl -n agrobank delete job agrobank-seed --ignore-not-found
kubectl apply -f agrobank-seed/job.yaml
kubectl -n agrobank logs -f job/agrobank-seed     # "waiting to start"? run it again in a few seconds

# After the real seed: mirror device/disease links into oxus-models
kubectl -n agrobank exec deploy/oxus-models-api -c api -- sh -c 'cd /app && python -m scripts.import_from_backend'

# Revoke the seeder token in Vault, then:
kubectl -n agrobank delete secret agrobank-seed-vault
```

### Option B: no Vault, the Job brings its own

```bash
# Fill the DB + S3 credentials
kubectl apply -f agrobank-seed/secret-local-vault.yaml

# Run the Job; change `args:` in agrobank-seed/job-local-vault.yaml between runs:
#   ["preflight"] -> ["seed", "--dry-run"] -> ["seed"]
kubectl -n agrobank delete job agrobank-seed-local-vault --ignore-not-found
kubectl apply -f agrobank-seed/job-local-vault.yaml
kubectl -n agrobank logs job/agrobank-seed-local-vault -c vault       # "loaded 17 keys into ..."
kubectl -n agrobank logs -f job/agrobank-seed-local-vault -c seed     # "waiting to start"? run it again in a few seconds

# After the real seed: mirror device/disease links into oxus-models
kubectl -n agrobank exec deploy/oxus-models-api -c api -- sh -c 'cd /app && python -m scripts.import_from_backend'

# Done: drop the elevated DB/S3 credentials from the cluster
kubectl -n agrobank delete secret agrobank-seed-env
```

How it works: the seeder only reads credentials from Vault (by design: they never sit in its
environment). So `job-local-vault.yaml` runs a throwaway `hashicorp/vault:1.18` in dev mode
next to it: in memory, listening on `127.0.0.1` inside the pod only, gone when the Job ends.
It loads `agrobank-seed-env` into `secret/amudario/agrobank/seed` (the seeder's default path),
and a startup probe holds the seeder back until that is done. This uses a native sidecar, so
the cluster must be **Kubernetes ≥ 1.29** (`kubectl version`).

Seeder exit codes (both options): `0` ok, `1` a check or load failed, `2` config/payload
problem, `3` Vault, a database or S3 unreachable.

```bash
# Prefect: server first, the worker waits for it and applies prefect.prod.yaml in an initContainer
kubectl apply -f oxus-prefect/deployment-server.yaml -f oxus-prefect/service.yaml -f oxus-prefect/ingress.yaml
kubectl -n agrobank rollout status deploy/oxus-prefect-server --timeout=5m
kubectl apply -f oxus-prefect/deployment-worker.yaml
kubectl -n agrobank rollout status deploy/oxus-prefect-worker --timeout=5m

# Frontend
kubectl apply -f oxus-frontend/deployment.yaml -f oxus-frontend/service.yaml -f oxus-frontend/ingress.yaml
kubectl -n agrobank rollout status deploy/oxus-frontend --timeout=3m
```

## 7. Verify

```bash
kubectl -n agrobank get pods,svc,ingress            # or: get httproute
kubectl -n agrobank logs deploy/oxus-prefect-worker | grep -i 'started'      # 'Worker ... started!'
kubectl -n agrobank exec deploy/oxus-prefect-server -- prefect work-pool ls
kubectl -n agrobank exec deploy/oxus-prefect-server -- prefect deployment ls

curl -sI https://<DOMAIN>/                          # frontend
curl -s  https://<DOMAIN>/api/health                # backend
curl -sI https://<DOMAIN>/prefect                   # Prefect UI
```

HTTPRoute not working? `kubectl -n agrobank describe httproute oxus-backend`: the `Accepted`
and `ResolvedRefs` conditions name the problem (usually the Gateway listener's `allowedRoutes`
does not allow namespace `agrobank`, or the hostname does not match the listener).

---

## Day-2

**New image:** change the tag and re-apply, or in one command:

```bash
kubectl -n agrobank set image deploy/oxus-backend app=registry.gitlab.com/amudario/development/backend2:agrobank-<IID> migrate=registry.gitlab.com/amudario/development/backend2:agrobank-<IID>
```

(Every container in the deployment — initContainers included — must get the new tag. For
models, bump `deployment-api.yaml` and `deployment-celery.yaml` together; for Prefect, both
deployments.)

**Secret changed:** apps read their secret only at start (env vars, and the backend's `.env`
is mounted with `subPath`, which never refreshes), same as with Vault Agent. Edit the
`secret*.yaml`, re-apply and restart what uses it:

```bash
kubectl replace -f oxus-models/secret.yaml        # replace, not apply: apply does not drop removed keys
kubectl -n agrobank rollout restart deploy/oxus-models-api deploy/oxus-models-celery
```

| Secret file | Restart |
|:------------|:--------|
| `oxus-backend/secret.yaml` | `deploy/oxus-backend` |
| `oxus-models/secret.yaml` | `deploy/oxus-models-api deploy/oxus-models-celery` |
| `oxus-prefect/secret-server.yaml` | `deploy/oxus-prefect-server` |
| `oxus-prefect/secret-worker.yaml` | `deploy/oxus-prefect-worker` |

<a id="scaling"></a>
**Scaling:** frontend scales freely. Backend and models-api run migrations in an
initContainer, so two pods starting together run them twice at once; backend also keeps
sessions in files per pod (`SESSION_DRIVER=file`). Before going above 1 replica there, move
sessions to Redis and the migrations into a Job. Prefect server and worker stay at 1.

---

## Differences from the VM setup, and gotchas

- **`php artisan config:cache`** from the backend unit is not carried over: it ran in a
  throw-away `--rm` container, so its cache never reached the running app anyway.
- **`/internal/*` calls** (Prefect → backend/models) are only accepted from private addresses
  without proxy headers. They go pod → Service directly, so this holds as long as the pod CIDR
  is RFC 1918 (`10.x`, `172.16–31.x`, `192.168.x`). If the cluster uses `100.64.0.0/10`
  (some CNIs/clouds do), these calls get rejected — check with
  `kubectl get pods -n agrobank -o wide`. Same if a service mesh sidecar injects
  `X-Forwarded-For` on the way in.
- **`enableServiceLinks: false`** is set on every pod. Without it Kubernetes injects
  `<SERVICE>_PORT=tcp://...` style variables, which Prefect (and anything reading `REDIS_PORT`)
  would pick up if someone ever creates a Service with a clashing name.
- **No Vault Agent.** Secrets are plain Kubernetes Secrets from the `secret*.yaml` manifests. If the
  cluster runs external-secrets and can reach Vault, the same Vault paths can feed
  `ExternalSecret`s later; the Secret names above are what the deployments expect.
- **Docker network `oxus-models-agrobank`** is not needed: every pod reaches every Service in
  the namespace.
- **Pods `ImagePullBackOff`** → `gitlab-registry` secret missing/wrong, or it was created in
  another namespace.
- **Pods stuck in `Init:`** → `kubectl -n agrobank logs <pod> -c migrate` (or `deploy-seed`,
  `wait-for-server`, `deploy-flows`). Almost always DB reachability or a wrong secret value.
