# Guide 09 — OLTP · Percona XtraDB Cluster (3 PXC + 2 ProxySQL + VRRP VIP)

> **Mirrors:** `nexus-infra-oltp` — the `oltp-pxc-node` + `oltp-proxysql-node`
> Packer templates + the `oltp-percona` env overlays (`percona-tls`,
> `percona-config`, `percona-galera-bootstrap`, `proxysql-config`,
> `proxysql-keepalived`, `percona-nftables-backplane`). Where the automated lab
> renders configs via Vault Agents and drives the bootstrap over SSH, this guide
> installs by hand and **issues the cert + seeds the secrets directly**.

---

## 1. Overview & purpose

A highly-available MySQL service: a **3-node Percona XtraDB Cluster** (synchronous
**Galera** multi-master replication) behind a **2-node ProxySQL** layer, fronted
by a single **VRRP floating VIP** (`192.168.70.50`) so clients have one stable
endpoint. **5 nodes total.**

- **PXC (`pxc-node-1/2/3`, `.51/.52/.53`)** — Galera replicates **every** write
  to all 3 nodes synchronously before commit (a write isn't acked until the
  cluster has it). This is **replication, not sharding** — every node holds the
  full dataset. Single-writer mode (ProxySQL routes writes to one node) avoids
  multi-master write conflicts.
- **ProxySQL (`proxysql-1/2`, `.54/.55`)** — a MySQL-protocol proxy that
  health-probes the Galera backends and routes: writes → the current writer node,
  reads → the Synced reader nodes. It hides node failover from clients.
- **VRRP VIP (`.50`)** — keepalived floats one IP between the 2 ProxySQL nodes;
  clients connect to `mysql://…@192.168.70.50:6033`. If the MASTER ProxySQL dies,
  the VIP moves to the BACKUP in ~1 s.

**Why this shape:** it exercises the LB-tier HA promise (ADR-0025) — the cluster's
HA is only as good as its front door, so the front door (ProxySQL) is itself HA
via the VIP. mTLS throughout.

**Dependency:**
- **Guides 00–04** — foundation alive; Guide 04 PKI + Part-D scaffolding (this
  guide creates the `percona-server` PKI role + seeds the cluster secrets in KV).
- 5 `deb13` nodes baselined per Guide 00 §5.B, dual-NIC static.

> **By-hand divergence:** issue the cert with the `vault` CLI, seed the 4 cluster
> secrets in Vault KV + place them on the nodes — no per-node Vault Agent.

---

## 2. Component primer

- **Percona XtraDB Cluster (PXC) + Galera (`wsrep`).** PXC is MySQL + the
  **Galera** synchronous-replication library (`wsrep` = write-set replication).
  Every transaction is certified + applied on **all** nodes before commit, so all
  nodes are always consistent (no replica lag). *Why (vs. async MySQL
  replication):* no stale reads, no lost writes on failover. *Otherwise:* primary/
  replica async (Guide 10's Patroni does that for Postgres) or sharding (no —
  Galera replicates the whole set; sharding is Vitess/Citus, Guides 21/22).
- **SST / IST.** When a node joins, it syncs state: **SST** (State Snapshot
  Transfer — a full copy, via `xtrabackup-v2`, online/non-blocking) for a fresh
  node; **IST** (Incremental State Transfer — just the missing write-sets) for a
  node that was briefly away. *Why xtrabackup-v2:* no donor read-lock.
- **`galera_new_cluster` / bootstrap.** A Galera cluster has to be *started* by
  exactly one node declaring itself the primary component (`--wsrep-new-cluster`,
  i.e. `wsrep_cluster_address=gcomm://` with no members). The others then join it.
  **Running bootstrap when the cluster already exists causes split-brain** — so
  the build probes `wsrep_cluster_size` first.
- **`pxc_strict_mode=ENFORCING`.** Refuses Galera anti-patterns (MyISAM writes,
  tables without a primary key, explicit `LOCK TABLES`) at write time instead of
  letting them silently diverge the cluster.
- **ProxySQL + `mysql_galera_hostgroups`.** ProxySQL groups the PXC backends by
  Galera state (read from each via the `clustercheck` user): **writer** (hg 10,
  exactly one), **backup_writer** (hg 20, failover writers), **reader** (hg 30),
  **offline** (hg 40, Donor/Desync). It auto-shuffles nodes between groups as
  their `wsrep` state changes. *Why:* clients never need to know which node is the
  writer.
- **keepalived VRRP VIP.** A floating IP advertised by the higher-priority
  (MASTER) ProxySQL; a `vrrp_script` health-check demotes priority if ProxySQL
  stops serving, flipping the VIP to the BACKUP. **Unicast VRRP** (not multicast)
  — multicast `224.0.0.18` doesn't reliably traverse VMware Workstation's VMnet11,
  so both nodes would go split-brain MASTER. *Why a VIP:* one stable client
  endpoint that survives a ProxySQL node loss (ADR-0025).

---

## 3. Prerequisites

