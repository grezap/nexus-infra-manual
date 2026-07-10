# Guide 08 — OLTP · MongoDB replica set (3-node, keyFile + mTLS x509)

> **Mirrors:** `nexus-infra-oltp` — the `oltp-mongo-node` Packer template
> (`nexus-mongo.service`) + the `oltp-mongo` env overlays (`mongo-tls`,
> `mongo-config`, `mongo-rs-initiate`, `mongo-nftables-backplane`,
> `mongo-vault-agents`). Where the automated lab renders certs/keyFile via a
> Vault Agent and bootstraps over SSH, this guide installs MongoDB by hand and
> **issues the cert + generates the keyFile directly**.

---

## 1. Overview & purpose

A 3-node **MongoDB replica set** (`nexus-rs`) — one PRIMARY + two SECONDARYs —
running **mutual TLS** for the wire **and** **keyFile internal auth** for
member-to-member traffic, with authorization enabled. It's the lab's document
store.

A replica set gives MongoDB its HA: all writes go to the PRIMARY and replicate to
the SECONDARYs; if the PRIMARY fails, the members elect a new one. Two security
layers: **mTLS x509** (every client + member presents a cert) and a shared
**keyFile** (the internal-auth secret members use to authenticate to each other).

**The headline gotcha — and why this guide is interesting:** on MongoDB 8.0, with
`keyFile` + `authorization: enabled`, the **localhost exception does not
activate** — so you cannot just create the first user from localhost. The
bootstrap path is to authenticate as the internal **`__system`** user
(SCRAM-SHA-256, password = the keyFile content) to create the first real user,
then switch to that user. This guide reproduces that exactly.

**Dependency:**
- **Guides 00–04** — foundation alive; Guide 04's PKI + Part-D scaffolding (this
  guide creates the `mongo-server` PKI role).
- 3 `deb13` nodes baselined per Guide 00 §5.B, dual-NIC static (`.71/.72/.73`).

> **By-hand divergence:** issue the cert with the `vault` CLI, generate the
> keyFile once + distribute it, and place the smoke-user password file — no
> per-node Vault Agent.

---

## 2. Component primer

- **MongoDB replica set.** A group of `mongod` instances holding the same data:
  one **PRIMARY** (takes writes) + N **SECONDARYs** (replicate + serve reads). A
  failed PRIMARY triggers an election among the survivors. *Why:* HA + read
  scaling. *Otherwise:* standalone `mongod` (no failover) or a sharded cluster
  (Guide 12 — built on top of replica sets).
- **`rs.initiate()`.** The one-time command that tells the members their topology
  (the `members[]` list, by host:port). It's special: it's allowed **without
  auth** even when auth is on, because it's the bootstrap pre-condition.
