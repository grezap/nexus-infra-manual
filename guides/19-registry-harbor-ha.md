# Guide 19 — Registry · Harbor HA (2 app nodes + PG/Redis HA datastore + MinIO blobs)

> **Mirrors:** `nexus-infra-registry` (a **separate repo** from the lakehouse) — the
> `registry-harbor-node` + `registry-pg-node` Packer templates + the
> `registry-harbor` Terraform env overlays (`…-nftables-backplane`,
> `…-vault-agents`, `…-tls`, `…-registry-pg-replication`, `…-harbor-config`,
> `…-cluster-bootstrap`) — Phase 0.L.4 / ADR-0036, tier `09-platform`. The platform's
> **container registry** + supply-chain showcase.

> 🔗 **Depends on Guide 16 (MinIO).** Harbor stores image layers (blobs) in the
> `harbor` bucket on `minio.nexus.lab:9000` — MinIO is a **hard runtime
> dependency**. Also depends on the **Vault OIDC provider** (foundation, Guide 04)
> for SSO. Build 16 first; MinIO must be **powered on** when you apply this tier.

---

## 1. Overview & purpose

**Harbor** — an OCI-compliant container registry, deployed **highly available** and
wired into the platform's storage, auth, and supply-chain primitives. **4 nodes,
two roles:**

- **App nodes (`registry-1/2`, `.115/.116`)** — two **stateless** Harbor instances
  (each a Docker-Compose stack of ~8 containers: core, registry, portal, jobservice,
  Trivy, nginx, redis-proxy, …). Stateless because **all durable state lives
  elsewhere** — blobs in MinIO, metadata in PG, cache in Redis. Either node can serve
  any request; round-robin DNS `registry.nexus.lab` fronts them. HA requires the two
  nodes share identical secrets (`/data/secret` — the secretkey + CSRF + registry
  http.secret), so a token issued by one is accepted by the other.
- **Datastore nodes (`registry-pg-1/2`, `.117/.118`)** — a **dedicated**
  PostgreSQL 17 master-replica HA pair **co-located with Redis** master-replica,
  fronted by a keepalived **VRRP VIP** `registry-db.nexus.lab` (`.119`). On failover
  the VIP holder promotes **both** PG (standby→primary) and Redis (replica→master).
  Harbor reaches DB `:5432` + cache `:6379` through the VIP.
- **Blob backend** = MinIO `s3://harbor` (Guide 16). **Auth** = Vault **OIDC SSO**
  (`auth_mode=oidc_auth`); the local admin stays as break-glass. **Supply chain** =
  **Trivy** (CVE scanning) + **cosign** (Sigstore image signing).

**Why HA + external state:** a registry is critical shared infra — every CI push and
every node's image pull goes through it. Making the app tier stateless + the
datastore HA means a node (or a DB) can die without an outage. **Why it matters:**
this is where the lab's images live, scanned and signed.

---

## 2. Component primer

- **Harbor.** A CNCF OCI registry: image storage + RBAC projects + replication +
  vulnerability scanning + signing + a web UI. Runs as a Docker-Compose stack. *Why:*
  a full-featured private registry with the supply-chain features built in.
  *Otherwise:* a plain `registry:2` (Distribution — no UI/scan/RBAC), GitLab/GitHub
  registries (tied to a forge), or a cloud registry (defeats on-prem).
- **Stateless app + external state.** Harbor supports **external** PostgreSQL, Redis,
  and S3 storage instead of its bundled ones — so the app containers hold no durable
  data and scale/replace freely. *Why:* that's what makes 2-node HA possible. *vs.
  the default single-node install:* bundled PG/Redis/filesystem = a single point of
  failure.
- **PostgreSQL + Redis HA datastore.** Same streaming-replication + keepalived-VIP
  pattern as **Guide 17's catalog PG**, here **co-located with Redis** and with a VIP
  that promotes *both* engines. PG holds Harbor's metadata (projects, users, tags);
  Redis holds the cache + job queue (non-authoritative — a cold cache after failover
  is fine). *Why a dedicated pair:* the registry gets its own datastore, isolated from
  the OLTP/catalog DBs.
- **MinIO S3 storage backend.** Harbor's `storage_service.s3` points image layers at
  `s3://harbor` on MinIO (path-style, v4 auth, the Vault CA as `ca_bundle`). *Why:*
  durable, erasure-coded blob storage shared with the rest of the lakehouse. *vs.
  `data_volume` (local disk):* not HA — two app nodes couldn't share it.
- **Trivy.** Harbor's built-in CVE scanner; `./install.sh --with-trivy` adds the
  scanner container. A push can be scanned on demand or by policy. *Otherwise:* Clair
  (the older Harbor scanner) or external scanning.
- **cosign / Sigstore.** Signs image digests so consumers can verify provenance
  (`cosign sign` → a signature accessory stored in Harbor; `cosign verify` checks it).
  *Why:* supply-chain integrity. *Otherwise:* Notary v1 (Harbor's legacy signer,
  deprecated).
- **Vault OIDC SSO.** Harbor's `auth_mode=oidc_auth` federates login to Vault's OIDC
  provider (Vault is the IdP, backed by AD via `auth/ldap`). *Why:* one identity plane
  for the platform. *Otherwise:* Harbor-local DB users, or LDAP directly.
- **keepalived dual-engine promotion.** The VRRP `notify_master` script promotes PG
  *and* Redis on the new VIP holder; `notify_backup` demotes the loser's Redis to a
  replica. State **`BACKUP` + `nopreempt`** so a recovered node doesn't flap the VIP.

---

## 3. Prerequisites

| # | Requirement | One-command verify |
|---|---|---|
| 1 | **Guide 16 (MinIO) built + POWERED ON** — the blob backend | `dig +short minio.nexus.lab @192.168.70.1` → 4 IPs; `mc admin info` healthy |
| 2 | **Foundation alive** (Guides 00–04) — Vault PKI + KV + **OIDC provider**; gateway DNS | `vault status` on `vault-1` → `Sealed: false`; `vault read identity/oidc/provider/nexus-registry` |
| 3 | **CA bundle** on the build host (`~/.nexus/vault-ca-bundle.crt`) | `Test-Path ~/.nexus/vault-ca-bundle.crt` → `True` |
| 4 | **4 `deb13` nodes** baselined (Guide 00), dual-NIC static `.115/.116` + `.117/.118` | the 4 answer `:22`; firstboot mapped `NEXUS_ROLE` / `NEXUS_PG_ROLE` |
| 5 | Egress to **download.docker.com**, **github.com/goharbor + sigstore**, **apt.postgresql.org** (or local caches) | `curl -sI https://download.docker.com | head -1` |

> **Versions:** **Harbor 2.14.4** (offline installer — images baked in), Docker CE
> (stable), **cosign 2.4.3**, **PostgreSQL 17** (PGDG) + **Redis** + keepalived.
> Front door: round-robin `registry.nexus.lab` → `.115/.116`. Datastore VIP
> `registry-db.nexus.lab` → `.119`. Blobs: `s3://harbor` at `https://minio.nexus.lab:9000`.

> **By-hand divergence:** read KV with `vault kv get` (no Vault Agent); issue certs
> with the `vault` CLI. Two cert layouts (Harbor `harbor.{crt,key}` owner
> `root:registry`; PG `server.{crt,key}` owner `postgres`, **key `0600`** — T1).
> Render `harbor.yml` by hand. The datastore HA mechanics are **the same as Guide 17
> §5.4** plus co-located Redis.

---

## 4. Target topology

| Node | Role | VMnet11 | VMnet10 | MAC (primary / secondary) | RAM | Ports |
|---|---|---|---|---|---|---|
| `registry-1` | Harbor app (seed) | `.115` | `.10.115` | `…3F:00:A4` / `…3F:01:A4` | 6 GB | 443 HTTPS |
| `registry-2` | Harbor app (join) | `.116` | `.10.116` | `…3F:00:AF` / `…3F:01:AF` | 6 GB | 443 |
| `registry-pg-1` | PG **primary** + Redis master | `.117` | `.10.117` | `…3F:00:B0` / `…3F:01:B0` | 2 GB | 5432 PG · 6379 Redis |
| `registry-pg-2` | PG **replica** + Redis replica | `.118` | `.10.118` | `…3F:00:B1` / `…3F:01:B1` | 2 GB | 5432 · 6379 |
| **VIP** | `registry-db.nexus.lab` (keepalived) | **`.119`** | — | — | — | 5432 + 6379 → current primary |