| # | Requirement | One-command verify |
|---|---|---|
| 1 | Foundation alive (Guides 00–04); Vault PKI usable | `vault read pki_int/cert/ca` on vault-1 returns the intermediate |
| 2 | 5 `deb13` nodes baselined, dual-NIC static `.51/.52/.53` (PXC) + `.54/.55` (ProxySQL) | those 5 answer `:22` |
| 3 | Vault root token on build host | `Test-Path ~/.nexus/secrets/vault-cluster-init.json` |
| 4 | Internet egress on the nodes | `ssh …@51 'curl -sI https://repo.percona.com \| head -1'` → `200` |

> Versions: **Percona XtraDB Cluster 8.0** (Galera 4), **ProxySQL 2.x**. Cluster
> name `nexus-pxc`. Galera traffic on the **VMnet10 backplane** (`4444` SST,
> `4567` replication, `4568` IST); client SQL on `3306` (PXC) / `6033` (ProxySQL)
> over VMnet11.

---

## 4. Target topology

| Node | Role | VMnet11 | VMnet10 | server_id / prio | vCPU/RAM/disk |
|---|---|---|---|:--:|---|
| `pxc-node-1` | PXC Galera (bootstrap node) | `.51` | `.10.51` | server_id 1 | 2 / 8 GB / 60 GB |
| `pxc-node-2` | PXC Galera | `.52` | `.10.52` | server_id 2 | 2 / 8 GB / 60 GB |
| `pxc-node-3` | PXC Galera | `.53` | `.10.53` | server_id 3 | 2 / 8 GB / 60 GB |
| `proxysql-1` | ProxySQL + keepalived **MASTER** | `.54` | `.10.54` | prio 110 | 2 / 2 GB / 30 GB |
| `proxysql-2` | ProxySQL + keepalived **BACKUP** | `.55` | `.10.55` | prio 100 | 2 / 2 GB / 30 GB |
| **VIP** | `proxysql.nexus.lab` client front door | **`.50`** | — | VRRP | — |

> Cluster secrets (4) in Vault KV `nexus/oltp/percona/`: `cluster-password`
> (wsrep_sst + VRRP auth), `monitor-password` (clustercheck), `smoke-rw-password`,
> `proxysql-admin-password`. PKI role **`percona-server`**. Client connection:
> `mysql://smoke-rw@192.168.70.50:6033/nexus_smoke`.

---

## 5. Step-by-step build

> **WHERE:** node steps as `nexusadmin`→`sudo -i`; `mysql` under `sudo` (TLS
> material is `root:mysql`). `vault` on **`vault-1`** (root token). Bootstrap from
> `pxc-node-1`.

### 5.1 — PXC nodes: base install + nftables (all 3)

> **Step 5.1.1 — Install Percona XtraDB Cluster 8.0**
> **WHERE:** each PXC node, root shell.
> **WHY:** the PXC server + Galera 4 lib + `xtrabackup`. Disable the stock unit
> (we run `nexus-percona`).
> **WHAT:**
> ```bash
> apt-get update -qq && apt-get install -y curl gnupg openssl
> curl -fsSL https://repo.percona.com/apt/percona-release_latest.generic_all.deb -o /tmp/percona-release.deb
> apt-get install -y /tmp/percona-release.deb
> percona-release setup pxc-80
> apt-get update -qq && apt-get install -y percona-xtradb-cluster percona-xtrabackup-80
> systemctl disable --now mysql 2>/dev/null || true
> install -d -o mysql -g mysql -m0750 /etc/nexus-percona /etc/nexus-percona/tls /var/lib/nexus-percona
> install -d -o mysql -g mysql -m0755 /var/log/nexus-percona /var/run/nexus-percona
> ```
> **EXPECTED:** PXC installs.
> **VERIFY:** `mysqld --version` → `8.0.x-...Percona XtraDB Cluster`;
> `ls /usr/lib/galera4/libgalera_smm.so` exists.

> **Step 5.1.2 — Install the `nexus-percona` units + open the Galera ports**
> **WHERE:** each PXC node, root shell.
> **WHY:** two units — `nexus-percona.service` (normal start, reads the canonical
> `wsrep_cluster_address`) + `nexus-percona-bootstrap.service` (adds
> `--wsrep-new-cluster` for the one bootstrap node). nftables trusts the VMnet10
> backplane (Galera `4444/4567/4568`) + opens `3306` on VMnet11.
> **WHAT:**
> ```bash
> cat > /etc/systemd/system/nexus-percona.service <<'EOF'
> [Unit]
> Description=Nexus Percona XtraDB Cluster node
> After=network-online.target
> Wants=network-online.target
> [Service]
> User=mysql
> Group=mysql
> RuntimeDirectory=nexus-percona
> RuntimeDirectoryMode=0755
> ExecStart=/usr/sbin/mysqld --defaults-file=/etc/nexus-percona/my.cnf
> Restart=on-failure
> LimitNOFILE=1048576
> [Install]
> WantedBy=multi-user.target
> EOF
> # bootstrap unit = same but with --wsrep-new-cluster
> sed 's/Nexus Percona XtraDB Cluster node/Nexus PXC BOOTSTRAP node/; s#--defaults-file=/etc/nexus-percona/my.cnf#--defaults-file=/etc/nexus-percona/my.cnf --wsrep-new-cluster#' \
>   /etc/systemd/system/nexus-percona.service > /etc/systemd/system/nexus-percona-bootstrap.service
> systemctl daemon-reload
> # nftables: add to chain input before counter drop:
> #   iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 3306 accept comment "MySQL"
> #   iifname "nic1" ip saddr 192.168.10.0/24 accept comment "trusted cluster backplane (VMnet10: Galera 4444/4567/4568)"
> nft -c -f /etc/nftables.conf && systemctl reload nftables
> ```
> **EXPECTED:** both units installed; nftables valid.
> **VERIFY:** `systemctl cat nexus-percona-bootstrap.service | grep wsrep-new-cluster`.

