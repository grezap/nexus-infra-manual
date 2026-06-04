# Guide 14 — Analytics · StarRocks shared-nothing (3 FE + 3 BE)

> **Mirrors:** `nexus-infra-analytics` — the `analytics-starrocks` env overlays
> (`starrocks-fe-bootstrap`, `starrocks-be-join`, `starrocks-schema-bootstrap`,
> `starrocks-tls`, `starrocks-backup-repo`, `starrocks-nftables-backplane`) —
> Phase 0.G.6 / ADR-0030. The automated lab drives the FE `--helper` join + the
> `ALTER SYSTEM` registration over SSH; by hand we run the same `start_fe.sh` +
> `mysql` commands directly.

---

## 1. Overview & purpose

The second Analytics engine: a 6-node **StarRocks** MPP (massively-parallel)
analytics database in **shared-nothing** mode — each backend stores its own
data locally. **3 FE + 3 BE.**

- **Frontends (`sr-fe-leader`, `sr-fe-follower-1/2`, `.31/.32/.33`)** — the FE
  holds cluster **metadata** (catalog, tablet map) in an embedded **BDB-JE**
  store with a majority quorum: **1 leader + 2 followers**. FEs accept queries on
  the **MySQL protocol** (`:9030`) and forward DDL to the leader.
- **Backends (`sr-be-1/2/3`, `.34/.35/.36`)** — the BE stores tablets + executes
  query fragments. A table is `DISTRIBUTED BY HASH(...) BUCKETS n` (sharded into
  tablets across the BEs) × `replication_num=3` (each tablet replicated to all 3
  BEs).
- **Front door** = **round-robin DNS** `starrocks-fe.nexus.lab` → the 3 FE
  (ADR-0031, no VIP — any FE serves queries + forwards DDL to the leader).

Runs on **JDK 21**. Cluster-internal FE↔BE traffic binds the **VMnet10
backplane** (`priority_networks`) and is gated by nftables; the MySQL query
protocol uses `--skip-ssl` (see the MariaDB-client quirk in §2). This is the
**shared-nothing** topology — the **shared-data** variant (FE + stateless CN over
MinIO object storage) is Guide 15.

**Why a second analytics engine** (alongside ClickHouse): StarRocks is a
MySQL-protocol MPP engine with a cost-based optimizer + a different sharding model
(FE/BE separation, tablet replication) — a distinct point in the analytics design
space.

**Dependency:**
- **Guides 00 + 04** — foundation alive; Vault PKI.
- **Guide 01** — gateway dnsmasq (round-robin record) + NFS export (backup repo).
- 6 `deb13` nodes baselined per Guide 00 §5.B, dual-NIC static.

> **By-hand divergence:** issue certs with the `vault` CLI, download the StarRocks
> tarball + run `start_fe.sh`/`mysql` directly — no Vault Agent. The VMware
> cold-rebuild flakes (vmrun power-on storm, FE-leader-no-service-NIC) are
> clone-time artifacts.

---

## 2. Component primer

- **StarRocks (MPP, FE/BE split).** A columnar MPP OLAP engine. The **FE**
  (frontend, Java) is the brain — SQL parsing, planning, metadata, and the MySQL
  protocol endpoint; the **BE** (backend, C++) is the muscle — stores tablets +
  runs query fragments in parallel. *Why the split:* metadata HA (FE quorum) is
  decoupled from data scale (add BEs). *Otherwise:* ClickHouse (Guide 13, no
  central metadata node) — a different MPP shape.
- **FE quorum (BDB-JE).** The FEs replicate metadata via **Berkeley DB Java
  Edition** with a majority quorum: one **leader** (takes metadata writes) + N
  **followers**. 3 FEs tolerate one loss. *Why:* the catalog must survive an FE
  failure.
- **The `--helper` one-shot join.** A fresh follower first-starts with
  `start_fe.sh --helper <leader>:9010` — a one-shot that pulls + persists the
  BDB-JE metadata from the leader, then you hand it to systemd (subsequent
  restarts read the persisted meta, no `--helper`). The leader must already know
  the follower (`ALTER SYSTEM ADD FOLLOWER`) before the join.