> MAC block `:A4` (canon) + `:AF/:B0/:B1`. **PG + Redis replication ride the VMnet10
> backplane**; Harbor→datastore (via the VIP) + the VIP + VRRP ride VMnet11. VRRP
> `virtual_router_id 73`. VMs under `H:\VMS\NexusPlatform\09-platform\<node>\` (the
> **platform** tier, not `08-spark`).

---

## 5. Step-by-step build

> **WHERE:** node steps as `nexusadmin` → `sudo -i` (root). `vault` runs on
> **`vault-1`**. Order: datastore HA first (Harbor needs DB+Redis), then Harbor app
> tier, then the OIDC + push/scan/sign exit gate.

### 5.0 — Seed KV + the Vault OIDC client (once)

> **Step 5.0.1 — Write the registry secrets to Vault KV**
> **WHERE:** `vault-1` (`.121`), root shell with an operator `VAULT_TOKEN`.
> **WHY:** all registry creds live in KV; the nodes read them. Hex passwords are
> inline-safe in SQL. The S3 blob creds reuse Guide 16's MinIO app key (its
> `readwrite` policy covers the new `harbor` bucket).
> **WHAT:**
> ```bash
> export VAULT_ADDR=https://127.0.0.1:8200 VAULT_CACERT=$HOME/.nexus/vault-ca-bundle.crt
> vault kv put nexus/registry/harbor-admin           value="$(openssl rand -hex 16)"
> vault kv put nexus/registry/harbor-secret-key      value="$(openssl rand -hex 16)"
> vault kv put nexus/registry/pg-superuser-password  value="$(openssl rand -hex 24)"
> vault kv put nexus/registry/pg-replication-password value="$(openssl rand -hex 24)"
> vault kv put nexus/registry/harbor-db-password     value="$(openssl rand -hex 24)"
> vault kv put nexus/registry/redis-password         value="$(openssl rand -hex 24)"
> ```
> **VERIFY:** `vault kv get -field=value nexus/registry/harbor-admin` returns 32 hex chars.

> **Step 5.0.2 — Create the Vault OIDC client for Harbor + seed its id/secret**
> **WHERE:** `vault-1`, root shell.
> **WHY:** Harbor federates login to Vault's OIDC provider. Create a confidential
> OIDC **client** whose redirect URI is Harbor's callback, expose it via the
> `nexus-registry` provider, and write the generated `client_id`/`client_secret` to
> KV (Harbor reads them in §5.5). (This is the OIDC instance of Guide 04's
> scaffolding pattern; the assignment/key/provider may already exist from the
> foundation build — these writes are idempotent.)
> **WHAT:**
> ```bash
> # key + assignment (idempotent) -- allow all entities for the lab
> vault write identity/oidc/key/nexus-registry-key allowed_client_ids="*" rotation_period=24h verification_ttl=24h
> vault write identity/oidc/assignment/nexus-registry-all entity_ids="" group_ids=""
> # the client (confidential) -- redirect URI = Harbor's OIDC callback
> vault write identity/oidc/client/nexus-registry \
>   key=nexus-registry-key \
>   redirect_uris="https://registry.nexus.lab/c/oidc/callback" \
>   assignments=allow_all \
>   id_token_ttl=30m access_token_ttl=1h
> # the provider that issues for this client
> vault write identity/oidc/provider/nexus-registry \
>   allowed_client_ids="$(vault read -field=client_id identity/oidc/client/nexus-registry)" \
>   scopes_supported="openid,profile,email,groups"
> # seed id + secret to KV for Harbor
> vault kv put nexus/registry/oidc-client-id     value="$(vault read -field=client_id     identity/oidc/client/nexus-registry)"
> vault kv put nexus/registry/oidc-client-secret value="$(vault read -field=client_secret identity/oidc/client/nexus-registry)"
> ```
> **EXPECTED:** the provider's discovery doc is served.
> **VERIFY:** `curl -s --cacert $VAULT_CACERT https://vault-1.nexus.lab:8200/v1/identity/oidc/provider/nexus-registry/.well-known/openid-configuration | jq .issuer`
> → `…/identity/oidc/provider/nexus-registry`.

### 5.1 — Create the 4 VMs + install packages

> **Step 5.1.1 — Create the 4 VMs + baseline**
> **WHERE:** VMware GUI on the build host.
> **WHY:** standard Guide 00 deb13 shape (no extra data disk). App nodes 6 GB (the
> Compose stack is heavy), datastore nodes 2 GB.
> **WHAT:** create `registry-1/2` (6 GB) + `registry-pg-1/2` (2 GB) per Guide 00 §5.A
> under the `09-platform` tier folder; pin the §4 MACs; install Debian 13 + baseline.
> firstboot maps `NEXUS_ROLE` (`registry-harbor`/`registry-pg`) + `NEXUS_PG_ROLE`
> (`primary`/`replica`).
> **VERIFY:** `ssh nexusadmin@192.168.70.117 'sudo grep NEXUS_PG_ROLE /etc/nexus-registry-pg/node-identity.env'`
> → `primary` (and `.118` → `replica`).

> **Step 5.1.2 — Install PostgreSQL 17 + Redis + keepalived on the datastore nodes (`.117/.118`)**
> **WHERE:** `registry-pg-1` + `registry-pg-2`, root shell.
> **WHY:** identical PGDG install to **Guide 17 §5.1.3** (bookworm PGDG repo +
> bookworm fallback for `libicu72`/`libldap-2.5-0`), plus **Redis** here. Units
> disabled — §5.4 configures + starts them.
> **WHAT (run on both datastore nodes — PGDG as Guide 17 §5.1.3, then):**
> ```bash
> apt-get install -y postgresql-17 postgresql-client-17 postgresql-contrib-17 keepalived redis-server
> systemctl disable --now postgresql postgresql@17-main keepalived redis-server 2>/dev/null || true
> install -d -o root -g postgres -m 0750 /etc/nexus-registry-pg /etc/nexus-registry-pg/tls
> ```
> **VERIFY:** `/usr/lib/postgresql/17/bin/postgres --version` → `17.x`; `redis-server --version`; `keepalived --version`.

> **Step 5.1.3 — Install Docker + Harbor 2.14.4 + cosign on the app nodes (`.115/.116`)**
> **WHERE:** `registry-1` + `registry-2`, root shell.
> **WHY:** Harbor runs as a Docker-Compose stack. The **offline** installer bakes the
> images in (no registry egress at install). `docker.service` stays disabled until
> §5.5 (after NIC config).
> **WHAT (run on both app nodes):**
> ```bash
> apt-get update && apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
> install -d -m 0755 /etc/apt/keyrings
> curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
> echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
> apt-get update && apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
> usermod -aG docker nexusadmin
> systemctl disable --now docker 2>/dev/null || true
> groupadd --system registry 2>/dev/null || true
> install -d -o root -g registry -m 0750 /etc/nexus-registry /etc/nexus-registry/tls
> # cosign
> curl -fSL https://github.com/sigstore/cosign/releases/download/v2.4.3/cosign-linux-amd64 -o /usr/local/bin/cosign
> chmod 0755 /usr/local/bin/cosign
> # Harbor 2.14.4 offline installer (images baked in; ~700 MB)
> curl -fSL https://github.com/goharbor/harbor/releases/download/v2.14.4/harbor-offline-installer-v2.14.4.tgz -o /tmp/harbor.tgz
> tar xzf /tmp/harbor.tgz -C /opt && rm /tmp/harbor.tgz   # -> /opt/harbor (install.sh, harbor.yml.tmpl, harbor.*.tar.gz)
> ```
> **VERIFY:** `docker --version`; `cosign version`; `test -s /opt/harbor/install.sh && ls /opt/harbor/harbor*.tar.gz`.

### 5.2 — nftables (all 4: backplane trust + ports + Docker forward-chain)