### 5.2 — PKI cert + cluster secrets

> **Step 5.2.1 — Create the `percona-server` PKI role + seed the 4 KV secrets**
> **WHERE:** `vault-1`, root shell.
> **WHY:** the cert role (server+client EKU, all 5 hostnames) + the four shared
> secrets. The `cluster-password` doubles as the VRRP AH auth (first 8 chars).
> **WHAT:**
> ```bash
> vault write pki_int/roles/percona-server \
>   allowed_domains='nexus.lab,percona.nexus.lab,pxc-node-1,pxc-node-2,pxc-node-3,proxysql-1,proxysql-2,localhost' \
>   allow_subdomains=true allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
>
> vault kv put nexus/oltp/percona/cluster-password       password="$(openssl rand -hex 16)"
> vault kv put nexus/oltp/percona/monitor-password       password="$(openssl rand -hex 16)"
> vault kv put nexus/oltp/percona/smoke-rw-password       password="$(openssl rand -base64 24 | tr -dc 'a-zA-Z0-9' | head -c 24)"
> vault kv put nexus/oltp/percona/proxysql-admin-password password="$(openssl rand -base64 24 | tr -dc 'a-zA-Z0-9' | head -c 24)"
> ```
> **EXPECTED:** role + 4 secrets written.
> **VERIFY:** `vault kv list nexus/oltp/percona` lists the 4 paths.

> **Step 5.2.2 — Issue + place each node's cert; place the secret files**
> **WHERE:** issue on `vault-1`; place on each node (all 5).
> **WHY:** PXC takes **3 separate** PEM files (`ssl-cert` leaf, `ssl-key` PKCS#8,
> `ssl-ca` chain). ProxySQL reuses the same set. Place `cluster-password` (PXC +
> ProxySQL keepalived) + the others where each role needs them.
> **WHAT (per node — issue on vault-1, substitute host/IPs):**
> ```bash
> ISSUED=$(vault write -format=json pki_int/issue/percona-server \
>   common_name="<host>.percona.nexus.lab" alt_names="<host>,<host>.nexus.lab,localhost" \
>   ip_sans="<vmnet11>,<vmnet10>,127.0.0.1" ttl=2160h)
> echo "$ISSUED" | jq -r '.data.certificate' > /tmp/server-cert.pem
> echo "$ISSUED" | jq -r '.data.private_key' | openssl pkcs8 -topk8 -nocrypt > /tmp/server-key.pem
> { echo "$ISSUED" | jq -r '.data.issuing_ca'; cat ~/.nexus/secrets/vault-root-ca.pem; } > /tmp/ca.pem
> # scp the 3 to <host>:/etc/nexus-percona/tls/ (root:mysql; cert/ca 0644, key 0640)
> # PXC nodes also need: /etc/nexus-percona/cluster-password (0400 root:mysql)
> ```
> **EXPECTED:** 3 cert files per node + the secret files where needed.
> **VERIFY:** `sudo openssl x509 -in /etc/nexus-percona/tls/server-cert.pem -noout -subject`
> → `CN=<host>.percona.nexus.lab`.

### 5.3 — Render PXC config (my.cnf + wsrep.cnf, all 3)

> **Step 5.3.1 — Write `my.cnf` + `wsrep.cnf` on each PXC node**
> **WHERE:** each PXC node, root shell.
> **WHY:** Percona's two-file split — `my.cnf` (MySQL/InnoDB/TLS, identical except
> `server_id`) `!include`s `wsrep.cnf` (Galera, per-node identity). **Two
> trailing-newline traps:** (a) `my.cnf` must end with a newline or MySQL's
> `!include` parser eats the last char (`wsrep.cnf` → `wsrep.cn`); (b) `wsrep.cnf`
> must end with a blank line so the bootstrap's `!include sst-auth.cnf` append
> lands on its own line.
> **WHAT (substitute `<server_id>`, `<host>`, `<vmnet10>`):**
> ```bash
> cat > /etc/nexus-percona/my.cnf <<'EOF'
> [mysqld]
> server_id                = <server_id>
> bind-address             = 0.0.0.0
> port                     = 3306
> datadir                  = /var/lib/nexus-percona
> socket                   = /var/run/nexus-percona/mysqld.sock
> pid-file                 = /var/run/nexus-percona/mysqld.pid
> log_error                = /var/log/nexus-percona/mysqld.log
> ssl-cert                 = /etc/nexus-percona/tls/server-cert.pem
> ssl-key                  = /etc/nexus-percona/tls/server-key.pem
> ssl-ca                   = /etc/nexus-percona/tls/ca.pem
> require_secure_transport = ON
> tls_version              = TLSv1.2,TLSv1.3
> default_storage_engine   = InnoDB
> innodb_buffer_pool_size  = 1G
> innodb_log_file_size     = 256M
> innodb_flush_log_at_trx_commit = 2
> innodb_flush_method      = O_DIRECT
> max_connections          = 200
> binlog_format            = ROW
> innodb_autoinc_lock_mode = 2
> pxc_strict_mode          = ENFORCING
> [client]
> socket                   = /var/run/nexus-percona/mysqld.sock
> ssl-ca                   = /etc/nexus-percona/tls/ca.pem
> !include /etc/nexus-percona/wsrep.cnf
> EOF
>
> cat > /etc/nexus-percona/wsrep.cnf <<'EOF'
> [mysqld]
> wsrep_provider          = /usr/lib/galera4/libgalera_smm.so
> wsrep_cluster_name      = nexus-pxc
> wsrep_node_name         = <host>
> wsrep_node_address      = <vmnet10>
> wsrep_cluster_address   = gcomm://192.168.10.51,192.168.10.52,192.168.10.53
> wsrep_sst_method        = xtrabackup-v2
> wsrep_provider_options  = "gcache.size=512M; gcache.recover=yes"
> pxc-encrypt-cluster-traffic = ON
>
> EOF
> chown root:mysql /etc/nexus-percona/*.cnf ; chmod 644 /etc/nexus-percona/*.cnf
> ```
> **EXPECTED:** both files written (note `my.cnf` ends with a newline after the
> `!include`; `wsrep.cnf` ends with a blank line).
> **VERIFY:** `tail -c1 /etc/nexus-percona/my.cnf | xxd` shows `0a` (LF);
> `tail -c2 /etc/nexus-percona/wsrep.cnf | xxd` ends `0a0a` (blank line).
> **Do not start the service yet** — §5.4 owns the cluster-aware start.

