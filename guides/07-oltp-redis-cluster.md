# Guide 07 — OLTP · Redis Cluster (6-node, 3 shards × 2 replicas, mTLS)

> **Mirrors:** `nexus-infra-oltp` — the `oltp-redis-node` Packer template
> (`nexus-redis.service`) + the `oltp-redis` env overlays (`redis-tls`,
> `redis-config`, `redis-cluster-create`, `redis-nftables-backplane`,
> `redis-vault-agents`). Where the automated lab renders the per-node cert via a
> Vault Agent and forms the cluster from the build host, this guide installs
> Redis by hand and **issues each cert directly with the `vault` CLI**.

---

## 1. Overview & purpose

This is the first **OLTP-tier** guide: a 6-node **Redis Cluster** — **3 shards
× 2 replicas** (3 masters + 3 replicas) — running **mutual-TLS-only** from cold
start. It's the lab's low-latency key/value + cache store.

Redis Cluster shards the keyspace into **16384 hash slots** spread across the 3
masters; each master has one replica for failover. Clients are routed to the
owning shard via `MOVED` redirects (cluster-aware clients cache the slot map).
The whole thing is locked to mTLS: plain port disabled, TLS on `6379`, the
cluster gossip bus on `16379` also TLS, and every client must present a cert.

**Why:** a sharded + replicated KV store exercises horizontal partitioning +
intra-cluster failover — the simplest member of the OLTP tier, and a clean second
use of Guide 04's scaffolding pattern.

**Dependency:**
- **Guides 00–04** — foundation alive; Guide 04's PKI (`pki_int/issue/...`) +
  the Part-D scaffolding pattern (this guide creates the `redis-server` PKI role).
- 6 `deb13` nodes baselined per Guide 00 §5.B, dual-NIC static.

> **By-hand divergence:** issue each node's cert with the `vault` CLI + place the
> three PEM files — no per-node Vault Agent. (The automated lab's Vault Agent
> auto-renews the 90-day leaf; by hand you re-issue when it nears expiry.)

---

## 2. Component primer

- **Redis Cluster.** Redis's native sharding + HA mode. The keyspace is split
  into **16384 hash slots**; each **master** owns a contiguous slot range, and a
  key's slot is `CRC16(key) % 16384`. Nodes gossip topology + health over a
  separate **cluster bus** port (`16379`). With **3 masters + 3 replicas** the
  cluster survives any single master loss (its replica is promoted). *Why
  cluster mode (not a single instance + Sentinel):* horizontal partitioning +
  built-in failover in one mechanism. *Otherwise:* standalone Redis (no
  sharding) or Sentinel-managed replication (HA but no sharding).
- **Hash slots + `MOVED` redirects.** A client that `SET`s a key to the "wrong"
  node gets a `MOVED <slot> <node>` reply; cluster-aware clients (`redis-cli -c`)
  follow it. *Why slots (not consistent hashing):* explicit slot ownership makes
  resharding deterministic.
- **mTLS-only.** `port 0` disables the plain port; `tls-port 6379` +
  `tls-cluster yes` + `tls-replication yes` put **data, the cluster bus, and
  replication** all on TLS; `tls-auth-clients yes` means every client presents a
  cert. The `redis-server` PKI role has `client_flag=true`, so a node's own cert
  doubles as its client identity (for self-`PING` and the intra-cluster bus).
  *Why `protected-mode no`:* Redis 8's protected mode rejects non-loopback
  connections when no `requirepass` is set — and that trips even a node reaching
  *its own* announce IP during cluster-create. It's safe to disable here because
  nftables + TLS-only + client-cert auth are already three layers of defense.
- **AOF persistence.** `appendonly yes` logs every write for durability across
  restarts; the cluster topology lives in `nodes.conf`. *Otherwise:* RDB
  snapshots (coarser) or no persistence (cache-only).
- **The `sudo` rule.** The TLS material is `0640 root:redis`; `nexusadmin` can't
  read it, so **every `redis-cli` that passes `--cert/--key` must be `sudo`**.

---