- **`DISTRIBUTED BY HASH … BUCKETS n` + `replication_num`.** A table's rows are
  hash-distributed into `n` **tablets** (sharding); `replication_num=3` keeps 3
  copies of each tablet across the BEs (HA + read parallelism). *vs. ClickHouse:*
  StarRocks does tablet-level sharding + replication transparently — you don't
  hand-build a `Distributed`-over-`Replicated` pair.
- **MySQL protocol + the `--skip-ssl` quirk.** StarRocks speaks the MySQL wire
  protocol on `:9030`, so you query it with the `mysql` client. But Debian 13
  ships the **MariaDB 11.8** client, which **requires TLS for password auth by
  default** — and StarRocks' MySQL endpoint here isn't TLS-wrapped — so every
  `mysql` invocation needs **`--skip-ssl`** (transient S5).
- **JDK 21 + JAVA_HOME.** The FE is Java; Debian 13 has no `openjdk-17`, so the
  lab uses **openjdk-21** with `JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64`
  (transient S2). The `--helper` one-shot must be invoked with that `JAVA_HOME`.
- **`priority_networks`.** FE + BE bind their cluster-internal traffic to the
  **VMnet10 backplane** (`priority_networks = 192.168.10.0/24`) so FE↔BE +
  BDB-JE replication ride the trusted segment.
- **The binary source (transient S1).** StarRocks' old release CDN 403s
  superseded versions; the **download-portal API**
  (`download.starrocks.io/en-US/download/releases`) lists current per-line patches
  with an OS-suffixed `fileUrl` (`StarRocks-<ver>-ubuntu-amd64.tar.gz`) + md5.
  The lab landed **3.5.17**. (The GitHub "Source code" tarball is uncompiled —
  not usable.)

---

## 3. Prerequisites

| # | Requirement | One-command verify |
|---|---|---|
| 1 | Foundation alive (Guides 00 + 04); Vault PKI usable | `vault read pki_int/cert/ca` on vault-1 returns the intermediate |
| 2 | 6 `deb13` nodes baselined, dual-NIC static `.31–.36` | those 6 answer `:22` |
| 3 | Gateway NFS export `/srv/nfs/analytics-backups` (`fsid=0`) | `ssh …@1 'sudo exportfs -v \| grep analytics-backups'` |
| 4 | The StarRocks 3.5.17 ubuntu tarball reachable (download-portal `fileUrl`) | `ssh …@31 'curl -sI <fileUrl> \| head -1'` → `200`/`206` |
| 5 | Internet egress on the nodes (JDK + tarball) | `ssh …@31 'curl -sI https://download.starrocks.io \| head -1'` → `200` |

> StarRocks **3.5.17**, **JDK 21**. Cluster front door: round-robin DNS
> `starrocks-fe.nexus.lab` → `.31/.32/.33`. FE ports: `9030` (MySQL query),
> `8030` (HTTP), `9020` (RPC), `9010` (edit log / `--helper`). BE ports: `9060`
> (RPC), `9050` (heartbeat — the `ADD BACKEND` port), `8060`, `8040`.

---

## 4. Target topology

| Node | Role | VMnet11 | VMnet10 | RAM |
|---|---|---|---|---|
| `sr-fe-leader` | FE (initial leader, BDB-JE) | `.31` | `.10.31` | 4 GB |
| `sr-fe-follower-1` | FE follower | `.32` | `.10.32` | 4 GB |
| `sr-fe-follower-2` | FE follower | `.33` | `.10.33` | 4 GB |
| `sr-be-1` | BE (tablets + storage) | `.34` | `.10.34` | 6 GB |
| `sr-be-2` | BE | `.35` | `.10.35` | 6 GB |
| `sr-be-3` | BE | `.36` | `.10.36` | 6 GB |