### 5.4 — Bootstrap the Galera cluster (the exit gate)

> **Step 5.4.1 — Bootstrap `pxc-node-1` + create the cluster users**
> **WHERE:** `pxc-node-1` (`.51`), root shell.
> **WHY:** exactly one node bootstraps the cluster. Start the **bootstrap** unit
> (`--wsrep-new-cluster`), set the root password, create the three users, write
> `sst-auth.cnf`. **Only ever bootstrap when no cluster exists** (else split-brain).
> **WHAT:**
> ```bash
> CLUSTER_PWD=$(sudo cat /etc/nexus-percona/cluster-password)
> MON_PWD=$(vault kv get -field=password nexus/oltp/percona/monitor-password)   # or place a file
> SMOKE_PWD=$(vault kv get -field=password nexus/oltp/percona/smoke-rw-password)
>
> systemctl start nexus-percona-bootstrap.service
> # wait for mysqld ready:
> until mysql -S /var/run/nexus-percona/mysqld.sock -e 'SELECT 1' >/dev/null 2>&1; do sleep 2; done
>
> mysql -S /var/run/nexus-percona/mysqld.sock <<SQL
> ALTER USER 'root'@'localhost' IDENTIFIED BY '${CLUSTER_PWD}';
> CREATE USER IF NOT EXISTS 'wsrep_sst'@'%'    IDENTIFIED BY '${CLUSTER_PWD}';
> GRANT RELOAD, LOCK TABLES, PROCESS, REPLICATION CLIENT ON *.* TO 'wsrep_sst'@'%';
> CREATE USER IF NOT EXISTS 'clustercheck'@'%' IDENTIFIED BY '${MON_PWD}';
> GRANT USAGE, PROCESS ON *.* TO 'clustercheck'@'%';
> CREATE DATABASE IF NOT EXISTS nexus_smoke;
> CREATE USER IF NOT EXISTS 'smoke-rw'@'%'     IDENTIFIED BY '${SMOKE_PWD}';
> GRANT ALL ON nexus_smoke.* TO 'smoke-rw'@'%';
> FLUSH PRIVILEGES;
> SQL
>
> # SST auth (separate 0640 file, included by wsrep.cnf):
> printf '[mysqld]\nwsrep_sst_auth = wsrep_sst:%s\n' "$CLUSTER_PWD" > /etc/nexus-percona/sst-auth.cnf
> chown root:mysql /etc/nexus-percona/sst-auth.cnf ; chmod 640 /etc/nexus-percona/sst-auth.cnf
> echo '!include /etc/nexus-percona/sst-auth.cnf' >> /etc/nexus-percona/wsrep.cnf
> ```
> **EXPECTED:** mysqld up; users created; `sst-auth.cnf` written + included.
> **VERIFY:** `mysql -S /var/run/nexus-percona/mysqld.sock -e "SHOW STATUS LIKE 'wsrep_cluster_size'"`
> → `1`.