## 3. Prerequisites

| # | Requirement | One-command verify |
|---|---|---|
| 1 | Foundation alive (Guides 00–04); Vault PKI usable | `vault read pki_int/cert/ca` on vault-1 returns the intermediate |
| 2 | 6 `deb13` nodes baselined, dual-NIC static `.81–.84`, `.87`, `.89` | those 6 answer `:22` |
| 3 | Vault root token on build host | `Test-Path ~/.nexus/secrets/vault-cluster-init.json` |
| 4 | Internet egress on the nodes | `ssh …@81 'curl -sI https://packages.redis.io \| head -1'` → `200` |

> Redis version: **8.0.2**. Cluster traffic here uses **VMnet11** (not the
> VMnet10 backplane) for both data + cluster bus — a deliberate simplification
> (cluster traffic in a 6-node lab is negligible); the dual-NIC backplane is kept
> for future scale-out.

---

## 4. Target topology

| Node | Role (after cluster-create) | VMnet11 | VMnet10 | vCPU/RAM/disk |
|---|---|---|---|---|
| `redis-1` | master (shard 1) | `.81` | `.10.81` | 2 / 2 GB / 40 GB |
| `redis-2` | master (shard 2) | `.82` | `.10.82` | 2 / 2 GB / 40 GB |
| `redis-3` | master (shard 3) | `.83` | `.10.83` | 2 / 2 GB / 40 GB |
| `redis-4` | replica | `.84` | `.10.84` | 2 / 2 GB / 40 GB |
| `redis-5` | replica | `.87` | `.10.87` | 2 / 2 GB / 40 GB |
| `redis-6` | replica | `.89` | `.10.89` | 2 / 2 GB / 40 GB |