> FE↔BE + BDB-JE on the **VMnet10 backplane** (`priority_networks`). PKI role
> **`starrocks-server`** (per-node certs for HTTP/future use). Demo DB `nexus`,
> table `nexus.events` (`BUCKETS 6` × `replication_num 3`). Backup repo: NFS
> `/srv/nfs/analytics-backups` (broker-less `file://`).

---

## 5. Step-by-step build

> **WHERE:** node steps as `nexusadmin`→`sudo -i`. `mysql` against the FE always
> uses **`--skip-ssl`**. `vault` on **`vault-1`**.

### 5.1 — Per-node base install

> **Step 5.1.1 — Install JDK 21 + the StarRocks 3.5.17 tarball (all 6)**
> **WHERE:** each node, root shell.
> **WHY:** the FE needs JDK 21 (Debian 13 has no openjdk-17, S2). Download the
> StarRocks tarball **from the download-portal `fileUrl`** (the old CDN 403s, S1)
> to `/var/tmp` (the FE/BE root has small `/tmp` tmpfs that ENOSPCs on the 2.2 GB
> tarball, S3). Both FE + BE binaries are in the one tarball.
> **WHAT:**
> ```bash
> apt-get update -qq && apt-get install -y openjdk-21-jdk curl openssl
> # JAVA_HOME (FE needs it; bake into the env)
> echo 'JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64' >> /etc/environment
> # Download to /var/tmp (NOT /tmp -- tmpfs ENOSPC). <fileUrl> from the portal API.
> cd /var/tmp
> curl -fSL '<StarRocks-3.5.17-ubuntu-amd64 fileUrl from download.starrocks.io portal>' -o starrocks.tar.gz
> tar xzf starrocks.tar.gz && rm starrocks.tar.gz
> mv StarRocks-3.5.17-ubuntu-amd64 /opt/starrocks    # contains fe/ + be/
> id starrocks >/dev/null 2>&1 || useradd --system -m -d /opt/starrocks -s /usr/sbin/nologin starrocks
> chown -R starrocks:starrocks /opt/starrocks
> ```
> **EXPECTED:** `/opt/starrocks/fe` + `/opt/starrocks/be` exist.
> **VERIFY:** `java -version` → `21`; `ls /opt/starrocks/fe/bin/start_fe.sh /opt/starrocks/be/bin/start_be.sh`.

