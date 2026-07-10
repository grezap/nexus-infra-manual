# Guide 12 — OLTP · MongoDB sharded (3 config + 2 shards×3 + 2 mongos)

> **Mirrors:** `nexus-infra-oltp` — the `oltp-mongo-node` Packer template
> (extended with `mongodb-org-mongos`) + the `oltp-mongo-sharded` env overlays
> (`mongo-keyfile`, `mongo-tls`, `mongo-config`, `mongo-rs-initiate`, `mongo-add-shards`,
> `mongo-nftables-backplane`, `mongo-vault-agents`) — Phase 0.N + the **0.N.1 wire-mTLS
> hardening** (sealed 2026-07-10) / ADR-0040. The automated lab renders the per-host
> certs + keyFile via Vault Agents and drives `rs.initiate`/`sh.addShard` over SSH; by
> hand we issue the certs with the `vault` CLI + distribute the keyFile, then run
> `mongosh` directly. Builds on **Guide 08**'s replica-set + keyFile + mTLS x509 +
> `__system` patterns.

---

## 1. Overview & purpose

The **last OLTP tier**: a horizontally **sharded** MongoDB cluster — **11 nodes**
that distribute a collection's documents across two shards, each shard itself a
3-member replica set. This is sharding proper (data partitioned across shards),
in contrast to Guide 08's single replica set (full dataset replicated).

- **Config-server replica set (`config`, port `27019`)** — `mongo-cfg-1/2/3`
  (`.74/.75/.76`). Holds the cluster metadata (which chunk lives on which shard).
- **2 shard replica sets (port `27018`)** — each a 3-member RS:
  - **`shard-1`**: `mongo-shard-1-1/2/3` (`.77/.78/.79`)
  - **`shard-2`**: `mongo-shard-2-1/2/3` (`.80/.56/.57`)
- **2 mongos routers (port `27017`)** — `mongo-mongos-1/2` (`.58/.59`). Stateless
  query routers; clients connect here, and mongos routes each query to the right
  shard using the config servers' metadata.

A **hashed shard key** spreads documents evenly; the balancer migrates chunks to
keep the shards balanced. Security is **two layers**: **keyFile internal auth** (the
same shared secret across all 11 nodes) for member-to-member traffic, **and mutual
TLS x509 (`requireTLS`)** on the wire — every client + member presents a per-host
leaf cert issued from Vault PKI (the **0.N.1 wire-mTLS hardening**, sealed
2026-07-10). keyFile stays as the *member*-auth layer (`clusterAuthMode: keyFile`);
mTLS is the *wire* layer on top — the same posture as Guide 08, now ported onto the
11-node sharded topology (3 config-RS + 2 shard-RS + 2 mongos).

**Why a separate cluster from Guide 08:** to demonstrate the *sharding* topology
(config servers + shards + routers) distinctly from simple replication. Two shards
is the minimum that proves chunk distribution + cross-shard routing.

**Dependency:**
- **Guides 00 + 04** — foundation alive (Vault KV for the keyFile).
- 11 `deb13` nodes baselined per Guide 00 §5.B, dual-NIC static.
- Conceptually builds on **Guide 08** (keyFile, `__system` bootstrap, `rs.initiate`).

> **By-hand divergence:** generate/reuse the keyFile + distribute it, and run
> `rs.initiate`/`sh.addShard` with `mongosh` directly — no Vault Agent. The
> automated lab's VMware-specific transients (11-concurrent-clone power-on storm →
> `-parallelism=3`) don't apply by hand — you power VMs on as you build them.

---

## 2. Component primer

- **Sharding vs. replication.** *Replication* (Guide 08) keeps full copies of the
  data for HA. *Sharding* **partitions** the data across shards for horizontal
  scale — each shard holds a *subset*. A production sharded cluster combines both:
  each shard is itself a replica set (HA), and the shards together hold the whole
  (scale). *Otherwise:* a single replica set (no horizontal partitioning).
- **Config servers.** A dedicated replica set (`config`, `configsvr: true`,
  port `27019`) storing the cluster's **metadata** — the shard list + the chunk
  map (which range of shard-key values lives on which shard). *Why a replica set:*
  the metadata is critical, so it's HA. *Otherwise:* the cluster can't route.
- **Shard.** A replica set (here `shardsvr: true`, port `27018`) holding a subset
  of every sharded collection's data. Two shards × 3 members = 6 data nodes.
- **mongos router.** A **stateless** process (port `27017`) that clients connect
  to; it reads the chunk map from the config servers and routes each operation to
  the owning shard(s). Run ≥2 behind round-robin DNS for HA. *Note:* a mongos
  **cannot bind its port until the config-server RS is initiated** (a real
  ordering trap, N7).
- **Shard key + chunks + balancer.** A collection is sharded on a **shard key**;
  MongoDB splits the key space into **chunks** and the **balancer** spreads chunks
  across shards. A **hashed** shard key (`{k: 'hashed'}`) distributes inserts
  evenly (vs. a range key, which can hot-spot on monotonic inserts).