> The master/replica split is what `--cluster-replicas 1` produces from the
> 6-node list (first 3 masters, last 3 replicas, with anti-affinity). Ports:
> **`6379`** (TLS data) + **`16379`** (TLS cluster bus), both on VMnet11. Cert
> files per node: `/etc/nexus-redis/tls/{server.crt,server.key,ca.crt}`
> (key in PKCS#8). PKI role: **`redis-server`** (`client_flag=true`, 90-day TTL).

---

## 5. Step-by-step build

> **WHERE:** node steps as `nexusadmin`→`sudo -i`; `redis-cli` under `sudo`.
> `vault` commands on **`vault-1`** (root token). "bootstrap node" = `redis-1`.

### 5.1 — Per-node base install (all 6)

> **Step 5.1.1 — Install Redis 8.0.2 + create the `redis` user and dirs**
> **WHERE:** each node, root shell.
> **WHY:** the Redis server + the service account + the config/data/log dirs the
> rendered `redis.conf` references.
> **WHAT:**
> ```bash
> apt-get update -qq && apt-get install -y lsb-release curl gpg openssl
> curl -fsSL https://packages.redis.io/gpg | gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
> echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" \
>   > /etc/apt/sources.list.d/redis.list
> apt-get update -qq && apt-get install -y redis-tools=6:8.0.2-* redis-server=6:8.0.2-* || apt-get install -y redis
> systemctl disable --now redis-server 2>/dev/null || true   # we run our own nexus-redis unit
> id redis >/dev/null 2>&1 || useradd --system -g redis -s /usr/sbin/nologin -M redis
> install -d -o redis -g redis -m0750 /etc/nexus-redis /etc/nexus-redis/tls /var/lib/nexus-redis /var/log/nexus-redis
> install -d -o redis -g redis -m0755 /var/run/nexus-redis
> ```
> **EXPECTED:** Redis installs; the stock unit is disabled.
> **VERIFY:** `redis-server --version` → `v=8.0.2`.

> **Step 5.1.2 — Install the `nexus-redis` systemd unit**
> **WHERE:** each node, root shell.
> **WHY:** a dedicated unit pointing at our config (the stock `redis-server` unit
> uses `/etc/redis/redis.conf`). `RuntimeDirectory=` recreates `/var/run/nexus-redis`
> on every start (it's tmpfs, wiped on reboot).
> **WHAT:**
> ```bash
> cat > /etc/systemd/system/nexus-redis.service <<'EOF'
> [Unit]
> Description=Nexus Redis Cluster node
> After=network-online.target
> Wants=network-online.target
> [Service]
> User=redis
> Group=redis
> RuntimeDirectory=nexus-redis
> RuntimeDirectoryMode=0755
> ExecStart=/usr/bin/redis-server /etc/nexus-redis/redis.conf
> Restart=on-failure
> LimitNOFILE=65536
> [Install]
> WantedBy=multi-user.target
> EOF
> systemctl daemon-reload
> ```
> **EXPECTED:** unit installed. (Do **not** start yet — no config + no certs.)
> **VERIFY:** `systemctl cat nexus-redis.service | grep ExecStart` → our config path.

> **Step 5.1.3 — Open `6379` + `16379` on VMnet11 (nftables)**
> **WHERE:** each node, root shell.
> **WHY:** Redis data + cluster bus run on VMnet11 here. Add the two ports from
> the lab network; keep the VMnet10 backplane trust for future scale-out.
> **WHAT:** add to `/etc/nftables.conf` `chain input` (before `counter drop`):
> ```
> iifname "nic0" ip saddr 192.168.70.0/24 tcp dport { 6379, 16379 } accept comment "Redis data + cluster bus"
> iifname "nic1" ip saddr 192.168.10.0/24 accept comment "trusted cluster backplane (VMnet10)"
> ```
> then `nft -c -f /etc/nftables.conf && systemctl reload nftables`.
> **EXPECTED:** ruleset valid.
> **VERIFY:** `nft list chain inet filter input | grep 6379`.

### 5.2 — PKI role + per-node certs (Guide 04 Part D)

> **Step 5.2.1 — Create the `redis-server` PKI role**
> **WHERE:** `vault-1`, root shell.
> **WHY:** server **and** client EKU (a node's cert is also its client identity);
> all 6 hostnames + their `.redis.nexus.lab` forms allowed; 90-day leaves.
> **WHAT:**
> ```bash
> vault write pki_int/roles/redis-server \
>   allowed_domains='nexus.lab,redis.nexus.lab,redis-1,redis-2,redis-3,redis-4,redis-5,redis-6,localhost' \
>   allow_subdomains=true allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
> ```
> **EXPECTED:** role written.
> **VERIFY:** `vault read pki_int/roles/redis-server | grep client_flag` → `true`.

> **Step 5.2.2 — Issue + place each node's three PEM files (PKCS#8 key)**
> **WHERE:** issue on `vault-1`; place on each node.
> **WHY:** Redis takes three separate PEM paths — leaf (`server.crt` = leaf +
> issuing CA, full chain), key (`server.key`, PKCS#8), CA (`ca.crt`). Standardize
> the key on PKCS#8 (the platform convention).
> **WHAT (per node — issue on vault-1, substitute host/IPs):**
> ```bash
> ISSUED=$(vault write -format=json pki_int/issue/redis-server \
>   common_name="<host>.redis.nexus.lab" alt_names="<host>,<host>.nexus.lab,localhost" \
>   ip_sans="<vmnet11>,<vmnet10>,127.0.0.1" ttl=2160h)
> { echo "$ISSUED" | jq -r '.data.certificate'; echo "$ISSUED" | jq -r '.data.issuing_ca'; } > /tmp/server.crt
> echo "$ISSUED" | jq -r '.data.private_key' | openssl pkcs8 -topk8 -nocrypt -out /tmp/server.key
> echo "$ISSUED" | jq -r '.data.issuing_ca' > /tmp/ca.crt
> # scp the three to <host>:/etc/nexus-redis/tls/ (owned root:redis, 0640)
> ```
> **EXPECTED:** three files per node.
> **VERIFY:** on a node, `sudo openssl x509 -in /etc/nexus-redis/tls/server.crt -noout -subject`
> → `CN=<host>.redis.nexus.lab`; `grep -c 'BEGIN PRIVATE KEY' /etc/nexus-redis/tls/server.key`
> → `1` (PKCS#8, not `RSA PRIVATE KEY`).

### 5.3 — Render `redis.conf` + start each node

> **Step 5.3.1 — Write `redis.conf` and start `nexus-redis` (all 6, TLS PING)**
> **WHERE:** each node, root shell.
> **WHY:** mTLS-only cluster-member config. The only per-host directive is
> `cluster-announce-ip` (what this node tells peers/clients to reach it on).
> **WHAT (substitute `<host>` + `<vmnet11>`):**
> ```bash
> cat > /etc/nexus-redis/redis.conf <<'EOF'
> # --- Network ---
> bind 0.0.0.0 -::*
> port 0
> tls-port 6379
> protected-mode no
> # --- TLS ---
> tls-cert-file       /etc/nexus-redis/tls/server.crt
> tls-key-file        /etc/nexus-redis/tls/server.key
> tls-ca-cert-file    /etc/nexus-redis/tls/ca.crt
> tls-cluster         yes
> tls-replication     yes
> tls-auth-clients    yes
> tls-protocols       "TLSv1.2 TLSv1.3"
> # --- Cluster ---
> cluster-enabled         yes
> cluster-config-file     /var/lib/nexus-redis/nodes.conf
> cluster-node-timeout    5000
> cluster-announce-ip     <vmnet11>
> cluster-announce-port   6379
> cluster-announce-bus-port 16379
> # --- Persistence ---
> appendonly  yes
> dir         /var/lib/nexus-redis
> # --- Lifecycle (systemd manages) ---
> daemonize   no
> supervised  systemd
> logfile     /var/log/nexus-redis/redis.log
> pidfile     /var/run/nexus-redis/redis.pid
> loglevel    notice
> EOF
> chown root:redis /etc/nexus-redis/redis.conf ; chmod 640 /etc/nexus-redis/redis.conf
> systemctl enable --now nexus-redis.service
> ```
> **EXPECTED:** the service starts (each node standalone-but-cluster-enabled,
> not yet joined).
> **VERIFY (self-mTLS PING — `sudo` for the cert):**
> ```bash
> sudo redis-cli -h 127.0.0.1 -p 6379 --tls \
>   --cacert /etc/nexus-redis/tls/ca.crt --cert /etc/nexus-redis/tls/server.crt --key /etc/nexus-redis/tls/server.key PING
> ```
> → `PONG`. Repeat on all 6.

### 5.4 — Form the cluster (the exit gate)

> **Step 5.4.1 — `redis-cli --cluster create` (3 masters + 3 replicas)**
> **WHERE:** `redis-1` (`.81`), root shell.
> **WHY:** form the cluster across all 6 nodes; `--cluster-replicas 1` makes 3
> shards (3 masters + 3 replicas) with anti-affinity. `--cluster-yes` skips the
> interactive layout prompt. Idempotent guard: if `cluster info` already shows
> `cluster_state:ok`, skip (create refuses to overwrite a live cluster).
> **WHAT:**
> ```bash
> TLS='--tls --cacert /etc/nexus-redis/tls/ca.crt --cert /etc/nexus-redis/tls/server.crt --key /etc/nexus-redis/tls/server.key'
> sudo redis-cli $TLS --cluster create \
>   192.168.70.81:6379 192.168.70.82:6379 192.168.70.83:6379 \
>   192.168.70.84:6379 192.168.70.87:6379 192.168.70.89:6379 \
>   --cluster-replicas 1 --cluster-yes
> ```
> **EXPECTED:** `[OK] All 16384 slots covered.`
> **VERIFY:** `sudo redis-cli -h 127.0.0.1 -p 6379 $TLS cluster info | grep cluster_state`
> → `cluster_state:ok`.

> **Step 5.4.2 — Reconcile orphan masters (Redis 8.0.2 quirk)**
> **WHERE:** `redis-1`, root shell.
> **WHY:** Redis 8.0.2's `--cluster create --cluster-replicas 1` can silently
> leave **all 6 as masters** (3 with slots + 3 orphan masters with none) —
> `cluster_state:ok` + 16384 slots, but the *shape* is wrong (6 masters / 0
> replicas). Convert each orphan (a master with **no** slots) into a replica of a
> slotted master. Idempotent: finds zero orphans on a correctly-shaped cluster.
> **WHAT:**
> ```bash
> TLS='--tls --cacert /etc/nexus-redis/tls/ca.crt --cert /etc/nexus-redis/tls/server.crt --key /etc/nexus-redis/tls/server.key'
> NODES=$(sudo redis-cli -h 127.0.0.1 -p 6379 $TLS cluster nodes)
> echo "$NODES" | awk '/master/ && NF>8  {print $1}'      > /tmp/masters.txt   # masters WITH slots
> echo "$NODES" | awk '/master/ && NF==8 {print $1" "$2}' > /tmp/orphans.txt   # masters WITHOUT slots
> if [ -s /tmp/orphans.txt ]; then
>   MCOUNT=$(wc -l < /tmp/masters.txt) ; n=0
>   while read -r OID OADDR; do
>     IDX=$(( n % MCOUNT + 1 )); TARGET=$(awk -v k="$IDX" 'NR==k{print;exit}' /tmp/masters.txt)
>     OIP=$(echo "$OADDR" | cut -d@ -f1 | cut -d: -f1); OPORT=$(echo "$OADDR" | cut -d@ -f1 | cut -d: -f2)
>     sudo redis-cli -h "$OIP" -p "$OPORT" $TLS CLUSTER REPLICATE "$TARGET"
>     n=$(( n + 1 ))
>   done < /tmp/orphans.txt
> fi
> rm -f /tmp/masters.txt /tmp/orphans.txt
> ```
> **EXPECTED:** orphans (if any) replicate to masters; a healthy cluster is a no-op.
> **VERIFY:** `sudo redis-cli -h 127.0.0.1 -p 6379 $TLS cluster nodes | grep -c master`
> → `3`; `… | grep -c slave` → `3`.

> **Step 5.4.3 — Verify health + cross-shard round-trip (the exit gate)**
> **WHERE:** `redis-1`, root shell.
> **WHY:** prove 3 shards, full slot coverage, 6 known nodes, and that
> cluster-mode routing (`-c`, follows `MOVED`) works end-to-end over mTLS.
> **WHAT:**
> ```bash
> TLS='--tls --cacert /etc/nexus-redis/tls/ca.crt --cert /etc/nexus-redis/tls/server.crt --key /etc/nexus-redis/tls/server.key'
> sudo redis-cli -h 127.0.0.1 -p 6379 $TLS cluster info | grep -E 'cluster_state|cluster_size|cluster_known_nodes|cluster_slots'
> # cross-shard SET/GET (4 keys hash to >1 shard; -c follows MOVED redirects)
> for n in 1 2 3 4; do sudo redis-cli -h 127.0.0.1 -p 6379 $TLS -c SET nexus-smoke-$n "val-$n" >/dev/null; done
> for n in 1 2 3 4; do echo -n "key-$n="; sudo redis-cli -h 127.0.0.1 -p 6379 $TLS -c GET nexus-smoke-$n; done
> ```
> **EXPECTED:** `cluster_state:ok`, `cluster_size:3`, `cluster_known_nodes:6`,
> `cluster_slots_assigned:16384`, `cluster_slots_ok:16384`; the 4 GETs return
> `val-1`…`val-4`.
> **VERIFY:** all five `cluster_*` values as above + the 4 round-trip values match.

---

## 6. Validation — by-hand acceptance smoke

From the **host** (via `ssh …@81` + `sudo redis-cli`).

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | All 6 reachable on TLS data port | `81,82,83,84,87,89 \| % { Test-NetConnection 192.168.70.$_ -Port 6379 }` | all `True` |
| 2 | Plain port disabled | `ssh …@81 'redis-cli -h 127.0.0.1 -p 6379 PING'` (no `--tls`) | error / no PONG (TLS-only) |
| 3 | Self-mTLS PING (each node) | `sudo redis-cli … PING` on each | `PONG` ×6 |
| 4 | Cluster state | `sudo redis-cli … cluster info \| grep cluster_state` | `cluster_state:ok` |
| 5 | 3 shards | `… \| grep cluster_size` | `cluster_size:3` |
| 6 | 6 nodes known | `… \| grep cluster_known_nodes` | `cluster_known_nodes:6` |
| 7 | Full slot coverage | `… \| grep -E 'slots_assigned\|slots_ok'` | both `16384` |
| 8 | Shape = 3 masters + 3 replicas | `sudo redis-cli … cluster nodes \| grep -c master/slave` | 3 + 3 |
| 9 | Cross-shard round-trip | SET/GET 4 keys via `-c` | all 4 values return |
| 10 | **Replica failover** | `sudo redis-cli -h <a master> … DEBUG SLEEP 30` (or stop its node), then `cluster info` from another node | the replica is promoted; `cluster_state:ok` holds (then restore) |

**1–9 green ⇒ Guide 07 satisfied.** 10 is the HA proof — a master's replica takes
over.

---

## 7. Teardown / reset

```bash
# Stop the service on each node (cluster state in nodes.conf + AOF is preserved
# until the disk is deleted):
for ip in 81 82 83 84 87 89; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-redis.service'; done
# then vmrun stop + deleteVM each of the 6 (Guide 00 §7).
```
To wipe cluster state but keep the VMs: on each node
`sudo rm -f /var/lib/nexus-redis/nodes.conf /var/lib/nexus-redis/appendonly*`,
then re-run §5.3–5.4.

> Cold rebuild has no stale-KV prerequisite — the Redis tier's Vault state is the
> `redis-server` PKI role (an upsert) + per-node certs (re-issue). A fresh
> cluster-create forms new node IDs; the old `nodes.conf` is gone with the disks.

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `redis-cli` create: `DENIED Redis is running in protected mode` | Redis 8 protected mode rejects non-loopback connections with no `requirepass` — trips even on a node's own announce IP | `protected-mode no` in `redis.conf` (§5.3.1); safe behind nftables + TLS-only + client-cert auth. |
| Cluster forms but shape is 6 masters / 0 replicas | Redis 8.0.2 `--cluster create --cluster-replicas 1` quirk | Reconcile: `CLUSTER REPLICATE` each orphan (slot-less) master onto a slotted master (§5.4.2). |
| `redis-cli` errors `AccessDeniedException` / can't read the key | TLS material is `0640 root:redis`; `nexusadmin` can't read it | `sudo` every `redis-cli` that passes `--cert/--key`. |
| Node won't start, TLS load error | key is PKCS#1, or a path is wrong | Convert the key with `openssl pkcs8 -topk8` (§5.2.2); confirm the three `tls-*-file` paths exist + are `root:redis 0640`. |
| `--cluster create` refuses: "node already knows other nodes" | a previous cluster is still formed | Probe `cluster info` first; if `cluster_state:ok`, skip create (it's already done). |
| `MOVED` errors from a client | client isn't cluster-aware | use `redis-cli -c` (follows `MOVED`) or a cluster-mode client library. |
| Replica not promoted on master loss | `cluster-node-timeout` not elapsed, or the replica is unhealthy | wait `cluster-node-timeout` (5000 ms) + check the replica's `cluster nodes` link state. |

---

### Cross-references

- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (Redis `.81`–`.89` decade)
- **Automated equivalents:** `nexus-infra-oltp/packer/oltp-redis-node/` + `terraform/envs/oltp-redis/role-overlay-redis-*.tf`
- **Scaffolding pattern reused:** [`04-foundation-vault-pki-ldap.md`](./04-foundation-vault-pki-ldap.md) Part D
- **Previous guide:** [`06-kafka-ecosystem.md`](./06-kafka-ecosystem.md)
- **Next guide:** Guide 08 — OLTP · MongoDB replica set (3-node, keyFile + mTLS x509). See [`INDEX.md`](../INDEX.md).
