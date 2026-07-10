# Guide 13 — Analytics · ClickHouse (3 shards × 2 replicas + 3-node Keeper)

> **Mirrors:** `nexus-infra-analytics` — the `analytics-clickhouse` env overlays
> (`clickhouse-keeper-config`, `clickhouse-server-config`,
> `clickhouse-schema-bootstrap`, `clickhouse-tls`, `clickhouse-backup-repo`,
> `clickhouse-nftables-backplane`) — Phase 0.G.5 / ADRs 0028–0032. The automated
> lab renders the XML + drives the DDL over SSH; by hand we do the same with
> `clickhouse-client` directly.

---

## 1. Overview & purpose

The first **Analytics-tier** guide: a 9-node **ClickHouse** cluster — a columnar
OLAP database for fast aggregations over large datasets. It's **3 shards × 2
replicas** (data partitioned across shards, each shard mirrored on 2 replicas)
plus a dedicated **3-node ClickHouse Keeper** ensemble for coordination.

- **ClickHouse Keeper (`ch-keeper-1/2/3`, `.41/.42/.43`)** — a C++ RAFT quorum
  (Keeper's `mntr` reports one leader + two followers) that ClickHouse uses for
  replication coordination. It's the **ZooKeeper replacement** (ADR-0028) — same
  protocol, native to ClickHouse, no JVM.
- **6 data nodes (`ch-shardN-repM`, `.44–.49`)** — 3 shards × 2 replicas. Each
  table is a **`ReplicatedMergeTree`** (the replicas of a shard stay in sync via
  Keeper) fronted by a **`Distributed`** table that fans queries across all
  shards. `internal_replication=true` means an insert to the Distributed table
  writes to **one** replica per shard and lets ReplicatedMergeTree replicate it.
- **Front door** = **round-robin DNS** `clickhouse.nexus.lab` → all 6 data nodes
  (ADR-0031, no VIP — every node is an equal `Distributed` entry point).

Everything is **mTLS** (secure-only listeners; plain ports removed). DDL is
applied **`ON CLUSTER`** so it lands on every node at once via the macros
(`{shard}`/`{replica}`).

**Dependency:**
- **Guides 00 + 04** — foundation alive; Vault PKI (per-node certs).
- **Guide 01** — the gateway's dnsmasq (round-robin record) + NFS export
  (`/srv/nfs/analytics-backups`, for the backup repo).
- 9 `deb13` nodes baselined per Guide 00 §5.B, dual-NIC static.

> **By-hand divergence:** issue certs with the `vault` CLI, render the XML +
> run `clickhouse-client` directly — no Vault Agent. The VMware cold-rebuild
> flakes (vmrun power-on storm, nic1 NO-CARRIER) are clone-time artifacts, not
> relevant by hand.

---

## 2. Component primer

- **ClickHouse.** A columnar OLAP DBMS — stores data by column + compresses
  heavily, so analytical scans/aggregations over billions of rows are fast.
  *Why:* the analytics workload (vs. the OLTP row-stores in Guides 07–12).
  *Otherwise:* a row store (slow at column aggregations) or StarRocks (Guide 14,
  a parallel analytics engine in this lab).
- **ClickHouse Keeper (not ZooKeeper).** ClickHouse's built-in RAFT coordination
  service (ADR-0028) — replicas register + coordinate replication through it.
  *Why Keeper:* native C++, no JVM/ZooKeeper to operate; same client protocol.
  Runs on 3 dedicated nodes for quorum. Ports: `9181` (4-letter-word health),
  `9281` (secure client — what the servers connect to), `9234` (RAFT, TLS).
- **Shard vs. replica.** A **shard** holds a *subset* of the data (horizontal
  partition); a **replica** is a full copy of a shard (HA + read scale). 3 shards
  × 2 replicas = the whole dataset, twice. *Otherwise:* replicas-only (no
  partitioning) or shards-only (no HA).
- **`ReplicatedMergeTree` + macros.** The replicated table engine; its first two
  args are the **Keeper path** and the **replica name**, templated with the
  per-node **macros** `{shard}`/`{replica}`. So one `ON CLUSTER` `CREATE` makes
  every node build the right local table. Using `{uuid}` in the path
  (`/clickhouse/tables/{uuid}/{shard}`) avoids `REPLICA_ALREADY_EXISTS` collisions
  on `RESTORE AS <new>` (transient #7).
- **`Distributed` table + `internal_replication`.** A virtual table that fans
  reads across shards + routes writes. With `internal_replication=true`, a write
  goes to **one** replica per shard and ReplicatedMergeTree replicates it (vs.
  the Distributed table writing to *both* replicas → double-write). *This is the
  ADR-0029 model.*
- **`ON CLUSTER` DDL + the distributed-DDL queue.** DDL submitted `ON CLUSTER
  <name>` is queued in Keeper (`/clickhouse/task_queue/ddl`) and executed on every
  cluster host. A **readiness gate** (a throwaway `ON CLUSTER` DB create/drop)
  confirms every host can complete a distributed task before the real schema runs.
- **The interserver-FQDN gotcha (#4 — the hard one).** ClickHouse advertises each
  replica's **FQDN** in its Keeper `/host` znode regardless of
  `interserver_http_host`; that FQDN resolves (via DNS) to the **firewall-closed
  VMnet11** IP, so part-fetch (port `9010`, open only on the backplane) times out
  and replicas never converge. **Fix:** write an `/etc/hosts` block on every data
  node mapping each peer FQDN → its **VMnet10 backplane** IP, so the advertised
  FQDN routes over the trusted backplane.
- **Round-robin DNS front door.** `clickhouse.nexus.lab` resolves to all 6 data
  nodes. dnsmasq's `host-record` keeps only the *last* IP for a name, so the lab
  uses the **hosts-file form** (`addn-hosts` → one `IP name` line per node) for a
  true multi-A round-robin (transient #8).

---

## 3. Prerequisites

| # | Requirement | One-command verify |
|---|---|---|
| 1 | Foundation alive (Guides 00 + 04); Vault PKI usable | `vault read pki_int/cert/ca` on vault-1 returns the intermediate |
| 2 | 9 `deb13` nodes baselined, dual-NIC static `.41–.49` | those 9 answer `:22` |
| 3 | Gateway NFS export `/srv/nfs/analytics-backups` (Guide 01-style, `fsid=0`) | `ssh …@1 'sudo exportfs -v \| grep analytics-backups'` |
| 4 | Vault root token on build host | `Test-Path ~/.nexus/secrets/vault-cluster-init.json` |
| 5 | Internet egress on the nodes | `ssh …@41 'curl -sI https://packages.clickhouse.com \| head -1'` → `200` |

> ClickHouse **26.5** (current LTS). Cluster name `nexus_analytics`. Backplane:
> Keeper RAFT `9234` + interserver `9010` on VMnet10; native TLS `9440` +
> HTTPS `8443` reachable on VMnet11.

---

## 4. Target topology

| Node | Role | shard/replica | VMnet11 | VMnet10 | RAM |
|---|---|---|---|---|---|
| `ch-keeper-1/2/3` | ClickHouse Keeper (RAFT) | — | `.41/.42/.43` | `.10.41/.42/.43` | 2 GB |
| `ch-shard1-rep1` | data | s1 r1 | `.44` | `.10.44` | 6 GB |
| `ch-shard1-rep2` | data | s1 r2 | `.45` | `.10.45` | 6 GB |
| `ch-shard2-rep1` | data | s2 r1 | `.46` | `.10.46` | 6 GB |
| `ch-shard2-rep2` | data | s2 r2 | `.47` | `.10.47` | 6 GB |
| `ch-shard3-rep1` | data | s3 r1 | `.48` | `.10.48` | 6 GB |
| `ch-shard3-rep2` | data | s3 r2 | `.49` | `.10.49` | 6 GB |

> Front door: round-robin DNS `clickhouse.nexus.lab` → `.44–.49`. PKI roles
> **`clickhouse-keeper-server`** + **`clickhouse-server`**. Backup repo:
> NFS `/srv/nfs/analytics-backups` (`file://` disk). Cluster secret
> `nexus_analytics_internal` (inter-node DDL auth).

---

## 5. Step-by-step build

> **WHERE:** node steps as `nexusadmin`→`sudo -i`; `clickhouse-client` over TLS
> (`--secure --port 9440 --accept-invalid-certificate`). `vault` on **`vault-1`**.

### 5.1 — Per-node base install

> **Step 5.1.1 — Keeper nodes: install `clickhouse-keeper`**
> **WHERE:** `ch-keeper-1/2/3`, root shell.
> **WHAT:**
> ```bash
> apt-get update -qq && apt-get install -y apt-transport-https ca-certificates gnupg curl openssl
> curl -fsSL https://packages.clickhouse.com/rpm/lts/repodata/repomd.xml.key | gpg --dearmor -o /usr/share/keyrings/clickhouse-keyring.gpg
> echo "deb [signed-by=/usr/share/keyrings/clickhouse-keyring.gpg] https://packages.clickhouse.com/deb stable main" > /etc/apt/sources.list.d/clickhouse.list
> apt-get update -qq && apt-get install -y clickhouse-keeper
> install -d -o clickhouse -g clickhouse -m0750 /etc/nexus-clickhouse-keeper/tls /var/lib/nexus-clickhouse-keeper /var/log/nexus-clickhouse-keeper
> ```
> **VERIFY:** `clickhouse-keeper --version` prints `26.5.x`.

> **Step 5.1.2 — Data nodes: install `clickhouse-server` + client**
> **WHERE:** `ch-shard{1,2,3}-rep{1,2}`, root shell.
> **WHAT (same repo as 5.1.1):**
> ```bash
> # (add the clickhouse apt repo as in 5.1.1, then:)
> DEBIAN_FRONTEND=noninteractive apt-get install -y clickhouse-server clickhouse-client
> systemctl disable --now clickhouse-server 2>/dev/null || true   # we run nexus-clickhouse-server
> install -d -o clickhouse -g clickhouse -m0750 /etc/clickhouse-server/tls
> ```
> **VERIFY:** `clickhouse-server --version` → `26.5.x`.

> **Step 5.1.3 — nftables: backplane trust + service ports**
> **WHERE:** each node, root shell.
> **WHY:** Keeper RAFT (`9234`) + ClickHouse interserver (`9010`) ride the VMnet10
> backplane (trust the whole segment); HTTPS `8443` + native TLS `9440` are
> reachable on VMnet11.
> **WHAT:** add to `/etc/nftables.conf` `chain input` (before `counter drop`):
> ```
> iifname "nic1" ip saddr 192.168.10.0/24 accept comment "trusted cluster backplane (VMnet10)"
> # keeper:  iifname "nic0" ip saddr 192.168.70.0/24 tcp dport { 9181, 9281 } accept
> # data:    iifname "nic0" ip saddr 192.168.70.0/24 tcp dport { 8443, 9440 } accept
> ```
> **VERIFY:** `nft list chain inet filter input | grep nic1`.

### 5.2 — Per-node TLS certs

> **Step 5.2.1 — PKI roles + per-node certs (keeper + data)**
> **WHERE:** issue on `vault-1`; place on each node.
> **WHY:** mTLS everywhere. Keeper certs go to `/etc/nexus-clickhouse-keeper/tls/`,
> data certs to `/etc/clickhouse-server/tls/` — each as `server.crt`/`server.key`/
> `ca.crt`. SANs must include the node FQDN + the round-robin name
> `clickhouse.nexus.lab` (for the data nodes).
> **WHAT (once, on `vault-1` — create the two roles):**
> ```bash
> export VAULT_ADDR=https://127.0.0.1:8200 VAULT_CACERT=$HOME/.nexus/vault-ca-bundle.crt
> for role in clickhouse-keeper-server clickhouse-server; do
>   vault write pki_int/roles/$role \
>     allowed_domains='nexus.lab,clickhouse.nexus.lab,ch-keeper-1,ch-keeper-2,ch-keeper-3,ch-shard1-rep1,ch-shard1-rep2,ch-shard2-rep1,ch-shard2-rep2,ch-shard3-rep1,ch-shard3-rep2,localhost' \
>     allow_subdomains=true allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>     server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
> done
> ```
> Per-node values (the 3 **keeper** nodes use the `clickhouse-keeper-server` role +
> the keeper tls dir; the 6 **data** nodes use `clickhouse-server` + add the
> round-robin name `clickhouse.nexus.lab` to the SANs):
>
> | Node | VMnet11 | PKI role | CN | extra `alt_names` | `ip_sans` | cert dir |
> |---|---|---|---|---|---|---|
> | `ch-keeper-1` | `.41` | `clickhouse-keeper-server` | `ch-keeper-1.nexus.lab` | — | `192.168.10.41,192.168.70.41,127.0.0.1` | `/etc/nexus-clickhouse-keeper/tls` |
> | `ch-keeper-2` | `.42` | `clickhouse-keeper-server` | `ch-keeper-2.nexus.lab` | — | `192.168.10.42,192.168.70.42,127.0.0.1` | `/etc/nexus-clickhouse-keeper/tls` |
> | `ch-keeper-3` | `.43` | `clickhouse-keeper-server` | `ch-keeper-3.nexus.lab` | — | `192.168.10.43,192.168.70.43,127.0.0.1` | `/etc/nexus-clickhouse-keeper/tls` |
> | `ch-shard1-rep1` | `.44` | `clickhouse-server` | `ch-shard1-rep1.nexus.lab` | `clickhouse.nexus.lab` | `192.168.10.44,192.168.70.44,127.0.0.1` | `/etc/clickhouse-server/tls` |
> | `ch-shard1-rep2` | `.45` | `clickhouse-server` | `ch-shard1-rep2.nexus.lab` | `clickhouse.nexus.lab` | `192.168.10.45,192.168.70.45,127.0.0.1` | `/etc/clickhouse-server/tls` |
> | `ch-shard2-rep1` | `.46` | `clickhouse-server` | `ch-shard2-rep1.nexus.lab` | `clickhouse.nexus.lab` | `192.168.10.46,192.168.70.46,127.0.0.1` | `/etc/clickhouse-server/tls` |
> | `ch-shard2-rep2` | `.47` | `clickhouse-server` | `ch-shard2-rep2.nexus.lab` | `clickhouse.nexus.lab` | `192.168.10.47,192.168.70.47,127.0.0.1` | `/etc/clickhouse-server/tls` |
> | `ch-shard3-rep1` | `.48` | `clickhouse-server` | `ch-shard3-rep1.nexus.lab` | `clickhouse.nexus.lab` | `192.168.10.48,192.168.70.48,127.0.0.1` | `/etc/clickhouse-server/tls` |
> | `ch-shard3-rep2` | `.49` | `clickhouse-server` | `ch-shard3-rep2.nexus.lab` | `clickhouse.nexus.lab` | `192.168.10.49,192.168.70.49,127.0.0.1` | `/etc/clickhouse-server/tls` |
>
> **WHAT (issue on `vault-1` — example `ch-shard1-rep1`, a data node; keeper rows use the keeper role + drop the RR name):**
> ```bash
> vault write -format=json pki_int/issue/clickhouse-server \
>   common_name=ch-shard1-rep1.nexus.lab \
>   alt_names='ch-shard1-rep1,ch-shard1-rep1.nexus.lab,clickhouse.nexus.lab,localhost' \
>   ip_sans='192.168.10.44,192.168.70.44,127.0.0.1' ttl=2160h > /tmp/ch-shard1-rep1.json
> # a keeper row instead:  vault write -format=json pki_int/issue/clickhouse-keeper-server \
> #   common_name=ch-keeper-1.nexus.lab alt_names='ch-keeper-1,ch-keeper-1.nexus.lab,localhost' ip_sans='192.168.10.41,192.168.70.41,127.0.0.1' ...
> vault read -field=certificate pki_int/cert/ca_chain > /tmp/nexus-ca-chain.pem
> ```
> **WHAT (place on each node — set `D` from the table's cert dir):**
> ```bash
> # copy /tmp/<host>.json + /tmp/nexus-ca-chain.pem to the node, then as root:
> D=/etc/clickhouse-server/tls          # keeper nodes: D=/etc/nexus-clickhouse-keeper/tls
> install -d -o clickhouse -g clickhouse -m 0750 "$D"
> jq -r '.data.certificate' /tmp/<host>.json > /tmp/leaf.crt
> jq -r '.data.issuing_ca'  /tmp/<host>.json > /tmp/int.crt
> jq -r '.data.private_key' /tmp/<host>.json > /tmp/leaf.key
> cat /tmp/leaf.crt /tmp/int.crt > "$D/server.crt"
> openssl pkcs8 -topk8 -nocrypt -in /tmp/leaf.key -out "$D/server.key"
> cp /tmp/nexus-ca-chain.pem "$D/ca.crt"
> chown -R clickhouse:clickhouse "$D" ; chmod 0644 "$D/server.crt" "$D/ca.crt" ; chmod 0640 "$D/server.key"
> rm -f /tmp/leaf.crt /tmp/int.crt /tmp/leaf.key /tmp/<host>.json
> ```
> **EXPECTED:** 3 cert files per node in the role's tls dir.
> **VERIFY (each node):** `sudo openssl x509 -in $D/server.crt -noout -subject` → the node CN.

### 5.3 — Bring up the Keeper ensemble

> **Step 5.3.1 — Render `keeper_config.xml` on each Keeper node, parallel start**
> **WHERE:** `ch-keeper-1/2/3`, root shell; start all 3 together.
> **WHY:** the RAFT quorum needs all 3 to elect. Per-node `server_id` (1/2/3); the
> shared `raft_configuration` lists all 3 by **backplane** IP; RAFT + client are
> TLS.
> **WHAT (substitute `<id>` per node):**
> ```bash
> cat > /etc/nexus-clickhouse-keeper/keeper_config.xml <<'EOF'
> <?xml version="1.0"?>
> <clickhouse>
>     <logger><level>information</level><log>/var/log/nexus-clickhouse-keeper/keeper.log</log></logger>
>     <listen_host>::</listen_host>
>     <keeper_server>
>         <tcp_port>9181</tcp_port>
>         <tcp_port_secure>9281</tcp_port_secure>
>         <server_id><id></server_id>
>         <log_storage_path>/var/lib/nexus-clickhouse-keeper/coordination/log</log_storage_path>
>         <snapshot_storage_path>/var/lib/nexus-clickhouse-keeper/coordination/snapshots</snapshot_storage_path>
>         <coordination_settings>
>             <operation_timeout_ms>10000</operation_timeout_ms>
>             <session_timeout_ms>30000</session_timeout_ms>
>         </coordination_settings>
>         <raft_configuration>
>             <server><id>1</id><hostname>192.168.10.41</hostname><port>9234</port><secure>true</secure></server>
>             <server><id>2</id><hostname>192.168.10.42</hostname><port>9234</port><secure>true</secure></server>
>             <server><id>3</id><hostname>192.168.10.43</hostname><port>9234</port><secure>true</secure></server>
>         </raft_configuration>
>     </keeper_server>
>     <openSSL>
>         <server>
>             <certificateFile>/etc/nexus-clickhouse-keeper/tls/server.crt</certificateFile>
>             <privateKeyFile>/etc/nexus-clickhouse-keeper/tls/server.key</privateKeyFile>
>             <caConfig>/etc/nexus-clickhouse-keeper/tls/ca.crt</caConfig>
>             <verificationMode>relaxed</verificationMode>
>             <loadDefaultCAFile>false</loadDefaultCAFile>
>         </server>
>         <client>
>             <certificateFile>/etc/nexus-clickhouse-keeper/tls/server.crt</certificateFile>
>             <privateKeyFile>/etc/nexus-clickhouse-keeper/tls/server.key</privateKeyFile>
>             <caConfig>/etc/nexus-clickhouse-keeper/tls/ca.crt</caConfig>
>             <verificationMode>relaxed</verificationMode>
>             <invalidCertificateHandler><name>RejectCertificateHandler</name></invalidCertificateHandler>
>         </client>
>     </openSSL>
> </clickhouse>
> EOF
> chown clickhouse:clickhouse /etc/nexus-clickhouse-keeper/keeper_config.xml
> systemctl enable --now clickhouse-keeper
> ```
> **EXPECTED:** RAFT elects a leader.
> **VERIFY:** `echo mntr | timeout 3 openssl s_client -connect 127.0.0.1:9281 -quiet 2>/dev/null | grep zk_server_state`
> → one node `leader`, two `follower`. (Or `echo mntr | nc 127.0.0.1 9181` on the plain health port.)

### 5.4 — Configure the ClickHouse servers

> **Step 5.4.1 — Write the `/etc/hosts` backplane block on every data node**
> **WHERE:** each data node, root shell.
> **WHY:** the load-bearing fix for interserver part-fetch (#4) — map every peer
> FQDN to its **VMnet10 backplane** IP so the FQDN ClickHouse advertises in Keeper
> routes over the backplane (where `9010` is open), not the firewall-closed
> VMnet11 IP. **Write this before the server starts.**
> **WHAT (identical block on all 6 data nodes):**
> ```bash
> cat >> /etc/hosts <<'EOF'
> # >>> nexus-analytics backplane (interserver replication over VMnet10) >>>
> 192.168.10.44 ch-shard1-rep1.clickhouse.nexus.lab ch-shard1-rep1.nexus.lab ch-shard1-rep1
> 192.168.10.45 ch-shard1-rep2.clickhouse.nexus.lab ch-shard1-rep2.nexus.lab ch-shard1-rep2
> 192.168.10.46 ch-shard2-rep1.clickhouse.nexus.lab ch-shard2-rep1.nexus.lab ch-shard2-rep1
> 192.168.10.47 ch-shard2-rep2.clickhouse.nexus.lab ch-shard2-rep2.nexus.lab ch-shard2-rep2
> 192.168.10.48 ch-shard3-rep1.clickhouse.nexus.lab ch-shard3-rep1.nexus.lab ch-shard3-rep1
> 192.168.10.49 ch-shard3-rep2.clickhouse.nexus.lab ch-shard3-rep2.nexus.lab ch-shard3-rep2
> # <<< nexus-analytics backplane <<<
> EOF
> ```
> **VERIFY:** `getent hosts ch-shard2-rep1.nexus.lab` → `192.168.10.46`.

> **Step 5.4.2 — Render `config.d/nexus-cluster.xml` + `users.d/nexus-bootstrap.xml`, then enable+restart**
> **WHERE:** each data node, root shell.
> **WHY:** the cluster topology (`remote_servers`: 3 shards × 2 replicas on the
> backplane, secure `9440`, `internal_replication=true`, a shared `secret`), the
> per-node **macros** (`{shard}`/`{replica}`/`{cluster}`), the `zookeeper` section
> pointing at the 3 Keepers' secure port, secure-only listeners, and the
> `default` user gaining `access_management` + `show_named_collections_secrets`
> (so the schema bootstrap can `GRANT ALL`). **`enable` then `restart`** — `enable
> --now` no-ops a running unit so config.d changes never load (#5).
> **WHAT (substitute `<shard>` + `<host>` per node):**
> ```bash
> cat > /etc/clickhouse-server/config.d/nexus-cluster.xml <<'EOF'
> <?xml version="1.0"?>
> <clickhouse>
>     <listen_host>::</listen_host>
>     <https_port>8443</https_port>
>     <tcp_port_secure>9440</tcp_port_secure>
>     <tcp_port remove="1"/>
>     <http_port remove="1"/>
>     <interserver_http_port remove="1"/>
>     <interserver_https_port>9010</interserver_https_port>
>     <interserver_http_host replace="replace"><this node's VMnet10 IP></interserver_http_host>
>     <macros>
>         <shard><shard></shard>
>         <replica><host></replica>
>         <cluster>nexus_analytics</cluster>
>     </macros>
>     <remote_servers replace="replace">
>         <nexus_analytics>
>             <secret>nexus_analytics_internal</secret>
>             <shard>
>                 <internal_replication>true</internal_replication>
>                 <replica><host>192.168.10.44</host><port>9440</port><secure>1</secure></replica>
>                 <replica><host>192.168.10.45</host><port>9440</port><secure>1</secure></replica>
>             </shard>
>             <shard>
>                 <internal_replication>true</internal_replication>
>                 <replica><host>192.168.10.46</host><port>9440</port><secure>1</secure></replica>
>                 <replica><host>192.168.10.47</host><port>9440</port><secure>1</secure></replica>
>             </shard>
>             <shard>
>                 <internal_replication>true</internal_replication>
>                 <replica><host>192.168.10.48</host><port>9440</port><secure>1</secure></replica>
>                 <replica><host>192.168.10.49</host><port>9440</port><secure>1</secure></replica>
>             </shard>
>         </nexus_analytics>
>     </remote_servers>
>     <zookeeper replace="replace">
>         <node><host>192.168.10.41</host><port>9281</port><secure>1</secure></node>
>         <node><host>192.168.10.42</host><port>9281</port><secure>1</secure></node>
>         <node><host>192.168.10.43</host><port>9281</port><secure>1</secure></node>
>     </zookeeper>
>     <distributed_ddl><path>/clickhouse/task_queue/ddl</path></distributed_ddl>
>     <openSSL>
>         <server>
>             <certificateFile>/etc/clickhouse-server/tls/server.crt</certificateFile>
>             <privateKeyFile>/etc/clickhouse-server/tls/server.key</privateKeyFile>
>             <caConfig>/etc/clickhouse-server/tls/ca.crt</caConfig>
>             <verificationMode>relaxed</verificationMode>
>             <loadDefaultCAFile>false</loadDefaultCAFile>
>         </server>
>         <client>
>             <certificateFile>/etc/clickhouse-server/tls/server.crt</certificateFile>
>             <privateKeyFile>/etc/clickhouse-server/tls/server.key</privateKeyFile>
>             <caConfig>/etc/clickhouse-server/tls/ca.crt</caConfig>
>             <verificationMode>relaxed</verificationMode>
>             <invalidCertificateHandler><name>RejectCertificateHandler</name></invalidCertificateHandler>
>         </client>
>     </openSSL>
> </clickhouse>
> EOF
>
> cat > /etc/clickhouse-server/users.d/nexus-bootstrap.xml <<'EOF'
> <?xml version="1.0"?>
> <clickhouse><users><default>
>     <networks replace="replace"><ip>127.0.0.1</ip><ip>::1</ip></networks>
>     <access_management>1</access_management>
>     <named_collection_control>1</named_collection_control>
>     <show_named_collections_secrets>1</show_named_collections_secrets>
> </default></users></clickhouse>
> EOF
> chown root:clickhouse /etc/clickhouse-server/config.d/nexus-cluster.xml /etc/clickhouse-server/users.d/nexus-bootstrap.xml
> chmod 0640 /etc/clickhouse-server/config.d/nexus-cluster.xml /etc/clickhouse-server/users.d/nexus-bootstrap.xml
> systemctl enable nexus-clickhouse-server 2>/dev/null || systemctl enable clickhouse-server
> systemctl restart clickhouse-server     # restart (not enable --now) so config.d loads
> ```
> **EXPECTED:** the server starts on the secure listeners.
> **VERIFY:** `clickhouse-client --secure --port 9440 --accept-invalid-certificate --query "SELECT count() FROM system.clusters WHERE cluster='nexus_analytics'"`
> → `6`. (And no `:8443/:9440` errors in `journalctl -u clickhouse-server`.)

### 5.5 — Round-robin DNS front door

> **Step 5.5.1 — Publish `clickhouse.nexus.lab` → all 6 data nodes (gateway)**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** a true multi-A round-robin. dnsmasq's `host-record` keeps only the
> *last* IP for a name, so use the **hosts-file form** via `addn-hosts` — one
> `IP name` line per node — in a file **outside** `/etc/dnsmasq.d/` (dnsmasq parses
> every file in the conf-dir as config) (#8).
> **WHAT:**
> ```bash
> cat > /etc/dnsmasq-analytics.hosts <<'EOF'
> 192.168.70.44 clickhouse.nexus.lab
> 192.168.70.45 clickhouse.nexus.lab
> 192.168.70.46 clickhouse.nexus.lab
> 192.168.70.47 clickhouse.nexus.lab
> 192.168.70.48 clickhouse.nexus.lab
> 192.168.70.49 clickhouse.nexus.lab
> EOF
> echo 'addn-hosts=/etc/dnsmasq-analytics.hosts' > /etc/dnsmasq.d/analytics-records.conf
> dnsmasq --test && systemctl reload dnsmasq
> ```
> **EXPECTED:** the name resolves to all 6 (rotating).
> **VERIFY:** `dig @192.168.70.1 clickhouse.nexus.lab +short` lists all 6 IPs.

### 5.6 — Schema bootstrap (the exit gate)

> **Step 5.6.1 — DDL-readiness gate → RBAC → schema → fan-out → convergence**
> **WHERE:** any data node (e.g. `ch-shard1-rep1`), root shell.
> **WHY:** `ON CLUSTER` DDL lands on every node via the distributed-DDL queue.
> First a **readiness gate** (a throwaway `ON CLUSTER` DB) confirms every host can
> complete a distributed task (fails fast if a node's Keeper session is stuck).
> Then SQL RBAC, then the `ReplicatedMergeTree` local table + the `Distributed`
> front, then a 600-row fan-out + a convergence check.
> **WHAT (`CH` = `clickhouse-client --secure --port 9440 --accept-invalid-certificate`):**
> ```bash
> CH="clickhouse-client --secure --port 9440 --accept-invalid-certificate"
> C=nexus_analytics
> # readiness gate
> $CH --distributed_ddl_task_timeout=15 --query "CREATE DATABASE IF NOT EXISTS nexus_ddlready ON CLUSTER $C"
> $CH --query "DROP DATABASE IF EXISTS nexus_ddlready ON CLUSTER $C SYNC"
> # RBAC
> $CH --query "CREATE ROLE IF NOT EXISTS app_ro ON CLUSTER $C"
> $CH --query "CREATE ROLE IF NOT EXISTS app_rw ON CLUSTER $C"
> $CH --query "CREATE DATABASE IF NOT EXISTS nexus ON CLUSTER $C"
> $CH --query "GRANT ON CLUSTER $C SELECT ON nexus.* TO app_ro"
> $CH --query "GRANT ON CLUSTER $C SELECT, INSERT ON nexus.* TO app_rw"
> $CH --query "CREATE USER IF NOT EXISTS admin ON CLUSTER $C IDENTIFIED WITH sha256_password BY '<admin-pw>'"
> $CH --query "GRANT ON CLUSTER $C ALL ON *.* TO admin WITH GRANT OPTION"
> # schema: ReplicatedMergeTree local + Distributed front
> $CH --query "CREATE TABLE nexus.events_local ON CLUSTER $C (event_id UInt64, ts DateTime, bucket UInt32, payload String) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{uuid}/{shard}', '{replica}') ORDER BY (ts, event_id)"
> $CH --query "CREATE TABLE nexus.events ON CLUSTER $C AS nexus.events_local ENGINE = Distributed($C, nexus, events_local, rand())"
> # fan-out 600 rows via the Distributed table
> $CH --query "INSERT INTO nexus.events SELECT number, now(), number % 3, concat('demo-', toString(number)) FROM numbers(600)"
> $CH --query "SELECT count() FROM nexus.events"
> ```
> **EXPECTED:** the final `count()` → `600` (the Distributed table sums all shards).
> **VERIFY:** each shard's two replicas converged — on every data node:
> `$CH --query "SELECT count() FROM nexus.events_local"` returns the **same**
> number for both replicas of a shard (≈200 each), and the cluster total is 600.

### 5.7 — (Optional) NFS backup repo

> **Step 5.7.1 — Mount the analytics-backups NFS export + register a backup disk**
> **WHERE:** each data node, root shell.
> **WHY:** ClickHouse `BACKUP … TO Disk('backups', …)` to the gateway's NFS
> export (`fsid=0` → mount via `:/`); the disk is declared in a `storage.xml`.
> **WHAT (outline):**
> ```bash
> apt-get install -y nfs-common
> echo '192.168.70.1:/  /var/lib/nexus-analytics-backups  nfs4  rw,hard,bg,_netdev,vers=4.2,sec=sys  0  0' >> /etc/fstab && mount -a
> # config.d/backups.xml declaring <backups><allowed_disk>backups</allowed_disk></backups>
> #   + a <storage_configuration><disks><backups><type>local</type><path>/var/lib/nexus-analytics-backups/</path>…
> # then: BACKUP TABLE nexus.events_local ON CLUSTER nexus_analytics TO Disk('backups','events.zip')
> ```
> **EXPECTED:** a backup writes to the NFS export + `RESTORE` round-trips.
> **VERIFY:** `findmnt /var/lib/nexus-analytics-backups`; a `BACKUP`/`RESTORE`
> succeeds (use a distinct `RESTORE … AS` name — the `{uuid}` Keeper path avoids
> `REPLICA_ALREADY_EXISTS`, #7).

---

## 6. Validation — by-hand acceptance smoke

From the **host** (`ssh` + `clickhouse-client --secure …`). Condensed from the
129-check `smoke-0.G.5.ps1`.

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | 9 nodes reachable | `Test-NetConnection` per node (keeper 9281 / data 9440) | all `True` |
| 2 | Keeper quorum | `mntr` on each keeper | 1 leader + 2 follower |
| 3 | Cluster sees 6 hosts | `SELECT count() FROM system.clusters WHERE cluster='nexus_analytics'` | `6` |
| 4 | Distributed total | `SELECT count() FROM nexus.events` | `600` |
| 5 | Sharded ×3 | per-shard `events_local` counts sum to 600 across 3 shards | ≈200 per shard |
| 6 | Replicated ×2 | both replicas of each shard report equal `events_local` count | equal per shard |
| 7 | mTLS enforced | plain `clickhouse-client --port 9000` (no `--secure`) | refused |
| 8 | Round-robin DNS | `dig @192.168.70.1 clickhouse.nexus.lab +short` | all 6 IPs |
| 9 | RBAC | `clickhouse-client -u app_ro …` can SELECT, cannot INSERT | as expected |
| 10 | **Replica failover** | stop one replica of a shard; query via the Distributed table | still returns 600 (the other replica serves; then restart) |

**1–9 green ⇒ Guide 13 satisfied.** 10 is the replica-HA proof.

---

## 7. Teardown / reset

```bash
for ip in 44 45 46 47 48 49; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now clickhouse-server'; done
for ip in 41 42 43; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now clickhouse-keeper'; done
# then vmrun stop + deleteVM each of the 9 (Guide 00 §7).
```
To reset the schema but keep the cluster: `DROP DATABASE nexus ON CLUSTER nexus_analytics SYNC` + drop the users/roles, then re-run §5.6.

> No stale-state prerequisite — replication metadata lives in Keeper + each node's
> data dir; a fresh cluster + `ON CLUSTER` DDL rebuilds it. The gateway DNS record
> + NFS export belong to Guide 01.

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Replicas never converge; `GET_PART … connect timed out :9010` | CH advertises the replica **FQDN** in Keeper; it resolves to the firewall-closed VMnet11 IP | write the `/etc/hosts` backplane block (FQDN → `192.168.10.x`) on every data node **before** start (§5.4.1, #4). |
| config.d change didn't take effect on re-apply | `enable --now` no-ops a running unit | use `enable` + `restart` (§5.4.2, #5). |
| `default` user can't `GRANT ALL` to admin | missing `show_named_collections_secrets` | add it to the `default` user (§5.4.2, #3). |
| ClickHouse won't parse the config | an XML comment contains `--` | XML comments may not contain a double hyphen (#6). |
| `clickhouse.nexus.lab` resolves to **one** node | dnsmasq `host-record` keeps only the last IP | use the `addn-hosts` hosts-file form (§5.5.1, #8). |
| `ON CLUSTER` DDL hangs / Code 159 | a node's Keeper session is stuck (e.g. nic1 came up NO-CARRIER) | the readiness gate fails fast; reconnect `nic1` (`vmrun connectNamedDevice … ethernet1` on the host) + `systemctl restart clickhouse-server`, then retry. |
| `RESTORE … AS <new>` fails `REPLICA_ALREADY_EXISTS` | static ZK path collides | use `{uuid}` in the ReplicatedMergeTree path (§5.6.1, #7). |
| Keeper won't elect | nodes started sequentially, or RAFT `9234` blocked on the backplane | start all 3 in parallel (§5.3.1); confirm nftables trusts VMnet10. |

---

## 9. Production tuning — ClickHouse

> **Everything below is *beyond the lab replica*.** The §5 configs (`config.d/nexus-cluster.xml`,
> `users.d/nexus-bootstrap.xml`, `keeper_config.xml`) ship **zero** memory/cache/pool
> tuning — the data nodes are 6 GB and the Keepers 2 GB, so ClickHouse's built-in defaults
> (RAM-ratio caps, default caches, core-count-derived threads) are fine and staying at
> defaults keeps this guide a faithful 1:1 replay. This section is what you would set on a
> **production** ClickHouse cluster (data nodes with tens–hundreds of GB of RAM, dozens of
> cores, fast NVMe) and *why*. **Do not paste these onto the 2 GB/6 GB lab VMs blindly** —
> most values assume production-sized RAM/cores and will OOM or starve a lab node.

Two placement surfaces, and they are **not** interchangeable — get this wrong and the knob
silently does nothing:

- **Server-level** settings (memory ceilings, caches, background pools, MergeTree, listener
  limits) live in a **`config.d/*.xml`** file, at the `<clickhouse>` root. They apply to the
  whole server process; a change needs a `systemctl restart clickhouse-server`.
- **Query/session** settings (per-query memory, threads, timeouts, spill-to-disk, partition
  guards) live in a **`users.d/*.xml`** file, inside a `<profiles>` block — they are *profile*
  defaults a client inherits, hot-reloaded (no restart) and overridable per-`SET`/per-user.

**Keeper nodes are separate.** `ch-keeper-1/2/3` run `clickhouse-keeper`, not
`clickhouse-server` — none of the server caches/pools/profiles below apply to them. Their
only tunables are the `<coordination_settings>` + JVM-free memory footprint covered in §9.5.

### 9.1 OS layer — inherit Guide 00 §9, with two ClickHouse overrides

Apply the whole OS-layer tuning from **[Guide 00 §9](./00-lab-host-and-base-vm.md#9-production-tuning--the-os-layer-feeds-every-linux-tier)**
(sysctl, ulimits, I/O scheduler, journald) on every data node first. ClickHouse then
**overrides two of those defaults** — do not take Guide 00's values verbatim here:

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| Transparent Huge Pages | **`madvise`** | unset (distro `madvise`/`always`) | **⚠️ ClickHouse is the ONE engine that overrides Guide 00 §9's `THP=never`.** CH's allocator (jemalloc) uses `madvise(MADV_HUGEPAGE)` deliberately for its large arenas; forcing THP fully off (as MongoDB/Redis require) *hurts* CH by denying those hugepages. Set `echo madvise > /sys/kernel/mm/transparent_hugepage/enabled` — **not** `never`, and **not** `always` (always causes `khugepaged` stalls). If a node runs *only* ClickHouse, leave the distro `madvise` default and skip Guide 00's disable-thp unit on it. |
| `read_ahead_kb` | `4096` (scan-heavy OLAP) | unset (`128`–`256`) | Guide 00 recommends *low* readahead for the random-access OLTP engines; **ClickHouse is the exception** — its `MergeTree` scans are large sequential reads, so raise readahead back up on the data volume to pull big contiguous granule ranges in one go. |
| `LimitNOFILE` (systemd) | **`500000`** | unset (unit default) | **⚠️** A data node holds an FD per open part file across many partitions + every client/interserver socket; the default `1024`/unit default is exhausted fast → `Too many open files`, failed merges, refused queries. Set it on the **unit**, not `limits.conf` (systemd ignores the latter — Guide 00 §9.3). Pair with `nofile 500000` in `limits.d` for non-systemd shells. |

```bash
# PRODUCTION — not applied in the lab. Raise the FD ceiling on the clickhouse-server unit.
mkdir -p /etc/systemd/system/clickhouse-server.service.d
cat > /etc/systemd/system/clickhouse-server.service.d/90-nofile.conf <<'EOF'
[Service]
LimitNOFILE=500000
EOF
systemctl daemon-reload && systemctl restart clickhouse-server
# VERIFY: cat /proc/$(pgrep -n clickhouse-serv)/limits | grep 'open files' -> 500000
# (the clickhouse deb also ships nofile=500000 in /etc/security/limits.d/clickhouse.conf for login shells)
```

### 9.2 Server memory & caches — `config.d/*.xml` (data nodes only)

Server-wide ceilings and caches. In the lab these are all **unset**, so CH uses its
RAM-ratio default (`0.9`) and default cache sizes — which on a 6 GB VM CH auto-clamps.

```bash
# PRODUCTION — not applied in the lab. Data nodes only. As root:
cat > /etc/clickhouse-server/config.d/nexus-tuning-memory.xml <<'EOF'
<?xml version="1.0"?>
<clickhouse>
    <!-- cap the whole server at 90% of host RAM (leave headroom for page cache + OS) -->
    <max_server_memory_usage_to_ram_ratio>0.9</max_server_memory_usage_to_ram_ratio>
    <!-- or pin an absolute byte ceiling instead of the ratio (0 = use the ratio above): -->
    <max_server_memory_usage>0</max_server_memory_usage>
    <!-- primary-key marks cache: keep hot; 5 GB is the default and a sane floor -->
    <mark_cache_size>5368709120</mark_cache_size>
    <!-- decompressed-block cache: OFF by default; only size it if you enable it per-profile -->
    <uncompressed_cache_size>8589934592</uncompressed_cache_size>
    <max_concurrent_queries>100</max_concurrent_queries>
</clickhouse>
EOF
chown root:clickhouse /etc/clickhouse-server/config.d/nexus-tuning-memory.xml
chmod 0640 /etc/clickhouse-server/config.d/nexus-tuning-memory.xml
systemctl restart clickhouse-server
```

| Setting (`config.d` root) | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `max_server_memory_usage_to_ram_ratio` | `0.9` | unset (default `0.9`) | The server-wide RAM ceiling as a fraction of total RAM; above it, queries are killed with `MEMORY_LIMIT_EXCEEDED` before the OS OOM-killer nukes the whole process. `0.9` leaves 10% for OS page cache. On a co-located node lower it (e.g. `0.7`). |
| `max_server_memory_usage` | `0` (use the ratio) *or* an absolute byte value | unset (`0`) | Absolute alternative to the ratio — set this (e.g. `120000000000` for 120 GB) when the host runs other services and you want a fixed byte cap regardless of total RAM. `0` = defer to the ratio. |
| `mark_cache_size` | `5368709120` (5 GB) — raise for many-column/many-part tables | unset (default 5 GB) | Caches the primary-key **marks** (granule offsets) so range scans skip the index read; a too-small mark cache re-reads `.mrk` files on every query → I/O amplification. 5 GB suits most; size it toward the working set of `system.parts` mark bytes on wide schemas. |
| `uncompressed_cache_size` | `8589934592` (8 GB) | unset (default 8 GB, but **unused**) | Caches *decompressed* blocks. Only helps when `use_uncompressed_cache=1` (see §9.4) — pointless for the typical full-scan OLAP pattern, valuable for repeated point/short-range lookups. Size it *only* if you turn the cache on. |
| `max_concurrent_queries` | `100` | unset (default `100`) | Hard cap on simultaneously-executing queries; the 101st gets `TOO_MANY_SIMULTANEOUS_QUERIES` instead of thrashing every query into memory pressure. Raise with core count; keep below the point where `max_memory_usage × this` exceeds the server ceiling. |

### 9.3 Background pools — `config.d/*.xml` (data nodes only)

Merges, mutations, fetches, and the replication/DDL schedulers run on background thread
pools. Undersized pools on a busy `ReplicatedMergeTree` cluster mean merges fall behind →
part count climbs → `parts_to_delay/throw_insert` (§9.5) starts throttling/rejecting inserts.

```bash
# PRODUCTION — not applied in the lab. Data nodes only. As root:
cat > /etc/clickhouse-server/config.d/nexus-tuning-pools.xml <<'EOF'
<?xml version="1.0"?>
<clickhouse>
    <background_pool_size>16</background_pool_size>
    <background_merges_mutations_concurrency_ratio>2</background_merges_mutations_concurrency_ratio>
    <background_schedule_pool_size>128</background_schedule_pool_size>
</clickhouse>
EOF
chown root:clickhouse /etc/clickhouse-server/config.d/nexus-tuning-pools.xml
chmod 0640 /etc/clickhouse-server/config.d/nexus-tuning-pools.xml
systemctl restart clickhouse-server
```

| Setting (`config.d` root) | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `background_pool_size` | `16` (≈ #cores) | unset (default `16`) | Threads available for background merges + mutations. Too small → merges lag behind ingest, part count explodes, inserts stall; scale it toward core count on merge-heavy ingest. |
| `background_merges_mutations_concurrency_ratio` | `2` | unset (default `2`) | Multiplier of *tasks* over threads (`pool_size × ratio` = concurrent merge/mutation slots) so many small merges can be scheduled without one giant merge hogging a thread; `2` lets short merges interleave. |
| `background_schedule_pool_size` | `128` | unset (default `128`/`512`) | Threads for the *lightweight periodic* jobs — replication queue processing, Keeper session upkeep, DDL-queue polling, part cleanup. A replicated cluster with many tables needs headroom here or replication/DDL fan-out visibly lags. |

### 9.4 Query profiles & quotas — `users.d/*.xml` (data nodes only)

Per-query resource limits and a **separate readonly profile**. The lab's
`users.d/nexus-bootstrap.xml` only grants the `default` user `access_management` — it sets
**no** `<profiles>`/`<quotas>`, so every query runs with CH's stock profile. Production
splits an analyst-facing readonly profile off from the ingest/admin `default`.

```bash
# PRODUCTION — not applied in the lab. Data nodes only (identical file on all 6). As root:
cat > /etc/clickhouse-server/users.d/nexus-profiles.xml <<'EOF'
<?xml version="1.0"?>
<clickhouse>
    <profiles>
        <default>
            <max_memory_usage>10000000000</max_memory_usage>             <!-- 10 GB / query -->
            <max_threads>16</max_threads>                                <!-- = physical cores -->
            <max_execution_time>0</max_execution_time>                   <!-- ingest/admin: no cap -->
            <use_uncompressed_cache>0</use_uncompressed_cache>
            <max_bytes_before_external_group_by>8000000000</max_bytes_before_external_group_by>
            <max_bytes_before_external_sort>8000000000</max_bytes_before_external_sort>
            <max_partitions_per_insert_block>100</max_partitions_per_insert_block>
        </default>
        <readonly>
            <readonly>1</readonly>                                       <!-- SELECT + SET only -->
            <max_memory_usage>5000000000</max_memory_usage>             <!-- tighter: 5 GB -->
            <max_threads>8</max_threads>
            <max_execution_time>60</max_execution_time>                  <!-- kill runaway analyst queries -->
            <max_partitions_to_read>100</max_partitions_to_read>
            <use_uncompressed_cache>1</use_uncompressed_cache>
        </readonly>
    </profiles>
    <quotas>
        <analyst_quota>
            <interval>
                <duration>3600</duration>
                <queries>10000</queries>
                <read_rows>1000000000000</read_rows>
                <execution_time>3600</execution_time>
            </interval>
        </analyst_quota>
    </quotas>
    <users>
        <!-- bind app_ro-style users to the readonly profile + quota -->
        <!-- <analyst><profile>readonly</profile><quota>analyst_quota</quota>...</analyst> -->
    </users>
</clickhouse>
EOF
chown root:clickhouse /etc/clickhouse-server/users.d/nexus-profiles.xml
chmod 0640 /etc/clickhouse-server/users.d/nexus-profiles.xml
# hot-reloaded — no restart needed. VERIFY: SELECT name,value FROM system.settings WHERE name='max_threads'
```

| Setting (profile in `users.d`) | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `max_memory_usage` | `10000000000` (10 GB) in `default`; tighter in `readonly` | unset (default 10 GB) | **Per-query** RAM ceiling (distinct from the §9.2 *server* ceiling). Bounds a single runaway `GROUP BY`/`JOIN` so it fails with `MEMORY_LIMIT_EXCEEDED` instead of starving every other query. Set it *below* `server_ceiling ÷ max_concurrent_queries`. |
| `max_threads` | `16` (= physical cores) | unset (default = #cores) | Threads *per query* for parallel scan/aggregation. CH defaults to the core count; pin it explicitly to keep a fat query from oversubscribing all cores, or lower it in the `readonly` profile so ad-hoc analysts don't monopolise the box. |
| `max_execution_time` | `0` (ingest) / `60` (readonly) | unset (`0` = no limit) | Wall-clock cap that kills long queries. Leave `0` for trusted ingest/admin; set a bound (e.g. `60`) in the analyst profile so a bad `SELECT *` can't run for an hour. |
| `use_uncompressed_cache` | `0` (default) / `1` (point-lookup profiles) | unset (`0`) | Toggles the §9.2 uncompressed cache **per profile**. Off is right for scan-heavy OLAP (would thrash the cache); turn it on only for a profile doing repeated small-range lookups, and size `uncompressed_cache_size` to match. |
| `max_bytes_before_external_group_by` / `_sort` | `8000000000` (8 GB) | unset (`0` = spill disabled) | **Spill-to-disk thresholds.** With `0`, a `GROUP BY`/`ORDER BY` bigger than `max_memory_usage` simply fails; set these to ~half of `max_memory_usage` so big aggregations spill to temp disk and *complete* (slower) instead of erroring. |
| `max_partitions_to_read` | `100` (readonly guard) | unset (`-1` = unlimited) | Rejects a query that would touch more than N partitions — a guard-rail against an accidental full-history scan across every partition. Set it in the analyst profile, leave unlimited for admin/backfills. |
| `max_partitions_per_insert_block` | `100` | unset (default `100`) | Caps how many partitions a single `INSERT` block may create; a badly-keyed insert spraying rows across thousands of partitions is a classic "too many parts" self-DoS. `100` catches the mistake early. |

### 9.5 MergeTree engine settings — `config.d/*.xml` (data nodes) + Keeper notes

`MergeTree`-family defaults that govern part sizing and the insert back-pressure curve. Set
them server-wide via a `<merge_tree>` block in `config.d` (or per-table `SETTINGS`); the lab
tables in §5.6 take all defaults.

```bash
# PRODUCTION — not applied in the lab. Data nodes only. As root:
cat > /etc/clickhouse-server/config.d/nexus-tuning-mergetree.xml <<'EOF'
<?xml version="1.0"?>
<clickhouse>
    <merge_tree>
        <index_granularity>8192</index_granularity>
        <parts_to_delay_insert>150</parts_to_delay_insert>
        <parts_to_throw_insert>300</parts_to_throw_insert>
    </merge_tree>
</clickhouse>
EOF
chown root:clickhouse /etc/clickhouse-server/config.d/nexus-tuning-mergetree.xml
chmod 0640 /etc/clickhouse-server/config.d/nexus-tuning-mergetree.xml
systemctl restart clickhouse-server
```

| Setting (`<merge_tree>` in `config.d`) | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `index_granularity` | `8192` | unset (default `8192`) | Rows per primary-key mark (one index entry per granule). `8192` is the sweet spot; lower it (e.g. `1024`) only for point-lookup-heavy tables to shrink the granule scanned per lookup, at the cost of a bigger mark cache footprint. |
| `parts_to_delay_insert` | `150` | unset (default `150`) | Once an active-part count per partition crosses this, CH **throttles** inserts (sleeps) to let merges catch up — the first line of ingest back-pressure. Raise it only if merges genuinely keep pace on faster storage. |
| `parts_to_throw_insert` | `300` | unset (default `300`) | The hard stop: past this, inserts are **rejected** with `TOO_MANY_PARTS`. Hitting it means merges are losing — fix by enlarging `background_pool_size` (§9.3) or batching inserts, not by blindly raising this. |
| `max_bytes_before_external_group_by`/`_sort` | see §9.4 | — | (Query-scope spill, lives in the profile — cross-referenced here because it pairs with part/merge sizing under memory pressure.) |

**Keeper nodes (`ch-keeper-1/2/3`) — `keeper_config.xml`, not server config.** None of §9.2–9.5
applies to Keeper. Its footprint is the in-memory znode set + snapshot/log storage, so
production tuning is confined to `<coordination_settings>` (already present in §5.3.1) and
sizing the data disk:

| Setting (`<coordination_settings>` in `keeper_config.xml`) | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `session_timeout_ms` | `30000` | `30000` (§5.3.1) | How long a server's Keeper session survives a network blip before its ephemeral znodes drop (triggering replica re-registration). Too low → spurious replica flaps on a busy backplane. Lab already sane. |
| `operation_timeout_ms` | `10000` | `10000` (§5.3.1) | Per-op deadline; raise only if a large `ON CLUSTER` DDL fan-out legitimately runs long. |
| `snapshot_distance` / `reserved_log_items` | tune for log-disk headroom | unset (defaults) | On a high-DDL/high-replication cluster the RAFT log grows between snapshots; give the Keeper `log`/`snapshots` dirs their own fast disk and keep an eye on `du` — a full Keeper disk stalls **all** coordination (and thus every insert). Keeper is CPU-light but latency-critical: keep the 3 nodes on low-contention hosts. |

> **The two-file rule, restated:** a *server* ceiling/cache/pool/MergeTree knob that ends up
> in a `users.d` profile is ignored, and a *query/session* knob placed at the `config.d`
> `<clickhouse>` root is ignored. When a setting misbehaves, check `system.server_settings`
> (config.d scope) vs `system.settings` (profile scope) to confirm it actually loaded.

---

### Cross-references

- **0.G.5 architecture + transients:** memory `project_nexus_infra_analytics_phase`; ADRs 0028 (Keeper) / 0029 (shard×replica) / 0031 (round-robin DNS) / 0032 (NFS backup)
- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (ClickHouse `.41`–`.49`)
- **Automated equivalents:** `nexus-infra-analytics/terraform/envs/analytics-clickhouse/role-overlay-clickhouse-*.tf`
- **Gateway NFS + DNS consumed:** [`01-foundation-nexus-gateway.md`](./01-foundation-nexus-gateway.md)
- **Previous guide:** [`12-oltp-mongodb-sharded.md`](./12-oltp-mongodb-sharded.md)
- **Next guide:** Guide 14 — Analytics · StarRocks shared-nothing (3 FE + 3 BE). See [`INDEX.md`](../INDEX.md).