- **keyFile internal auth + mTLS x509 wire (both layers).** The same 6–1024-char
  shared secret on all 11 nodes authenticates member-to-member traffic and implicitly
  enables authorization (`clusterAuthMode: keyFile` — the **member** layer). On top of
  it, **mutual TLS** (`net.tls.mode: requireTLS`, `allowConnectionsWithoutCertificates:
  false`) encrypts + mutually authenticates the **wire**: every mongod, mongos, and
  client presents a per-host leaf cert from Vault PKI. `--tlsCertificateKeyFile` takes
  **one** PEM (leaf + PKCS#8 key concatenated); the CA file is intermediate + root.
  *This is the 0.N.1 hardening (sealed 2026-07-10)* — the keyFile member layer and the
  mTLS wire layer are independent, and both stay on.
- **The `__system`-through-mongos limitation (N9 — the headline gotcha).** As in
  Guide 08, the localhost exception is off (8.0 + keyFile + authz), so you bootstrap
  via `__system`. **But `__system` auth uses the `local` DB, and mongos rejects it**
  (`Can't use 'local' database through mongos`). Sharded-cluster **client users
  live in `admin` on the config servers**. So: create a `root` user
  (`nexus-sharded-admin`, password = the keyFile content) on the **config-server
  PRIMARY** via `__system`+`local` (allowed on `mongod`), then auth all **mongos**
  operations as that user against `admin`.

---

## 3. Prerequisites

| # | Requirement | One-command verify |
|---|---|---|
| 1 | Foundation alive (Guides 00 + 04); Vault KV reachable | `vault kv get nexus/oltp/mongo/keyfile` on vault-1 (reuse Guide 08's keyFile, or seed fresh) |
| 2 | Vault PKI usable + the `mongo-sharded-server` role (created in §5.2.1) | `vault read pki_int/cert/ca` on vault-1 returns the intermediate; after §5.2.1, `vault read pki_int/roles/mongo-sharded-server` → `client_flag true` |
| 3 | Root CA bundle on the build host (to assemble `ca.crt` = intermediate + root) | `Test-Path ~/.nexus/secrets/vault-root-ca.pem` |
| 4 | 11 `deb13` nodes baselined, dual-NIC static (`.74–.80`, `.56/.57/.58/.59`) | all 11 answer `:22` |
| 5 | Vault root token on build host | `Test-Path ~/.nexus/secrets/vault-cluster-init.json` |
| 6 | Internet egress on the nodes | `ssh …@74 'curl -sI https://repo.mongodb.org \| head -1'` → `200` |

> MongoDB **8.0**. Ports: config `27019`, shards `27018`, mongos `27017`. keyFile
> at `/etc/nexus-mongo/keyfile` (`0400 mongodb:mongodb`); mTLS material at
> `/etc/nexus-mongo/tls/` — `server.pem` (leaf + PKCS#8 key) + `ca.crt` (intermediate +
> root), `0640 root:mongodb`. PKI role **`mongo-sharded-server`** (`client_flag=true`,
> 90-day leaves). MAC pool `:C0–:CA`.

---

## 4. Target topology

| Node | Role | RS / port | VMnet11 | VMnet10 |
|---|---|---|---|---|
| `mongo-cfg-1/2/3` | config server | `config` / `27019` | `.74/.75/.76` | `.10.74/.75/.76` |
| `mongo-shard-1-1/2/3` | shard-1 data | `shard-1` / `27018` | `.77/.78/.79` | `.10.77/.78/.79` |
| `mongo-shard-2-1/2/3` | shard-2 data | `shard-2` / `27018` | `.80/.56/.57` | `.10.80/.56/.57` |
| `mongo-mongos-1/2` | query router | mongos / `27017` | `.58/.59` | `.10.58/.59` |

> All on **2 vCPU / 2 GB / 40 GB**. Config-DB URI:
> `config/192.168.70.74:27019,192.168.70.75:27019,192.168.70.76:27019`. Client
> front door = round-robin DNS `mongos.nexus.lab` → `.58/.59` (port `27017`).
> The `.80→.56` decade-spill on shard-2 is intentional (the `.7x` decade ran out).

---

## 5. Step-by-step build

> **WHERE:** node steps as `nexusadmin`→`sudo -i`; `mongosh` under `sudo` (keyFile
> is `0400`). `vault` on **`vault-1`** (root token). Bootstrap config RS from
> `mongo-cfg-1`; add shards from `mongo-mongos-1`.

### 5.1 — Per-node base install (all 11)

> **Step 5.1.1 — Install MongoDB 8.0 (incl. `mongodb-org-mongos`) + dirs**
> **WHERE:** each node, root shell.
> **WHY:** the full `mongodb-org` metapackage gives both `mongod` (data/config
> nodes) and `mongos` (routers) — all 11 nodes get the same package; the *role*
> is set by config.
> **WHAT:**
> ```bash
> apt-get update -qq && apt-get install -y gnupg curl openssl
> curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | gpg --dearmor -o /usr/share/keyrings/mongodb-8.0.gpg
> echo "deb [signed-by=/usr/share/keyrings/mongodb-8.0.gpg] https://repo.mongodb.org/apt/debian trixie/mongodb-org/8.0 main" \
>   > /etc/apt/sources.list.d/mongodb-org-8.0.list
> apt-get update -qq && apt-get install -y mongodb-org mongodb-org-mongos
> systemctl disable --now mongod 2>/dev/null || true
> install -d -o mongodb -g mongodb -m0750 /etc/nexus-mongo /etc/nexus-mongo/tls /var/lib/nexus-mongo /var/log/nexus-mongo
> install -d -o mongodb -g mongodb -m0755 /var/run/nexus-mongo
> ```
> **VERIFY:** `mongod --version` + `mongos --version` both → `v8.0.x`.

> **Step 5.1.2 — Install the per-role systemd unit + nftables**
> **WHERE:** each node, root shell.
> **WHY:** data/config nodes run `nexus-mongo.service` (mongod + `mongod.conf`);
> mongos routers run `nexus-mongos.service` (mongos + `mongos.conf`) — different
> ports so they never collide. nftables opens the role port from VMnet11 + trusts
> the backplane.
> **WHAT (data/config nodes — `nexus-mongo.service`):**
> ```bash
> cat > /etc/systemd/system/nexus-mongo.service <<'EOF'
> [Unit]
> Description=Nexus MongoDB (sharded data/config node)
> After=network-online.target
> Wants=network-online.target
> [Service]
> User=mongodb
> Group=mongodb
> RuntimeDirectory=nexus-mongo
> RuntimeDirectoryMode=0755
> ExecStart=/usr/bin/mongod --config /etc/nexus-mongo/mongod.conf
> Restart=on-failure
> LimitNOFILE=64000
> [Install]
> WantedBy=multi-user.target
> EOF
> systemctl daemon-reload
> ```
> **WHAT (mongos nodes — `nexus-mongos.service`):**
> ```bash
> cat > /etc/systemd/system/nexus-mongos.service <<'EOF'
> [Unit]
> Description=Nexus MongoDB sharded cluster mongos router
> Requires=network-online.target
> After=network-online.target
> [Service]
> Type=simple
> User=mongodb
> Group=mongodb
> RuntimeDirectory=nexus-mongo
> RuntimeDirectoryMode=0750
> ExecStart=/usr/bin/mongos --config /etc/nexus-mongo/mongos.conf
> Restart=on-failure
> RestartSec=5
> KillMode=process
> LimitNOFILE=64000
> [Install]
> WantedBy=multi-user.target
> EOF
> systemctl daemon-reload
> ```
> **WHAT (nftables — add to chain input before `counter drop`, role port varies):**
> ```
> iifname "nic1" ip saddr 192.168.10.0/24 accept comment "trusted cluster backplane (VMnet10)"
> # config nodes:  iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 27019 accept
> # shard nodes:   iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 27018 accept
> # mongos nodes:  iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 27017 accept
> ```
> **VERIFY:** `systemctl cat nexus-mongo.service` (or `nexus-mongos.service`) shows
> the right ExecStart; `nft list chain inet filter input` shows the role port.

### 5.2 — PKI certs + shared keyFile (all 11)

> **Step 5.2.1 — Create the `mongo-sharded-server` PKI role**
> **WHERE:** `vault-1`, root shell.
> **WHY:** server **and** client EKU (each node's cert is also its client identity for
> mTLS member auth); the 11 sharded hostnames + `.mongo.nexus.lab` + the
> `mongos.nexus.lab` client front door; 90-day leaves. A **separate** role from Guide
> 08's `mongo-server` so this tier's `allowed_domains` carry all 11 sharded hostnames.
> **WHAT:**
> ```bash
> vault write pki_int/roles/mongo-sharded-server \
>   allowed_domains='nexus.lab,mongo.nexus.lab,mongos.nexus.lab,mongo-cfg-1,mongo-cfg-2,mongo-cfg-3,mongo-shard-1-1,mongo-shard-1-2,mongo-shard-1-3,mongo-shard-2-1,mongo-shard-2-2,mongo-shard-2-3,mongo-mongos-1,mongo-mongos-2,localhost' \
>   allow_subdomains=true allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
> ```
> **EXPECTED:** role written.
> **VERIFY:** `vault read pki_int/roles/mongo-sharded-server | grep client_flag` → `true`.

> **Step 5.2.2 — Issue + place each node's `server.pem` + `ca.crt` (all 11)**
> **WHERE:** issue on `vault-1`; place on each node.
> **WHY:** `requireTLS` needs, on **every** node, one combined PEM
> (`server.pem` = leaf + PKCS#8 key) plus the CA chain (`ca.crt` = intermediate + root;
> OpenSSL strict verify needs the full chain to a self-signed anchor). Each leaf carries
> **IP-SANs for BOTH NICs** (VMnet11 + VMnet10) + `127.0.0.1`, and DNS SANs for the
> node's hostname(s) — so mTLS validates whether a peer dials the service IP, the
> backplane IP, or by name. The two mongos also carry `mongos.nexus.lab` (the round-robin
> client front door). Same combined-PEM shape as Guide 08 §5.2.3.
> **WHAT — per node (CN + SANs):**
>
> | Node | CN (`common_name`) | `ip_sans` (both NICs + loopback) | `alt_names` (DNS) |
> |---|---|---|---|
> | `mongo-cfg-1`     | `mongo-cfg-1.mongo.nexus.lab`     | `192.168.70.74,192.168.10.74,127.0.0.1` | `mongo-cfg-1,mongo-cfg-1.nexus.lab,localhost` |
> | `mongo-cfg-2`     | `mongo-cfg-2.mongo.nexus.lab`     | `192.168.70.75,192.168.10.75,127.0.0.1` | `mongo-cfg-2,mongo-cfg-2.nexus.lab,localhost` |
> | `mongo-cfg-3`     | `mongo-cfg-3.mongo.nexus.lab`     | `192.168.70.76,192.168.10.76,127.0.0.1` | `mongo-cfg-3,mongo-cfg-3.nexus.lab,localhost` |
> | `mongo-shard-1-1` | `mongo-shard-1-1.mongo.nexus.lab` | `192.168.70.77,192.168.10.77,127.0.0.1` | `mongo-shard-1-1,mongo-shard-1-1.nexus.lab,localhost` |
> | `mongo-shard-1-2` | `mongo-shard-1-2.mongo.nexus.lab` | `192.168.70.78,192.168.10.78,127.0.0.1` | `mongo-shard-1-2,mongo-shard-1-2.nexus.lab,localhost` |
> | `mongo-shard-1-3` | `mongo-shard-1-3.mongo.nexus.lab` | `192.168.70.79,192.168.10.79,127.0.0.1` | `mongo-shard-1-3,mongo-shard-1-3.nexus.lab,localhost` |
> | `mongo-shard-2-1` | `mongo-shard-2-1.mongo.nexus.lab` | `192.168.70.80,192.168.10.80,127.0.0.1` | `mongo-shard-2-1,mongo-shard-2-1.nexus.lab,localhost` |
> | `mongo-shard-2-2` | `mongo-shard-2-2.mongo.nexus.lab` | `192.168.70.56,192.168.10.56,127.0.0.1` | `mongo-shard-2-2,mongo-shard-2-2.nexus.lab,localhost` |
> | `mongo-shard-2-3` | `mongo-shard-2-3.mongo.nexus.lab` | `192.168.70.57,192.168.10.57,127.0.0.1` | `mongo-shard-2-3,mongo-shard-2-3.nexus.lab,localhost` |
> | `mongo-mongos-1`  | `mongo-mongos-1.mongo.nexus.lab`  | `192.168.70.58,192.168.10.58,127.0.0.1` | `mongo-mongos-1,mongo-mongos-1.nexus.lab,mongos.nexus.lab,localhost` |
> | `mongo-mongos-2`  | `mongo-mongos-2.mongo.nexus.lab`  | `192.168.70.59,192.168.10.59,127.0.0.1` | `mongo-mongos-2,mongo-mongos-2.nexus.lab,mongos.nexus.lab,localhost` |
>
> For **each** node, issue on `vault-1` (substitute that row's `<CN>`/`<ip_sans>`/
> `<alt_names>`), split into `server.pem` + `ca.crt`, then scp both to the node:
> ```bash
> ISSUED=$(vault write -format=json pki_int/issue/mongo-sharded-server \
>   common_name="<CN>" alt_names="<alt_names>" ip_sans="<ip_sans>" ttl=2160h)
> # server.pem = leaf + PKCS#8 key (mongod/mongos combined shape)
> echo "$ISSUED" | jq -r '.data.certificate' > /tmp/server.pem
> echo "$ISSUED" | jq -r '.data.private_key' | openssl pkcs8 -topk8 -nocrypt >> /tmp/server.pem
> # ca.crt = intermediate + root
> { echo "$ISSUED" | jq -r '.data.issuing_ca'; cat ~/.nexus/secrets/vault-root-ca.pem; } > /tmp/ca.crt
> # scp /tmp/server.pem /tmp/ca.crt to <node>, then ON the node:
> #   sudo install -o root -g mongodb -m 0640 /tmp/server.pem /etc/nexus-mongo/tls/server.pem
> #   sudo install -o root -g mongodb -m 0640 /tmp/ca.crt     /etc/nexus-mongo/tls/ca.crt
> ```
> **EXPECTED:** `server.pem` + `ca.crt` on each of the 11 nodes, `0640 root:mongodb`.
> **VERIFY (per node):**
> `sudo openssl x509 -in /etc/nexus-mongo/tls/server.pem -noout -subject`
> → `CN=<host>.mongo.nexus.lab`;
> `grep -c 'BEGIN PRIVATE KEY' /etc/nexus-mongo/tls/server.pem` → `1` (PKCS#8);
> `sudo openssl x509 -in /etc/nexus-mongo/tls/server.pem -noout -text | grep -A1 'Subject Alternative Name'`
> lists **both** IPs (`.70.x` + `.10.x`).

> **Step 5.2.3 — Place the shared keyFile on every node (`0400 mongodb:mongodb`)**
> **WHERE:** build host (fetch from KV), then each node.
> **WHY:** all 11 members authenticate to each other with the **same** keyFile — this is
> the **member**-auth layer, independent of the mTLS wire layer above.
> Reuse Guide 08's keyFile from Vault KV (or seed a fresh one). mongod/mongos
> refuse to start unless it's `0400` and `mongodb`-owned.
> **WHAT:**
> ```bash
> # On vault-1 / build host: get the keyFile content (reuse Guide 08's, or seed fresh):
> vault kv get -field=content nexus/oltp/mongo/keyfile > /tmp/nexus-mongo-keyfile
> #   (to seed fresh instead: openssl rand -base64 756 | tr -d '\n' | vault kv put nexus/oltp/mongo/keyfile content=-)
> # Place on EACH of the 11 nodes (scp /tmp/nexus-mongo-keyfile to the node, then):
> #   sudo install -o mongodb -g mongodb -m 0400 /tmp/nexus-mongo-keyfile /etc/nexus-mongo/keyfile
> ```
> **EXPECTED:** identical keyFile on all 11, mode `0400`.
> **VERIFY:** `sudo stat -c '%U:%G %a' /etc/nexus-mongo/keyfile` → `mongodb:mongodb 400`;
> `sudo md5sum` matches across nodes.

### 5.3 — Render configs + start the engines

> **Step 5.3.1 — Render `mongod.conf` on config + shard nodes; start**
> **WHERE:** each config + shard node (9 nodes), root shell.
> **WHY:** the `mongod` config differs only in `clusterRole` (`configsvr` vs
> `shardsvr`), `replSetName`, and `port`. keyFile auth + `clusterAuthMode: keyFile`
> (member layer) **plus** `net.tls.mode: requireTLS` (the 0.N.1 wire layer) — the TLS
> block is identical on all 9 mongod nodes (per-host identity lives in the leaf cert).
> **WHAT (config node — `clusterRole: configsvr`, `replSetName: config`, port `27019`):**
> ```bash
> cat > /etc/nexus-mongo/mongod.conf <<'EOF'
> net:
>   bindIp: 0.0.0.0,::1
>   port: 27019
>   tls:
>     mode: requireTLS
>     certificateKeyFile: /etc/nexus-mongo/tls/server.pem
>     CAFile: /etc/nexus-mongo/tls/ca.crt
>     allowConnectionsWithoutCertificates: false
>     disabledProtocols: TLS1_0,TLS1_1
> replication:
>   replSetName: config
> security:
>   keyFile: /etc/nexus-mongo/keyfile
>   authorization: enabled
>   clusterAuthMode: keyFile
> sharding:
>   clusterRole: configsvr
> storage:
>   dbPath: /var/lib/nexus-mongo
>   wiredTiger:
>     engineConfig:
>       cacheSizeGB: 0.5
> systemLog:
>   destination: file
>   path: /var/log/nexus-mongo/mongod.log
>   logAppend: true
> processManagement:
>   fork: false
>   pidFilePath: /var/run/nexus-mongo/mongod.pid
> EOF
> chown root:mongodb /etc/nexus-mongo/mongod.conf ; chmod 640 /etc/nexus-mongo/mongod.conf
> systemctl enable --now nexus-mongo
> ```
> For **shard nodes**, the only differences are `port: 27018`,
> `replSetName: shard-1` (or `shard-2`), and `clusterRole: shardsvr`.
> **EXPECTED:** the service starts (each mongod standalone, RS not yet formed).
> **VERIFY (self-mTLS ping — `sudo` for the `0640` cert):** `systemctl is-active nexus-mongo` → `active`;
> `sudo mongosh --quiet --tls --tlsCAFile /etc/nexus-mongo/tls/ca.crt --tlsCertificateKeyFile /etc/nexus-mongo/tls/server.pem --port <role-port> --eval 'db.adminCommand({ping:1}).ok'` → `1`.

> **Step 5.3.2 — Render `mongos.conf` on the routers (do NOT start yet)**
> **WHERE:** `mongo-mongos-1/2`, root shell.
> **WHY:** mongos has no storage/replication — just `sharding.configDB` pointing
> at the config RS. **It can't bind `27017` until the config RS is initiated**
> (§5.4.1), so write the config now and start it in §5.4.3.
> **WHAT:**
> ```bash
> cat > /etc/nexus-mongo/mongos.conf <<'EOF'
> net:
>   bindIp: 0.0.0.0,::1
>   port: 27017
>   tls:
>     mode: requireTLS
>     certificateKeyFile: /etc/nexus-mongo/tls/server.pem
>     CAFile: /etc/nexus-mongo/tls/ca.crt
>     allowConnectionsWithoutCertificates: false
>     disabledProtocols: TLS1_0,TLS1_1
> security:
>   keyFile: /etc/nexus-mongo/keyfile
>   clusterAuthMode: keyFile
> sharding:
>   configDB: config/192.168.70.74:27019,192.168.70.75:27019,192.168.70.76:27019
> systemLog:
>   destination: file
>   path: /var/log/nexus-mongo/mongos.log
>   logAppend: true
> processManagement:
>   fork: false
>   pidFilePath: /var/run/nexus-mongo/mongos.pid
> EOF
> chown root:mongodb /etc/nexus-mongo/mongos.conf ; chmod 640 /etc/nexus-mongo/mongos.conf
> ```
> **EXPECTED:** `mongos.conf` written.
> **VERIFY:** `grep configDB /etc/nexus-mongo/mongos.conf` shows the 3 config IPs.

### 5.4 — Initiate the 3 replica sets + start mongos

> **Step 5.4.1 — `rs.initiate()` the config RS (`configsvr: true`)**
> **WHERE:** `mongo-cfg-1` (`.74`), root shell.
> **WHY:** the config RS must exist first (mongos depends on it). The init doc
> sets `configsvr: true`. `rs.initiate` is allowed without auth (bootstrap
> exception).
> **WHAT:**
> ```bash
> TLS='--tls --tlsCAFile /etc/nexus-mongo/tls/ca.crt --tlsCertificateKeyFile /etc/nexus-mongo/tls/server.pem'
> sudo mongosh --quiet $TLS --port 27019 --eval "printjson(rs.initiate({_id:'config', configsvr:true, members:[{_id:0,host:'192.168.70.74:27019'},{_id:1,host:'192.168.70.75:27019'},{_id:2,host:'192.168.70.76:27019'}]}))"
> ```
> **EXPECTED:** `ok: 1`; a PRIMARY elects in ~5 s.
> **VERIFY (auth as `__system`, over mTLS):**
> `KEY=$(sudo cat /etc/nexus-mongo/keyfile); TLS='--tls --tlsCAFile /etc/nexus-mongo/tls/ca.crt --tlsCertificateKeyFile /etc/nexus-mongo/tls/server.pem'; sudo mongosh --quiet $TLS --port 27019 --username __system --password "$KEY" --authenticationDatabase local --authenticationMechanism SCRAM-SHA-256 --eval "rs.status().members.filter(m=>m.stateStr=='PRIMARY').length"`
> → `1`.

> **Step 5.4.2 — `rs.initiate()` shard-1 + shard-2**
> **WHERE:** `mongo-shard-1-1` (`.77`) + `mongo-shard-2-1` (`.80`), root shell.
> **WHY:** each shard is its own 3-member RS. (No `configsvr` flag.)
> **WHAT (on `mongo-shard-1-1`; repeat on `mongo-shard-2-1` with shard-2's members):**
> ```bash
> TLS='--tls --tlsCAFile /etc/nexus-mongo/tls/ca.crt --tlsCertificateKeyFile /etc/nexus-mongo/tls/server.pem'
> # shard-1 (on .77):
> sudo mongosh --quiet $TLS --port 27018 --eval "printjson(rs.initiate({_id:'shard-1', members:[{_id:0,host:'192.168.70.77:27018'},{_id:1,host:'192.168.70.78:27018'},{_id:2,host:'192.168.70.79:27018'}]}))"
> # shard-2 (on .80):
> #   rs.initiate({_id:'shard-2', members:[{_id:0,host:'192.168.70.80:27018'},{_id:1,host:'192.168.70.56:27018'},{_id:2,host:'192.168.70.57:27018'}]})
> ```
> **EXPECTED:** `ok: 1` for each; each elects a PRIMARY.
> **VERIFY:** on each shard PRIMARY (auth `__system` over mTLS — the `$TLS` triplet +
> `--username __system …`, port 27018): `rs.status()` → 1 PRIMARY + 2 SECONDARY.

> **Step 5.4.3 — Start the mongos routers**
> **WHERE:** `mongo-mongos-1/2`, root shell.
> **WHY:** now that the config RS is live, mongos can bind `27017` + connect.
> **WHAT:**
> ```bash
> systemctl enable --now nexus-mongos
> ```
> **EXPECTED:** mongos binds `27017` (it couldn't before the config RS was up).
> **VERIFY:** `systemctl is-active nexus-mongos` → `active`;
> `ss -ltn '( sport = :27017 )'` shows a listener.

### 5.5 — Register shards + shard a collection (the exit gate)

> **Step 5.5.1 — Create the `nexus-sharded-admin` cluster user (the N9 workaround)**
> **WHERE:** `mongo-cfg-1` (`.74`), root shell.
> **WHY:** sharded-cluster client users live in `admin` **on the config servers**;
> mongos validates client auth against them. `__system` can't authenticate
> *through* mongos (it uses `local`, which mongos rejects). So create a `root`
> user — password = the keyFile content — on the **config-server PRIMARY** via
> `__system`+`local` (allowed on mongod), then use it for all mongos ops.
> **WHAT:**
> ```bash
> KEY=$(sudo cat /etc/nexus-mongo/keyfile)
> TLS='--tls --tlsCAFile /etc/nexus-mongo/tls/ca.crt --tlsCertificateKeyFile /etc/nexus-mongo/tls/server.pem'
> CFG_RS="mongodb://192.168.70.74:27019,192.168.70.75:27019,192.168.70.76:27019/admin?replicaSet=config"
> SYS="--username __system --password $KEY --authenticationDatabase local --authenticationMechanism SCRAM-SHA-256"
> sudo mongosh --quiet $TLS $SYS "$CFG_RS" --eval \
>   "db.getSiblingDB('admin').createUser({user:'nexus-sharded-admin', pwd:'$KEY', roles:[{role:'root',db:'admin'}]}); print('CREATED')"
> ```
> **EXPECTED:** `CREATED` (or "already exists" on a re-run — fine).
> **VERIFY:** auth as the new user **through mongos** (over mTLS) works:
> `sudo mongosh --quiet $TLS --host 192.168.70.58:27017 --username nexus-sharded-admin --password "$KEY" --authenticationDatabase admin --eval 'db.adminCommand({ping:1}).ok'`
> → `1`.

> **Step 5.5.2 — `sh.addShard` both shards (via mongos, as the cluster admin)**
> **WHERE:** `mongo-mongos-1` (`.58`), root shell.
> **WHY:** register each shard RS with the cluster. The shard URI is
> `<rs>/host:port,…`. Idempotent (errors "already exists" if re-run).
> **WHAT:**
> ```bash
> KEY=$(sudo cat /etc/nexus-mongo/keyfile)
> TLS='--tls --tlsCAFile /etc/nexus-mongo/tls/ca.crt --tlsCertificateKeyFile /etc/nexus-mongo/tls/server.pem'
> AUTH="--username nexus-sharded-admin --password $KEY --authenticationDatabase admin"
> M="sudo mongosh --quiet $TLS --host 192.168.70.58:27017 $AUTH --eval"
> # addShard URIs stay by-IP — the leaf certs carry IP-SANs, so mTLS validates by IP.
> $M "sh.addShard('shard-1/192.168.70.77:27018,192.168.70.78:27018,192.168.70.79:27018')"
> $M "sh.addShard('shard-2/192.168.70.80:27018,192.168.70.56:27018,192.168.70.57:27018')"
> ```
> **EXPECTED:** each `addShard` returns `ok: 1`.
> **VERIFY:** `$M "sh.status()"` lists **both** `shard-1` + `shard-2` with the
> balancer enabled.

> **Step 5.5.3 — Shard a collection + 200-doc round-trip (exit gate)**
> **WHERE:** `mongo-mongos-1`, root shell.
> **WHY:** prove sharding works end-to-end — enable sharding on a DB, shard a
> collection on a **hashed** key, insert 200 docs, and confirm they're **chunked
> across both shards** and routable from either mongos.
> **WHAT:**
> ```bash
> $M "sh.enableSharding('nexus_n_smoke'); \
>     db.getSiblingDB('nexus_n_smoke').samples.createIndex({k:'hashed'}); \
>     sh.shardCollection('nexus_n_smoke.samples',{k:'hashed'}); \
>     var b=[]; for(var i=0;i<200;i++){b.push({k:i,v:'data-'+i})}; \
>     db.getSiblingDB('nexus_n_smoke').samples.insertMany(b); \
>     print('INSERTED='+db.getSiblingDB('nexus_n_smoke').samples.countDocuments())"
> ```
> **EXPECTED:** `INSERTED=200`.
> **VERIFY:** chunks landed on **both** shards + the **other** mongos routes too:
> ```bash
> $M "db.getSiblingDB('nexus_n_smoke').samples.getShardDistribution()"   # both shards listed, non-zero docs each
> # via mongos-2 (.59), over mTLS:
> sudo mongosh --quiet $TLS --host 192.168.70.59:27017 $AUTH --eval "print(db.getSiblingDB('nexus_n_smoke').samples.countDocuments())"   # 200
> ```

---

## 6. Validation — by-hand acceptance smoke

Mirrors the 50-check `smoke-0.N.ps1`, condensed. From the **host** (`ssh` +
`sudo mongosh`; auth `__system` on mongod ports, `nexus-sharded-admin` on mongos).
**Every `mongosh` carries the `--tls` triplet** (`--tls --tlsCAFile
/etc/nexus-mongo/tls/ca.crt --tlsCertificateKeyFile /etc/nexus-mongo/tls/server.pem`)
— `requireTLS` refuses any plain connection.

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | 11 nodes reachable on their role port | `Test-NetConnection` per node/port (27019/27018/27017) | all `True` |
| 2 | **TLS required** (plain, non-TLS connection refused) | `ssh …@74 'sudo mongosh --quiet --host 127.0.0.1:27019 --eval "1"'` (**no** `--tls`) | connection error (`requireTLS`) |
| 3 | Config RS healthy | `rs.status()` on `mongo-cfg-1:27019` (`$TLS` + `__system`) | 1 PRIMARY + 2 SECONDARY |
| 4 | shard-1 RS healthy | `rs.status()` on `.77:27018` (`$TLS` + `__system`) | 1 PRIMARY + 2 SECONDARY |
| 5 | shard-2 RS healthy | `rs.status()` on `.80:27018` (`$TLS` + `__system`) | 1 PRIMARY + 2 SECONDARY |
| 6 | mongos bound | `ss -ltn` on `.58`/`.59` | `:27017` listening on both |
| 7 | Both shards registered | `sh.status()` via mongos (`$TLS` + admin user) | `shard-1` + `shard-2` present, balancer on |
| 8 | Collection sharded + distributed | `getShardDistribution()` (`$TLS` + admin) | docs on **both** shards |
| 9 | 200 docs present | `countDocuments()` via mongos (`$TLS` + admin) | `200` |
| 10 | **Both routers route** | `countDocuments()` via mongos-1 **and** mongos-2 (`$TLS`) | `200` from each |
| 11 | **Shard-primary failover** | stop `nexus-mongo` on a shard PRIMARY; re-query via mongos | a SECONDARY promotes; queries still return (then restart) |

**1–10 green ⇒ Guide 12 satisfied** (= the spirit of the 50/50 gate). 11 is the
per-shard HA proof. **This completes the OLTP tier (guides 07–12).**

---

## 7. Teardown / reset

```bash
for ip in 58 59; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-mongos'; done
for ip in 74 75 76 77 78 79 80 56 57; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-mongo'; done
# then vmrun stop + deleteVM each of the 11 (Guide 00 §7).
```

> Cold rebuild has no stale-state prerequisite beyond the keyFile (in Vault KV,
> reused). A fresh `rs.initiate` + `sh.addShard` rebuilds the topology; old data
> is gone with the disks. (The automated lab's `-parallelism=3` first-apply trick
> is a VMware-clone-storm workaround — irrelevant by hand, where you power VMs on
> as you build them.)

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Auth through mongos fails / `Can't use 'local' database through mongos` | `__system` auth uses `local`, which mongos rejects | create `nexus-sharded-admin` (root) in `admin` on the **config-server PRIMARY** via `__system`, then auth mongos ops as that user (§5.5.1, N9). |
| mongos won't start / can't bind `27017` | the **config RS isn't initiated yet** | initiate the config RS first (§5.4.1), *then* start mongos (§5.4.3) — N7. |
| `createUser`/first user fails "Unauthorized" from localhost | MongoDB 8.0 + keyFile + authz disables the localhost exception | bootstrap via `__system` (same as Guide 08). |
| mongod/mongos won't start: "permissions on keyFile are too open" | keyFile not `0400`/`mongodb`-owned | `sudo install -o mongodb -g mongodb -m 0400 …` (§5.2.1). |
| Shards register but data lands on one shard only | range shard key + monotonic inserts hot-spot | use a **hashed** shard key (`{k:'hashed'}`) (§5.5.3). |
| `sh.addShard` errors "already exists" on re-run | idempotent re-apply | benign — the shard is already registered. |
| `mongosh` AccessDenied reading the keyFile **or** the TLS key | keyFile is `0400 mongodb:mongodb`; TLS material is `0640 root:mongodb` | `sudo` every `mongosh` that reads them. |
| mongod/mongos won't start: TLS cert error | `server.pem` missing the key, or key is PKCS#1, or CA chain incomplete | `server.pem` = leaf + **PKCS#8** key concatenated; `ca.crt` = intermediate + root (§5.2.2). |
| `mongosh` connection refused / "no SSL certificate provided" | `requireTLS` is on but the client sent no cert | add the `--tls --tlsCAFile … --tlsCertificateKeyFile …` triplet to **every** `mongosh` (§5.4/§5.5). |
| TLS handshake fails when dialing a backplane/service IP | leaf lacks that IP-SAN | reissue from `mongo-sharded-server` with `ip_sans` covering **both** NICs + `127.0.0.1` (§5.2.2); `addShard` URIs stay by-IP since the certs carry IP-SANs. |
| Leaves near expiry (90-day TTL) — rotate without downtime | on-disk `server.pem` must be swapped + reloaded | reissue (§5.2.2), replace the on-disk `server.pem`, then `db.adminCommand({rotateCertificates:1})` on each **mongod** and **mongos** — it reloads the cert **online, with NO re-election** (0.N.1; a sharded-topology win Guide 08's replica-set rotation doesn't cover). |
| Config-DB URI rejected by mongos | wrong RS name or port in `configDB` | must be `config/<ip>:27019,…` matching the config RS name + port (§5.3.2). |

---

## 9. Production tuning — MongoDB (sharded)

> **Everything below is *beyond the lab replica*.** §5 ships the lab-scale values on
> 2 GB VMs (WiredTiger cache `0.5 GB`, uniform across config + shard nodes; no oplog
> sizing; default write/read concern); this section is what you would change for a
> **production** sharded cluster and *why*. It **never alters the §5 configs**. Do not
> paste these onto the 2 GB lab VMs blindly — several assume production-sized RAM +
> dedicated disks. For the **OS layer** (THP, swappiness, `nofile`, XFS, readahead, I/O
> scheduler), set it **once per Guide 00 §9**; below restates only the MongoDB-specific
> overrides.

### 9.1 OS layer (per Guide 00 §9 — MongoDB-specific requirements)

Set on **all 11 nodes** as a dedicated DB-host role. Full snippets live in Guide 00 §9;
only the Mongo-relevant knobs are called out here.

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| Transparent Huge Pages | **`never`** (Guide 00 §9.2) | unset (`madvise`/`always`) | **⚠️ required by MongoDB** — WiredTiger logs a startup warning under THP; `khugepaged` compaction causes multi-ms allocation stalls + RSS bloat under DB access patterns. |
| Filesystem for `dbPath` | **XFS** (Guide 00 §9.4) | ext4 (lab default) | MongoDB explicitly recommends XFS — WiredTiger + XFS avoids ext4 stall pathologies at high write concurrency on the shard data nodes. |
| `read_ahead_kb` | `128` (Guide 00 §9.4) | unset (`128`–`256`) | WiredTiger does random-access I/O; large readahead wastes bandwidth pulling pages you won't use. |
| `vm.swappiness` | `1` (Guide 00 §9.1) | unset (`60`) | Keeps the WiredTiger cache + working set in RAM; `60` swaps hot pages under cache pressure → latency cliffs. |
| `nofile` (open files) | soft `65536` / hard `1048576` | **PRESENT** — `LimitNOFILE=64000` in **both** units (§5.1.2) | A sharded node fans out many inter-member + client connections; the default `1024` soft cap → `Too many open files`. The lab already sets it via systemd, so it is *not* deferred. |
| `numactl --interleave=all` | wrap the `mongod`/`mongos` ExecStart | unset (single-socket lab VMs) | On a multi-socket box, an unbalanced NUMA allocation forces WiredTiger onto remote memory (or swaps when one node fills) → erratic latency; Mongo prescribes interleaved allocation. |

```bash
# PRODUCTION — NUMA interleave (multi-socket hosts only); NOT applied in the lab.
apt-get install -y numactl
# Edit BOTH units' ExecStart (nexus-mongo.service AND nexus-mongos.service), e.g.:
#   ExecStart=/usr/bin/numactl --interleave=all /usr/bin/mongod --config /etc/nexus-mongo/mongod.conf
#   ExecStart=/usr/bin/numactl --interleave=all /usr/bin/mongos --config /etc/nexus-mongo/mongos.conf
systemctl daemon-reload && systemctl restart nexus-mongo    # (or nexus-mongos)
```

### 9.2 Engine layer — mongod / mongos

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `storage.wiredTiger.engineConfig.cacheSizeGB` (**shard** nodes) | **≈ 50% of (RAM − 1 GB)**, pinned per RAM | `0.5` | The shards hold the actual collection data + do the query/index work — give WiredTiger the cache. Too small → constant eviction + disk reads. |
| `storage.wiredTiger.engineConfig.cacheSizeGB` (**config** servers) | **small (`1–2 GB`)**, well below the shards | `0.5` (uniform) | Config servers only hold cluster metadata (shard list + chunk map) — tiny vs. shard data. **The lab uses a uniform `0.5` everywhere; production sizes the shard cache far above the config-server cache** because the shards do the real work. |
| `replication.oplogSizeMB` (each RS) | **sized for peak write burst** (often 10–50 GB) | unset (default ≈ 5% of free disk) | The oplog is the replication + resync window. Too small on a write-heavy shard → a lagging SECONDARY falls off the oplog and needs a **full resync**. Size per shard's write rate. |
| cluster-wide default **write concern** | **`{w:'majority'}`** (`setDefaultRWConcern`) | unset (engine default, not pinned) | Guarantees an ack'd write survives a shard-primary failover — critical while the balancer + failovers move data. Pin it so it can't drift. |
| default **`readConcern`** | **`majority`** | unset (`local`) | `local` can read data a subsequent failover rolls back; `majority` reads only committed data — matches the write concern above. |
| `chunkSize` (sharding, MB) | **default `128`**; lower (`64`) only for many small shards | unset (default `128`) | Chunk granularity for the balancer. Smaller = more even distribution but more migration churn + metadata; leave at `128` unless the shard key yields jumbo chunks. |
| balancer window | **off-peak only** (`config.settings` `activeWindow`) | unset (balancer runs anytime) | Chunk migrations add I/O + lock overhead; confine them to a low-traffic window so they don't compete with peak OLTP load. |

```bash
# PRODUCTION — cluster-wide majority write+read concern (run once via a mongos, as the
# cluster admin, over mTLS). NOT applied in the lab.
db.adminCommand({ setDefaultRWConcern: 1,
  defaultWriteConcern: { w: 'majority', wtimeout: 5000 },
  defaultReadConcern:  { level: 'majority' } })

# PRODUCTION — confine the balancer to a 01:00–05:00 window (via a mongos). NOT applied in the lab.
use config
db.settings.updateOne({ _id: 'balancer' },
  { $set: { activeWindow: { start: '01:00', stop: '05:00' } } }, { upsert: true })

# PRODUCTION — per-shard WiredTiger cache (SHARD-node mongod.conf; config servers stay small).
# NOT applied in the lab (lab uses a uniform 0.5 everywhere):
#   storage:
#     wiredTiger:
#       engineConfig:
#         cacheSizeGB: <≈ 0.5 × (RAM_GB − 1)>   # shards
#         # config servers: keep 1–2 GB, far below the shards
```

> **The OS layer is shared** — set THP-off + XFS + low readahead + swappiness once per
> Guide 00 §9; the tables above add only the MongoDB-specific engine overrides
> (WiredTiger cache split, oplog, write/read concern, chunk/balancer, NUMA).

---

### Cross-references

- **0.N architecture + transients:** memory `project_nexus_infra_0n_phase`; ADR-0040 (sharded topology)
- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (Mongo-sharded `.56`–`.59`, `.74`–`.80`)
- **Automated equivalents:** `nexus-infra-oltp/packer/oltp-mongo-node/` + `terraform/envs/oltp-mongo-sharded/role-overlay-*.tf`
- **Builds on:** [`08-oltp-mongodb-replica-set.md`](./08-oltp-mongodb-replica-set.md) (keyFile + `__system` patterns)
- **Previous guide:** [`11-oltp-sqlserver-fci-ag.md`](./11-oltp-sqlserver-fci-ag.md)
- **Next guide:** Guide 13 — Analytics · ClickHouse (3 shards × 2 replicas + 3-node ClickHouse Keeper). See [`INDEX.md`](../INDEX.md). **(OLTP tier 07–12 complete; the guides now move into Analytics.)**