> **Step 5.4.2 — Restart `pxc-node-1` normally, then join `pxc-node-2`, then `pxc-node-3`**
> **WHERE:** pxc-node-1, then -2, then -3 — sequential.
> **WHY:** flip node-1 from the bootstrap unit to the normal unit (now it reads
> the canonical 3-member `wsrep_cluster_address`), then bring up the joiners
> **one at a time** so each SSTs cleanly from the live donor.
> **WHAT:**
> ```bash
> # on pxc-node-1: switch bootstrap -> normal
> systemctl stop nexus-percona-bootstrap.service
> systemctl start nexus-percona.service
> # wait for Synced:
> until mysql -S /var/run/nexus-percona/mysqld.sock -e "SHOW STATUS LIKE 'wsrep_local_state_comment'" 2>/dev/null | grep -q Synced; do sleep 2; done
>
> # on pxc-node-2 (then -3): copy sst-auth.cnf + add the !include, then start
> #   scp pxc-node-1:/etc/nexus-percona/sst-auth.cnf -> this node (0640 root:mysql)
> #   echo '!include /etc/nexus-percona/sst-auth.cnf' >> /etc/nexus-percona/wsrep.cnf
> systemctl start nexus-percona.service     # joins via SST from node-1
> ```
> Wait for each joiner to reach `Synced` and the cluster size to climb (2, then 3)
> before starting the next.
> **EXPECTED:** node-2 SSTs + syncs, then node-3.
> **VERIFY (on any node):** `SHOW STATUS LIKE 'wsrep_cluster_size'` → `3`;
> `wsrep_local_state_comment` → `Synced`; `wsrep_cluster_status` → `Primary`.

> **Step 5.4.3 — Write/read round-trip across the cluster (exit gate)**
> **WHERE:** pxc-node-1 (write), pxc-node-2/3 (read).
> **WHY:** prove synchronous Galera replication + mTLS + `smoke-rw` auth — a write
> on one node is instantly readable on the others.
> **WHAT:**
> ```bash
> SMOKE_PWD=$(vault kv get -field=password nexus/oltp/percona/smoke-rw-password)
> TOKEN="smoke-$(date +%s)"
> # write on node-1 (mTLS, smoke-rw):
> mysql -h 127.0.0.1 -P 3306 --ssl-ca=/etc/nexus-percona/tls/ca.pem -u smoke-rw -p"$SMOKE_PWD" nexus_smoke \
>   -e "CREATE TABLE IF NOT EXISTS t(id INT PRIMARY KEY, v VARCHAR(64)); REPLACE INTO t VALUES (1,'$TOKEN');"
> # read on node-2 + node-3:
> for ip in 192.168.70.52 192.168.70.53; do
>   mysql -h $ip -P 3306 --ssl-ca=/etc/nexus-percona/tls/ca.pem -u smoke-rw -p"$SMOKE_PWD" nexus_smoke -BNe "SELECT v FROM t WHERE id=1"
> done
> ```
> **EXPECTED:** both reads return `smoke-<ts>` — the write replicated synchronously.
> **VERIFY:** both `SELECT`s print the token (PXC exit gate met).

### 5.5 — ProxySQL nodes: install + config (both)

> **Step 5.5.1 — Install ProxySQL + render `/etc/proxysql.cnf` + start**
> **WHERE:** `proxysql-1/2` (`.54/.55`), root shell.
> **WHY:** ProxySQL groups the 3 PXC backends via `mysql_galera_hostgroups`
> (writer hg 10 / backup-writer 20 / reader 30 / offline 40, `max_writers=1`,
> `writer_is_also_reader=0`), monitors them as `clustercheck`, and serves clients
> on `6033`. The `.cnf` is the **bootstrap** config (ProxySQL then keeps state in
> its SQLite DB).
> **WHAT:**
> ```bash
> apt-get install -y proxysql mysql-client
> ADMIN_PWD=$(vault kv get -field=password nexus/oltp/percona/proxysql-admin-password)
> MON_PWD=$(vault kv get -field=password nexus/oltp/percona/monitor-password)
> SMOKE_PWD=$(vault kv get -field=password nexus/oltp/percona/smoke-rw-password)
>
> cat > /etc/proxysql.cnf <<EOF
> datadir="/var/lib/proxysql"
> admin_variables= { admin_credentials="admin:${ADMIN_PWD}"; mysql_ifaces="0.0.0.0:6032" }
> mysql_variables=
> {
>   threads=4
>   interfaces="0.0.0.0:6033"
>   monitor_username="clustercheck"
>   monitor_password="${MON_PWD}"
>   monitor_galera_healthcheck_interval=1000
>   have_ssl=true
>   ssl_p2s_ca="/etc/nexus-percona/tls/ca.pem"
>   ssl_p2s_cert="/etc/nexus-percona/tls/server-cert.pem"
>   ssl_p2s_key="/etc/nexus-percona/tls/server-key.pem"
> }
> mysql_servers=
> (
>   { address="192.168.70.51", port=3306, hostgroup=10, use_ssl=1 },
>   { address="192.168.70.52", port=3306, hostgroup=10, use_ssl=1 },
>   { address="192.168.70.53", port=3306, hostgroup=10, use_ssl=1 }
> )
> mysql_galera_hostgroups=
> (
>   { writer_hostgroup=10, backup_writer_hostgroup=20, reader_hostgroup=30, offline_hostgroup=40,
>     active=1, max_writers=1, writer_is_also_reader=0, max_transactions_behind=100 }
> )
> mysql_users=
> (
>   { username="smoke-rw", password="${SMOKE_PWD}", default_hostgroup=10, active=1 }
> )
> EOF
> chown root:proxysql /etc/proxysql.cnf ; chmod 640 /etc/proxysql.cnf
> # nftables: open 6032 + 6033 on VMnet11
> systemctl enable --now proxysql
> ```
> **EXPECTED:** ProxySQL starts, probes the backends.
> **VERIFY (admin :6032):**
> ```bash
> mysql -h 127.0.0.1 -P 6032 -u admin -p"$ADMIN_PWD" -e "SELECT hostname,hostgroup_id,status FROM runtime_mysql_servers"
> ```
> shows the 3 backends shuffled into writer(10)/reader(30); a query via
> `mysql -h 127.0.0.1 -P 6033 -u smoke-rw -p"$SMOKE_PWD" -e 'SELECT @@hostname'` returns a PXC hostname.

