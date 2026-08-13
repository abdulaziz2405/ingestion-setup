# Deploying Vault (internal-infra)

Variables that are used here and require change:
- `<TOOL_SERVER>`: IP of the tool server — Vault and the plain PostgreSQL it uses ([PostgreSQL](../postgres.md)) both run here.
- `<VAULT_DB_PASSWORD>`: password you pick for the `vault` PostgreSQL role.

## 1. Preparing the Postgres storage backend

Vault keeps everything in one Postgres table. Create a dedicated database and role for it.
Run this against the shared instance from [PostgreSQL](../postgres.md), as `postgres`:

```bash
docker exec -i postgres psql -U postgres <<'EOF'
CREATE ROLE vault WITH LOGIN PASSWORD '<VAULT_DB_PASSWORD>';
CREATE DATABASE vault OWNER vault;
EOF
```

Then create the table **inside the `vault` database, as the `vault` role**. It has to be owned
by `vault`, otherwise Vault can read it but not write to it:

```bash
docker exec -i postgres psql -U vault -d vault <<'EOF'
CREATE TABLE vault_kv_store (
  parent_path TEXT COLLATE "C" NOT NULL,
  path        TEXT COLLATE "C",
  key         TEXT COLLATE "C",
  value       BYTEA,
  CONSTRAINT pkey PRIMARY KEY (path, key)
);

CREATE INDEX parent_path_idx ON vault_kv_store (parent_path);
EOF
```

Confirm the table is present and owned by `vault`:
```bash
docker exec -i postgres psql -U vault -d vault -c '\d vault_kv_store'
docker exec -i postgres psql -U vault -d vault -c '\dt vault_kv_store'
```

## 2. Server config

Create directory:
```bash
mkdir -p /etc/vault
```

Create `/etc/vault/config.hcl`:

```hcl
storage "postgresql" {
  connection_url = "postgres://vault:<VAULT_DB_PASSWORD>@<TOOL_SERVER>:5432/vault?sslmode=disable"
  table          = "vault_kv_store"
  ha_enabled     = "false"
}

listener "tcp" {
  address     = "<TOOL_SERVER>:8200"
  tls_disable = "true"
}

api_addr      = "http://<TOOL_SERVER>:8200"
disable_mlock = true
ui            = true
```

The file holds the DB password, so keep it unreadable to everyone else. The container runs as
the image's `vault` user (uid 100, gid 1000), so hand the file to that uid. A root-owned `600`
file mounted into the container cannot be read by it, and Vault fails to start:

```bash
chown 100:1000 /etc/vault/config.hcl
chmod 600 /etc/vault/config.hcl
```

## 3. Systemd unit

Create `/etc/systemd/system/internal-infra-secretstore-vault-01.service` and paste the following code:

```ini
[Unit]
Description=Vault in Docker
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=0
Restart=on-failure
RestartSec=5s

ExecStartPre=-/usr/bin/docker rm -f vault
ExecStart=/usr/bin/docker run --rm --name vault \
  --network host \
  -e VAULT_ADDR=http://<TOOL_SERVER>:8200 \
  -v /etc/vault:/etc/vault:ro \
  hashicorp/vault:1.18 \
  server -config=/etc/vault/config.hcl
ExecStop=/usr/bin/docker stop vault

[Install]
WantedBy=multi-user.target
```

Start it and verify:
```bash
systemctl daemon-reload
systemctl enable --now internal-infra-secretstore-vault-01
systemctl status internal-infra-secretstore-vault-01 
docker logs vault --tail=40
```

## 4. Initialize Vault

All admin commands go through the running container:
```bash
docker exec -e VAULT_ADDR=http://<TOOL_SERVER>:8200 vault vault status
```

At this point `vault status` reports `Initialized false`, `Sealed true`. 
That is expected.

Now we have to initialize the Vault. This is done **exactly once**.
```bash
docker exec -e VAULT_ADDR=http://<TOOL_SERVER>:8200 vault \
  vault operator init -key-shares=5 -key-threshold=3
```

Output is 5 unseal keys plus the initial root token:
```
Unseal Key 1: <...>
Unseal Key 2: <...>
Unseal Key 3: <...>
Unseal Key 4: <...>
Unseal Key 5: <...>

Initial Root Token: hvs.<...>
```

**Copy all 5 keys and the root token into safe place before you proceed any further.**

## 5. Unseal

Run this three times, each time with a different key:
```bash
docker exec -it -e VAULT_ADDR=http://<TOOL_SERVER>:8200 vault vault operator unseal
```

Watch the progress counter, it should keep going up:
```
Sealed          true
Unseal Progress 1/3
```

After the third key, it should look like this:
```
Sealed          false
```

Once you're done, verify it's unsealed and initialized:
```bash
docker exec -e VAULT_ADDR=http://<TOOL_SERVER>:8200 vault vault status
```

Expected output should look something like this:
```
Initialized     true
Sealed          false
Storage Type    postgresql
HA Enabled      false
```

Now, using the root token, create the KV-mount where you will store your future secrets.
The mount is named `secret`, and every path the apps and Vault Agent reference sits under it:

```bash
export VAULT_TOKEN='hvs.xxxxxxxx'

docker exec -e VAULT_ADDR=http://<TOOL_SERVER>:8200 -e VAULT_TOKEN="$VAULT_TOKEN" vault \
  vault secrets enable -path=secret kv-v2

docker exec -e VAULT_ADDR=http://<TOOL_SERVER>:8200 -e VAULT_TOKEN="$VAULT_TOKEN" vault \
  vault secrets list
```

## 6. Create admin user

The UI is served by the same listener, at `http://<TOOL_SERVER>:8200`. Open it and log
in with the root token.

Add an `admin` policy under "Policies -> Create ACL Policy", naming it `admin`.
The single wildcard rule already covers every path, `sys/` and `auth/` included:

```
path "*" {
  capabilities = ["create", "read", "update", "delete", "list", "patch", "sudo"]
}
```

Enable the userpass method and create the **admin** user.
`token_policies=admin` refers to the policy by the name you gave it above:
```
export VAULT_TOKEN='hvs.xxxxxxxx'

docker exec -e VAULT_ADDR=http://<TOOL_SERVER>:8200 -e VAULT_TOKEN="$VAULT_TOKEN" vault \
  vault auth enable userpass

docker exec -e VAULT_ADDR=http://<TOOL_SERVER>:8200 -e VAULT_TOKEN="$VAULT_TOKEN" vault \
  vault write auth/userpass/users/admin password='MyPassword123' token_policies=admin
```