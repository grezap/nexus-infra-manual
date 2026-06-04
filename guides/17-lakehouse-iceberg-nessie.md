# Guide 17 — Lakehouse · Iceberg / Nessie (REST catalog + dedicated PG HA)

> **Mirrors:** `nexus-infra-lakehouse` — the `lakehouse-iceberg-rest-node` +
> `lakehouse-iceberg-pg-node` Packer templates + the `lakehouse-iceberg` Terraform
> env overlays (`…-nftables-backplane`, `…-vault-agents`, `…-tls`,
> `…-iceberg-pg-replication`, `…-nessie-config`, `…-catalog-bootstrap`) — Phase
> 0.L.2 / ADR-0034. The **second lakehouse-tier guide**: the metadata catalog that
> turns MinIO's raw buckets into versioned **Apache Iceberg** tables.

> 🔗 **Depends on Guide 16 (MinIO).** Nessie's warehouse is `s3://warehouse` on
> MinIO; table data + metadata land there. Build Guide 16 first. **Produces for
> Guide 18 (Spark)** — Spark reads/writes Iceberg tables *through* this REST
> catalog. (The definitive table→S3 write is proven by the Spark gate; this guide's
> exit gate proves the catalog is reachable + PG-backed.)

---

## 1. Overview & purpose

The **Iceberg REST catalog** — the layer that gives MinIO's object store *table*
semantics (schemas, snapshots, branches, time-travel) instead of just files.
**4 nodes, two roles:**

- **Catalog server (`iceberg-rest-1/2`, `.147/.148`)** — two HA instances of
  **Project Nessie**, a Quarkus app serving the **Iceberg REST API** (`/iceberg/v1/…`)
  *and* Nessie's native Git-like versioning API (`/api/v2/…`) over HTTPS `:19120`.
  Stateless app tier, fronted by round-robin DNS `iceberg.nexus.lab`.