> **Step 5.2.1 — Apply the combined ruleset**
> **WHERE:** each of the 4 nodes, root shell.
> **WHY:** trust the VMnet10 backplane (PG + Redis replication); open Harbor `:443`
> + PG `:5432` + Redis `:6379` (from the 4 registry IPs) + VRRP on VMnet11. ⚠️ **The
> Docker forward-chain accepts** are essential on the app nodes — Harbor's
> intra-container + published-port traffic crosses the `forward` hook, which defaults
> to `drop`; accept `docker0` + the compose `br*` bridges or the stack can't talk to
> itself ([[feedback_nftables_flush_ruleset_wipes_docker]]). And **`nft -f` wipes
> Docker's iptables-nft chains** — restart docker after if it's already running.
> **WHAT (run on all 4 nodes — opening a port a node doesn't use is harmless):**
> ```bash
> cat > /etc/nftables.conf <<'EOF'
> #!/usr/sbin/nft -f
> flush ruleset
> table inet filter {
>     chain input {
>         type filter hook input priority 0; policy drop;
>         iif "lo" accept
>         ct state { established, related } accept
>         ct state invalid drop
>         ip protocol icmp   accept
>         ip6 nexthdr icmpv6 accept
>         ip protocol vrrp accept comment "keepalived VRRP unicast (registry-db datastore VIP)"
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 22   accept comment "SSH from VMnet11"
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 9100 accept comment "node_exporter from VMnet11"
>         iifname "nic1" ip saddr 192.168.10.0/24 accept comment "trusted datastore backplane (VMnet10) -- PG + Redis replication"
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 443 accept comment "Harbor HTTPS from VMnet11 (round-robin registry.nexus.lab)"
>         iifname "nic0" ip saddr { 192.168.70.115, 192.168.70.116, 192.168.70.117, 192.168.70.118 } tcp dport { 5432, 6379 } accept comment "PG + Redis from the 4 registry IPs (Harbor app -> datastore VIP; PG/Redis peers)"
>         counter drop
>     }
>     chain forward {
>         type filter hook forward priority 0; policy drop;
>         ct state { established, related } accept
>         iifname "docker0" accept comment "Docker default bridge (in)"
>         oifname "docker0" accept comment "Docker default bridge (out)"
>         iifname "br*" accept comment "Docker compose bridges (in) -- Harbor intra-container"
>         oifname "br*" accept comment "Docker compose bridges (out) -- Harbor published-port DNAT"
>     }
>     chain output  { type filter hook output priority 0; policy accept; }
> }
> EOF
> nft -f /etc/nftables.conf ; systemctl enable nftables 2>/dev/null || true
> if systemctl is-active --quiet docker; then systemctl restart docker; fi
> ```
> **VERIFY:** `nft list chain inet filter input | grep '192.168.10.0/24 accept'` (all 4);
> app nodes show the `docker0` + `br*` accepts in the `forward` chain.

### 5.3 — Per-node mTLS certs from Vault PKI

> **Step 5.3.1 — Create the `registry-server` PKI role (once)**
> **WHERE:** `vault-1`, root shell.
> **WHY:** one role issues all 4 leaves. App leaves add `registry.nexus.lab`
> (round-robin); PG leaves add `registry-db.nexus.lab` + the **VIP `.119`** so clients
> validate across failover. 90-day TTL.
> **WHAT:**
> ```bash
> vault write pki_int/roles/registry-server \
>   allowed_domains='nexus.lab,registry.nexus.lab,registry-db.nexus.lab,registry-1,registry-2,registry-pg-1,registry-pg-2,localhost' \
>   allow_subdomains=true allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
> ```
> **VERIFY:** `vault read pki_int/roles/registry-server`.

> **Step 5.3.2 — Issue + place the app certs (`registry-1`, then `registry-2`)**
> **WHERE:** issue on `vault-1`; place on each app node.
> **WHY:** Harbor's nginx serves HTTPS `:443` with `harbor.crt`/`harbor.key`; `ca.crt`
> is what Harbor trusts for MinIO S3. Owner `root:registry`. Also install the chain
> **world-readable** at `/etc/ssl/certs/registry-ca.pem` so `nexusadmin` can
> `--cacert` without sudo (T5).
> **WHAT (on vault-1 — for `registry-1`):**
> ```bash
> vault write -format=json pki_int/issue/registry-server \
>   common_name=registry-1.nexus.lab \
>   alt_names='registry-1,registry-1.nexus.lab,registry.nexus.lab,localhost' \
>   ip_sans='192.168.10.115,192.168.70.115,127.0.0.1' ttl=2160h > /tmp/r1.json
> vault read -field=certificate pki_int/cert/ca_chain > /tmp/nexus-ca-chain.pem
> ```
> **WHAT (place on `registry-1`, as root):**
> ```bash
> D=/etc/nexus-registry/tls
> jq -r '.data.certificate' /tmp/r1.json > /tmp/leaf.crt
> jq -r '.data.issuing_ca'  /tmp/r1.json > /tmp/int.crt
> jq -r '.data.private_key' /tmp/r1.json > /tmp/leaf.key
> cat /tmp/leaf.crt /tmp/int.crt > "$D/harbor.crt"
> openssl pkcs8 -topk8 -nocrypt -in /tmp/leaf.key -out "$D/harbor.key"
> cp /tmp/nexus-ca-chain.pem "$D/ca.crt"
> install -m 0644 -o root -g root /tmp/nexus-ca-chain.pem /etc/ssl/certs/registry-ca.pem
> chown root:registry "$D/harbor.crt" "$D/harbor.key" "$D/ca.crt"
> chmod 0644 "$D/harbor.crt" "$D/ca.crt" ; chmod 0640 "$D/harbor.key"
> rm -f /tmp/leaf.crt /tmp/int.crt /tmp/leaf.key /tmp/r1.json
> ```
> **Repeat for `registry-2`** — `common_name=registry-2.nexus.lab`,
> `alt_names='registry-2,registry-2.nexus.lab,registry.nexus.lab,localhost'`,
> `ip_sans='192.168.10.116,192.168.70.116,127.0.0.1'`, place on `.116`.
> **VERIFY:** `openssl x509 -in /etc/nexus-registry/tls/harbor.crt -noout -ext subjectAltName`
> → SAN has `registry.nexus.lab`.

> **Step 5.3.3 — Issue + place the PG certs (`registry-pg-1`, then `registry-pg-2`)**
> **WHERE:** issue on `vault-1`; place on each PG node.
> **WHY:** PG TLS layout = `server.crt`/`server.key`/`ca.crt`, **owner `postgres`**.
> ⚠️ **The key must be `0600`** — PG rejects a db-user-owned key that's group/world
> readable (it allows `0640` only when *root*-owned) (T1). SANs add
> `registry-db.nexus.lab` + the VIP `.119`.
> **WHAT (on vault-1 — for `registry-pg-1`):**
> ```bash
> vault write -format=json pki_int/issue/registry-server \
>   common_name=registry-pg-1.nexus.lab \
>   alt_names='registry-pg-1,registry-pg-1.nexus.lab,registry-db.nexus.lab,localhost' \
>   ip_sans='192.168.10.117,192.168.70.117,192.168.70.119,127.0.0.1' ttl=2160h > /tmp/pg1.json
> ```
> **WHAT (place on `registry-pg-1`, as root):**
> ```bash
> D=/etc/nexus-registry-pg/tls
> jq -r '.data.certificate' /tmp/pg1.json > /tmp/leaf.crt
> jq -r '.data.issuing_ca'  /tmp/pg1.json > /tmp/int.crt
> jq -r '.data.private_key' /tmp/pg1.json > /tmp/leaf.key
> cat /tmp/leaf.crt /tmp/int.crt > "$D/server.crt"
> openssl pkcs8 -topk8 -nocrypt -in /tmp/leaf.key -out "$D/server.key"
> cp /tmp/nexus-ca-chain.pem "$D/ca.crt"
> install -m 0644 -o root -g root /tmp/nexus-ca-chain.pem /etc/ssl/certs/registry-ca.pem
> chown -R postgres:postgres "$D"
> chmod 0644 "$D/server.crt" "$D/ca.crt" ; chmod 0600 "$D/server.key"     # 0600 -- T1
> rm -f /tmp/leaf.crt /tmp/int.crt /tmp/leaf.key /tmp/pg1.json
> ```
> **Repeat for `registry-pg-2`** — `common_name=registry-pg-2.nexus.lab`,
> `alt_names='registry-pg-2,registry-pg-2.nexus.lab,registry-db.nexus.lab,localhost'`,
> `ip_sans='192.168.10.118,192.168.70.118,192.168.70.119,127.0.0.1'`, place on `.118`.
> **VERIFY:** `openssl x509 -in /etc/nexus-registry-pg/tls/server.crt -noout -ext subjectAltName`
> → SAN has `registry-db.nexus.lab` + `192.168.70.119`; `stat -c '%a' …/server.key` → `600`.

### 5.4 — PostgreSQL + Redis HA datastore

> **Step 5.4.1 — Configure the PRIMARY (`registry-pg-1`): PG + Redis master**
> **WHERE:** `registry-pg-1` (`.117`), root shell with `VAULT_ADDR`/`VAULT_TOKEN` + CA.
> **WHY:** standard PostgreSQL streaming-replication setup (the same shape used by
> the catalog PG in Guide 17, but everything is spelled out here): a `conf.d`
> drop-in (`wal_level=replica` + SSL), a `pg_hba` block (replication over the
> backplane + harbor/admin over VMnet11 TLS), the `repluser` + `harbor` roles, and
> the **`registry`** database. Plus a **Redis master** (requirepass, bind `0.0.0.0`).
> **WHAT (PG primary — the full `conf.d`, `pg_hba`, roles + DB are all inlined here):**
> ```bash
> export VAULT_ADDR=https://192.168.70.121:8200 VAULT_CACERT=/etc/ssl/certs/registry-ca.pem
> SUPERPW=$(vault kv get -field=value nexus/registry/pg-superuser-password)
> REPLPW=$(vault kv get -field=value nexus/registry/pg-replication-password)
> HARBORPW=$(vault kv get -field=value nexus/registry/harbor-db-password)
> CONF=/etc/postgresql/17/main ; mkdir -p "$CONF/conf.d"
> cat > "$CONF/conf.d/nexus-registry.conf" <<'EOF'
> listen_addresses = '*'
> wal_level = replica
> max_wal_senders = 10
> max_replication_slots = 10
> hot_standby = on
> password_encryption = scram-sha-256
> ssl = on
> ssl_cert_file = '/etc/nexus-registry-pg/tls/server.crt'
> ssl_key_file = '/etc/nexus-registry-pg/tls/server.key'
> ssl_ca_file = '/etc/nexus-registry-pg/tls/ca.crt'
> EOF
> grep -q "include_dir = 'conf.d'" "$CONF/postgresql.conf" || echo "include_dir = 'conf.d'" >> "$CONF/postgresql.conf"
> grep -q 'NEXUS-REGISTRY-HBA' "$CONF/pg_hba.conf" || cat >> "$CONF/pg_hba.conf" <<'EOF'
> # NEXUS-REGISTRY-HBA
> host    replication   repluser   192.168.10.0/24   scram-sha-256
> hostssl registry      harbor     192.168.70.0/24   scram-sha-256
> hostssl all           postgres   192.168.70.0/24   scram-sha-256
> EOF
> pg_ctlcluster 17 main start || systemctl start postgresql@17-main ; systemctl enable postgresql@17-main
> for i in $(seq 1 30); do sudo -u postgres pg_isready -q && break; sleep 2; done
> sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD '$SUPERPW'"
> sudo -u postgres psql -c "CREATE ROLE repluser WITH REPLICATION LOGIN PASSWORD '$REPLPW'"
> sudo -u postgres psql -c "CREATE ROLE harbor   WITH LOGIN PASSWORD '$HARBORPW'"
> sudo -u postgres psql -c "CREATE DATABASE registry OWNER harbor"
> pg_ctlcluster 17 main reload || systemctl reload postgresql@17-main
> ```
> **WHAT (Redis master):**
> ```bash
> REDISPW=$(vault kv get -field=value nexus/registry/redis-password)
> echo "$REDISPW" > /etc/nexus-registry-pg/redis-password ; chmod 0640 /etc/nexus-registry-pg/redis-password ; chown root:postgres /etc/nexus-registry-pg/redis-password
> cat > /etc/redis/nexus-registry-redis.conf <<EOF
> bind 0.0.0.0 -::1
> protected-mode no
> port 6379
> requirepass $REDISPW
> masterauth $REDISPW
> appendonly no
> save ""
> EOF
> grep -q 'nexus-registry-redis.conf' /etc/redis/redis.conf || echo "include /etc/redis/nexus-registry-redis.conf" >> /etc/redis/redis.conf
> systemctl enable redis-server ; systemctl restart redis-server ; sleep 2
> redis-cli -a "$REDISPW" --no-auth-warning replicaof no one >/dev/null
> ```
> **VERIFY:** `sudo -u postgres psql -tAc "SELECT 1 FROM pg_database WHERE datname='registry'"` → `1`;
> `redis-cli -a "$REDISPW" --no-auth-warning info replication | grep role` → `role:master`.

> **Step 5.4.2 — Clone the REPLICA (`registry-pg-2`): PG standby + Redis replica**
> **WHERE:** `registry-pg-2` (`.118`), root shell.
> **WHY:** clone the primary into a hot standby — write the standby's `conf.d` (same
> as the primary's), a `.pgpass` so the walreceiver can authenticate (`pg_basebackup
> -R` does **not** embed the replication password in `primary_conninfo`), then
> `pg_basebackup -R` from the primary's **backplane** IP `.10.117`. Plus a **Redis
> replica** (`replicaof <primary-backplane> 6379`).
> **WHAT (Redis replica config, then the PG standby — all inlined):**
> ```bash
> export VAULT_ADDR=https://192.168.70.121:8200 VAULT_CACERT=/etc/ssl/certs/registry-ca.pem
> REDISPW=$(vault kv get -field=value nexus/registry/redis-password)
> echo "$REDISPW" > /etc/nexus-registry-pg/redis-password ; chmod 0640 /etc/nexus-registry-pg/redis-password ; chown root:postgres /etc/nexus-registry-pg/redis-password
> cat > /etc/redis/nexus-registry-redis.conf <<EOF
> bind 0.0.0.0 -::1
> protected-mode no
> port 6379
> requirepass $REDISPW
> masterauth $REDISPW
> appendonly no
> save ""
> replicaof 192.168.10.117 6379
> EOF
> grep -q 'nexus-registry-redis.conf' /etc/redis/redis.conf || echo "include /etc/redis/nexus-registry-redis.conf" >> /etc/redis/redis.conf
> systemctl enable redis-server ; systemctl restart redis-server
> # PG standby: write the same conf.d as the primary, a .pgpass for the walreceiver, then pg_basebackup
> REPLPW=$(vault kv get -field=value nexus/registry/pg-replication-password)
> CONF=/etc/postgresql/17/main ; mkdir -p "$CONF/conf.d"
> cat > "$CONF/conf.d/nexus-registry.conf" <<'EOF'
> listen_addresses = '*'
> wal_level = replica
> max_wal_senders = 10
> max_replication_slots = 10
> hot_standby = on
> password_encryption = scram-sha-256
> ssl = on
> ssl_cert_file = '/etc/nexus-registry-pg/tls/server.crt'
> ssl_key_file = '/etc/nexus-registry-pg/tls/server.key'
> ssl_ca_file = '/etc/nexus-registry-pg/tls/ca.crt'
> EOF
> grep -q "include_dir = 'conf.d'" "$CONF/postgresql.conf" || echo "include_dir = 'conf.d'" >> "$CONF/postgresql.conf"
> echo "192.168.10.117:5432:replication:repluser:$REPLPW" > /var/lib/postgresql/.pgpass
> chown postgres:postgres /var/lib/postgresql/.pgpass ; chmod 0600 /var/lib/postgresql/.pgpass
> pg_ctlcluster 17 main stop || systemctl stop postgresql@17-main
> rm -rf /var/lib/postgresql/17/main ; install -d -m 0700 -o postgres -g postgres /var/lib/postgresql/17/main
> sudo -u postgres env PGPASSWORD="$REPLPW" pg_basebackup -h 192.168.10.117 -p 5432 -U repluser -D /var/lib/postgresql/17/main -Fp -Xs -P -R
> pg_ctlcluster 17 main start || systemctl start postgresql@17-main ; systemctl enable postgresql@17-main
> ```
> **VERIFY (replica):** `psql -tAc 'SELECT pg_is_in_recovery()'` → `t`; `redis-cli … info replication | grep role` → `role:slave`.
> **(primary):** `psql -tAc "SELECT count(*) FROM pg_stat_replication"` → `1`.

> **Step 5.4.3 — keepalived VRRP VIP `.119` (dual PG + Redis promotion) on both PG nodes**
> **WHERE:** `registry-pg-1` (priority 110) + `registry-pg-2` (priority 100), root shell.
> **WHY:** the same keepalived pattern as Guide 17 §5.4.3 — **but the
> `notify_master` script promotes BOTH PG and Redis**, and a **`notify_backup`**
> script demotes the loser's Redis to a replica of the peer. `BACKUP` + `nopreempt`;
> the check execs the **versioned** `pg_isready` (T3 in Guide 17). VRRP
> `virtual_router_id 73`.
> **WHAT (the check + promote + demote scripts — identical on both nodes; set `PEER_BP` per node):**
> ```bash
> cat > /usr/local/sbin/nexus-registry-pg-check.sh <<'EOS'
> #!/bin/bash
> exec /usr/lib/postgresql/17/bin/pg_isready -q -h 127.0.0.1 -p 5432
> EOS
> chmod 0755 /usr/local/sbin/nexus-registry-pg-check.sh
> cat > /etc/keepalived/nexus-registry-promote.sh <<'EOS'
> #!/bin/bash
> # became VIP holder (MASTER): promote PG if standby + Redis to master
> if sudo -u postgres psql -tAc "SELECT pg_is_in_recovery()" 2>/dev/null | grep -qi t; then
>   /usr/bin/pg_ctlcluster 17 main promote
> fi
> RPW=$(sudo cat /etc/nexus-registry-pg/redis-password 2>/dev/null)
> /usr/bin/redis-cli -a "$RPW" --no-auth-warning replicaof no one >/dev/null 2>&1 || true
> EOS
> chmod 0755 /etc/keepalived/nexus-registry-promote.sh
> # demote: PEER_BP = the OTHER node's backplane IP (.10.118 on pg-1; .10.117 on pg-2)
> PEER_BP=192.168.10.118     # <-- on registry-pg-2 set this to 192.168.10.117
> cat > /etc/keepalived/nexus-registry-demote.sh <<EOS
> #!/bin/bash
> # lost the VIP (BACKUP): make Redis a replica of the new master (the peer)
> RPW=\$(sudo cat /etc/nexus-registry-pg/redis-password 2>/dev/null)
> /usr/bin/redis-cli -a "\$RPW" --no-auth-warning replicaof $PEER_BP 6379 >/dev/null 2>&1 || true
> EOS
> chmod 0755 /etc/keepalived/nexus-registry-demote.sh
> ```
> **WHAT (on `registry-pg-1` — `priority 110`, `unicast_src_ip .117`, peer `.118`):**
> ```bash
> cat > /etc/keepalived/keepalived.conf <<'EOF'
> global_defs { script_user root }
> vrrp_script chk_pg {
>   script "/usr/local/sbin/nexus-registry-pg-check.sh"
>   interval 5
>   fall 2
>   rise 2
> }
> vrrp_instance VI_REGISTRY_DB {
>   state BACKUP
>   nopreempt
>   interface nic0
>   virtual_router_id 73
>   priority 110
>   advert_int 1
>   unicast_src_ip 192.168.70.117
>   unicast_peer { 192.168.70.118 }
>   authentication { auth_type PASS ; auth_pass regdbvrp }
>   virtual_ipaddress { 192.168.70.119/24 dev nic0 }
>   notify_master "/etc/keepalived/nexus-registry-promote.sh"
>   notify_backup "/etc/keepalived/nexus-registry-demote.sh"
>   track_script { chk_pg }
> }
> EOF
> systemctl enable --now keepalived ; systemctl restart keepalived
> ```
> **WHAT (on `registry-pg-2` — identical but `priority 100`, `unicast_src_ip
> 192.168.70.118`, `unicast_peer { 192.168.70.117 }`, and `PEER_BP=192.168.10.117` in
> the demote script).**
> **EXPECTED:** the VIP binds on the primary (`.117`).
> **VERIFY:** `ip -4 -o addr show nic0 | grep 192.168.70.119` on **exactly one** PG node;
> `redis-cli -h registry-db.nexus.lab -a "$REDISPW" --no-auth-warning ping` → `PONG`.

### 5.5 — Harbor app tier (seed → join → OIDC)

> **Step 5.5.1 — Create the `harbor` bucket in MinIO**
> **WHERE:** `minio-1` (`.141`), root shell.
> **WHY:** the S3 blob backend. The MinIO app key (Guide 16) covers it.
> **WHAT:**
> ```bash
> export VAULT_ADDR=https://192.168.70.121:8200 VAULT_CACERT=/etc/ssl/certs/minio-ca.pem
> RU=$(vault kv get -field=value nexus/lakehouse/minio/root-user)
> RP=$(vault kv get -field=value nexus/lakehouse/minio/root-password)
> sudo mc alias set nexuslocal https://localhost:9000 "$RU" "$RP"
> sudo mc mb --ignore-existing nexuslocal/harbor
> ```
> **VERIFY:** `sudo mc ls nexuslocal | grep harbor`.

> **Step 5.5.2 — Render `harbor.yml` + the shared secretkey, install (SEED: `registry-1`)**
> **WHERE:** `registry-1` (`.115`), root shell with `VAULT_ADDR`/`VAULT_TOKEN` + CA.
> **WHY:** render Harbor's config from KV — HTTPS with the Vault cert, **external** DB
> (→ the VIP), **external** Redis (→ the VIP), **S3** storage (→ MinIO), Trivy. ⚠️
> The `harbor.yml` must carry the **full** field set Harbor 2.14's `prepare`
> templates require (full `jobservice` incl. `job_loggers`, `notification`, `trivy`,
> `upload_purging`, `cache`, `proxy`) and **`_version: 2.14.0`** — the X.Y.**0**
> config-schema version, **not** the installer patch `2.14.4` (T3). Write the shared
> HA `secretkey` to `/data/secret/keys/secretkey` (registry-2 must get the *same* one).
> **WHAT:**
> ```bash
> export VAULT_ADDR=https://192.168.70.121:8200 VAULT_CACERT=/etc/ssl/certs/registry-ca.pem
> ADMINPW=$(vault kv get -field=value nexus/registry/harbor-admin)
> SECRETKEY=$(vault kv get -field=value nexus/registry/harbor-secret-key)
> HARBORDBPW=$(vault kv get -field=value nexus/registry/harbor-db-password)
> REDISPW=$(vault kv get -field=value nexus/registry/redis-password)
> S3AK=$(vault kv get -field=value nexus/lakehouse/minio/app-access-key)
> S3SK=$(vault kv get -field=value nexus/lakehouse/minio/app-secret-key)
> systemctl enable --now docker
> mkdir -p /data/secret/keys /var/log/harbor
> printf '%s' "$SECRETKEY" > /data/secret/keys/secretkey ; chmod 0600 /data/secret/keys/secretkey
> cat > /opt/harbor/harbor.yml <<EOF
> hostname: registry.nexus.lab
> external_url: https://registry.nexus.lab
> https:
>   port: 443
>   certificate: /etc/nexus-registry/tls/harbor.crt
>   private_key: /etc/nexus-registry/tls/harbor.key
> harbor_admin_password: $ADMINPW
> data_volume: /data
> storage_service:
>   ca_bundle: /etc/nexus-registry/tls/ca.crt
>   s3:
>     accesskey: $S3AK
>     secretkey: $S3SK
>     region: us-east-1
>     regionendpoint: https://minio.nexus.lab:9000
>     bucket: harbor
>     secure: true
>     v4auth: true
>     forcepathstyle: true
> external_database:
>   harbor:
>     host: registry-db.nexus.lab
>     port: 5432
>     db_name: registry
>     username: harbor
>     password: $HARBORDBPW
>     ssl_mode: require
>     max_idle_conns: 50
>     max_open_conns: 100
> external_redis:
>   host: registry-db.nexus.lab:6379
>   password: $REDISPW
>   registry_db_index: 1
>   jobservice_db_index: 2
>   trivy_db_index: 5
>   idle_timeout_seconds: 30
> trivy:
>   ignore_unfixed: false
>   skip_update: false
>   skip_java_db_update: false
>   offline_scan: false
>   security_check: vuln
>   insecure: false
>   timeout: 5m0s
> jobservice:
>   max_job_workers: 10
>   max_job_duration_hours: 24
>   job_loggers:
>     - STD_OUTPUT
>     - FILE
>   logger_sweeper_duration: 1
> notification:
>   webhook_job_max_retry: 3
>   webhook_job_http_client_timeout: 3
> log:
>   level: info
>   local:
>     rotate_count: 50
>     rotate_size: 200M
>     location: /var/log/harbor
> upload_purging:
>   enabled: true
>   age: 168h
>   interval: 24h
>   dryrun: false
> cache:
>   enabled: false
>   expire_hours: 24
> proxy:
>   http_proxy:
>   https_proxy:
>   no_proxy:
>   components:
>     - core
>     - jobservice
>     - trivy
> _version: 2.14.0
> EOF
> cd /opt/harbor && ./install.sh --with-trivy 2>&1 | tail -25
> ```
> **EXPECTED:** `install.sh` finishes; the 8 containers come up.
> **VERIFY:** `curl -fsS -k https://localhost/api/v2.0/health | grep -o '"status":"healthy"'`
> (the node's own FQDN resolves to `127.0.1.1`, so probe **localhost -k** — T4); wait
> up to ~6 min for all components healthy.

> **Step 5.5.3 — Copy the HA secret seed → `registry-2`, render + install (JOIN)**
> **WHERE:** `registry-1` (capture) → `registry-2` (`.116`) (render/install).
> **WHY:** HA requires **identical** `/data/secret` across app nodes (the secretkey +
> CSRF + registry http.secret + jobservice secret) — else a token minted by one is
> rejected by the other. So: capture registry-1's `/data/secret`, restore it on
> registry-2, then install registry-2 with the **same** `harbor.yml`. ✅ **registry-2's
> `harbor.yml` is byte-for-byte identical to registry-1's** — there are *no*
> node-specific values in it (it uses the round-robin `hostname: registry.nexus.lab`
> and the external DB/Redis/S3 endpoints, which are the same from either node). So you
> literally re-run the §5.5.2 render block on registry-2, unchanged.
> **WHAT — do these four things in order:**
> ```bash
> # 1. On registry-1: capture /data/secret (the shared HA secrets) to the build host.
> ssh nexusadmin@192.168.70.115 'sudo tar czf - -C /data secret | base64 -w0' > /tmp/secret.b64
>
> # 2. On registry-2: start docker + create /data.
> ssh nexusadmin@192.168.70.116 'sudo systemctl enable --now docker; sudo mkdir -p /data'
>
> # 3. On registry-2: restore the captured /data/secret.
> cat /tmp/secret.b64 | ssh nexusadmin@192.168.70.116 'base64 -d | sudo tar xzf - -C /data'
> ```
> ```bash
> # 4. SSH to registry-2 (192.168.70.116), become root, and run the ENTIRE §5.5.2
> #    render block verbatim (read the KV creds, write the identical /opt/harbor/harbor.yml,
> #    write /data/secret/keys/secretkey), then install:
> cd /opt/harbor && ./install.sh --with-trivy 2>&1 | tail -25
> ```
> **EXPECTED:** registry-2's stack comes up with the shared secrets.
> **VERIFY (on registry-2):** `curl -fsS -k https://localhost/api/v2.0/health | grep -o '"status":"healthy"'` → `healthy`.

> **Step 5.5.4 — Configure Vault OIDC SSO via the Harbor API**
> **WHERE:** `registry-1` (against the front door), root shell.
> **WHY:** flip Harbor to `auth_mode=oidc_auth` federated to Vault's
> `nexus-registry` OIDC provider; the local admin stays as break-glass. Use the
> world-readable CA for `--cacert` (T5).
> **WHAT:**
> ```bash
> export VAULT_ADDR=https://192.168.70.121:8200 VAULT_CACERT=/etc/ssl/certs/registry-ca.pem
> ADMINPW=$(vault kv get -field=value nexus/registry/harbor-admin)
> OIDCID=$(vault kv get -field=value nexus/registry/oidc-client-id)
> OIDCSEC=$(vault kv get -field=value nexus/registry/oidc-client-secret)
> curl -s -o /dev/null -w '%{http_code}\n' --cacert /etc/ssl/certs/registry-ca.pem \
>   -u "admin:$ADMINPW" -X PUT "https://registry.nexus.lab/api/v2.0/configurations" \
>   -H 'Content-Type: application/json' -d "{
>     \"auth_mode\": \"oidc_auth\",
>     \"oidc_name\": \"nexus-vault\",
>     \"oidc_endpoint\": \"https://vault-1.nexus.lab:8200/v1/identity/oidc/provider/nexus-registry\",
>     \"oidc_client_id\": \"$OIDCID\",
>     \"oidc_client_secret\": \"$OIDCSEC\",
>     \"oidc_scope\": \"openid,profile,email,groups\",
>     \"oidc_groups_claim\": \"groups\",
>     \"oidc_user_claim\": \"preferred_username\",
>     \"oidc_auto_onboard\": true,
>     \"oidc_verify_cert\": false }"
> ```
> **EXPECTED:** `200`.
> **VERIFY:** `curl -s --cacert /etc/ssl/certs/registry-ca.pem -u "admin:$ADMINPW" https://registry.nexus.lab/api/v2.0/configurations | grep -o '"auth_mode":[^,]*'`
> → contains `oidc_auth`.

### 5.6 — Cluster bootstrap (the exit gate: push + S3 blob + Trivy + cosign)

> **Step 5.6.1 — Push an image, scan it, sign it**
> **WHERE:** `registry-1` (`.115`) — it has docker + cosign + NAT egress.
> **WHY:** the deterministic exit gate proving the core deliverable end-to-end: push
> an image (blob lands in MinIO `s3://harbor`), Trivy scans it to `Success`, cosign
> signs + verifies it. ⚠️ Resolve the digest from **Harbor's API**, not `docker
> inspect` — Harbor re-stores the manifest (e.g. picking the amd64 entry from a
> multi-arch list), so its digest differs from the upstream one and the scan/cosign
> would 404 (T6).
> **WHAT:**
> ```bash
> export VAULT_ADDR=https://192.168.70.121:8200 VAULT_CACERT=/etc/ssl/certs/registry-ca.pem
> ADMINPW=$(vault kv get -field=value nexus/registry/harbor-admin)
> REG=registry.nexus.lab ; IMG="$REG/library/smoke:v1"
> # trust the registry CA for the docker daemon
> install -d -m 0755 /etc/docker/certs.d/$REG
> cp /etc/nexus-registry/tls/ca.crt /etc/docker/certs.d/$REG/ca.crt
> # push
> docker pull busybox:latest ; docker tag busybox:latest "$IMG"
> echo "$ADMINPW" | docker login "$REG" -u admin --password-stdin
> docker push "$IMG"
> # Harbor's AUTHORITATIVE digest (T6)
> DIG=$(curl -s --cacert /etc/ssl/certs/registry-ca.pem -u "admin:$ADMINPW" \
>   "https://$REG/api/v2.0/projects/library/repositories/smoke/artifacts" | grep -o '"digest":"sha256:[a-f0-9]*"' | head -1 | cut -d'"' -f4)
> # Trivy scan -> await Success
> curl -s -o /dev/null --cacert /etc/ssl/certs/registry-ca.pem -u "admin:$ADMINPW" \
>   -X POST "https://$REG/api/v2.0/projects/library/repositories/smoke/artifacts/$DIG/scan"
> for i in $(seq 1 30); do
>   S=$(curl -s --cacert /etc/ssl/certs/registry-ca.pem -u "admin:$ADMINPW" \
>     "https://$REG/api/v2.0/projects/library/repositories/smoke/artifacts/$DIG?with_scan_overview=true" | grep -o '"scan_status":"[^"]*"' | head -1)
>   echo "$S" | grep -q Success && { echo "scan=$S"; break; }; sleep 6
> done
> # cosign sign + verify (key-based)
> export COSIGN_PASSWORD=nexus-cosign-smoke ; WD=$(mktemp -d); cd "$WD"
> cosign generate-key-pair
> echo "$ADMINPW" | cosign login "$REG" -u admin --password-stdin
> cosign sign   --tlog-upload=false        --key cosign.key --yes "$REG/library/smoke@$DIG"
> cosign verify --insecure-ignore-tlog=true --key cosign.pub      "$REG/library/smoke@$DIG"
> cd / ; rm -rf "$WD" ; docker logout "$REG"
> ```
> **EXPECTED:** push succeeds; `scan_status` reaches `Success`; `cosign verify` passes.
> **VERIFY:** on `minio-1`, `sudo mc ls --recursive nexuslocal/harbor | wc -l` ≥ 1
> (blobs on the S3 backend). **➡ The HA registry + supply-chain path is proven.**

---

## 6. Validation — by-hand acceptance smoke (demo / playbook)

Condensed from `smoke-0.L.4.ps1`. Per-node SSH probes from the **build host**.

- **Input:** the 4 nodes up; datastore HA streaming; both Harbor healthy; exit gate green.
- **Where observed:** SSH to each node / `psql`+`redis-cli` on PG nodes / `curl` on
  app nodes / `mc` on `minio-1` / `dig` on the gateway.
- **Proves:** an HA Harbor registry with external HA PG/Redis, S3 blobs, scanning,
  signing, and Vault SSO.
- **Prerequisites:** Guides 00–04 + 16 (powered on) alive; §5 complete.

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | 4 nodes reachable | `ssh …@115/116/117/118 'echo ok'` | all `ok` |
| 2 | Identity | `grep NEXUS_ROLE / NEXUS_PG_ROLE …node-identity.env` | app `registry-harbor`; pg-1 `primary`, pg-2 `replica` |
| 3 | nftables backplane + Docker fwd | `nft list ruleset` | backplane trust; app nodes have `docker0`/`br*` forward accepts |
| 4 | mTLS SANs | `openssl x509 -in <crt> -noout -ext subjectAltName` | app `registry.nexus.lab`; PG `registry-db.nexus.lab` + `.119`; PG key `0600` |
| 5 | PG replication | `SELECT count(*) FROM pg_stat_replication` (pg-1) | `1` streaming |
| 6 | Replica recovery + DB | `pg_is_in_recovery()` (pg-2); `registry` DB exists | `t`; `1` |
| 7 | VRRP VIP | `ip addr show nic0 \| grep .119` (each PG) | on **exactly one** node |
| 8 | Redis HA + VIP | `info replication` (master/slave); `redis-cli -h registry-db… ping` | `master`/`slave`; `PONG` |
| 9 | Harbor API health | `curl -k https://localhost/api/v2.0/health` (both app) | `healthy` |
| 10 | DNS | `dig registry.nexus.lab +short` / `dig registry-db.nexus.lab +short` | 2 app IPs / VIP `.119` |
| 11 | MinIO blobs | `mc ls --recursive nexuslocal/harbor \| wc -l` | `>= 1` |
| 12 | Pushed image + Trivy | `…/artifacts?with_scan_overview=true` | `library/smoke` present; `scan_status":"Success"` |
| 13 | cosign signature | `…/artifacts?with_accessory=true` | signature accessory present |
| 14 | Vault OIDC SSO | `…/configurations` | `auth_mode` == `oidc_auth` |
| 15 | **App-node loss** (chaos) | stop registry-2's stack; pull from registry-1 | pull succeeds (shared state) |

**1–14 green ⇒ Guide 19 satisfied.** 15 is the HA payoff (a node can die; the other
serves the shared state). **This completes the Registry/platform tier (0.L.4).**

---

## 7. Teardown / reset

```bash
for ip in 115 116; do ssh nexusadmin@192.168.70.$ip 'cd /opt/harbor && sudo docker compose down; sudo rm -rf /data/secret /data/database /opt/harbor/harbor.yml'; done
for ip in 117 118; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now keepalived redis-server postgresql@17-main; sudo rm -f /etc/keepalived/keepalived.conf /etc/redis/nexus-registry-redis.conf'; done
# gateway DNS records (registry.nexus.lab + registry-db.nexus.lab) belong to Guide 01.
# then vmrun stop + deleteVM each of the 4 (Guide 00 §7).
```

> **The image blobs persist in MinIO `s3://harbor`; the metadata in PG.** Tearing
> down the app tier removes only the compute; a fresh Harbor pointed at the same
> bucket + a restored DB re-reads the images. The Vault OIDC client + the
> `nexus/registry/*` KV creds survive (sticky). To wipe the registry data: empty the
> `harbor` bucket *and* drop the `registry` DB.

---

## 8. Troubleshooting

| # | Symptom | Cause | Fix |
|---|---|---|---|
| **T1** | `postgresql@17-main` won't start: `private key file "…/server.key" has group or world access` | the PG key was installed `0640` owned by `postgres`; PG accepts a db-user-owned key only at `0600` | install `server.key` **`0600`** (root reads it fine for Harbor too) (§5.3.3). |
| **T2** | A `bash -s`-piped setup script silently stops early (Redis stays `inactive`, no journal) | an **input-less** `tee`/stdin-reader in the script consumes the rest of the piped script as its stdin | never leave a stdin-reading command without redirected input in a piped script; here every `tee` has a heredoc/redirect. |
| **T3** | `./install.sh` fails: Harbor `prepare` → `KeyError: 'job_loggers'` (or similar) | the hand-written `harbor.yml` omitted fields Harbor 2.14's `prepare` templates require, or used the wrong `_version` | render the **full** field set (full `jobservice` incl. `job_loggers`, `notification`, `trivy`, `upload_purging`, `cache`, `proxy`) and `_version: 2.14.0` (the X.Y.**0** schema, NOT the patch `2.14.4`) (§5.5.2). |
| **T4** | "Harbor API not healthy" though all containers are `(healthy)` | the node's own FQDN `registry-N.nexus.lab` resolves to `127.0.1.1` (firstboot `/etc/hosts`) | per-node liveness via `https://localhost/... -k`; the front door `registry.nexus.lab` round-robins to the real IPs (§5.5.2). |
| **T5** | `curl --cacert /etc/nexus-registry/tls/ca.crt` → `HTTP=000 error setting certificate file` | that cert dir is `0750 root:<group>`; `nexusadmin` can't read the CA without sudo | use the **world-readable** `/etc/ssl/certs/registry-ca.pem` for all `nexusadmin --cacert` calls (§5.3). |
| **T6** | Trivy/cosign 404: `artifact library/smoke@sha256:… not found` | `docker inspect .RepoDigests` returned the **upstream** busybox digest, not Harbor's re-stored manifest digest | resolve the digest from **Harbor's API** (`…/repositories/smoke/artifacts` → `.digest`) (§5.6.1). |
| **T7** | (Vault side) the OIDC provider setup intermittently errors on first run | a first-run MemDB read-after-write race on a fresh Vault identity store | the writes are idempotent — re-run §5.0.2; gate on the discovery endpoint serving the issuer. |
| **—** | Harbor stack can't reach itself / published ports dead | the `forward` chain drops Docker bridge traffic, or `nft -f` wiped Docker's chains | add the `docker0`/`br*` forward accepts (§5.2.1) and `systemctl restart docker` after any `nft -f` ([[feedback_nftables_flush_ruleset_wipes_docker]]). |
| **—** | Backplane down (replication can't connect) | VMware left `ethernet1`/nic1 NO-CARRIER at power-on | reconnect the 2nd NIC + `systemctl restart systemd-networkd`; confirm `ip addr show nic1` has `192.168.10.11X` (as Guides 16–18). |
| **—** | After a datastore failover, old primary won't rejoin | it diverged from the new timeline; Redis still thinks it's master | re-seed it as the new standby (`pg_basebackup -R` from the new primary) + point its Redis `replicaof` at the new master (handbook §3.3 manual runbook). |

---

## 9. Production tuning — Harbor registry

> **Everything below is *beyond the lab replica*.** The lab renders the §5.5.2 `harbor.yml`
> verbatim at lab-scale on 2 GB VMs — `max_job_workers: 10`, `cache.enabled: false`, and a
> single 2-node bundled PG + single bundled Redis at default sizing. This section is what you
> would change for a **production** registry carrying real CI push/pull + scan traffic, and
> *why*. **Do not paste these onto the 2 GB lab VMs blindly.** Frame it as two layers: the
> registry's **internal datastore** — size it exactly like the standalone engines (so link
> out, don't duplicate) — plus these **Harbor-app knobs** the standalone engines don't have.

### 9.1 The bundled datastore — size it like the standalone engines

Harbor is stateless; every durable byte lives in its **bundled PostgreSQL 17** (the
`registry-db` pair — `external_database` in §5.5.2) and its **bundled Redis** (the
`external_redis` block, running in Harbor's **CACHE** role: session state, job queues,
registry metadata cache). These are the *same* engines Guides 10 and 07 cover, so tune them
**there** — this guide does not restate their knobs:

- **Bundled PostgreSQL** (`registry` DB) → tune per **[Guide 10 §9](./10-oltp-patroni-postgresql-ha.md)**:
  `shared_buffers` (≈25 % of RAM), `effective_cache_size` (≈50–75 % of RAM), `work_mem`,
  checkpoint/WAL, plus the Guide 00 §9 OS layer. The lab leaves PG at defaults (128 MB
  `shared_buffers`); a busy registry's metadata queries want it sized like any OLTP node.
  Also raise `external_database.max_open_conns` (lab `100`) in step with PG `max_connections`.
- **Bundled Redis** (CACHE role) → tune per **[Guide 07 §9](./07-oltp-redis-cluster.md)**:
  set a hard **`maxmemory`** (≈50–60 % of the node's RAM) and — because this Redis is a
  cache, not a data-of-record store — **`maxmemory-policy allkeys-lru`** so it evicts cold
  keys instead of OOM-ing when the working set exceeds RAM. The lab sets neither (unbounded,
  `noeviction`), which is fine at lab volume but a memory-exhaustion risk under real load.

The rest of §9 is the layer the standalone engines have no equivalent for — Harbor's own
application knobs, all in `harbor.yml` (re-render + `./install.sh` to apply).

### 9.2 Harbor application knobs (`harbor.yml`)

| Setting | Production value | Lab value (§5.5.2) | Why it matters |
|---|---|---|---|
| `jobservice.max_job_workers` | `10`–`50`, sized to registry-node cores × ~2 | `10` | Concurrency ceiling for **all** async jobs — replication, GC, retention, and Trivy scan dispatch share this pool. Too low serialises scans behind a GC; too high starves the DB/Redis it drives. Scale with core count and PG `max_open_conns`. |
| **Garbage collection** schedule (UI/API: *Administration → Clean Up → GC*, or `PUT /api/v2.0/system/gc/schedule`) | Off-peak cron, e.g. `0 0 3 * * *` daily, ✅ *delete untagged* | none (manual only) | Blob GC is what actually reclaims MinIO/S3 space after tags are removed; without a schedule the `s3://harbor` bucket grows forever. GC takes a brief **read-only** window on the registry — run it off-peak. |
| **Tag-retention policy** (per-project: *Project → Policy → Tag Retention*) | e.g. keep last **10** pulled + last **30 days**, per repo | none (unbounded) | Bounds how many artifact versions each repo keeps so GC has something to collect; without it every CI push accumulates forever. Retention only *marks* — GC (above) frees the blobs. |
| `cache.enabled` (registry metadata cache) | `true`, `expire_hours: 24` | `false` | Caches manifest/metadata lookups in the bundled Redis so hot pulls skip a PG round-trip — materially cuts pull latency and DB load on a busy registry. Off in the lab to keep the Redis role minimal. |
| `storage_service.s3` (registry blob backend) | keep `region`/`regionendpoint` local; add a CDN/pull-through cache for geo-distributed pulls | S3 → MinIO `s3://harbor`, `v4auth`, `forcepathstyle` | Blobs already live in object storage (correct); at scale the tuning is on the *object store* side (MinIO erasure/throughput) + a front cache, not Harbor. |
| **Trivy scanner** — `trivy.skip_update: false`, adapter `SCANNER_TRIVY_VULN_TYPE`, and scan concurrency (bounded by `max_job_workers`) | Pre-warm/mirror the Trivy DB internally; keep `skip_update:false` but pin a mirror; raise workers so scans don't queue | `skip_update:false`, `offline_scan:false`, `timeout:5m0s` | Every scan is a jobservice job, so **scan concurrency = `max_job_workers`** minus whatever GC/replication is using. A slow/rate-limited Trivy DB pull (GitHub) stalls scans; production mirrors the DB and gives scans headroom. |
| `upload_purging` (`enabled`/`age`/`interval`) | `enabled: true`, `age: 168h`, `interval: 24h` | **PRESENT** — `enabled:true, age:168h, interval:24h` | Purges orphaned partial-upload blobs (failed/aborted pushes) so they don't leak storage. The lab already ships the recommended values — noted here so it's not mistaken for a gap. |
| `external_database.max_idle_conns` / `max_open_conns` | raise with PG capacity (e.g. `100`/`300`) | `50` / `100` | Harbor's PG connection pool; must track PG `max_connections` and `jobservice.max_job_workers` or jobs block on connection acquisition under load. |

```yaml
# PRODUCTION harbor.yml deltas — re-render on BOTH registry nodes (byte-identical, §5.5.2),
# then `cd /opt/harbor && ./install.sh --with-trivy`. Not applied in the lab.
jobservice:
  max_job_workers: 25
cache:
  enabled: true          # lab: false — turn the bundled-Redis metadata cache on
  expire_hours: 24
external_database:
  max_idle_conns: 100    # lab: 50
  max_open_conns: 300    # lab: 100 — keep <= PG max_connections
# GC + tag-retention are runtime policy, not harbor.yml — set them via the API/UI:
#   curl -u admin:$PW -X PUT https://registry.nexus.lab/api/v2.0/system/gc/schedule \
#     -d '{"schedule":{"type":"Daily","cron":"0 0 3 * * *"}}'
```

> **OS layer:** the registry + `registry-db` nodes are Debian hosts — apply
> **[Guide 00 §9](./00-lab-host-and-base-vm.md)** (`nofile`, swappiness, THP-off for the
> bundled Redis) exactly as for any data node before the engine knobs above.

---

### Cross-references

- **0.L.4 architecture:** memory `project_nexus_infra_lakehouse_phase` (0.L.4 row);
  ADR-0036 (Harbor registry HA); the `nexus-infra-registry` handbook §3.2 (the 7 transients)
- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (registry `.115/.116`,
  PG `.117/.118`, VIP `.119`, MAC `:A4/:AF/:B0/:B1`); `vms.yaml` (tier `09-platform`)
- **Automated equivalents:** `nexus-infra-registry/packer/registry-{harbor,pg}-node/`
  + `terraform/envs/registry-harbor/role-overlay-{registry-pg-replication,harbor-config,registry-cluster-bootstrap,registry-tls}.tf`
- **Smoke mirror:** `nexus-infra-registry/scripts/smoke-0.L.4.ps1`
- **Depends on:** [`16-lakehouse-minio.md`](./16-lakehouse-minio.md) (the `s3://harbor` blob backend) + the Vault OIDC provider (Guide 04 scaffolding)
- **Related PG HA pattern:** [`17-lakehouse-iceberg-nessie.md`](./17-lakehouse-iceberg-nessie.md) §5.4 (the streaming-replication + keepalived-VIP base this extends with Redis)
- **Transients:** [[feedback_nftables_flush_ruleset_wipes_docker]] · [[feedback_cluster_template_nftables_backplane]] · [[feedback_nftables_runtime_add_after_drop]]
- **Previous guide:** [`18-lakehouse-spark-ha.md`](./18-lakehouse-spark-ha.md)
- **Next guide:** Guide 20 — Observability · Grafana LGTM (Prometheus + Loki + Tempo + Grafana + OTel, VRRP VIP). See [`INDEX.md`](../INDEX.md). **(Registry tier 0.L.4 complete.)**