- **mTLS x509 (`requireTLS`).** `net.tls.mode: requireTLS` refuses any non-TLS
  connection; `allowConnectionsWithoutCertificates: false` makes every client
  present a cert. `--tlsCertificateKeyFile` takes **one** PEM with the leaf +
  key concatenated (unlike Redis's three files). *Why:* encrypted + mutually
  authenticated wire.
- **keyFile internal auth.** A shared secret (6–1024 base64 chars) that replica-
  set members use to authenticate to **each other**. Setting `security.keyFile`
  *implicitly enables* `authorization`. *Why:* member traffic must be
  authenticated, not just clients. *Otherwise:* x509 member auth (more cert
  plumbing; keyFile is simpler for a lab).
- **The localhost exception (and why it's off here).** Normally MongoDB lets you
  create the *first* user from localhost before auth is enforced. **On 8.0 with
  `keyFile` + `authorization: enabled`, that exception does not activate** — the
  runtime check still requires auth. So the first-user bootstrap can't use it.
- **`__system` bootstrap.** `__system` is MongoDB's internal cluster user
  (root-equivalent). You can authenticate as it with **SCRAM-SHA-256**, username
  `__system`, authdb `local`, password = **the keyFile content**. Used **once**
  to `createUser` the real `smoke-rw` account; after that everything auths as
  `smoke-rw` (`readWrite` on `nexus_smoke`). *MongoDB calls `__system` operator
  use "discouraged but supported"* — fine for a one-shot bootstrap.
- **The `sudo` + `$set`-escaping rules.** TLS material is `0640 root:mongodb` →
  `sudo` every `mongosh` that reads it. And in an `updateOne({$set:...})` issued
  through an SSH double-quoted envelope, bash expands `$set` to empty → escape it
  (`\$set`) — a real transient.

---

## 3. Prerequisites

| # | Requirement | One-command verify |
|---|---|---|
| 1 | Foundation alive (Guides 00–04); Vault PKI usable | `vault read pki_int/cert/ca` on vault-1 returns the intermediate |
| 2 | 3 `deb13` nodes baselined, dual-NIC static `.71/.72/.73` | those 3 answer `:22` |
| 3 | Vault root token on build host | `Test-Path ~/.nexus/secrets/vault-cluster-init.json` |
| 4 | Internet egress on the nodes | `ssh …@71 'curl -sI https://repo.mongodb.org \| head -1'` → `200` |

> MongoDB version: **8.0**. Replica set name `nexus-rs`. Data port `27017` (TLS).

---

## 4. Target topology

| Node | Role (after init) | VMnet11 | VMnet10 | vCPU/RAM/disk |
|---|---|---|---|---|
| `mongo-1` | initial PRIMARY (member `_id:0`) | `.71` | `.10.71` | 2 / 2 GB / 40 GB |
| `mongo-2` | SECONDARY (member `_id:1`) | `.72` | `.10.72` | 2 / 2 GB / 40 GB |
| `mongo-3` | SECONDARY (member `_id:2`) | `.73` | `.10.73` | 2 / 2 GB / 40 GB |

> Data + replication on `27017` (TLS), VMnet11. Cert: `/etc/nexus-mongo/tls/server.pem`
> (leaf + PKCS#8 key combined) + `ca.crt` (intermediate + root). keyFile:
> `/etc/nexus-mongo/keyfile` (`0400 mongodb:mongodb`). PKI role **`mongo-server`**
> (`client_flag=true`, 90-day TTL). Smoke user `smoke-rw` (`readWrite` on
> `nexus_smoke`).

---

## 5. Step-by-step build

> **WHERE:** node steps as `nexusadmin`→`sudo -i`; `mongosh` under `sudo`. `vault`
> commands on **`vault-1`** (root token). Bootstrap from `mongo-1`.

### 5.1 — Per-node base install (all 3)

> **Step 5.1.1 — Install MongoDB 8.0 + create dirs**
> **WHERE:** each node, root shell.
> **WHY:** the `mongod` + `mongosh` packages (the package creates the `mongodb`
> user); the dedicated config/data/log dirs.
> **WHAT:**
> ```bash
> apt-get update -qq && apt-get install -y gnupg curl openssl
> curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | gpg --dearmor -o /usr/share/keyrings/mongodb-8.0.gpg
> echo "deb [signed-by=/usr/share/keyrings/mongodb-8.0.gpg] https://repo.mongodb.org/apt/debian trixie/mongodb-org/8.0 main" \
>   > /etc/apt/sources.list.d/mongodb-org-8.0.list
> apt-get update -qq && apt-get install -y mongodb-org
> systemctl disable --now mongod 2>/dev/null || true   # we run our own nexus-mongo unit
> install -d -o mongodb -g mongodb -m0750 /etc/nexus-mongo /etc/nexus-mongo/tls /var/lib/nexus-mongo /var/log/nexus-mongo
> install -d -o mongodb -g mongodb -m0755 /var/run/nexus-mongo
> ```
> **EXPECTED:** MongoDB installs; the stock `mongod` unit disabled.
> **VERIFY:** `mongod --version | head -1` → `db version v8.0.x`.

> **Step 5.1.2 — Install the `nexus-mongo` systemd unit + open `27017`**
> **WHERE:** each node, root shell.
> **WHY:** our unit points at our config; `RuntimeDirectory=` recreates the tmpfs
> `/var/run/nexus-mongo`. nftables opens `27017` on VMnet11.
> **WHAT:**
> ```bash
> cat > /etc/systemd/system/nexus-mongo.service <<'EOF'
> [Unit]
> Description=Nexus MongoDB replica-set node
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
> # nftables: 27017 from VMnet11 + backplane trust
> # add to /etc/nftables.conf chain input, before counter drop:
> #   iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 27017 accept comment "MongoDB"
> #   iifname "nic1" ip saddr 192.168.10.0/24 accept comment "trusted cluster backplane (VMnet10)"
> nft -c -f /etc/nftables.conf && systemctl reload nftables
> ```
> **EXPECTED:** unit installed; nftables valid.
> **VERIFY:** `nft list chain inet filter input | grep 27017`.

### 5.2 — PKI cert + keyFile + smoke-user password

> **Step 5.2.1 — Create the `mongo-server` PKI role**
> **WHERE:** `vault-1`, root shell.
> **WHY:** server + client EKU (a node's cert is also its client identity); the 3
> hostnames + `.mongo.nexus.lab` forms; 90-day leaves.
> **WHAT:**
> ```bash
> vault write pki_int/roles/mongo-server \
>   allowed_domains='nexus.lab,mongo.nexus.lab,mongo-1,mongo-2,mongo-3,localhost' \
>   allow_subdomains=true allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
> ```
> **EXPECTED:** role written.
> **VERIFY:** `vault read pki_int/roles/mongo-server | grep client_flag` → `true`.

> **Step 5.2.2 — Generate the shared keyFile once + distribute to all 3 nodes**
> **WHERE:** build host (or vault-1), then each node.
> **WHY:** the keyFile is a **single shared secret** all 3 members use for
> internal auth — generate it **once** and place the identical file on every
> node, `0400 mongodb:mongodb` (mongod refuses to start on looser perms). Store
> the content in Vault KV for the record (the `__system` bootstrap needs it).
> **WHAT:**
> ```bash
> # Generate once (1024-char base64 is the MongoDB max; 756 raw bytes -> ~1024 b64):
> openssl rand -base64 756 | tr -d '\n' > /tmp/nexus-mongo-keyfile
> # Save the content in Vault KV (so the __system bootstrap + future nodes can read it):
> vault kv put nexus/oltp/mongo/keyfile content=@/tmp/nexus-mongo-keyfile
> # Place on EACH node (scp /tmp/nexus-mongo-keyfile to the node, then):
> #   sudo install -o mongodb -g mongodb -m 0400 /tmp/nexus-mongo-keyfile /etc/nexus-mongo/keyfile
> ```
> **EXPECTED:** the same keyFile on all 3 nodes, mode `0400`.
> **VERIFY:** `sudo stat -c '%U:%G %a' /etc/nexus-mongo/keyfile` → `mongodb:mongodb 400`;
> the file is byte-identical across nodes (`sudo md5sum` matches).

> **Step 5.2.3 — Issue + place each node's `server.pem` + `ca.crt`; seed the smoke password**
> **WHERE:** issue on `vault-1`; place on each node.
> **WHY:** mongod's `--tlsCertificateKeyFile` takes **one** PEM = leaf + key
> (PKCS#8) concatenated; the CA file = intermediate + root (OpenSSL strict verify
> needs the full chain to a self-signed anchor). The smoke-user password is a
> generated secret stored in KV + a node-local file the bootstrap reads.
> **WHAT (per node — issue on vault-1, substitute host/IPs):**
> ```bash
> ISSUED=$(vault write -format=json pki_int/issue/mongo-server \
>   common_name="<host>.mongo.nexus.lab" alt_names="<host>,<host>.nexus.lab,localhost" \
>   ip_sans="<vmnet11>,<vmnet10>,127.0.0.1" ttl=2160h)
> # server.pem = leaf + PKCS#8 key (mongod's combined shape)
> echo "$ISSUED" | jq -r '.data.certificate' > /tmp/server.pem
> echo "$ISSUED" | jq -r '.data.private_key' | openssl pkcs8 -topk8 -nocrypt >> /tmp/server.pem
> # ca.crt = intermediate + root
> { echo "$ISSUED" | jq -r '.data.issuing_ca'; cat ~/.nexus/secrets/vault-root-ca.pem; } > /tmp/ca.crt
> # scp both to <host>:/etc/nexus-mongo/tls/ (owned root:mongodb, 0640)
> ```
> ```bash
> # Smoke-rw password: generate once, seed KV, place a node-local copy on each node:
> vault kv put nexus/oltp/mongo/smoke-user-password password="$(openssl rand -base64 24 | tr -dc 'a-zA-Z0-9' | head -c 24)"
> SMOKE=$(vault kv get -field=password nexus/oltp/mongo/smoke-user-password)
> # on each node: echo -n "$SMOKE" | sudo tee /etc/nexus-mongo/smoke-user-password ; sudo chmod 640 …
> ```
> **EXPECTED:** `server.pem` + `ca.crt` on each node; smoke password seeded.
> **VERIFY:** `sudo openssl x509 -in /etc/nexus-mongo/tls/server.pem -noout -subject`
> → `CN=<host>.mongo.nexus.lab`; `grep -c 'BEGIN PRIVATE KEY' /etc/nexus-mongo/tls/server.pem`
> → `1` (PKCS#8).

### 5.3 — Render `mongod.conf` + start each node

> **Step 5.3.1 — Write `mongod.conf` and start `nexus-mongo` (all 3, TLS ping)**
> **WHERE:** each node, root shell.
> **WHY:** `requireTLS` + keyFile + `authorization: enabled` + the `nexus-rs`
> replSet name. The config is **identical** on all 3 (per-member identity lives in
> `rs.initiate`'s `members[]`). WiredTiger cache capped at 0.5 GB for the 2 GB
> nodes.
> **WHAT:**
> ```bash
> cat > /etc/nexus-mongo/mongod.conf <<'EOF'
> net:
>   bindIp: 0.0.0.0,::1
>   port: 27017
>   tls:
>     mode: requireTLS
>     certificateKeyFile: /etc/nexus-mongo/tls/server.pem
>     CAFile: /etc/nexus-mongo/tls/ca.crt
>     allowConnectionsWithoutCertificates: false
>     disabledProtocols: TLS1_0,TLS1_1
> replication:
>   replSetName: nexus-rs
> security:
>   keyFile: /etc/nexus-mongo/keyfile
>   authorization: enabled
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
> systemctl enable --now nexus-mongo.service
> ```
> **EXPECTED:** the service starts (each node standalone, RS not yet formed).
> **VERIFY (self-mTLS ping — `sudo` for the cert):**
> ```bash
> sudo mongosh --quiet --tls --tlsCAFile /etc/nexus-mongo/tls/ca.crt \
>   --tlsCertificateKeyFile /etc/nexus-mongo/tls/server.pem --host 127.0.0.1:27017 \
>   --eval 'db.adminCommand({ping:1}).ok'
> ```
> → `1`. Repeat on all 3.

### 5.4 — Initiate the replica set + bootstrap the user (exit gate)

> **Step 5.4.1 — `rs.initiate()` the 3-member set**
> **WHERE:** `mongo-1` (`.71`), root shell.
> **WHY:** form `nexus-rs` with all 3 members by IP. `rs.initiate()` is the one
> command allowed **without** auth (the bootstrap exception). MongoDB elects a
> PRIMARY in ~3–5 s (mongo-1 usually wins first).
> **WHAT:**
> ```bash
> TLS='--tls --tlsCAFile /etc/nexus-mongo/tls/ca.crt --tlsCertificateKeyFile /etc/nexus-mongo/tls/server.pem'
> sudo mongosh --quiet $TLS --host 127.0.0.1:27017 --eval "printjson(rs.initiate({_id:'nexus-rs',members:[{_id:0,host:'192.168.70.71:27017'},{_id:1,host:'192.168.70.72:27017'},{_id:2,host:'192.168.70.73:27017'}]}))"
> ```
> **EXPECTED:** `ok: 1`.
> **VERIFY:** wait ~5 s, then check status (auth as `__system`, next step's args).

> **Step 5.4.2 — Wait for 1 PRIMARY + 2 SECONDARY (auth as `__system`)**
> **WHERE:** `mongo-1`, root shell.
> **WHY:** once the RS forms, `rs.status()` requires auth — the localhost
> exception is off (8.0 + keyFile + authz). Authenticate as **`__system`**
> (password = keyFile content) to read status.
> **WHAT:**
> ```bash
> TLS='--tls --tlsCAFile /etc/nexus-mongo/tls/ca.crt --tlsCertificateKeyFile /etc/nexus-mongo/tls/server.pem'
> KEY=$(sudo cat /etc/nexus-mongo/keyfile)
> SYS="--username __system --password $KEY --authenticationDatabase local --authenticationMechanism SCRAM-SHA-256"
> sudo mongosh --quiet $TLS $SYS --host 127.0.0.1:27017 --eval \
>   "var s=rs.status(); var p=0,sec=0,h=0; s.members.forEach(m=>{if(m.stateStr=='PRIMARY')p++;if(m.stateStr=='SECONDARY')sec++;if(m.health==1)h++}); print('PRIMARY='+p+' SECONDARY='+sec+' HEALTH='+h+' MEMBERS='+s.members.length)"
> ```
> **EXPECTED:** `PRIMARY=1 SECONDARY=2 HEALTH=3 MEMBERS=3`.
> **VERIFY:** the line above (retry a few times if an election is still settling).

> **Step 5.4.3 — Bootstrap the `smoke-rw` user via `__system` (the localhost-exception workaround)**
> **WHERE:** `mongo-1`, root shell.
> **WHY:** create the first real user. Since the localhost exception is off, do it
> authenticated as `__system`. `createUser` is a **write** → route it via the RS
> connection string so it lands on whichever member is currently PRIMARY.
> **WHAT:**
> ```bash
> TLS='--tls --tlsCAFile /etc/nexus-mongo/tls/ca.crt --tlsCertificateKeyFile /etc/nexus-mongo/tls/server.pem'
> KEY=$(sudo cat /etc/nexus-mongo/keyfile) ; SMOKE=$(sudo cat /etc/nexus-mongo/smoke-user-password)
> SYS="--username __system --password $KEY --authenticationDatabase local --authenticationMechanism SCRAM-SHA-256"
> RS_URI="mongodb://192.168.70.71:27017,192.168.70.72:27017,192.168.70.73:27017/admin?replicaSet=nexus-rs"
> sudo mongosh --quiet $TLS $SYS "$RS_URI" --eval \
>   "db.getSiblingDB('admin').createUser({user:'smoke-rw',pwd:'$SMOKE',roles:[{role:'readWrite',db:'nexus_smoke'}]}); print('CREATE_OK')"
> ```
> **EXPECTED:** `CREATE_OK` (or "already exists" on a re-run — treat as success).
> **VERIFY:** auth as `smoke-rw` works:
> `sudo mongosh --quiet $TLS --username smoke-rw --password "$SMOKE" --authenticationDatabase admin --host 127.0.0.1:27017 --eval 'db.runCommand({ping:1}).ok'`
> → `1`.

> **Step 5.4.4 — Write/read round-trip across the RS (the exit gate)**
> **WHERE:** `mongo-1`, root shell.
> **WHY:** prove replication + mTLS + auth end-to-end: write on the PRIMARY (auto-
> routed), read from a SECONDARY with `readConcern: majority`. **Note the
> `\$set` escape** — through the SSH double-quote envelope bash would expand
> `$set` to empty; escape it.
> **WHAT:**
> ```bash
> TLS='--tls --tlsCAFile /etc/nexus-mongo/tls/ca.crt --tlsCertificateKeyFile /etc/nexus-mongo/tls/server.pem'
> SMOKE=$(sudo cat /etc/nexus-mongo/smoke-user-password)
> AUTH="--username smoke-rw --password $SMOKE --authenticationDatabase admin"
> TOKEN="smoke-$(date +%s)"
> # write to PRIMARY (readPreference=primary default):
> WURI="mongodb://192.168.70.71:27017,192.168.70.72:27017,192.168.70.73:27017/nexus_smoke?replicaSet=nexus-rs"
> sudo mongosh --quiet $TLS $AUTH "$WURI" --eval "db.smoke.updateOne({_id:'k'},{\$set:{token:'$TOKEN'}},{upsert:true}); print('WROTE')"
> # read from a SECONDARY with majority read concern:
> RURI="mongodb://192.168.70.71:27017,192.168.70.72:27017,192.168.70.73:27017/nexus_smoke?replicaSet=nexus-rs&readPreference=secondary&readConcernLevel=majority"
> sleep 3
> sudo mongosh --quiet $TLS $AUTH "$RURI" --eval "print('READ='+db.smoke.findOne({_id:'k'}).token)"
> ```
> **EXPECTED:** `WROTE` then `READ=smoke-<ts>` matching the token.
> **VERIFY:** the read token equals the written token → replication confirmed.
> This is the Phase 0.G.2 exit gate.

---

## 6. Validation — by-hand acceptance smoke

From the **host** (via `ssh …@71` + `sudo mongosh`).

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | All 3 reachable on `27017` | `71,72,73 \| % { Test-NetConnection 192.168.70.$_ -Port 27017 }` | all `True` |
| 2 | TLS required | `ssh …@71 'sudo mongosh --quiet --host 127.0.0.1 --eval "1"'` (no TLS) | connection error |
| 3 | Self-mTLS ping (each node) | `sudo mongosh $TLS … ping` on each | `1` ×3 |
| 4 | RS shape | `rs.status()` via `__system` | 1 PRIMARY + 2 SECONDARY, all health 1 |
| 5 | `smoke-rw` auth works | `sudo mongosh $TLS --username smoke-rw … ping` | `1` |
| 6 | Write→PRIMARY, read←SECONDARY | round-trip (§5.4.4) | token matches |
| 7 | keyFile perms | `sudo stat -c '%a %U:%G' /etc/nexus-mongo/keyfile` (each node) | `400 mongodb:mongodb` |
| 8 | **Election on PRIMARY loss** | `sudo systemctl stop nexus-mongo` on the PRIMARY; `rs.status()` from another node | a SECONDARY becomes PRIMARY; then restart the stopped node |

**1–6 green ⇒ Guide 08 satisfied.** 8 is the HA proof (failover election).

---

## 7. Teardown / reset

```bash
for ip in 71 72 73; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-mongo.service'; done
# then vmrun stop + deleteVM each of the 3 (Guide 00 §7).
```
To wipe RS state but keep the VMs: on each node `sudo rm -rf /var/lib/nexus-mongo/*`,
then re-run §5.3–5.4.

> Cold rebuild has no stale-KV prerequisite — the keyFile + smoke password live in
> Vault KV (reused on rebuild so the same secrets carry over), and the PKI role is
> an upsert. A fresh `rs.initiate` forms new member state; old data is gone with
> the disks.

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `createUser` fails "Unauthorized" from localhost | MongoDB 8.0 + keyFile + `authorization:enabled` does **not** activate the localhost exception | Bootstrap as `__system` (SCRAM-SHA-256, password = keyFile content, authdb `local`) — §5.4.3. See `feedback_mongodb_8_keyfile_localhost_exception`. |
| `updateOne({$set:…})` errors / `$set` empty over SSH | bash expanded `$set` inside the SSH double-quote envelope | Escape it: `\$set` (§5.4.4). See `feedback_bash_dollar_set_via_ssh`. |
| mongod won't start: "permissions on keyFile are too open" | keyFile isn't `0400` (or not `mongodb`-owned) | `sudo install -o mongodb -g mongodb -m 0400 … /etc/nexus-mongo/keyfile` (§5.2.2). |
| mongod won't start: TLS cert error | `server.pem` missing the key, or key is PKCS#1, or CA chain incomplete | `server.pem` = leaf + PKCS#8 key; `ca.crt` = intermediate + root (§5.2.3). |
| `mongosh` AccessDenied reading the cert | TLS material is `0640 root:mongodb`; `nexusadmin` can't read it | `sudo` every `mongosh` (§5.3.1). |
| `rs.status()` fails after init | the RS is formed → auth required (localhost exception off) | authenticate as `__system` or `smoke-rw` (§5.4.2). |
| `rs.initiate()` errors "already initialized" | re-running on a formed member | probe `rs.status().ok` first; skip init if `1`. |
| Replicated read returns null / stale | replication hasn't caught up | retry with a short delay; use `readConcernLevel=majority` (§5.4.4). |

---

## 9. Production tuning — MongoDB

> **Everything below is *beyond the lab replica*.** The §5.3.1 `mongod.conf` is the
> verbatim lab config — it pins the WiredTiger cache to `0.5` GB, takes the default oplog
> and concern levels, and runs on 2 GB / 2-vCPU VMs where those defaults keep the guide a
> faithful 1:1 replay. This section is what you would change on a **production** MongoDB
> replica-set member and *why*. **Do not paste these onto the 2 GB lab VMs blindly** — the
> cache and oplog sizes assume production-sized hosts and disks. The **OS layer** (kernel
> `sysctl`, THP, filesystem, ulimits, I/O scheduler) lives once in
> **[Guide 00 §9](./00-lab-host-and-base-vm.md#9-production-tuning--the-os-layer-feeds-every-linux-tier)**;
> only the MongoDB-specific overrides are restated here.

### 9.1 WiredTiger cache

The lab caps the internal cache at `0.5` GB so three data engines coexist on a 2 GB node.
In production this is the biggest performance lever — but bigger is **not** always better,
because WiredTiger also leans on the OS filesystem cache for compressed blocks.

```yaml
# PRODUCTION mongod.conf — WiredTiger cache (not applied in the lab).
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 15.5   # e.g. 50% of (32 GB - 1 GB); size to YOUR host's RAM
```

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `storage.wiredTiger.engineConfig.cacheSizeGB` | `≈ 50% of (RAM − 1 GB)` (mongod's own default formula) | **`0.5` (§5.3.1)** | The internal WiredTiger cache holds uncompressed working-set pages. Too large starves the OS page cache (which WiredTiger uses for *compressed* blocks) and risks OOM on a shared host; too small forces constant eviction + disk reads. The lab pins `0.5` only so three engines fit a 2 GB node. |

### 9.2 OS layer — THP, filesystem, readahead, swappiness, ulimits  ⚠️

MongoDB is the most OS-opinionated engine in the fleet: it **logs startup warnings** for
THP, non-XFS filesystems on some setups, and high readahead. These live once in
**[Guide 00 §9](./00-lab-host-and-base-vm.md#9-production-tuning--the-os-layer-feeds-every-linux-tier)**; restated here because they are MongoDB-driven.

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| Transparent Huge Pages | **`never`** | unset (`madvise`/`always`) | **⚠️ required by MongoDB** — logs a startup warning; THP's `khugepaged` compaction causes latency spikes and RSS bloat under WiredTiger's access pattern. Guide 00 §9.2. |
| Filesystem | **XFS** for the `dbPath` | ext4 (lab default) | **⚠️ recommended by MongoDB** — WiredTiger on XFS avoids ext4 write-stall pathologies at high concurrency; MongoDB's own production checklist calls for XFS. Guide 00 §9.4. |
| `read_ahead_kb` (block device) | low (`0`–`128`) | unset (`128`–`256`) | MongoDB's random-access pattern wastes bandwidth on large readahead pulling pages it won't use. Guide 00 §9.4. |
| `vm.swappiness` (sysctl) | `1` | unset (`60`) | Keeps the WiredTiger cache resident; the default `60` lets the kernel swap hot pages under cache pressure → latency cliffs. Guide 00 §9.1. |
| `nofile` (open files) | `64000` | **`64000` via `LimitNOFILE=` (§5.1.2 unit)** | Already set on the `nexus-mongo` unit — a busy mongod with many connections + data/journal files exhausts the default `1024` soft limit → refused connections. Note: `limits.conf` is ignored for systemd services, so it lives on the **unit**. |
| `nproc` / `memlock` | raised / high | unset | WiredTiger spawns many threads; MongoDB recommends lifting the per-user process cap (and `memlock` where memory is pinned). Guide 00 §9.3. |

### 9.3 Oplog sizing

```yaml
# PRODUCTION mongod.conf — oplog (set at first init; resize a running set with replSetResizeOplog).
replication:
  replSetName: nexus-rs
  oplogSizeMB: 51200   # 50 GB; size to cover peak-write-rate x longest replica downtime
```

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `replication.oplogSizeMB` | size for the **peak write window**, not the 5%-of-disk default (default = 5% of free disk, capped 50 GB) | unset (default 5% of disk) | The oplog is the capped replication log every SECONDARY tails. A member offline (or lagging) longer than the oplog **time window** falls off and needs a full, expensive **initial sync**. On a small disk 5% may be only minutes of writes — size it to cover your longest expected maintenance/backup window at peak write rate. Resize a live set with `db.adminCommand({replSetResizeOplog:1, size:51200})` (no restart). |

### 9.4 Durability — cluster-wide write & read concern defaults

```javascript
// PRODUCTION — run once on the PRIMARY; cluster-wide concern defaults (not applied in the lab).
db.adminCommand({
  setDefaultRWConcern: 1,
  defaultWriteConcern: { w: "majority", j: true },
  defaultReadConcern:  { level: "majority" }
})
```

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| cluster-wide default **write** concern | `{ w: "majority", j: true }` | unset (server default `w:"majority"` since 5.0; `j` per-op) | Guarantees an acknowledged write is journaled on a **majority** of members before returning — so it survives a PRIMARY failover without a rollback. 5.0+ already defaults `w:majority`; set it explicitly **plus** `j:true` to also require the on-disk journal flush. |
| default **read** concern | `majority` | unset (`local`) | `local` reads can surface writes that later **roll back** during a failover; `majority` returns only majority-committed data. §5.4.4's smoke already reads per-query with `readConcernLevel=majority` — this makes it the cluster-wide default so every client gets it. |

### 9.5 NUMA

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `numactl --interleave=all` (launch wrapper) | wrap the `mongod` launch on multi-socket hosts | not used (single-socket lab VMs) | **⚠️ recommended by MongoDB on NUMA hardware** — without interleaving, mongod's memory lands on one NUMA node and cross-node access + zone-reclaim thrashing tank performance; mongod logs a NUMA warning at startup. Pair with `vm.zone_reclaim_mode=0` (usually the default). |

```ini
# PRODUCTION — /etc/systemd/system/nexus-mongo.service.d/10-numa.conf (multi-socket hosts only).
[Service]
ExecStart=
ExecStart=/usr/bin/numactl --interleave=all /usr/bin/mongod --config /etc/nexus-mongo/mongod.conf
```

### 9.6 Networking — wire compression + connection ceiling

```yaml
# PRODUCTION mongod.conf — networking (not applied in the lab).
net:
  compression:
    compressors: zstd,snappy   # prefer zstd's higher ratio on the replication backplane
  maxIncomingConnections: 20000
```

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `net.compression.compressors` | `zstd,snappy` (list `zstd` first to prefer its ratio) | unset (server default: `snappy` offered) | Compresses both client and **intra-cluster replication** traffic on the wire — a real bandwidth saving for replication fan-out across the backplane. `zstd` trades a little CPU for a markedly better ratio than `snappy`; the peers negotiate the best mutually-supported codec. |
| `net.maxIncomingConnections` | a realistic ceiling above your total driver pool sizes (default `65536`) | unset (`65536`) | Caps concurrent incoming connections. The default is effectively unlimited, and each connection costs a thread + ~1 MB stack — a connection storm from a misconfigured driver pool can exhaust threads/RAM before the DB itself is stressed. |

---

### Cross-references

- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (Mongo `.71`–`.73`)
- **Automated equivalents:** `nexus-infra-oltp/packer/oltp-mongo-node/` + `terraform/envs/oltp-mongo/role-overlay-mongo-*.tf`
- **Scaffolding pattern reused:** [`04-foundation-vault-pki-ldap.md`](./04-foundation-vault-pki-ldap.md) Part D
- **Previous guide:** [`07-oltp-redis-cluster.md`](./07-oltp-redis-cluster.md)
- **Next guide:** Guide 09 — OLTP · Percona XtraDB Cluster (3 PXC + 2 ProxySQL + VRRP VIP). See [`INDEX.md`](../INDEX.md). (The sharded MongoDB build is Guide 12, on top of this replica-set pattern.)
