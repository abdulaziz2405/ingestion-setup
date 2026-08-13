# Deploying Vault Agent

Variables that are used here and require change:
- `<TOOL_SERVER>`: IP of the tool server where Vault runs, the same one used in [Vault](./vault.md).

Vault Agent authenticates to Vault and renders one `.env` per app into `/opt/app/shared/secrets/`.

Each app keeps its variables at `secret/amudario/applications/`, which is the path you type in
the Vault UI. The templates below spell it `secret/data/amudario/applications/...`: `data/` is
how the KV-v2 engine addresses the same secret over the API.

Applications read their `.env` only at container start, so a re-rendered secret does not reach
a running one. After editing anything in Vault, restart the unit that consumes it.

## 1. Setting up directories

On application server:

```bash
mkdir -p /opt/app/tools/vault-agent/data/{config,credentials,templates}
mkdir -p /opt/app/shared/secrets
```

## 2. Preparing Vault

Run these against Vault with a root token. 

```bash
export VAULT_TOKEN='hvs.xxxxxxxx'

docker exec -e VAULT_ADDR=http://<TOOL_SERVER>:8200 -e VAULT_TOKEN="$VAULT_TOKEN" vault \
  vault auth enable -path=amudario approle

cat <<'EOF' | docker exec -i -e VAULT_ADDR=http://<TOOL_SERVER>:8200 -e VAULT_TOKEN="$VAULT_TOKEN" vault \
  vault policy write amudario -
path "secret/data/amudario/applications/*" {
  capabilities = ["read"]
}
EOF

docker exec -e VAULT_ADDR=http://<TOOL_SERVER>:8200 -e VAULT_TOKEN="$VAULT_TOKEN" vault \
  vault write auth/amudario/role/amudario \
    token_policies="amudario" \
    token_ttl=1h token_max_ttl=4h secret_id_ttl=0
```

Read the role credentials and copy both values for the next step:

```bash
docker exec -e VAULT_ADDR=http://<TOOL_SERVER>:8200 -e VAULT_TOKEN="$VAULT_TOKEN" vault \
  vault read auth/amudario/role/amudario/role-id
docker exec -e VAULT_ADDR=http://<TOOL_SERVER>:8200 -e VAULT_TOKEN="$VAULT_TOKEN" vault \
  vault write -f auth/amudario/role/amudario/secret-id
```

## 3. Storing the credentials

Back on the application server, write the two values into files.
The `-n` matters — a trailing newline breaks the agent's reading of the value.

```bash
echo -n "<ROLE_ID>"   > /opt/app/tools/vault-agent/data/credentials/role_id
echo -n "<SECRET_ID>" > /opt/app/tools/vault-agent/data/credentials/secret_id
chmod 600 /opt/app/tools/vault-agent/data/credentials/{role_id,secret_id}
```

## 4. Agent config

Create `/opt/app/tools/vault-agent/data/config/vault-agent.hcl`.
It carries one `template` block per app: add a block whenever you onboard a new one.

```hcl
auto_auth {
  method {
    type       = "approle"
    mount_path = "auth/amudario"
    config = {
      role_id_file_path                   = "/vault-agent/credentials/role_id"
      secret_id_file_path                 = "/vault-agent/credentials/secret_id"
      remove_secret_id_file_after_reading = false
    }
  }
  sink { type = "file"  config = { path = "/vault-agent/credentials/token" } }
}

template_config {
  static_secret_render_interval = "10s"
}

cache { use_auto_auth_token = true }

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = true
}

vault { address = "http://<TOOL_SERVER>:8200" }

template {
  source      = "/vault-agent/templates/oxus-models-agrobank-template.tpl"
  destination = "/secrets/oxus-models-agro.env"
}

template {
  source      = "/vault-agent/templates/oxus-backend-agrobank-template.tpl"
  destination = "/secrets/oxus-backend-agro.env"
}

template {
  source      = "/vault-agent/templates/oxus-prefect-server-agrobank-template.tpl"
  destination = "/secrets/oxus-prefect-server-agro.env"
}

template {
  source      = "/vault-agent/templates/oxus-prefect-worker-agrobank-template.tpl"
  destination = "/secrets/oxus-prefect-worker-agro.env"
}
```

Prefect takes two blocks: the server and the worker hold separate secrets, and each unit reads
its own file.

Each app also needs its own template file.

Create `/opt/app/tools/vault-agent/data/templates/oxus-backend-agrobank-template.tpl`:

```hcl
{{- with secret "secret/data/amudario/applications/oxus-backend" -}}
{{- range $key, $value := .Data.data }}
{{ $key }}={{ $value }}
{{- end }}
{{ end -}}
```

Create `/opt/app/tools/vault-agent/data/templates/oxus-models-agrobank-template.tpl`:

```hcl
{{- with secret "secret/data/amudario/applications/oxus-models" -}}
{{- range $key, $value := .Data.data }}
{{ $key }}={{ $value }}
{{- end }}
{{ end -}}
```

Create `/opt/app/tools/vault-agent/data/templates/oxus-prefect-server-agrobank-template.tpl`:

```hcl
{{- with secret "secret/data/amudario/applications/oxus-prefect-server" -}}
{{- range $key, $value := .Data.data }}
{{ $key }}={{ $value }}
{{- end }}
{{ end -}}
```

Create `/opt/app/tools/vault-agent/data/templates/oxus-prefect-worker-agrobank-template.tpl`:

```hcl
{{- with secret "secret/data/amudario/applications/oxus-prefect-worker" -}}
{{- range $key, $value := .Data.data }}
{{ $key }}={{ $value }}
{{- end }}
{{ end -}}
```

## 5. Systemd unit

Create `/etc/systemd/system/dev-secretstore-vault-agent.service` and paste the following code:

```ini
[Unit]
Description=Vault Agent
Requires=docker.service network-online.target
After=docker.service network-online.target

[Service]
Type=simple
ExecStartPre=-/usr/bin/docker rm -f dev-secretstore-vault-agent
ExecStart=/usr/bin/docker run --rm \
    --name dev-secretstore-vault-agent \
    -p 127.0.0.1:18200:8200 \
    -v /opt/app/tools/vault-agent/data:/vault-agent:rw \
    -v /opt/app/shared/secrets:/secrets:rw \
    --cpus 0.1 \
    --memory 256m \
    --entrypoint vault \
    hashicorp/vault:1.18 \
    agent -log-level=info -config=/vault-agent/config/vault-agent.hcl
ExecStop=/usr/bin/docker stop dev-secretstore-vault-agent
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

Start and check the unit:

```bash
systemctl daemon-reload
systemctl enable --now dev-secretstore-vault-agent
systemctl status dev-secretstore-vault-agent
docker logs dev-secretstore-vault-agent --tail=30
```

Confirm the files are being rendered, one per template block:

```bash
ls -l /opt/app/shared/secrets/
# oxus-backend-agro.env
# oxus-models-agro.env
# oxus-prefect-server-agro.env
# oxus-prefect-worker-agro.env
```