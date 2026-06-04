# Guide 12 — OLTP · MongoDB sharded (3 config + 2 shards×3 + 2 mongos)

> **Mirrors:** `nexus-infra-oltp` — the `oltp-mongo-node` Packer template
> (extended with `mongodb-org-mongos`) + the `oltp-mongo-sharded` env overlays
> (`mongo-keyfile`, `mongo-config`, `mongo-rs-initiate`, `mongo-add-shards`,
> `mongo-nftables-backplane`) — Phase 0.N / ADR-0040. The automated lab distributes
> the keyFile + drives `rs.initiate`/`sh.addShard` over SSH; by hand we do the same
> with `mongosh` directly. Builds on **Guide 08**'s replica-set + keyFile +
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
keep the shards balanced. Security is **keyFile internal auth** (the same shared
secret across all 11 nodes) — **no mTLS in this v1** (deferred to a 0.N.1
hardening; contrast Guide 08, which had mTLS).

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
- **keyFile internal auth (no mTLS in v1).** The same 6–1024-char shared secret on
  all 11 nodes authenticates member-to-member traffic and implicitly enables
  authorization. `clusterAuthMode: keyFile`. *Why no TLS here:* v1 scope — keyFile
  proves internal auth + the sharding topology; mTLS is the 0.N.1 follow-up.
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
| 2 | 11 `deb13` nodes baselined, dual-NIC static (`.74–.80`, `.56/.57/.58/.59`) | all 11 answer `:22` |
| 3 | Vault root token on build host | `Test-Path ~/.nexus/secrets/vault-cluster-init.json` |
| 4 | Internet egress on the nodes | `ssh …@74 'curl -sI https://repo.mongodb.org \| head -1'` → `200` |

> MongoDB **8.0**. Ports: config `27019`, shards `27018`, mongos `27017`. keyFile
> at `/etc/nexus-mongo/keyfile` (`0400 mongodb:mongodb`). MAC pool `:C0–:CA`.

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
> install -d -o mongodb -g mongodb -m0750 /etc/nexus-mongo /var/lib/nexus-mongo /var/log/nexus-mongo
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

### 5.2 — Distribute the shared keyFile (all 11)

> **Step 5.2.1 — Place the keyFile on every node (`0400 mongodb:mongodb`)**
> **WHERE:** build host (fetch from KV), then each node.
> **WHY:** all 11 members authenticate to each other with the **same** keyFile.
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
> `shardsvr`), `replSetName`, and `port`. keyFile auth + `clusterAuthMode: keyFile`.
> **WHAT (config node — `clusterRole: configsvr`, `replSetName: config`, port `27019`):**
> ```bash
> cat > /etc/nexus-mongo/mongod.conf <<'EOF'
> net:
>   bindIp: 0.0.0.0,::1
>   port: 27019
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
> **VERIFY:** `systemctl is-active nexus-mongo` → `active`;
> `sudo mongosh --quiet --port <role-port> --eval 'db.adminCommand({ping:1}).ok'` → `1`.

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
> sudo mongosh --quiet --port 27019 --eval "printjson(rs.initiate({_id:'config', configsvr:true, members:[{_id:0,host:'192.168.70.74:27019'},{_id:1,host:'192.168.70.75:27019'},{_id:2,host:'192.168.70.76:27019'}]}))"
> ```
> **EXPECTED:** `ok: 1`; a PRIMARY elects in ~5 s.
> **VERIFY (auth as `__system`):**
> `KEY=$(sudo cat /etc/nexus-mongo/keyfile); sudo mongosh --quiet --port 27019 --username __system --password "$KEY" --authenticationDatabase local --authenticationMechanism SCRAM-SHA-256 --eval "rs.status().members.filter(m=>m.stateStr=='PRIMARY').length"`
> → `1`.