### 5.6 — keepalived VRRP VIP (`.50`) on both ProxySQL nodes

> **Step 5.6.1 — Install keepalived + the health check + the VRRP config**
> **WHERE:** `proxysql-1` (MASTER, prio 110) + `proxysql-2` (BACKUP, prio 100).
> **WHY:** float `192.168.70.50` to the healthy higher-priority ProxySQL.
> **Unicast** VRRP (multicast doesn't traverse VMnet11 → split-brain). The
> health script demotes priority (`weight -30`) if ProxySQL stops serving.
> **WHAT (on proxysql-1 — for proxysql-2 swap `state BACKUP`, `priority 100`, and
> the `unicast_src_ip`/`unicast_peer`):**
> ```bash
> apt-get install -y keepalived
> CLUSTER_PWD=$(sudo cat /etc/nexus-percona/cluster-password) ; VRRP_PWD=${CLUSTER_PWD:0:8}
> SMOKE_PWD=$(vault kv get -field=password nexus/oltp/percona/smoke-rw-password)
>
> cat > /etc/keepalived/check_proxysql.sh <<EOF
> #!/bin/bash
> systemctl is-active --quiet proxysql || exit 1
> mysql -h 127.0.0.1 -P 6033 -u smoke-rw -p'${SMOKE_PWD}' -BNe 'SELECT 1' >/dev/null 2>&1 || exit 1
> exit 0
> EOF
> chmod 700 /etc/keepalived/check_proxysql.sh
>
> cat > /etc/keepalived/keepalived.conf <<EOF
> vrrp_script chk_proxysql {
>     script "/etc/keepalived/check_proxysql.sh"
>     interval 2
>     weight -30
>     fall 3
>     rise 2
> }
> vrrp_instance VI_PROXYSQL_NEXUS {
>     state MASTER
>     interface nic0
>     virtual_router_id 51
>     priority 110
>     advert_int 1
>     unicast_src_ip 192.168.70.54
>     unicast_peer { 192.168.70.55 }
>     authentication { auth_type AH  auth_pass ${VRRP_PWD} }
>     virtual_ipaddress { 192.168.70.50/24 dev nic0 }
>     track_script { chk_proxysql }
> }
> EOF
> # nftables: allow VRRP (ip protocol 112) between the 2 ProxySQL nodes on nic0
> systemctl enable --now keepalived
> ```
> **EXPECTED:** keepalived starts; proxysql-1 claims the VIP.
> **VERIFY:** `ip addr show dev nic0` on proxysql-1 shows `192.168.70.50/24`; on
> proxysql-2 it does **not**. From the host:
> `mysql -h 192.168.70.50 -P 6033 -u smoke-rw -p… -BNe 'SELECT 1'` → `1`.

---

## 6. Validation — by-hand acceptance smoke

From the **host** (via `ssh` + `sudo mysql` / direct `mysql -h <VIP>`).

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | Galera cluster size | `ssh …@51 "sudo mysql -S … -e \"SHOW STATUS LIKE 'wsrep_cluster_size'\""` | `3` |
| 2 | All Synced + Primary | `wsrep_local_state_comment` + `wsrep_cluster_status` on all 3 | `Synced` / `Primary` |
| 3 | Synchronous replication | write node-1, read node-2 + node-3 (§5.4.3) | token on all 3 |
| 4 | mTLS enforced | `mysql -h .51 -P 3306` without `--ssl-ca` | rejected (require_secure_transport) |
| 5 | ProxySQL backends healthy | `runtime_mysql_servers` on each ProxySQL | 3 backends in writer(10)/reader(30) |
| 6 | Query via ProxySQL | `mysql -h .54 -P 6033 -u smoke-rw … 'SELECT @@hostname'` | a PXC hostname |
| 7 | VIP held by MASTER | `ip addr show nic0` on proxysql-1 / proxysql-2 | `.50` on -1 only |
| 8 | **Query via the VIP** | host: `mysql -h 192.168.70.50 -P 6033 -u smoke-rw … 'SELECT 1'` | `1` |
| 9 | **VIP failover** | `ssh …@54 'sudo systemctl stop proxysql'`; re-check #7 + #8 | VIP moves to proxysql-2; #8 still works (then restart) |
| 10 | **Galera node failover** | stop `nexus-percona` on the writer; ProxySQL promotes a new writer | #6/#8 still serve writes (then restart) |

**1–8 green ⇒ Guide 09 satisfied.** 9 + 10 are the two HA proofs (ProxySQL VIP
failover + Galera writer failover).

---

## 7. Teardown / reset

```bash
for ip in 54 55; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now keepalived proxysql'; done
for ip in 53 52 51; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-percona nexus-percona-bootstrap'; done
# then vmrun stop + deleteVM each of the 5 (Guide 00 §7).
```

> **Galera cold-start order matters.** The node with the most advanced state must
> bootstrap first. After a *clean* full shutdown, `/var/lib/nexus-percona/grastate.dat`
> on the last-stopped node has `safe_to_bootstrap: 1` — bootstrap **that** node
> (set it manually if needed), then join the others. Bootstrapping the wrong node
> loses writes. The cluster secrets stay in Vault KV (reused on rebuild).

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Both ProxySQL nodes hold the VIP (split-brain) | multicast VRRP doesn't traverse VMware VMnet11 | Use **unicast** VRRP (`unicast_src_ip` + `unicast_peer`) — §5.6.1. |
| mysqld won't start: `Can't get stat of '…/wsrep.cn'` | `my.cnf` lacks a trailing newline; `!include` parser ate the last char | end `my.cnf` with a newline (§5.3.1). |
| `wsrep_sst_auth` missing / SST fails on joiners | the `!include sst-auth.cnf` concatenated onto the last `wsrep.cnf` line | `wsrep.cnf` must end with a **blank line** so the append lands on its own line (§5.3.1). |
| mysqld starts standalone (no Galera); `wsrep_cluster_size` undefined | `my.cnf` didn't `!include wsrep.cnf` (wsrep_provider never loaded) | add the `!include /etc/nexus-percona/wsrep.cnf` line (§5.3.1). |
| Cluster split-brain / data divergence | `galera_new_cluster` run while a cluster already existed | **Probe `wsrep_cluster_size` before bootstrapping**; only bootstrap one node, only when none is up. |
| `mysqld --validate-config` hangs on a PXC node | on PXC, validate-config tries to *activate* wsrep + connect to gcomm peers | don't run it; the `SELECT 1` probe + cluster formation are the real checks. |
| Joiner stuck in `Donor/Desync` or won't SST | `wsrep_sst_auth` wrong, or backplane `4444/4567/4568` blocked | confirm `sst-auth.cnf` matches the `wsrep_sst` user password; confirm nftables trusts VMnet10. |
| ProxySQL shows backends `OFFLINE_HARD` | `clustercheck` user/password wrong, or `monitor_*` misconfigured | verify the `clustercheck` grant + `monitor_password` matches KV. |
| Cold-start lost writes after full shutdown | bootstrapped a node without `safe_to_bootstrap: 1` | bootstrap the node whose `grastate.dat` has `safe_to_bootstrap: 1` (§7). |

---

## 9. Production tuning — Percona XtraDB Cluster / Galera

> **Everything below is *beyond the lab replica*.** The §5 `my.cnf`/`wsrep.cnf` are
> what the lab actually ships — 8 GB PXC VMs sized to *form the cluster and prove
> synchronous Galera replication + mTLS*, not to carry production write load. This
> section is what you would change on a **production** PXC deployment and why. It does
> **not** alter the §5 renders. **Do not paste these onto the lab VMs blindly** — the
> buffer-pool and gcache sizes assume production-sized RAM.
>
> **OS layer first.** A production PXC host also needs the kernel / ulimit / THP tuning in
> [Guide 00 §9](./00-lab-host-and-base-vm.md#9-production-tuning--the-os-layer-feeds-every-linux-tier)
> — set that once per host, then add the engine overrides below. Two of its knobs are
> re-flagged in §9.3 because PXC is one of the engines that hard-needs them (**THP off**,
> **`vm.swappiness=1`**).

### 9.1 InnoDB — memory, redo log & I/O (`[mysqld]` in `/etc/nexus-percona/my.cnf`)

```ini
# PRODUCTION — not applied in the lab. Sized here for an 8-vCPU / 64 GB dedicated PXC node.
[mysqld]
innodb_buffer_pool_size        = 46G          # ~70-75% of RAM on a dedicated node
innodb_buffer_pool_instances   = 8            # 1 instance per ~1-2 GB of pool, cap ~8
innodb_flush_log_at_trx_commit = 1            # full durability (see the tradeoff below)
innodb_flush_method            = O_DIRECT     # already set in §5.3.1 — keep it
innodb_redo_log_capacity       = 4G           # 8.0.30+ single knob (replaces innodb_log_file_size)
innodb_io_capacity             = 2000         # SSD baseline background flush rate
innodb_io_capacity_max         = 4000         # SSD burst ceiling
```

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `innodb_buffer_pool_size` | **≈ 70-75 % of RAM** on a dedicated node (e.g. `46G` on 64 GB) | `1G` | The hot working set (data + index pages) lives here; too small ⇒ every read/write hits disk. The single most important MySQL knob. Leave ~25 % for the OS page cache, per-connection buffers, and Galera's gcache. |
| `innodb_buffer_pool_instances` | `8` (once the pool is >1 GB) | unset (`1`) | Splits the pool into N independent mutex-guarded arenas so concurrent threads don't serialise on one buffer-pool mutex — real throughput on many-core boxes. Size so each instance is ≥1 GB. |
| `innodb_flush_log_at_trx_commit` | **`1`** (flush + fsync the redo log at every commit) | `2` | ⚠️ **Durability tradeoff — read this.** `2` (lab) writes the redo log to the OS cache at commit but only fsyncs once/second, so a **host crash / power loss can lose ~1 s of committed transactions on that node**. The lab accepts `2` *only* because Galera synchronously replicates every write-set to all 3 nodes before commit, so a single node's lost second is normally recoverable from a peer — it is a lab-scale write-throughput shortcut, **not** a durability guarantee. Production that must survive a simultaneous multi-node power event (or that runs a smaller cluster) sets **`1`**: D-safe, at the cost of one fsync per commit (mitigated by a battery-backed / NVMe write cache and group commit). |
| `innodb_redo_log_capacity` | **≥ 1-2 GB** (e.g. `4G` write-heavy) | `innodb_log_file_size = 256M` | The redo log absorbs write bursts between checkpoints; too small ⇒ constant "furious flushing" checkpoint stalls under load. On Percona/MySQL **8.0.30+** set the single `innodb_redo_log_capacity` (it supersedes `innodb_log_file_size`/`innodb_log_files_in_group`); on older 8.0 use `innodb_log_file_size = 1G` (×2 files). |
| `innodb_flush_method` | `O_DIRECT` | **PRESENT** (`O_DIRECT`, §5.3.1) | Bypasses the OS page cache for data/redo files so pages aren't double-buffered (once in InnoDB's pool, once in the kernel). Already correct in the lab config — no change. |
| `innodb_io_capacity` / `innodb_io_capacity_max` | `2000` / `4000` (SATA/NVMe SSD) | unset (`200`/`2000`) | Tells InnoDB how many IOPS the storage can sustain so background flushing + purge keep up with the redo log; the default `200` assumes a single spinning disk and throttles a fast SSD into checkpoint stalls. |

### 9.2 Galera / wsrep — parallel apply & flow control (`/etc/nexus-percona/wsrep.cnf`)

```ini
# PRODUCTION — not applied in the lab. Tune to the node's core count + write working-set.
[mysqld]
wsrep_slave_threads     = 8      # = number of CPU cores (parallel write-set apply)
wsrep_provider_options  = "gcache.size=8G; gcache.recover=yes; gcs.fc_limit=512; gcs.fc_factor=0.8"
```

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `wsrep_slave_threads` | **= CPU core count** (e.g. `8`) | unset (`1`) | Applier threads that replay incoming write-sets in parallel. With `1`, a busy writer node out-paces the appliers on the others → replication lag → flow-control pauses that throttle the *whole* cluster to the slowest node. Match it to cores. |
| `gcache.size` | **sized to the write working-set** (e.g. `8G` — enough to hold the write-sets produced during the longest expected node outage) | `512M` | The on-disk ring buffer of recent write-sets. If a node is away longer than gcache can cover, its rejoin needs a **full SST** (blocking `xtrabackup` clone of the whole dataset) instead of a cheap **IST** (just the missing write-sets). Size it to `write-rate × max-node-downtime`. |
| `gcs.fc_limit` / `gcs.fc_factor` | `512` / `0.8` (raise from defaults `16` / `0.5` for write-heavy clusters) | unset (defaults) | Galera **flow control**: when a node's receive queue exceeds `fc_limit` write-sets it pauses the cluster until it drains to `fc_limit × fc_factor`. The default `16` is tiny — a momentarily-slow node stalls every writer. Raising the limit lets a node fall further behind before it halts the cluster, trading a little more failover-window lag for far fewer whole-cluster write pauses. Pair with more `wsrep_slave_threads` so nodes drain faster. |
| `pxc_strict_mode` | `ENFORCING` | **PRESENT** (`ENFORCING`, §5.3.1) | Rejects Galera anti-patterns (MyISAM writes, PK-less tables, explicit `LOCK TABLES`) at write time rather than letting them silently diverge the cluster. Already correct — keep it in production. |

### 9.3 OS-layer requirements for PXC (set per [Guide 00 §9](./00-lab-host-and-base-vm.md#9-production-tuning--the-os-layer-feeds-every-linux-tier))

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| Transparent Huge Pages | **`never`** | unset | ⚠️ **required** — THP's `khugepaged` compaction causes multi-ms allocation stalls against InnoDB's buffer-pool access pattern. Disable via the `disable-thp` unit in Guide 00 §9.2. |
| `vm.swappiness` | `1` | unset (`60`) | Keeps the InnoDB buffer pool + gcache resident; swapping them out under cache pressure turns every "cache hit" into a disk read. Guide 00 §9.1. |
| `nofile` (open files) | soft `65536` / hard `1048576` | **PRESENT** (`LimitNOFILE=1048576` on `nexus-percona.service`, §5.1.2) | A PXC node with hundreds of connections + data files + SST sockets exhausts the default `1024` soft limit → `Too many open files`. The lab already grants this in the systemd unit — no change needed. |

---

### Cross-references

- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (Percona `.50`–`.55`, VIP `.50`); ADR-0025 (LB-tier HA)
- **Automated equivalents:** `nexus-infra-oltp/packer/{oltp-pxc-node,oltp-proxysql-node}/` + `terraform/envs/oltp-percona/role-overlay-*.tf`
- **Scaffolding pattern reused:** [`04-foundation-vault-pki-ldap.md`](./04-foundation-vault-pki-ldap.md) Part D
- **Previous guide:** [`08-oltp-mongodb-replica-set.md`](./08-oltp-mongodb-replica-set.md)
- **Next guide:** Guide 10 — OLTP · Patroni PostgreSQL HA (3 Patroni + 3 etcd + 2 HAProxy + VRRP VIP). See [`INDEX.md`](../INDEX.md).