> **Step 5.1.2 — Install the systemd units (FE on .31–.33, BE on .34–.36)**
> **WHERE:** FE nodes + BE nodes, root shell.
> **WHY:** `start_fe.sh --daemon` / `start_be.sh --daemon` wrapped in systemd, run
> as `starrocks` with `JAVA_HOME` in the environment.
> **WHAT (FE nodes — `nexus-starrocks-fe.service`):**
> ```bash
> cat > /etc/systemd/system/nexus-starrocks-fe.service <<'EOF'
> [Unit]
> Description=Nexus StarRocks FE
> After=network-online.target
> Wants=network-online.target
> [Service]
> Type=forking
> User=starrocks
> Group=starrocks
> Environment=JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
> ExecStart=/opt/starrocks/fe/bin/start_fe.sh --daemon
> ExecStop=/opt/starrocks/fe/bin/stop_fe.sh
> Restart=on-failure
> LimitNOFILE=655350
> [Install]
> WantedBy=multi-user.target
> EOF
> systemctl daemon-reload
> ```
> **WHAT (BE nodes — `nexus-starrocks-be.service`):**
> ```bash
> cat > /etc/systemd/system/nexus-starrocks-be.service <<'EOF'
> [Unit]
> Description=Nexus StarRocks BE
> After=network-online.target
> Wants=network-online.target
> [Service]
> Type=forking
> User=starrocks
> Group=starrocks
> ExecStart=/opt/starrocks/be/bin/start_be.sh --daemon
> ExecStop=/opt/starrocks/be/bin/stop_be.sh
> Restart=on-failure
> LimitNOFILE=655350
> [Install]
> WantedBy=multi-user.target
> EOF
> systemctl daemon-reload
> ```
> **VERIFY:** `systemctl cat nexus-starrocks-fe.service` (or `-be`) shows the right
> ExecStart. (Don't start yet — config first.)

> **Step 5.1.3 — nftables: backplane trust + service ports + MySQL client**
> **WHERE:** each node, root shell.
> **WHY:** FE↔BE + BDB-JE ride the backplane (trust VMnet10); the MySQL query
> `:9030` + FE HTTP `:8030` are reachable on VMnet11.
> **WHAT:** add to `/etc/nftables.conf` `chain input` (before `counter drop`):
> ```
> iifname "nic1" ip saddr 192.168.10.0/24 accept comment "trusted cluster backplane (VMnet10)"
> # FE: iifname "nic0" ip saddr 192.168.70.0/24 tcp dport { 9030, 8030 } accept
> # BE: iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 8040 accept   (web UI)
> ```
> Install `mariadb-client` (for `mysql`) on the build host or a node:
> `apt-get install -y mariadb-client`.
> **VERIFY:** `nft list chain inet filter input | grep nic1`.

### 5.2 — Per-node PKI certs

> **Step 5.2.1 — PKI role + per-node certs**
> **WHERE:** issue on `vault-1`; place on each node.
> **WHY:** per-node certs (HTTP endpoints / future hardening). Cluster-internal
> FE/BE security here relies on the trusted backplane + nftables (SR's internal
> protocol isn't full mTLS like ClickHouse, and the MySQL endpoint uses
> `--skip-ssl`). Place at `/opt/starrocks/{fe,be}/conf/tls/` (or a node tls dir).
> **WHAT (on vault-1):**
> ```bash
> vault write pki_int/roles/starrocks-server \
>   allowed_domains='nexus.lab,starrocks-fe.nexus.lab,sr-fe-leader,sr-fe-follower-1,sr-fe-follower-2,sr-be-1,sr-be-2,sr-be-3,localhost' \
>   allow_subdomains=true allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
> # per node: issue (CN=<host>.nexus.lab, FE nodes add starrocks-fe.nexus.lab,
> #   ip_sans=<vmnet11>,<vmnet10>,127.0.0.1) -> server.crt/server.key/ca.crt
> ```
> **EXPECTED:** certs placed per node.
> **VERIFY:** `sudo openssl x509 -in <tls-dir>/server.crt -noout -subject` → node CN.

### 5.3 — Bring up the FE quorum (`--helper`)

> **Step 5.3.1 — Append `priority_networks` + heap to each FE's `fe.conf`; start the leader**
> **WHERE:** FE nodes, root shell; leader (`.31`) first.
> **WHY:** bind FE to the backplane + right-size the heap. The first FE started
> with empty metadata **becomes the leader**.
> **WHAT (on all 3 FE — idempotent append):**
> ```bash
> CONF=/opt/starrocks/fe/conf/fe.conf
> grep -q '^priority_networks' "$CONF" || cat >> "$CONF" <<'EOF'
>
> # --- nexus per-host settings ---
> priority_networks = 192.168.10.0/24
> JAVA_OPTS="-Xmx2g -XX:+UseG1GC"
> EOF
> ```
> **WHAT (on `sr-fe-leader` only — start + wait for MySQL :9030):**
> ```bash
> systemctl enable --now nexus-starrocks-fe.service
> # wait for the query port:
> until mysql --skip-ssl -h 127.0.0.1 -P 9030 -u root -N -e 'SHOW FRONTENDS' >/dev/null 2>&1; do sleep 8; done
> mysql --skip-ssl -h 127.0.0.1 -P 9030 -u root -e 'SHOW FRONTENDS'
> ```
> **EXPECTED:** the leader answers on `:9030`; `SHOW FRONTENDS` lists 1 FE (LEADER).
> **VERIFY:** `mysql --skip-ssl -h 192.168.70.31 -P 9030 -u root -N -e "SHOW FRONTENDS"`
> shows one row, role LEADER, Alive `true`.

> **Step 5.3.2 — Register + join each follower (`ADD FOLLOWER` → `--helper`)**
> **WHERE:** the leader (register) + each follower (join), root shell.
> **WHY:** the leader must know the follower (`ALTER SYSTEM ADD FOLLOWER
> '<b10>:9010'`) before the follower first-starts with `--helper` to pull the
> BDB-JE metadata; then hand it to systemd.
> **WHAT (per follower — sr-fe-follower-1 `.32`/`.10.32`, then -2 `.33`/`.10.33`):**
> ```bash
> # On the leader (.31): register the follower's backplane edit-log endpoint:
> mysql --skip-ssl -h 127.0.0.1 -P 9030 -u root -e "ALTER SYSTEM ADD FOLLOWER '192.168.10.32:9010'"
> ```
> ```bash
> # On the follower (.32): one-shot --helper join (persists BDB-JE meta), then systemd:
> export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
> if [ ! -d /opt/starrocks/fe/meta/bdb ]; then
>   sudo -u starrocks JAVA_HOME="$JAVA_HOME" /opt/starrocks/fe/bin/start_fe.sh --helper 192.168.10.31:9010 --daemon
>   sleep 15
>   sudo -u starrocks JAVA_HOME="$JAVA_HOME" /opt/starrocks/fe/bin/stop_fe.sh || true
>   sleep 3
> fi
> systemctl enable --now nexus-starrocks-fe.service
> ```
> **EXPECTED:** each follower joins the quorum.
> **VERIFY (on the leader):** `mysql --skip-ssl … -e "SHOW FRONTENDS"` → **3 FE**,
> 1 LEADER + 2 FOLLOWER, all Alive `true`.

### 5.4 — Join the BEs

> **Step 5.4.1 — Append `priority_networks` + `storage_root_path` to `be.conf`; start; register**
> **WHERE:** BE nodes (`.34/.35/.36`), then the FE leader.
> **WHY:** bind BE to the backplane + set its storage dir, start the BE, then
> register it on the FE leader (`ALTER SYSTEM ADD BACKEND '<b10>:9050'` — the
> **heartbeat** port).
> **WHAT (on each BE):**
> ```bash
> CONF=/opt/starrocks/be/conf/be.conf
> grep -q '^priority_networks' "$CONF" || cat >> "$CONF" <<'EOF'
>
> # --- nexus per-host settings ---
> priority_networks = 192.168.10.0/24
> storage_root_path = /opt/starrocks/be/storage
> EOF
> install -d -o starrocks -g starrocks /opt/starrocks/be/storage
> systemctl enable --now nexus-starrocks-be.service
> ```
> ```bash
> # On the FE leader (.31): register each BE by its backplane heartbeat endpoint:
> for b in 34 35 36; do
>   mysql --skip-ssl -h 127.0.0.1 -P 9030 -u root -e "ALTER SYSTEM ADD BACKEND '192.168.10.$b:9050'"
> done
> ```
> **EXPECTED:** 3 BEs come up + register.
> **VERIFY (on the leader):** `mysql --skip-ssl … -e "SHOW BACKENDS"` → **3 BE**,
> all Alive `true`.

### 5.5 — Round-robin DNS front door

> **Step 5.5.1 — Publish `starrocks-fe.nexus.lab` → the 3 FE (gateway)**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** any FE serves queries (forwards DDL to the leader), so round-robin DNS
> over the 3 FE is the stable client endpoint (ADR-0031). Use the `addn-hosts`
> hosts-file form (same as Guide 13 — `host-record` keeps only the last IP).
> **WHAT:**
> ```bash
> cat > /etc/dnsmasq-starrocks.hosts <<'EOF'
> 192.168.70.31 starrocks-fe.nexus.lab
> 192.168.70.32 starrocks-fe.nexus.lab
> 192.168.70.33 starrocks-fe.nexus.lab
> EOF
> echo 'addn-hosts=/etc/dnsmasq-starrocks.hosts' > /etc/dnsmasq.d/starrocks-records.conf
> dnsmasq --test && systemctl reload dnsmasq
> ```
> **EXPECTED:** the name resolves to all 3 FE.
> **VERIFY:** `dig @192.168.70.1 starrocks-fe.nexus.lab +short` → the 3 FE IPs.

### 5.6 — Schema bootstrap (the exit gate)

> **Step 5.6.1 — Create a sharded + replicated table, insert, verify**
> **WHERE:** the FE leader (or via the round-robin name), root shell.
> **WHY:** prove sharding + replication end-to-end — `DISTRIBUTED BY HASH(...)
> BUCKETS 6` shards rows into tablets across the 3 BEs, `replication_num=3`
> replicates each tablet to all 3 BEs.
> **WHAT (`RP` = `mysql --skip-ssl -h 127.0.0.1 -P 9030 -u root`):**
> ```bash
> RP="mysql --skip-ssl -h 127.0.0.1 -P 9030 -u root"
> $RP -e "CREATE DATABASE IF NOT EXISTS nexus"
> $RP -e "CREATE TABLE nexus.events (event_id BIGINT, ts DATETIME, bucket INT, payload VARCHAR(64)) \
>         DUPLICATE KEY(event_id) DISTRIBUTED BY HASH(event_id) BUCKETS 6 PROPERTIES(\"replication_num\"=\"3\")"
> # insert a few hundred rows (build the VALUES list however you like):
> $RP -e "INSERT INTO nexus.events VALUES (1,'2026-06-04 00:00:00',1,'demo-1'),(2,'2026-06-04 00:00:01',2,'demo-2') /* … up to 200 … */"
> $RP -N -e "SELECT count(*) FROM nexus.events"
> $RP -e "SHOW CREATE TABLE nexus.events\G" | grep '"replication_num" = "3"'
> ```
> **EXPECTED:** the `count(*)` matches the rows inserted; `SHOW CREATE TABLE`
> confirms `"replication_num" = "3"`.
> **VERIFY:** `$RP -e "SHOW PARTITIONS FROM nexus.events"` shows the tablets;
> `$RP -e "ADMIN SHOW REPLICA DISTRIBUTION FROM nexus.events"` shows replicas
> spread across all 3 BEs (sharded ×6 BUCKETS AND replicated ×3 — the exit gate).

### 5.7 — (Optional) NFS backup repo

> **Step 5.7.1 — Mount the analytics-backups NFS + register a broker-less repo**
> **WHERE:** FE leader (repo) + each BE (mount), root shell.
> **WHY:** StarRocks `BACKUP` to a broker-less `file://` repository on the
> gateway's NFS export (`fsid=0` → mount via `:/`, S6). Broker-less = no separate
> Broker process.
> **WHAT (outline):**
> ```bash
> # on each BE + FE: mount the NFS export
> apt-get install -y nfs-common
> echo '192.168.70.1:/  /var/lib/nexus-analytics-backups  nfs4  rw,hard,bg,_netdev,vers=4.2,sec=sys  0  0' >> /etc/fstab && mount -a
> # on the FE leader: register the repo + snapshot
> RP -e "CREATE REPOSITORY nexus_backup WITH BROKER ON LOCATION 'file:///var/lib/nexus-analytics-backups/starrocks'"
> #   (BROKER ON / broker-less: file:// repos don't need a Broker in SR 3.x)
> ```
> **EXPECTED:** the repo registers; a `BACKUP SNAPSHOT` writes to the NFS export
> (best-effort/soft on a fresh repo — non-fatal).
> **VERIFY:** `findmnt /var/lib/nexus-analytics-backups`; `SHOW REPOSITORIES`
> lists `nexus_backup`.

---

## 6. Validation — by-hand acceptance smoke

From the **host** (`mysql --skip-ssl`). Condensed from the 73-check `smoke-0.G.6.ps1`.

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | 6 nodes reachable | `Test-NetConnection` (FE `:9030`, BE `:8040`) | all `True` |
| 2 | FE quorum | `SHOW FRONTENDS` | 3 FE: 1 LEADER + 2 FOLLOWER, all Alive |
| 3 | BE cluster | `SHOW BACKENDS` | 3 BE, all Alive |
| 4 | Round-robin DNS | `dig @192.168.70.1 starrocks-fe.nexus.lab +short` | 3 FE IPs |
| 5 | Query via the FE name | `mysql --skip-ssl -h starrocks-fe.nexus.lab -P 9030 -u root -e 'SELECT 1'` | `1` |
| 6 | Table sharded | `SHOW PARTITIONS FROM nexus.events` / replica distribution | tablets across all 3 BE |
| 7 | Replicated ×3 | `SHOW CREATE TABLE nexus.events` | `"replication_num" = "3"` |
| 8 | Row round-trip | `SELECT count(*) FROM nexus.events` | matches inserted |
| 9 | **FE follower loss** | stop an FE follower; quorum stays (2/3) | leader still serves; queries work (then restart) |
| 10 | **BE loss** | stop one BE; query via FE | still returns (replicas on the other 2 BE; then restart) |

**1–8 green ⇒ Guide 14 satisfied.** 9 + 10 are the HA proofs (FE quorum majority +
BE tablet replication).

---

## 7. Teardown / reset

```bash
for ip in 34 35 36; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-starrocks-be'; done
for ip in 31 32 33; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-starrocks-fe'; done
# then vmrun stop + deleteVM each of the 6 (Guide 00 §7).
```
To reset metadata for a clean re-bootstrap: on FE nodes
`sudo rm -rf /opt/starrocks/fe/meta/bdb /opt/starrocks/fe/meta/image`; on BE nodes
`sudo rm -rf /opt/starrocks/be/storage/*`, then re-run §5.3–5.6.

> The gateway DNS record + NFS export belong to Guide 01. A fresh FE bootstrap +
> BE join rebuilds the cluster; no external stale-state to clean.

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `mysql` to the FE fails on auth / TLS | Debian 13's MariaDB 11.8 client requires TLS for password auth; the SR endpoint isn't TLS | always pass **`--skip-ssl`** (S5). |
| FE won't start / Java errors | no openjdk-17 on Debian 13; wrong `JAVA_HOME` | use **openjdk-21** + `JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64` (S2). |
| Tarball download fails / 403 | StarRocks' old release CDN 403s superseded versions | download from the **download-portal `fileUrl`** (`download.starrocks.io/en-US/download/releases`), OS-suffixed (S1). |
| Tarball extract: `No space left on device` | small `/tmp` tmpfs vs. the 2.2 GB tarball | download + extract in **`/var/tmp`** (disk-backed), S3. |
| Follower won't join / no metadata | started without `--helper`, or not registered on the leader first | `ALTER SYSTEM ADD FOLLOWER '<b10>:9010'` on the leader, **then** first-start the follower with `--helper <leader>:9010` (§5.3.2). |
| `ADD BACKEND` then BE shows Dead | wrong port (used `9060` not the heartbeat `9050`), or backplane blocked | register on `<b10>:9050`; confirm nftables trusts VMnet10 (§5.4.1). |
| FE leader boots with no service-NIC IP | VMware didn't attach the NIC at power-on | `vmrun reset` power-cycle the VM (S7) — clone-time only, not relevant to a by-hand build. |
| `starrocks-fe.nexus.lab` resolves to one FE | dnsmasq `host-record` keeps only the last IP | use the `addn-hosts` hosts-file form (§5.5.1). |

---

### Cross-references

- **0.G.6 architecture + transients (S1–S7):** memory `project_nexus_infra_analytics_phase`; ADR-0030 (FE/BE topology), ADR-0031 (round-robin DNS), ADR-0032 (NFS backup)
- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (StarRocks `.31`–`.36`)
- **Automated equivalents:** `nexus-infra-analytics/terraform/envs/analytics-starrocks/role-overlay-starrocks-*.tf`
- **Contrast:** [`13-analytics-clickhouse.md`](./13-analytics-clickhouse.md) (the other analytics engine)
- **Previous guide:** [`13-analytics-clickhouse.md`](./13-analytics-clickhouse.md)
- **Next guide:** Guide 15 — Analytics · StarRocks shared-data (3 FE + 2 CN over MinIO). See [`INDEX.md`](../INDEX.md). (Depends on Guide 16's MinIO for object storage — note the forward dependency.)