- **Catalog metadata DB (`iceberg-pg-1/2`, `.149/.150`)** — a **dedicated**
  PostgreSQL 17 **master-replica HA pair** (streaming replication + a keepalived
  **VRRP VIP** `iceberg-db.nexus.lab` `.151`). This is where Nessie *persists* the
  catalog (the table pointers, branches, commits). Separate from every other PG in
  the platform (the Patroni OLTP cluster, Harbor's PG) — the catalog gets its own.
- **Warehouse** = `s3://warehouse` on **MinIO** (Guide 16). Nessie writes Iceberg
  **table data + metadata files** to the bucket; the **PG** holds only the catalog
  *pointers* to those files.

**Why this split:** Iceberg separates **data** (Parquet + metadata JSON in object
storage) from the **catalog** (which snapshot is current, what branches exist).
Nessie is the catalog server; PG is its durable store; MinIO is the warehouse. Two
Nessie instances + a PG HA pair means **no single point of failure** in the
catalog path. **Why it matters:** every lake engine downstream (Spark, Trino,
StarRocks external catalogs) talks to *this* REST endpoint to discover + commit
tables.

---

## 2. Component primer

- **Apache Iceberg.** A table format over object storage — files in S3 + a metadata
  tree giving ACID snapshots, schema evolution, hidden partitioning, time-travel.
  *Why:* turns a bucket of Parquet into a real warehouse table. *Otherwise:* Delta
  Lake or Apache Hudi (same niche; Iceberg has the broadest engine support).
- **Project Nessie.** An Iceberg **catalog** server with Git-like semantics
  (branches, tags, commits across tables). Speaks the **Iceberg REST catalog**
  protocol *and* its own API. A Quarkus (JVM) app. *Why:* a transactional,
  multi-table catalog with versioning, not just a pointer store. *Otherwise:* the
  Hive Metastore (legacy, Thrift), AWS Glue (cloud), the standalone
  `iceberg-rest-fixture`, or JDBC catalog. Nessie adds branching the others lack.
- **Iceberg REST catalog protocol.** The vendor-neutral HTTP API
  (`/iceberg/v1/config`, `/namespaces`, `/tables`) every modern engine speaks. Nessie
  exposes it at `/iceberg/` — so Spark/Trino/Flink all point at one URL. *Why:* one
  catalog, many engines, no per-engine driver.
- **Nessie version store = JDBC (PostgreSQL).** Nessie persists its commits to a
  relational DB. We use `JDBC2` → the bundled **named `postgresql` datasource**
  (which must be *activated* — see §8 T1) → the catalog PG. *Otherwise:* RocksDB
  (single-node, no HA), Cassandra/DynamoDB/MongoDB (heavier). PG gives us a simple
  HA story with tools we already run.
- **PostgreSQL streaming replication.** The primary ships WAL to a hot-standby
  replica in real time (`pg_basebackup -R` to clone, then continuous streaming).
  *Why:* a warm copy of the catalog DB ready to take over. *vs. Patroni (Guide 10):*
  this is **plain** streaming replication + keepalived, not Patroni/etcd — the
  catalog DB is small + low-churn, so a 2-node pair with a VIP is enough.
  *Otherwise:* Patroni (overkill here), logical replication (wrong tool).
- **keepalived / VRRP VIP.** A floating IP (`.151`) that lives on whichever PG node
  is primary; Nessie always connects to the VIP, so a failover is transparent to it.
  `notify_master` runs a promote script on the node that wins the VIP. State
  **`BACKUP` + `nopreempt`** so a recovered old-primary doesn't yank the VIP back
  and split-brain. *Otherwise:* HAProxy with a PG health check (Guide 10's pattern;
  heavier for a 2-node pair). *vs. PXC's keepalived (Guide 9):* same tool, but here
  it also **promotes** the standby, not just moves the address.

---

## 3. Prerequisites

| # | Requirement | One-command verify |
|---|---|---|
| 1 | **Guide 16 (MinIO) built** — `warehouse` bucket + app creds in Vault KV; `minio.nexus.lab:9000` TLS-reachable | `mc ls nexuslocal/warehouse` on `minio-1`; `vault kv get nexus/lakehouse/minio/app-access-key` |
| 2 | **Foundation alive** (Guides 00–04) — Vault PKI + KV; gateway DNS | `vault status` on `vault-1` → `Sealed: false` |
| 3 | **CA bundle** on the build host (`~/.nexus/vault-ca-bundle.crt`) | `Test-Path ~/.nexus/vault-ca-bundle.crt` → `True` |
| 4 | **4 `deb13` nodes** baselined (Guide 00), dual-NIC static `.147–.150` / `.10.147–.150` | the 4 answer `:22`; firstboot mapped roles (`NEXUS_ROLE`) |
| 5 | **Nessie JAR + JDK 21** reachable (GitHub release, or a local cache) | `curl -sI https://github.com/projectnessie/nessie/releases/download/nessie-0.107.5/nessie-quarkus-0.107.5-runner.jar \| head -1` |

> **Versions:** Project **Nessie 0.107.5** (Quarkus runner JAR, **JDK 21**),
> **PostgreSQL 17** (PGDG). REST front door: round-robin DNS `iceberg.nexus.lab` →
> `.147/.148`, HTTPS `:19120`. Catalog DB front door: VRRP VIP
> `iceberg-db.nexus.lab` → `.151`. Warehouse: `s3://warehouse` at
> `https://minio.nexus.lab:9000`.

> **By-hand divergence:** the automated path uses a per-node Vault Agent to render
> certs + read KV. The manual path issues each leaf with `vault write
> pki_int/issue/iceberg-server` and reads KV with `vault kv get` from your operator
> session — no Vault Agent. The two roles use **different cert file layouts** (PG:
> `server.crt`/`server.key`/`ca.crt`; REST: `cert.pem`/`key.pem`/`ca.crt`), spelled
> out in §5.3.

---

## 4. Target topology

| Node | Role | VMnet11 | VMnet10 | MAC (primary / secondary) | RAM | Ports |
|---|---|---|---|---|---|---|
| `iceberg-rest-1` | Nessie (Iceberg REST catalog) | `.147` | `.10.147` | `…3F:00:A0` / `…3F:01:A0` | 4 GB | 19120 HTTPS (app) · 9000 HTTP (mgmt/health) |
| `iceberg-rest-2` | Nessie (HA peer) | `.148` | `.10.148` | `…3F:00:A1` / `…3F:01:A1` | 4 GB | 19120 · 9000 |
| `iceberg-pg-1` | catalog PG **primary** | `.149` | `.10.149` | `…3F:00:A2` / `…3F:01:A2` | 2 GB | 5432 (TLS) |
| `iceberg-pg-2` | catalog PG **replica** | `.150` | `.10.150` | `…3F:00:A3` / `…3F:01:A3` | 2 GB | 5432 (TLS) |
| **VIP** | `iceberg-db.nexus.lab` (keepalived) | **`.151`** | — | — | — | 5432 → current primary |

> MAC block `:A0–:A3` (after MinIO `:9D`). **Streaming replication rides the
> VMnet10 backplane** (`192.168.10.149` ↔ `.150`); client SQL (Nessie→PG) + the VIP
> ride VMnet11. VRRP `virtual_router_id 71`. VMs under
> `H:\VMS\NexusPlatform\08-spark\<node>\` (shared lakehouse tier folder + `.14x`
> decade). Quarkus health is on the **management** port `http :9000/q/health`, *not*
> the app port `:19120` (T5).

---

## 5. Step-by-step build

> **WHERE:** node steps as `nexusadmin` → `sudo -i` (root). `vault` runs on
> **`vault-1`**. Order: PG HA pair first (Nessie needs the DB), then Nessie, then
> the catalog exit gate. The two PG nodes differ by role (**primary** `.149` /
> **replica** `.150`) and are spelled out in full; the two REST nodes are symmetric.

### 5.0 — Seed the catalog secrets in Vault KV (once)

> **Step 5.0.1 — Write the PG + Nessie passwords to Vault KV**
> **WHERE:** `vault-1` (`.121`), root shell with an operator `VAULT_TOKEN`.
> **WHY:** the PG superuser, the replication role, and the Nessie DB-user passwords
> must exist in KV before the nodes configure replication + Nessie. Use **hex**
> passwords (`openssl rand -hex`) — they're inline-safe in SQL (no quoting traps).
> The S3 warehouse creds are reused from Guide 16 (`nexus/lakehouse/minio/app-*`).
> **WHAT:**
> ```bash
> export VAULT_ADDR=https://127.0.0.1:8200 VAULT_CACERT=$HOME/.nexus/vault-ca-bundle.crt
> vault kv put nexus/lakehouse/iceberg/pg-superuser-password   value="$(openssl rand -hex 24)"
> vault kv put nexus/lakehouse/iceberg/pg-replication-password value="$(openssl rand -hex 24)"
> vault kv put nexus/lakehouse/iceberg/nessie-db-password      value="$(openssl rand -hex 24)"
> ```
> **EXPECTED:** 3 KV entries written.
> **VERIFY:** `vault kv get -field=value nexus/lakehouse/iceberg/nessie-db-password` returns 48 hex chars.

### 5.1 — Create the 4 VMs + install packages

> **Step 5.1.1 — Create the 4 VMs + baseline**
> **WHERE:** VMware GUI on the build host.
> **WHY:** standard Guide 00 deb13 shape (no extra data disk this tier). REST nodes
> 4 GB (JVM), PG nodes 2 GB.
> **WHAT:** create `iceberg-rest-1/2` (4 GB) + `iceberg-pg-1/2` (2 GB) per Guide 00
> §5.A; pin the §4 MACs; install Debian 13 + baseline (firstboot maps each role +
> `NEXUS_PG_ROLE` for the PG nodes; dual-NIC static).
> **EXPECTED:** the 4 boot with their `.147–.150` leases + `.10.147–.150` backplane.
> **VERIFY:** `ssh nexusadmin@192.168.70.149 'sudo grep NEXUS_PG_ROLE /etc/nexus-iceberg-pg/node-identity.env'`
> → `NEXUS_PG_ROLE=primary` (and `.150` → `replica`).

> **Step 5.1.2 — Install Nessie + JDK 21 on the REST nodes (`.147/.148`)**
> **WHERE:** `iceberg-rest-1` + `iceberg-rest-2`, root shell.
> **WHY:** Nessie is a single Quarkus runner JAR on JDK 21; runs as a `nessie`
> system user (group `iceberg`). The service is installed **disabled** — §5.4
> renders `nessie.env` and starts it.
> **WHAT (run on both REST nodes):**
> ```bash
> apt-get update && apt-get install -y openjdk-21-jre-headless curl jq
> getent group iceberg >/dev/null || groupadd --system iceberg
> getent passwd nessie >/dev/null || useradd --system --gid iceberg --home-dir /opt/nessie --create-home --shell /usr/sbin/nologin nessie
> chmod 0755 /opt/nessie
> curl -fSL https://github.com/projectnessie/nessie/releases/download/nessie-0.107.5/nessie-quarkus-0.107.5-runner.jar \
>   -o /opt/nessie/nessie-quarkus-runner.jar
> chown nessie:iceberg /opt/nessie/nessie-quarkus-runner.jar
> install -d -o root -g iceberg -m 0750 /etc/nexus-iceberg-rest /etc/nexus-iceberg-rest/tls
> ```
> Then install the systemd unit (disabled):
> ```bash
> cat > /etc/systemd/system/nexus-nessie.service <<'EOF'
> [Unit]
> Description=Nexus Project Nessie (Iceberg REST catalog server)
> Documentation=https://projectnessie.org/
> After=network-online.target
> Wants=network-online.target
> ConditionPathExists=/etc/nexus-iceberg-rest/nessie.env
> [Service]
> Type=simple
> User=nessie
> Group=iceberg
> Environment=JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
> EnvironmentFile=-/etc/nexus-iceberg-rest/nessie.env
> ExecStart=/usr/lib/jvm/java-21-openjdk-amd64/bin/java -jar /opt/nessie/nessie-quarkus-runner.jar
> Restart=on-failure
> RestartSec=10
> LimitNOFILE=65536
> [Install]
> WantedBy=multi-user.target
> EOF
> systemctl daemon-reload ; systemctl disable nexus-nessie.service 2>/dev/null || true
> ```
> **EXPECTED:** JAR present; unit installed + disabled.
> **VERIFY:** `test -s /opt/nessie/nessie-quarkus-runner.jar && echo ok`;
> `systemctl is-enabled nexus-nessie.service` → `disabled`.

> **Step 5.1.3 — Install PostgreSQL 17 + keepalived on the PG nodes (`.149/.150`)**
> **WHERE:** `iceberg-pg-1` + `iceberg-pg-2`, root shell.
> **WHY:** PG 17 from PGDG. **deb13 (trixie) PGDG lags** → use the **bookworm**
> PGDG repo, and pull the two t64-renamed deps (`libicu72` + `libldap-2.5-0`) from a
> low-pinned bookworm fallback (T4). Keep the default `main` cluster but **disable**
> the units so a clone doesn't auto-start a stale cluster before §5.2 configures
> replication.
> **WHAT (run on both PG nodes):**
> ```bash
> apt-get install -y curl ca-certificates gnupg lsb-release
> install -d -m 0755 /etc/apt/keyrings
> curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor > /etc/apt/keyrings/pgdg.gpg
> echo 'deb [signed-by=/etc/apt/keyrings/pgdg.gpg] https://apt.postgresql.org/pub/repos/apt bookworm-pgdg main' > /etc/apt/sources.list.d/pgdg.list
> # bookworm fallback for the 2 t64-renamed libs only (pinned low)
> echo 'deb http://deb.debian.org/debian bookworm main' > /etc/apt/sources.list.d/bookworm-pg-deps.list
> cat > /etc/apt/preferences.d/bookworm-pg-deps.pref <<'EOF'
> Package: *
> Pin: release n=bookworm
> Pin-Priority: 100
>
> Package: libicu72 libldap-2.5-0
> Pin: release n=bookworm
> Pin-Priority: 990
> EOF
> apt-get update
> apt-get install -y -t bookworm libicu72 libldap-2.5-0
> apt-get install -y postgresql-17 postgresql-client-17 postgresql-contrib-17 keepalived
> systemctl disable --now postgresql postgresql@17-main keepalived 2>/dev/null || true
> install -d -o root -g postgres -m 0750 /etc/nexus-iceberg-pg /etc/nexus-iceberg-pg/tls
> ```
> **EXPECTED:** PG 17 + keepalived installed; units disabled.
> **VERIFY:** `/usr/lib/postgresql/17/bin/postgres --version` → `17.x`; `keepalived --version`.

### 5.2 — nftables (all 4: backplane trust + service ports)

> **Step 5.2.1 — Apply the per-node ruleset**
> **WHERE:** each of the 4 nodes, root shell.
> **WHY:** streaming replication + future peer traffic ride VMnet10 — trust the
> backplane ([[feedback_cluster_template_nftables_backplane]]). Open PG `5432` (PG
> nodes) / Nessie `19120` + `9000` (REST nodes) on VMnet11. Atomic `nft -f`
> ([[feedback_nftables_runtime_add_after_drop]]).
> **WHAT (PG nodes — `.149/.150`):**
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
>         ip protocol icmp accept
>         ip6 nexthdr icmpv6 accept
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 22   accept comment "SSH"
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 9100 accept comment "node_exporter"
>         iifname "nic1" ip saddr 192.168.10.0/24 accept comment "trusted cluster backplane (VMnet10)"
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 5432 accept comment "PostgreSQL (Nessie + admin)"
>         iif "nic0" ip protocol vrrp accept comment "keepalived VRRP"
>         counter drop
>     }
>     chain forward { type filter hook forward priority 0; policy drop; }
>     chain output  { type filter hook output priority 0; policy accept; }
> }
> EOF
> nft -f /etc/nftables.conf ; systemctl enable nftables 2>/dev/null || true
> ```
> **WHAT (REST nodes — `.147/.148`):** same ruleset but replace the `5432` + `vrrp`
> lines with:
> ```bash
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport { 19120, 9000 } accept comment "Nessie Iceberg REST (app) + mgmt/health"
> ```
> **EXPECTED:** ruleset loads clean on all 4.
> **VERIFY:** `nft list chain inet filter input | grep '192.168.10.0/24 accept'` (all 4);
> PG nodes show `dport 5432`; REST nodes show `19120, 9000`.

### 5.3 — Per-node mTLS certs from Vault PKI

> **Step 5.3.1 — Create the `iceberg-server` PKI role (once)**
> **WHERE:** `vault-1`, root shell.
> **WHY:** one role issues all 4 leaves. PG leaves add `iceberg-db.nexus.lab` + the
> **VIP IP `.151`** to the SANs (clients hit the catalog-DB front door); REST leaves
> add `iceberg.nexus.lab` (round-robin). 90-day TTL.
> **WHAT:**
> ```bash
> vault write pki_int/roles/iceberg-server \
>   allowed_domains='nexus.lab,iceberg.nexus.lab,iceberg-db.nexus.lab,iceberg-rest-1,iceberg-rest-2,iceberg-pg-1,iceberg-pg-2,localhost' \
>   allow_subdomains=true allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
> ```
> **VERIFY:** `vault read pki_int/roles/iceberg-server`.

> **Step 5.3.2 — Issue + place the PG certs (`iceberg-pg-1`, then `iceberg-pg-2`)**
> **WHERE:** issue on `vault-1`; place on each PG node.
> **WHY:** PG TLS layout = `server.crt` (leaf+intermediate) / `server.key` (**PKCS#8**)
> / `ca.crt` (full chain), **owner `postgres`** (PG refuses a key it can't read /
> that's group/world-readable). SANs cover the node, `iceberg-db.nexus.lab`, the
> VIP `.151`, and the backplane IP (replication connects over `.10.x`).
> **WHAT (on vault-1 — for `iceberg-pg-1`):**
> ```bash
> vault write -format=json pki_int/issue/iceberg-server \
>   common_name=iceberg-pg-1.nexus.lab \
>   alt_names='iceberg-pg-1,iceberg-pg-1.nexus.lab,iceberg-db.nexus.lab,localhost' \
>   ip_sans='192.168.10.149,192.168.70.149,192.168.70.151,127.0.0.1' ttl=2160h > /tmp/pg1.json
> vault read -field=certificate pki_int/cert/ca_chain > /tmp/nexus-ca-chain.pem
> ```
> **WHAT (copy `/tmp/pg1.json` + chain to `iceberg-pg-1`, then as root there):**
> ```bash
> D=/etc/nexus-iceberg-pg/tls
> jq -r '.data.certificate' /tmp/pg1.json > /tmp/leaf.crt
> jq -r '.data.issuing_ca'  /tmp/pg1.json > /tmp/int.crt
> jq -r '.data.private_key' /tmp/pg1.json > /tmp/leaf.key
> cat /tmp/leaf.crt /tmp/int.crt > "$D/server.crt"
> openssl pkcs8 -topk8 -nocrypt -in /tmp/leaf.key -out "$D/server.key"
> cp /tmp/nexus-ca-chain.pem "$D/ca.crt"
> cp /tmp/nexus-ca-chain.pem /etc/ssl/certs/iceberg-ca.pem
> chown -R postgres:postgres "$D"
> chmod 0644 "$D/server.crt" "$D/ca.crt" ; chmod 0600 "$D/server.key"
> rm -f /tmp/leaf.crt /tmp/int.crt /tmp/leaf.key /tmp/pg1.json
> ```
> **Repeat for `iceberg-pg-2`** — issue with `common_name=iceberg-pg-2.nexus.lab`,
> `alt_names='iceberg-pg-2,iceberg-pg-2.nexus.lab,iceberg-db.nexus.lab,localhost'`,
> `ip_sans='192.168.10.150,192.168.70.150,192.168.70.151,127.0.0.1'`, then place
> identically on `.150`.
> **VERIFY (each PG node):** `openssl x509 -in /etc/nexus-iceberg-pg/tls/server.crt -noout -ext subjectAltName`
> → SAN has `iceberg-db.nexus.lab` + IP `192.168.70.151`; key line 1 = `BEGIN PRIVATE KEY`.

> **Step 5.3.3 — Issue + place the REST certs (`iceberg-rest-1`, then `iceberg-rest-2`)**
> **WHERE:** issue on `vault-1`; place on each REST node.
> **WHY:** REST TLS layout = `cert.pem` (leaf+intermediate) / `key.pem` (**PKCS#8**)
> / `ca.crt`, **owner `nessie`**. SANs add `iceberg.nexus.lab` (the round-robin
> name Quarkus serves HTTPS for).
> **WHAT (on vault-1 — for `iceberg-rest-1`):**
> ```bash
> vault write -format=json pki_int/issue/iceberg-server \
>   common_name=iceberg-rest-1.nexus.lab \
>   alt_names='iceberg-rest-1,iceberg-rest-1.nexus.lab,iceberg.nexus.lab,localhost' \
>   ip_sans='192.168.10.147,192.168.70.147,127.0.0.1' ttl=2160h > /tmp/rest1.json
> ```
> **WHAT (place on `iceberg-rest-1`, as root):**
> ```bash
> D=/etc/nexus-iceberg-rest/tls
> jq -r '.data.certificate' /tmp/rest1.json > /tmp/leaf.crt
> jq -r '.data.issuing_ca'  /tmp/rest1.json > /tmp/int.crt
> jq -r '.data.private_key' /tmp/rest1.json > /tmp/leaf.key
> cat /tmp/leaf.crt /tmp/int.crt > "$D/cert.pem"
> openssl pkcs8 -topk8 -nocrypt -in /tmp/leaf.key -out "$D/key.pem"
> cp /tmp/nexus-ca-chain.pem "$D/ca.crt"
> cp /tmp/nexus-ca-chain.pem /etc/ssl/certs/iceberg-ca.pem
> chown -R nessie:iceberg "$D"
> chmod 0644 "$D/cert.pem" "$D/ca.crt" ; chmod 0600 "$D/key.pem"
> rm -f /tmp/leaf.crt /tmp/int.crt /tmp/leaf.key /tmp/rest1.json
> ```
> **Repeat for `iceberg-rest-2`** — `common_name=iceberg-rest-2.nexus.lab`,
> `alt_names='iceberg-rest-2,iceberg-rest-2.nexus.lab,iceberg.nexus.lab,localhost'`,
> `ip_sans='192.168.10.148,192.168.70.148,127.0.0.1'`, place on `.148`.
> **VERIFY (each REST node):** `openssl x509 -in /etc/nexus-iceberg-rest/tls/cert.pem -noout -ext subjectAltName`
> → SAN has `iceberg.nexus.lab`; key line 1 = `BEGIN PRIVATE KEY`.

### 5.4 — PostgreSQL master-replica HA pair

> **Step 5.4.1 — Configure the PRIMARY (`iceberg-pg-1`, `.149`)**
> **WHERE:** `iceberg-pg-1`, root shell with `VAULT_ADDR`/`VAULT_TOKEN` + CA.
> **WHY:** enable replication + TLS, set `pg_hba` (replication over the backplane;
> Nessie + admin over VMnet11 TLS), create the `repluser` + `nessie` roles + the
> `nessie` DB. The standby will clone from here.
> **WHAT:**
> ```bash
> export VAULT_ADDR=https://192.168.70.121:8200 VAULT_CACERT=/etc/ssl/certs/iceberg-ca.pem
> SUPERPW=$(vault kv get -field=value nexus/lakehouse/iceberg/pg-superuser-password)
> REPLPW=$(vault kv get -field=value nexus/lakehouse/iceberg/pg-replication-password)
> NESSIEPW=$(vault kv get -field=value nexus/lakehouse/iceberg/nessie-db-password)
> CONF=/etc/postgresql/17/main
> mkdir -p "$CONF/conf.d"
> cat > "$CONF/conf.d/nexus-iceberg.conf" <<'EOF'
> listen_addresses = '*'
> wal_level = replica
> max_wal_senders = 10
> max_replication_slots = 10
> hot_standby = on
> password_encryption = scram-sha-256
> ssl = on
> ssl_cert_file = '/etc/nexus-iceberg-pg/tls/server.crt'
> ssl_key_file = '/etc/nexus-iceberg-pg/tls/server.key'
> ssl_ca_file = '/etc/nexus-iceberg-pg/tls/ca.crt'
> EOF
> grep -q "include_dir = 'conf.d'" "$CONF/postgresql.conf" || echo "include_dir = 'conf.d'" >> "$CONF/postgresql.conf"
> grep -q 'NEXUS-ICEBERG-HBA' "$CONF/pg_hba.conf" || cat >> "$CONF/pg_hba.conf" <<'EOF'
> # NEXUS-ICEBERG-HBA
> host    replication   repluser   192.168.10.0/24   scram-sha-256
> hostssl nessie        nessie     192.168.70.0/24   scram-sha-256
> hostssl all           postgres   192.168.70.0/24   scram-sha-256
> EOF
> pg_ctlcluster 17 main start || systemctl start postgresql@17-main
> systemctl enable postgresql@17-main
> for i in $(seq 1 30); do sudo -u postgres pg_isready -q && break; sleep 2; done
> sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD '$SUPERPW'"
> sudo -u postgres psql -c "CREATE ROLE repluser WITH REPLICATION LOGIN PASSWORD '$REPLPW'"
> sudo -u postgres psql -c "CREATE ROLE nessie WITH LOGIN PASSWORD '$NESSIEPW'"
> sudo -u postgres psql -c "CREATE DATABASE nessie OWNER nessie"
> pg_ctlcluster 17 main reload || systemctl reload postgresql@17-main
> ```
> **EXPECTED:** PG up; `repluser` + `nessie` roles + `nessie` DB created; SSL on.
> **VERIFY:** `sudo -u postgres psql -tAc "SELECT 1 FROM pg_database WHERE datname='nessie'"` → `1`;
> `sudo -u postgres psql -tAc 'SHOW ssl'` → `on`.

> **Step 5.4.2 — Clone the REPLICA (`iceberg-pg-2`, `.150`) via `pg_basebackup`**
> **WHERE:** `iceberg-pg-2`, root shell with `VAULT_ADDR`/`VAULT_TOKEN` + CA.
> **WHY:** stop + wipe the replica's PGDATA, `pg_basebackup -R` from the primary's
> **backplane IP** (replication rides VMnet10), start as a hot standby. ⚠️
> `pg_basebackup -R` does **not** embed the replication password in
> `primary_conninfo` — the walreceiver authenticates via the `postgres` user's
> **`.pgpass`** (T6); write it first.
> **WHAT:**
> ```bash
> export VAULT_ADDR=https://192.168.70.121:8200 VAULT_CACERT=/etc/ssl/certs/iceberg-ca.pem
> REPLPW=$(vault kv get -field=value nexus/lakehouse/iceberg/pg-replication-password)
> CONF=/etc/postgresql/17/main ; DATA=/var/lib/postgresql/17/main
> mkdir -p "$CONF/conf.d"
> cat > "$CONF/conf.d/nexus-iceberg.conf" <<'EOF'
> listen_addresses = '*'
> wal_level = replica
> hot_standby = on
> password_encryption = scram-sha-256
> ssl = on
> ssl_cert_file = '/etc/nexus-iceberg-pg/tls/server.crt'
> ssl_key_file = '/etc/nexus-iceberg-pg/tls/server.key'
> ssl_ca_file = '/etc/nexus-iceberg-pg/tls/ca.crt'
> EOF
> grep -q "include_dir = 'conf.d'" "$CONF/postgresql.conf" || echo "include_dir = 'conf.d'" >> "$CONF/postgresql.conf"
> # walreceiver auth (T6) -- .pgpass for the postgres OS user, points at the primary backplane IP
> echo "192.168.10.149:5432:replication:repluser:$REPLPW" > /var/lib/postgresql/.pgpass
> chown postgres:postgres /var/lib/postgresql/.pgpass ; chmod 0600 /var/lib/postgresql/.pgpass
> # clone from the primary backplane IP
> pg_ctlcluster 17 main stop || systemctl stop postgresql@17-main
> rm -rf "$DATA" ; install -d -m 0700 -o postgres -g postgres "$DATA"
> sudo -u postgres env PGPASSWORD="$REPLPW" pg_basebackup -h 192.168.10.149 -p 5432 -U repluser -D "$DATA" -Fp -Xs -P -R
> pg_ctlcluster 17 main start || systemctl start postgresql@17-main
> systemctl enable postgresql@17-main
> ```
> **EXPECTED:** `pg_basebackup` streams the base backup; the node comes up in recovery.
> **VERIFY (on the replica):** `sudo -u postgres psql -tAc 'SELECT pg_is_in_recovery()'` → `t`;
> **(on the primary):** `sudo -u postgres psql -tAc "SELECT count(*) FROM pg_stat_replication WHERE state='streaming'"` → `1`.

> **Step 5.4.3 — keepalived VRRP VIP `.151` on both PG nodes**
> **WHERE:** `iceberg-pg-1` (priority 110) + `iceberg-pg-2` (priority 100), root shell.
> **WHY:** the floating catalog-DB front door. **State `BACKUP` + `nopreempt`** so a
> recovered old-primary doesn't reclaim the VIP and split-brain. The health check
> **must** exec the **versioned** `pg_isready` binary
> (`/usr/lib/postgresql/17/bin/pg_isready`), *not* the `/usr/bin/pg_isready`
> pg_wrapper symlink — the wrapper fails under keepalived's exec context → check
> fails → no MASTER → no VIP (T3). `notify_master` promotes a standby on failover.
> **WHAT (on BOTH PG nodes — the check + promote scripts are identical):**
> ```bash
> cat > /usr/local/sbin/nexus-pg-check.sh <<'EOS'
> #!/bin/bash
> exec /usr/lib/postgresql/17/bin/pg_isready -q -h 127.0.0.1 -p 5432
> EOS
> chmod 0755 /usr/local/sbin/nexus-pg-check.sh
> cat > /etc/keepalived/nexus-iceberg-promote.sh <<'EOS'
> #!/bin/bash
> # promote this PG node if it is a standby (failover)
> if sudo -u postgres psql -tAc "SELECT pg_is_in_recovery()" 2>/dev/null | grep -qi t; then
>   /usr/bin/pg_ctlcluster 17 main promote
> fi
> EOS
> chmod 0755 /etc/keepalived/nexus-iceberg-promote.sh
> ```
> **WHAT (on `iceberg-pg-1` — `priority 110`, `unicast_src_ip .149`, peer `.150`):**
> ```bash
> cat > /etc/keepalived/keepalived.conf <<'EOF'
> global_defs { script_user root }
> vrrp_script chk_pg {
>   script "/usr/local/sbin/nexus-pg-check.sh"
>   interval 5
>   fall 2
>   rise 2
> }
> vrrp_instance VI_ICEBERG_DB {
>   state BACKUP
>   nopreempt
>   interface nic0
>   virtual_router_id 71
>   priority 110
>   advert_int 1
>   unicast_src_ip 192.168.70.149
>   unicast_peer { 192.168.70.150 }
>   authentication { auth_type PASS ; auth_pass icebrgvr }
>   virtual_ipaddress { 192.168.70.151/24 dev nic0 }
>   notify_master "/etc/keepalived/nexus-iceberg-promote.sh"
>   track_script { chk_pg }
> }
> EOF
> systemctl enable --now keepalived ; systemctl restart keepalived
> ```
> **WHAT (on `iceberg-pg-2` — identical file but `priority 100`, `unicast_src_ip
> 192.168.70.150`, `unicast_peer { 192.168.70.149 }`):** write the same
> `keepalived.conf` with those three values changed, then `systemctl enable --now keepalived`.
> **EXPECTED:** the VIP binds on the primary (`.149`, higher priority) ~10–15 s after start.
> **VERIFY:** `ip -4 -o addr show nic0 | grep 192.168.70.151` returns a line on
> **exactly one** PG node; from any node `ping -c1 192.168.70.151` succeeds.

### 5.5 — Configure + start Nessie (both REST nodes)

> **Step 5.5.1 — Import the Vault CA into the JVM truststore + render `nessie.env` + `nessie.properties`**
> **WHERE:** `iceberg-rest-1` + `iceberg-rest-2`, root shell with `VAULT_ADDR`/`VAULT_TOKEN` + CA.
> **WHY:** Nessie's **S3 client** (AWS Java SDK) validates MinIO's TLS against the
> **JVM truststore** — import the Vault CA into `cacerts` or the warehouse writes
> fail TLS. The version store is **JDBC2** → the bundled **named `postgresql`
> datasource**, which must be **activated** (`…POSTGRESQL_ACTIVE=true`) and given the
> URL/creds — the *default* datasource stays inert (T1). The compound S3 access-key
> (name+secret) can't map from env vars (Quarkus `SRCFG00050`) → it goes in a
> **`nessie.properties`** file as a `urn:nessie-secret` reference (T2).
> **WHAT (run on both REST nodes):**
> ```bash
> export VAULT_ADDR=https://192.168.70.121:8200 VAULT_CACERT=/etc/ssl/certs/iceberg-ca.pem
> JH=/usr/lib/jvm/java-21-openjdk-amd64
> NESSIEPW=$(vault kv get -field=value nexus/lakehouse/iceberg/nessie-db-password)
> S3AK=$(vault kv get -field=value nexus/lakehouse/minio/app-access-key)
> S3SK=$(vault kv get -field=value nexus/lakehouse/minio/app-secret-key)
>
> # Trust the Vault CA in the JVM so Nessie's S3 client validates MinIO
> keytool -delete -alias nexus-ca -keystore "$JH/lib/security/cacerts" -storepass changeit 2>/dev/null || true
> keytool -importcert -noprompt -alias nexus-ca -file /etc/ssl/certs/iceberg-ca.pem \
>   -keystore "$JH/lib/security/cacerts" -storepass changeit
>
> cat > /etc/nexus-iceberg-rest/nessie.env <<EOF
> # Quarkus HTTPS-only on :19120
> QUARKUS_HTTP_PORT=19120
> QUARKUS_HTTP_INSECURE_REQUESTS=disabled
> QUARKUS_HTTP_SSL_PORT=19120
> QUARKUS_HTTP_SSL_CERTIFICATE_FILES=/etc/nexus-iceberg-rest/tls/cert.pem
> QUARKUS_HTTP_SSL_CERTIFICATE_KEY_FILES=/etc/nexus-iceberg-rest/tls/key.pem
> # Version store = JDBC2 (PostgreSQL via the iceberg-db VRRP VIP). Activate the
> # NAMED postgresql datasource (default stays inert).
> NESSIE_VERSION_STORE_TYPE=JDBC2
> NESSIE_VERSION_STORE_PERSIST_JDBC_DATASOURCE=postgresql
> QUARKUS_DATASOURCE_POSTGRESQL_ACTIVE=true
> QUARKUS_DATASOURCE_POSTGRESQL_DB_KIND=postgresql
> QUARKUS_DATASOURCE_POSTGRESQL_USERNAME=nessie
> QUARKUS_DATASOURCE_POSTGRESQL_PASSWORD=$NESSIEPW
> QUARKUS_DATASOURCE_POSTGRESQL_JDBC_URL=jdbc:postgresql://iceberg-db.nexus.lab:5432/nessie?sslmode=require
> # Iceberg REST catalog + S3 warehouse (MinIO)
> NESSIE_CATALOG_DEFAULT_WAREHOUSE=warehouse
> NESSIE_CATALOG_WAREHOUSES_WAREHOUSE_LOCATION=s3://warehouse/
> NESSIE_CATALOG_SERVICE_S3_DEFAULT_OPTIONS_REGION=us-east-1
> NESSIE_CATALOG_SERVICE_S3_DEFAULT_OPTIONS_ENDPOINT=https://minio.nexus.lab:9000/
> NESSIE_CATALOG_SERVICE_S3_DEFAULT_OPTIONS_PATH_STYLE_ACCESS=true
> NESSIE_CATALOG_SERVICE_S3_DEFAULT_OPTIONS_AUTH_TYPE=STATIC
> QUARKUS_CONFIG_LOCATIONS=/etc/nexus-iceberg-rest/nessie.properties
> EOF
> chown root:iceberg /etc/nexus-iceberg-rest/nessie.env ; chmod 0640 /etc/nexus-iceberg-rest/nessie.env
>
> cat > /etc/nexus-iceberg-rest/nessie.properties <<EOF
> # S3 STATIC creds -- the inline access-key.name/.secret form is rejected by
> # Quarkus config validation (SRCFG00050); use a urn:nessie-secret reference.
> nessie.catalog.service.s3.default-options.access-key=urn:nessie-secret:quarkus:lakehouse-s3-creds
> lakehouse-s3-creds.name=$S3AK
> lakehouse-s3-creds.secret=$S3SK
> EOF
> chown root:iceberg /etc/nexus-iceberg-rest/nessie.properties ; chmod 0640 /etc/nexus-iceberg-rest/nessie.properties
> ```
> **EXPECTED:** CA imported; `nessie.env` + `nessie.properties` rendered (`0640 root:iceberg`).
> **VERIFY:** `keytool -list -alias nexus-ca -keystore "$JH/lib/security/cacerts" -storepass changeit`
> shows the cert; `grep POSTGRESQL_ACTIVE /etc/nexus-iceberg-rest/nessie.env` → `true`.

> **Step 5.5.2 — Start Nessie on both nodes; wait for health**
> **WHERE:** both REST nodes (start), root shell.
> **WHY:** Nessie connects to the catalog PG via the VIP + bootstraps its schema on
> first start. Health is on the **management** port `http :9000/q/health`, *not* the
> app port (T5).
> **WHAT (on both REST nodes):**
> ```bash
> systemctl daemon-reload ; systemctl enable nexus-nessie.service ; systemctl restart nexus-nessie.service
> for i in $(seq 1 24); do
>   code=$(curl -fsS -o /dev/null -w '%{http_code}' http://localhost:9000/q/health 2>/dev/null || true)
>   [ "$code" = "200" ] && { echo "nessie healthy"; break; }
>   sleep 10
> done
> ```
> **EXPECTED:** `nessie healthy` within ~4 min on each node.
> **VERIFY:** `systemctl is-active nexus-nessie.service` → `active`;
> `curl -fsS -o /dev/null -w '%{http_code}' http://localhost:9000/q/health` → `200`;
> `curl -sk -o /dev/null -w '%{http_code}' https://localhost:19120/iceberg/v1/config` → `200`.

### 5.6 — DNS: round-robin REST + the VIP record (gateway)

> **Step 5.6.1 — Publish `iceberg.nexus.lab` (round-robin) + `iceberg-db.nexus.lab` (VIP)**
> **WHERE:** `nexus-gateway` (`.70.1`), root shell.
> **WHY:** `iceberg.nexus.lab` → the 2 REST nodes (round-robin, no VIP — the app
> tier is symmetric); `iceberg-db.nexus.lab` → the single keepalived VIP `.151`
> (Nessie's JDBC URL points here).
> **WHAT:**
> ```bash
> cat > /etc/dnsmasq-iceberg.hosts <<'EOF'
> 192.168.70.147 iceberg.nexus.lab
> 192.168.70.148 iceberg.nexus.lab
> 192.168.70.151 iceberg-db.nexus.lab
> EOF
> echo 'addn-hosts=/etc/dnsmasq-iceberg.hosts' > /etc/dnsmasq.d/iceberg-records.conf
> dnsmasq --test && systemctl reload dnsmasq
> ```
> **VERIFY:** `dig @192.168.70.1 iceberg.nexus.lab +short` → `.147` + `.148`;
> `dig @192.168.70.1 iceberg-db.nexus.lab +short` → `.151`.

### 5.7 — Catalog bootstrap (the exit gate)

> **Step 5.7.1 — Namespace round-trip via the Iceberg REST API**
> **WHERE:** `iceberg-rest-1` (`.147`), root shell.
> **WHY:** the deterministic exit gate — prove the catalog is reachable + PG-backed
> by creating a namespace through the Iceberg REST API (which writes to PG) and
> reading it back. A best-effort table create follows; the *definitive* table→S3
> write is the Guide 18 Spark gate (Spark is the natural Iceberg client).
> **WHAT (on iceberg-rest-1):**
> ```bash
> BASE=https://localhost:19120
> # 1. Iceberg REST config (proves the catalog is up)
> curl -sk "$BASE/iceberg/v1/config" -o /tmp/icfg.json -w 'iceberg-config:%{http_code}\n'
> # 2. Nessie native config (proves the PG backend is reachable)
> curl -sk -o /dev/null -w 'nessie-config:%{http_code}\n' "$BASE/api/v2/config"
> # 3. Namespace create + list round-trip (proves a catalog WRITE to PG)
> PREFIX=$(jq -r '.overrides.prefix // .defaults.prefix // "main"' /tmp/icfg.json)
> curl -sk -X POST "$BASE/iceberg/v1/$PREFIX/namespaces" \
>   -H 'Content-Type: application/json' -d '{"namespace":["nexus_lakehouse"]}' >/dev/null
> curl -sk "$BASE/iceberg/v1/$PREFIX/namespaces" | grep -q nexus_lakehouse && echo "namespace round-trip OK"
> # 4. best-effort: create an Iceberg table (warehouse write proven by Spark gate)
> curl -sk -X POST "$BASE/iceberg/v1/$PREFIX/namespaces/nexus_lakehouse/tables" \
>   -H 'Content-Type: application/json' \
>   -d '{"name":"smoke","schema":{"type":"struct","fields":[{"id":1,"name":"id","required":true,"type":"long"}]}}' \
>   -w '\ntable-create:%{http_code}\n'
> ```
> **EXPECTED:** `iceberg-config:200`, `nessie-config:200`, `namespace round-trip OK`.
> **VERIFY:** `curl -sk https://localhost:19120/iceberg/v1/$PREFIX/namespaces` lists
> `nexus_lakehouse`; if the table created, `mc ls --recursive nexuslocal/warehouse/nexus_lakehouse/`
> on `minio-1` shows metadata objects (else the Spark gate proves it). **➡ The
> catalog is live for Guide 18 (Spark).**

---

## 6. Validation — by-hand acceptance smoke (demo / playbook)

Condensed from `smoke-0.L.2.ps1`. Per-node SSH probes from the **build host**.

- **Input:** the 4 nodes up; PG HA pair streaming; Nessie healthy; DNS published.
- **Where observed:** SSH to each node / `psql` on the PG nodes / `curl` on the REST
  nodes / `dig` on the gateway.
- **Proves:** an HA Iceberg REST catalog (2 Nessie) backed by a PG master-replica
  pair with a failover VIP, warehouse in MinIO.
- **Prerequisites:** Guides 00–04 + 16 alive; §5 complete.

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | 4 nodes reachable | `ssh …@147..150 'echo ok'` | all `ok` |
| 2 | PG roles via firstboot | `grep NEXUS_PG_ROLE …/node-identity.env` | pg-1 `primary`, pg-2 `replica` |
| 3 | TLS material | `openssl x509 -in <crt> -noout -ext subjectAltName` | PG SAN has `iceberg-db.nexus.lab`+`.151`; REST SAN has `iceberg.nexus.lab`; keys PKCS#8 |
| 4 | nftables backplane trust | `nft list chain inet filter input` (all 4) | `192.168.10.0/24 accept` present |
| 5 | PG streaming replication | `SELECT count(*) FROM pg_stat_replication WHERE state='streaming'` (pg-1) | `>= 1` |
| 6 | Replica in recovery | `SELECT pg_is_in_recovery()` (pg-2) | `t` |
| 7 | Nessie DB exists | `SELECT 1 FROM pg_database WHERE datname='nessie'` (pg-1) | `1` |
| 8 | VRRP VIP bound | `ip addr show nic0 \| grep .151` on each PG node | on **exactly one** node |
| 9 | Nessie health + REST | `curl http://localhost:9000/q/health` + `https://localhost:19120/iceberg/v1/config` (both REST) | both `200` |
| 10 | PG backend reachable | `curl https://localhost:19120/api/v2/config` (both REST) | `200` |
| 11 | DNS | `dig iceberg.nexus.lab +short` / `dig iceberg-db.nexus.lab +short` | 2 REST IPs / the VIP `.151` |
| 12 | Namespace round-trip | `curl …/iceberg/v1/<prefix>/namespaces` | `nexus_lakehouse` present |
| 13 | **PG failover** (chaos) | stop PG on the primary; VIP moves to pg-2 + pg-2 promotes (`pg_is_in_recovery()` → `f`) | catalog stays up; **pg-1 then needs re-sync as a standby of pg-2** |

**1–12 green ⇒ Guide 17 satisfied.** 13 is the HA payoff (catalog survives a PG
primary loss). After chaos, re-sync the old primary as a standby (§8 note).

---

## 7. Teardown / reset

```bash
for ip in 147 148; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-nessie.service; sudo rm -f /etc/nexus-iceberg-rest/nessie.env'; done
for ip in 149 150; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now keepalived postgresql@17-main; sudo rm -f /etc/keepalived/keepalived.conf'; done
# gateway: rm /etc/dnsmasq.d/iceberg-records.conf /etc/dnsmasq-iceberg.hosts ; systemctl reload dnsmasq
# then vmrun stop + deleteVM each of the 4 (Guide 00 §7).
```

> **The catalog metadata is in the PG `nessie` DB; the table data is in MinIO
> `s3://warehouse`.** Deleting the VMs discards the PG copy; the warehouse objects
> survive in MinIO. A fresh catalog + an empty `nessie` DB starts clean (it won't
> "see" orphaned warehouse files unless you re-register tables). The MinIO bucket +
> the gateway DNS records belong to Guides 16/01.

---

## 8. Troubleshooting

| # | Symptom | Cause | Fix |
|---|---|---|---|
| **T1** | Nessie starts but commits fail / it uses an in-memory store | the bundled **named `postgresql` datasource is inert** — only the default datasource is active by default | set `NESSIE_VERSION_STORE_PERSIST_JDBC_DATASOURCE=postgresql` **and** `QUARKUS_DATASOURCE_POSTGRESQL_ACTIVE=true` (+ the named datasource's URL/user/pw) — §5.5.1. |
| **T2** | Nessie won't start: Quarkus `SRCFG00050` on the S3 access-key | the compound S3 access-key (name+secret) **can't map from env vars** | put it in `nessie.properties` as `…access-key=urn:nessie-secret:quarkus:lakehouse-s3-creds` + `lakehouse-s3-creds.name/.secret`, pulled in via `QUARKUS_CONFIG_LOCATIONS` (§5.5.1). |
| **T3** | keepalived never assigns the VIP (no node becomes MASTER) | the track_script called the `/usr/bin/pg_isready` **pg_wrapper symlink**, which fails under keepalived's exec context → `chk_pg` returns 1 | the check must exec the **versioned** binary `/usr/lib/postgresql/17/bin/pg_isready` (§5.4.3). |
| **T4** | PG 17 install fails / unmet deps on Debian 13 | trixie PGDG lags; PG 17 (bookworm-built) wants `libicu72` + `libldap-2.5-0`, both t64-renamed in trixie | use the **bookworm** PGDG repo + a low-pinned bookworm fallback for *only* those two libs (`apt-get -t bookworm`) — §5.1.3. |
| **T5** | Health probe at `https://…:19120/q/health` returns nothing | Quarkus serves `/q/health` on the **management** interface `http :9000`, not the app port | probe `http://localhost:9000/q/health`; the app API (`/iceberg/`, `/api/`) is on `https :19120` (§5.5.2). |
| **T6** | Replica never streams: walreceiver auth fails | `pg_basebackup -R` does **not** embed the replication password in `primary_conninfo` | write `/var/lib/postgresql/.pgpass` (`<primary-bp-ip>:5432:replication:repluser:<pw>`, `0600 postgres`) before/after basebackup (§5.4.2). |
| **T7** | After a failover, the VIP flaps back / split-brain | a recovered old-primary reclaimed the VIP | keepalived state **`BACKUP` + `nopreempt`** on both nodes (§5.4.3) — the survivor keeps the VIP until it dies. |
| **T8** | Nessie S3 writes fail TLS validation | the JVM truststore doesn't trust MinIO's Vault-PKI cert | import the Vault CA into `$JAVA_HOME/lib/security/cacerts` via `keytool` on both REST nodes (§5.5.1). |
| **—** | Backplane down (replication can't connect) | VMware left `ethernet1`/nic1 NO-CARRIER at power-on | reconnect the 2nd NIC + `systemctl restart systemd-networkd`; confirm `ip addr show nic1` has `192.168.10.14X` (same as Guide 16 T2). |
| **—** | After chaos failover, old primary won't rejoin | it diverged from the new timeline | re-clone it as a standby of the new primary: `pg_basebackup -R` from pg-2's backplane IP (§5.4.2 pattern, swapping roles). |

---

### Cross-references

- **0.L.2 architecture:** memory `project_nexus_infra_lakehouse_phase`; ADR-0034
  (Iceberg/Nessie REST catalog + dedicated PG HA)
- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (Iceberg REST
  `.147/.148`, PG `.149/.150`, VIP `.151`, MAC block `:A0–:A3`); `vms.yaml`
  (`cluster: iceberg`)
- **Automated equivalents:** `nexus-infra-lakehouse/packer/lakehouse-iceberg-{rest,pg}-node/`
  + `terraform/envs/lakehouse-iceberg/role-overlay-{iceberg-pg-replication,nessie-config,iceberg-catalog-bootstrap,iceberg-tls}.tf`
- **Smoke mirror:** `nexus-infra-lakehouse/scripts/smoke-0.L.2.ps1`
- **Depends on:** [`16-lakehouse-minio.md`](./16-lakehouse-minio.md) (the `s3://warehouse` bucket)
- **Produces for:** Guide 18 — Lakehouse · Spark HA (the natural Iceberg client; proves table→S3 writes)
- **Related PG HA pattern:** [`10-oltp-patroni-postgresql-ha.md`](./10-oltp-patroni-postgresql-ha.md) (Patroni — contrast: heavier, etcd-driven)
- **Transients:** [[feedback_cluster_template_nftables_backplane]] · [[feedback_nftables_runtime_add_after_drop]]
- **Previous guide:** [`16-lakehouse-minio.md`](./16-lakehouse-minio.md)
- **Next guide:** Guide 18 — Lakehouse · Spark HA (2 masters + ZooKeeper quorum + 3 workers). See [`INDEX.md`](../INDEX.md).