> **Step 5.4.2 — `rs.initiate()` shard-1 + shard-2**
> **WHERE:** `mongo-shard-1-1` (`.77`) + `mongo-shard-2-1` (`.80`), root shell.
> **WHY:** each shard is its own 3-member RS. (No `configsvr` flag.)
> **WHAT (on `mongo-shard-1-1`; repeat on `mongo-shard-2-1` with shard-2's members):**
> ```bash
> # shard-1 (on .77):
> sudo mongosh --quiet --port 27018 --eval "printjson(rs.initiate({_id:'shard-1', members:[{_id:0,host:'192.168.70.77:27018'},{_id:1,host:'192.168.70.78:27018'},{_id:2,host:'192.168.70.79:27018'}]}))"
> # shard-2 (on .80):
> #   rs.initiate({_id:'shard-2', members:[{_id:0,host:'192.168.70.80:27018'},{_id:1,host:'192.168.70.56:27018'},{_id:2,host:'192.168.70.57:27018'}]})
> ```
> **EXPECTED:** `ok: 1` for each; each elects a PRIMARY.
> **VERIFY:** on each shard PRIMARY (auth `__system`, port 27018): `rs.status()`
> → 1 PRIMARY + 2 SECONDARY.

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
> CFG_RS="mongodb://192.168.70.74:27019,192.168.70.75:27019,192.168.70.76:27019/admin?replicaSet=config"
> SYS="--username __system --password $KEY --authenticationDatabase local --authenticationMechanism SCRAM-SHA-256"
> sudo mongosh --quiet $SYS "$CFG_RS" --eval \
>   "db.getSiblingDB('admin').createUser({user:'nexus-sharded-admin', pwd:'$KEY', roles:[{role:'root',db:'admin'}]}); print('CREATED')"
> ```
> **EXPECTED:** `CREATED` (or "already exists" on a re-run — fine).
> **VERIFY:** auth as the new user **through mongos** works:
> `sudo mongosh --quiet --host 192.168.70.58:27017 --username nexus-sharded-admin --password "$KEY" --authenticationDatabase admin --eval 'db.adminCommand({ping:1}).ok'`
> → `1`.

> **Step 5.5.2 — `sh.addShard` both shards (via mongos, as the cluster admin)**
> **WHERE:** `mongo-mongos-1` (`.58`), root shell.
> **WHY:** register each shard RS with the cluster. The shard URI is
> `<rs>/host:port,…`. Idempotent (errors "already exists" if re-run).
> **WHAT:**
> ```bash
> KEY=$(sudo cat /etc/nexus-mongo/keyfile)
> AUTH="--username nexus-sharded-admin --password $KEY --authenticationDatabase admin"
> M="sudo mongosh --quiet --host 192.168.70.58:27017 $AUTH --eval"
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
> # via mongos-2 (.59):
> sudo mongosh --quiet --host 192.168.70.59:27017 $AUTH --eval "print(db.getSiblingDB('nexus_n_smoke').samples.countDocuments())"   # 200
> ```

---

## 6. Validation — by-hand acceptance smoke

Mirrors the 50-check `smoke-0.N.ps1`, condensed. From the **host** (`ssh` +
`sudo mongosh`; auth `__system` on mongod ports, `nexus-sharded-admin` on mongos).

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | 11 nodes reachable on their role port | `Test-NetConnection` per node/port (27019/27018/27017) | all `True` |
| 2 | Config RS healthy | `rs.status()` on `mongo-cfg-1:27019` (`__system`) | 1 PRIMARY + 2 SECONDARY |
| 3 | shard-1 RS healthy | `rs.status()` on `.77:27018` | 1 PRIMARY + 2 SECONDARY |
| 4 | shard-2 RS healthy | `rs.status()` on `.80:27018` | 1 PRIMARY + 2 SECONDARY |
| 5 | mongos bound | `ss -ltn` on `.58`/`.59` | `:27017` listening on both |
| 6 | Both shards registered | `sh.status()` via mongos (admin user) | `shard-1` + `shard-2` present, balancer on |
| 7 | Collection sharded + distributed | `getShardDistribution()` | docs on **both** shards |
| 8 | 200 docs present | `countDocuments()` via mongos | `200` |
| 9 | **Both routers route** | `countDocuments()` via mongos-1 **and** mongos-2 | `200` from each |
| 10 | **Shard-primary failover** | stop `nexus-mongo` on a shard PRIMARY; re-query via mongos | a SECONDARY promotes; queries still return (then restart) |

**1–9 green ⇒ Guide 12 satisfied** (= the spirit of the 50/50 gate). 10 is the
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
| `mongosh` AccessDenied reading the keyFile | keyFile is `0400 mongodb:mongodb` | `sudo` every `mongosh` that reads it. |
| Config-DB URI rejected by mongos | wrong RS name or port in `configDB` | must be `config/<ip>:27019,…` matching the config RS name + port (§5.3.2). |

---

### Cross-references

- **0.N architecture + transients:** memory `project_nexus_infra_0n_phase`; ADR-0040 (sharded topology)
- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (Mongo-sharded `.56`–`.59`, `.74`–`.80`)
- **Automated equivalents:** `nexus-infra-oltp/packer/oltp-mongo-node/` + `terraform/envs/oltp-mongo-sharded/role-overlay-*.tf`
- **Builds on:** [`08-oltp-mongodb-replica-set.md`](./08-oltp-mongodb-replica-set.md) (keyFile + `__system` patterns)
- **Previous guide:** [`11-oltp-sqlserver-fci-ag.md`](./11-oltp-sqlserver-fci-ag.md)
- **Next guide:** Guide 13 — Analytics · ClickHouse (3 shards × 2 replicas + 3-node ClickHouse Keeper). See [`INDEX.md`](../INDEX.md). **(OLTP tier 07–12 complete; the guides now move into Analytics.)**
