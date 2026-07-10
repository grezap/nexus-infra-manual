# Guide 10 — OLTP · Patroni PostgreSQL HA (3 Patroni + 3 etcd + 2 HAProxy + VRRP VIP)

> **Mirrors:** `nexus-infra-oltp` — the `oltp-{patroni,etcd,haproxy}-node` Packer
> templates + the `oltp-patroni` env overlays (`patroni-tls`, `etcd-bootstrap`,
> `patroni-bootstrap`, `haproxy-config`, `haproxy-keepalived`,
> `patroni-nftables-backplane`). Where the automated lab renders configs via Vault
> Agents and drives the bootstrap over SSH, this guide installs by hand and
> **issues the cert + seeds the secrets directly**.

---

## 1. Overview & purpose

A highly-available PostgreSQL service — the **asynchronous** counterpart to
Guide 09's synchronous Galera. **8 nodes:**

- **Patroni + PostgreSQL 17 (`pg-primary`, `pg-replica-1/2`, `.61/.62/.63`)** —
  one PRIMARY takes writes and **streams** WAL to the two replicas; **Patroni**
  is the HA controller that holds the leader lease, runs `initdb`/`pg_basebackup`,
  and performs **automatic failover** (promotes a replica if the primary dies).
- **etcd (`etcd-1/2/3`, `.64/.65/.66`)** — the **DCS** (distributed configuration
  store): a 3-node Raft cluster holding the leader lease + cluster state that
  Patroni reads/writes to coordinate.
- **HAProxy (`haproxy-pg-1/2`, `.67/.68`)** — routes `:5432` to **whichever node
  is currently the Patroni leader**, by health-probing each node's Patroni REST
  `/leader` endpoint (200 = leader, 503 = replica).
- **VRRP VIP (`.60`)** — keepalived floats one IP between the 2 HAProxy nodes;
  clients connect to `postgresql://…@192.168.70.60:5432`.

**Why Patroni (vs. Percona's Galera):** Postgres replication is **async streaming**
(a primary + read replicas), and Patroni adds the missing automatic-failover
brain on top via a consensus store. Same HA promise, completely different
mechanism — this tier exercises the etcd-DCS + leader-lease model.

**Dependency:**
- **Guides 00–04** — foundation alive; Guide 04 PKI + Part-D scaffolding (this
  guide creates the `patroni-server` PKI role + seeds the cluster secrets).
- 8 `deb13` nodes baselined per Guide 00 §5.B, dual-NIC static.

> **By-hand divergence:** issue the cert with the `vault` CLI, seed the 5 secrets
> in Vault KV + place them on the nodes — no per-node Vault Agent.

---

## 2. Component primer

- **PostgreSQL streaming replication.** The PRIMARY ships its WAL (write-ahead
  log) to replicas, which replay it — async by default (a tiny lag window). Read
  replicas serve `hot_standby` reads. *Why (vs. Galera's sync):* the standard
  Postgres HA model; lower write latency, at the cost of a small failover-time
  data-loss window. *Otherwise:* logical replication (row-level; heavier) or sync
  replication (higher latency).
- **Patroni.** A Python HA agent that wraps each `postgres`. It elects a leader
  via a **lease** in the DCS, `initdb`'s the first node, `pg_basebackup`'s the
  replicas, and on primary failure **promotes** the most-caught-up replica +
  reconfigures the others to follow it. Exposes a REST API on `:8008`
  (`/leader` returns 200 only on the current leader — HAProxy's routing signal).
  *Otherwise:* repmgr, pg_auto_failover (Patroni is the lab standard + DCS-based).
- **etcd as the DCS.** A 3-node Raft key/value store holding the leader lease +
  cluster config. Patroni nodes race to claim `/service/<scope>/leader`; the
  winner is primary. *Why etcd (vs. Consul/ZooKeeper):* Patroni's first-class
  backend; clean Raft semantics. Here etcd uses **TLS for the wire + HTTP
  basic-auth RBAC** (Patroni sends `username=root` + password) rather than
  client-cert-CN auth — simpler, one less rotation surface.
- **`pg_rewind` + replication slots.** `use_pg_rewind` lets a demoted old-primary
  rejoin without a full re-clone; `use_slots` gives each replica a replication
  slot so the primary retains WAL until the replica has it. *Otherwise:* full
  `pg_basebackup` on every rejoin (slow).
- **HAProxy `/leader` health-check routing.** All 3 Postgres nodes are in one
  backend; `option httpchk GET /leader` against Patroni's REST `:8008` marks
  **only the leader** UP (200), the replicas DOWN (503). So `:5432` always lands
  on the writer, and failover is automatic when the leader role moves. **`check`
  must be on `default-server`** or the probe never runs (a real transient).
- **keepalived VRRP VIP.** Same as Guide 09: a floating `.60` between the 2
  HAProxy nodes, **unicast** VRRP (multicast doesn't traverse VMnet11), priority
  110/100, health-script demotion. The front door's own HA (ADR-0025).

---

## 3. Prerequisites

| # | Requirement | One-command verify |
|---|---|---|
| 1 | Foundation alive (Guides 00–04); Vault PKI usable | `vault read pki_int/cert/ca` on vault-1 returns the intermediate |
| 2 | 8 `deb13` nodes baselined, dual-NIC static `.61–.68` | those 8 answer `:22` |
| 3 | Vault root token on build host | `Test-Path ~/.nexus/secrets/vault-cluster-init.json` |
| 4 | Gateway resolves the bare node hostnames | `ssh …@61 'getent hosts etcd-1 etcd-2 etcd-3'` resolves |
| 5 | Internet egress on the nodes | `ssh …@61 'curl -sI https://apt.postgresql.org \| head -1'` → `200` |

> Versions: **PostgreSQL 17** + **Patroni**, **etcd 3.x**, **HAProxy 2.x**.
> Patroni scope `nexus-pg`. etcd peers on the **VMnet10 backplane** (`:2380`),
> clients on VMnet11 (`:2379`); Postgres `:5432`, Patroni REST `:8008`, HAProxy
> stats `:8404`.
>
> **etcd uses BARE hostnames** (`etcd-1`, not `etcd-1.nexus.lab`) — dnsmasq serves
> `nexus.local`, not `nexus.lab`, but the bare names resolve and the cert SANs
> include the bare form (a real 0.G.4 transient).

---

## 4. Target topology

| Node | Role | VMnet11 | VMnet10 | prio | vCPU/RAM/disk |
|---|---|---|---|:--:|---|
| `pg-primary` | Patroni + PG 17 (initial leader) | `.61` | `.10.61` | — | 2 / 4 GB / 80 GB |
| `pg-replica-1` | Patroni + PG 17 | `.62` | `.10.62` | — | 2 / 4 GB / 80 GB |
| `pg-replica-2` | Patroni + PG 17 | `.63` | `.10.63` | — | 2 / 4 GB / 80 GB |
| `etcd-1/2/3` | etcd DCS (Raft) | `.64/.65/.66` | `.10.64/.65/.66` | — | 2 / 2 GB / 40 GB |
| `haproxy-pg-1` | HAProxy + keepalived **MASTER** | `.67` | `.10.67` | 110 | 2 / 2 GB / 30 GB |
| `haproxy-pg-2` | HAProxy + keepalived **BACKUP** | `.68` | `.10.68` | 100 | 2 / 2 GB / 30 GB |
| **VIP** | `pg.nexus.lab` client front door | **`.60`** | — | VRRP | — |

> Secrets (5) in Vault KV `nexus/oltp/patroni/`: `etcd-root-password`,
> `patroni-rest-password`, `postgres-superuser-password`,
> `postgres-replication-password`, `haproxy-stats-password`. One PKI role
> **`patroni-server`** covers all 8 nodes. Client:
> `postgresql://nexusops@192.168.70.60:5432/postgres`.

---

## 5. Step-by-step build

> **WHERE:** node steps as `nexusadmin`→`sudo -i`. `vault` on **`vault-1`** (root
> token). etcd + Patroni each start **in parallel** across their members (Raft /
> leader-race need simultaneous startup).

### 5.1 — Base install per role

> **Step 5.1.1 — etcd nodes: install etcd 3.x**
> **WHERE:** `etcd-1/2/3`, root shell.
> **WHAT:**
> ```bash
> apt-get update -qq && apt-get install -y etcd-server etcd-client openssl
> systemctl disable --now etcd 2>/dev/null || true
> install -d -o etcd -g etcd -m0750 /etc/nexus-etcd /etc/nexus-etcd/tls /var/lib/nexus-etcd
> cat > /etc/systemd/system/nexus-etcd.service <<'EOF'
> [Unit]
> Description=Nexus etcd (Patroni DCS)
> After=network-online.target
> Wants=network-online.target
> [Service]
> User=etcd
> Group=etcd
> ExecStart=/usr/bin/etcd --config-file /etc/nexus-etcd/etcd.conf.yml
> Restart=on-failure
> LimitNOFILE=65536
> [Install]
> WantedBy=multi-user.target
> EOF
> systemctl daemon-reload
> ```
> **VERIFY:** `etcd --version` prints a 3.x version.

> **Step 5.1.2 — Patroni nodes: install PostgreSQL 17 + Patroni**
> **WHERE:** `pg-primary`, `pg-replica-1/2`, root shell.
> **WHY:** PG 17 binaries (no auto-started cluster — Patroni owns the data dir) +
> Patroni with the etcd3 driver.
> **WHAT:**
> ```bash
> apt-get update -qq && apt-get install -y curl gnupg openssl python3-pip
> install -d /usr/share/postgresql-common/pgdg
> curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc
> echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt trixie-pgdg main" \
>   > /etc/apt/sources.list.d/pgdg.list
> apt-get update -qq && apt-get install -y postgresql-17 patroni python3-etcd3
> systemctl disable --now postgresql 2>/dev/null || true   # Patroni manages PG, not the distro unit
> install -d -o postgres -g postgres -m0750 /etc/nexus-patroni /etc/nexus-patroni/tls
> install -d -o postgres -g postgres -m0700 /var/lib/nexus-patroni
> install -d -o postgres -g postgres -m0755 /var/run/nexus-patroni
> cat > /etc/systemd/system/nexus-patroni.service <<'EOF'
> [Unit]
> Description=Nexus Patroni (PostgreSQL HA)
> After=network-online.target
> Wants=network-online.target
> [Service]
> User=postgres
> Group=postgres
> ExecStart=/usr/bin/patroni /etc/nexus-patroni/patroni.yml
> Restart=on-failure
> LimitNOFILE=65536
> [Install]
> WantedBy=multi-user.target
> EOF
> systemctl daemon-reload
> ```
> **VERIFY:** `ls /usr/lib/postgresql/17/bin/postgres` exists; `patroni --version`.

> **Step 5.1.3 — HAProxy nodes: install HAProxy + keepalived**
> **WHERE:** `haproxy-pg-1/2`, root shell.
> **WHAT:**
> ```bash
> apt-get update -qq && apt-get install -y haproxy keepalived postgresql-client-17 openssl
> systemctl disable --now haproxy 2>/dev/null || true
> install -d -o haproxy -g haproxy -m0750 /etc/nexus-haproxy /etc/nexus-haproxy/tls
> install -d -m0755 /run/nexus-haproxy
> cat > /etc/systemd/system/nexus-haproxy.service <<'EOF'
> [Unit]
> Description=Nexus HAProxy (Patroni LB)
> After=network-online.target
> Wants=network-online.target
> [Service]
> ExecStart=/usr/sbin/haproxy -Ws -f /etc/nexus-haproxy/haproxy.cfg
> Restart=on-failure
> [Install]
> WantedBy=multi-user.target
> EOF
> systemctl daemon-reload
> ```
> **VERIFY:** `haproxy -v` prints 2.x.

> **Step 5.1.4 — nftables backplane trust (all 8)**
> **WHERE:** each node, root shell.
> **WHY:** trust the VMnet10 backplane (etcd `2380`, Patroni replication) + open
> the per-role service ports on VMnet11.
> **WHAT:** add to `/etc/nftables.conf` `chain input` (before `counter drop`):
> ```
> iifname "nic1" ip saddr 192.168.10.0/24 accept comment "trusted cluster backplane (VMnet10)"
> # PG nodes:    iifname "nic0" ip saddr 192.168.70.0/24 tcp dport { 5432, 8008 } accept
> # etcd nodes:  iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 2379 accept
> # haproxy:     iifname "nic0" ip saddr 192.168.70.0/24 tcp dport { 5432, 8404 } accept
> #              + ip protocol 112 accept   (VRRP, haproxy nodes only)
> ```
> then `nft -c -f /etc/nftables.conf && systemctl reload nftables`.
> **VERIFY:** `nft list chain inet filter input | grep nic1`.

### 5.2 — PKI cert + cluster secrets

> **Step 5.2.1 — Create the `patroni-server` PKI role + seed the 5 KV secrets**
> **WHERE:** `vault-1`, root shell.
> **WHY:** one cert role for all 8 nodes (server+client EKU, bare + FQDN names);
> five shared secrets.
> **WHAT:**
> ```bash
> vault write pki_int/roles/patroni-server \
>   allowed_domains='nexus.lab,patroni.nexus.lab,pg-primary,pg-replica-1,pg-replica-2,etcd-1,etcd-2,etcd-3,haproxy-pg-1,haproxy-pg-2,localhost' \
>   allow_subdomains=true allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
>
> for s in etcd-root-password patroni-rest-password postgres-superuser-password postgres-replication-password haproxy-stats-password; do
>   vault kv put nexus/oltp/patroni/$s password="$(openssl rand -hex 16)"
> done
> ```
> **EXPECTED:** role + 5 secrets written.
> **VERIFY:** `vault kv list nexus/oltp/patroni` lists the 5.

> **Step 5.2.2 — Issue + place each node's cert (3 PEM files)**
> **WHERE:** issue on `vault-1`; place on each node (all 8).
> **WHY:** etcd, Patroni, and HAProxy all want `server-cert.pem` (leaf),
> `server-key.pem` (PKCS#8), `ca.pem` (intermediate + root). **Include the bare
> hostname in the SANs** (etcd dials peers by bare name).
> **WHAT (per node — issue on vault-1, substitute host/IPs + the per-role tls dir):**
> ```bash
> ISSUED=$(vault write -format=json pki_int/issue/patroni-server \
>   common_name="<host>.nexus.lab" alt_names="<host>,localhost" \
>   ip_sans="<vmnet11>,<vmnet10>,127.0.0.1" ttl=2160h)
> echo "$ISSUED" | jq -r '.data.certificate' > /tmp/server-cert.pem
> echo "$ISSUED" | jq -r '.data.private_key' | openssl pkcs8 -topk8 -nocrypt > /tmp/server-key.pem
> { echo "$ISSUED" | jq -r '.data.issuing_ca'; cat ~/.nexus/secrets/vault-root-ca.pem; } > /tmp/ca.pem
> # scp to <host>:/etc/nexus-{etcd|patroni|haproxy}/tls/ per role (owned by the role user, key 0640)
> ```
> **EXPECTED:** 3 files per node.
> **VERIFY:** `sudo openssl x509 -in <tls-dir>/server-cert.pem -noout -ext subjectAltName`
> shows the bare hostname + both IPs.

### 5.3 — Bring up the etcd DCS (parallel start + RBAC)

> **Step 5.3.1 — Render `etcd.conf.yml` on each etcd node, start all 3 in parallel**
> **WHERE:** `etcd-1/2/3`, root shell; start from the build host in parallel.
> **WHY:** Raft's `initial-cluster-state: new` needs all 3 starting together to
> form quorum (sequential isolates the first node). Peers on the VMnet10
> backplane (`:2380`); clients on VMnet11 (`:2379`). Peer TLS uses client-cert
> auth; the *client* listener uses TLS for encryption only (RBAC is basic-auth).
> **WHAT (per node — substitute `<host>`, `<vmnet11>`, `<vmnet10>`):**
> ```bash
> cat > /etc/nexus-etcd/etcd.conf.yml <<'EOF'
> name: <host>
> data-dir: /var/lib/nexus-etcd
> listen-peer-urls: https://<vmnet10>:2380
> listen-client-urls: https://<vmnet11>:2379,https://<vmnet10>:2379,https://127.0.0.1:2379
> initial-advertise-peer-urls: https://<vmnet10>:2380
> advertise-client-urls: https://<vmnet11>:2379
> initial-cluster: etcd-1=https://192.168.10.64:2380,etcd-2=https://192.168.10.65:2380,etcd-3=https://192.168.10.66:2380
> initial-cluster-token: nexus-patroni-etcd
> initial-cluster-state: new
> peer-transport-security:
>   cert-file: /etc/nexus-etcd/tls/server-cert.pem
>   key-file: /etc/nexus-etcd/tls/server-key.pem
>   trusted-ca-file: /etc/nexus-etcd/tls/ca.pem
>   client-cert-auth: true
> client-transport-security:
>   cert-file: /etc/nexus-etcd/tls/server-cert.pem
>   key-file: /etc/nexus-etcd/tls/server-key.pem
>   trusted-ca-file: /etc/nexus-etcd/tls/ca.pem
>   client-cert-auth: false
> EOF
> chown etcd:etcd /etc/nexus-etcd/etcd.conf.yml
> ```
> Then, from the build host, start all 3 together:
> ```powershell
> '64','65','66' | ForEach-Object -Parallel { ssh nexusadmin@192.168.70.$_ 'sudo systemctl enable --now nexus-etcd' }
> ```
> **EXPECTED:** Raft elects a leader within a few seconds.
> **VERIFY:** `sudo etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/nexus-etcd/tls/ca.pem --cert=… --key=… endpoint status -w table`
> shows 3 members, 1 leader.

> **Step 5.3.2 — Enable RBAC (root user + auth)**
> **WHERE:** `etcd-1`, root shell.
> **WHY:** Patroni authenticates to etcd as `root` over HTTP basic-auth. Create
> the root user (password from KV) + enable auth.
> **WHAT:**
> ```bash
> ROOT_PWD=$(vault kv get -field=password nexus/oltp/patroni/etcd-root-password)
> E="sudo etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/nexus-etcd/tls/ca.pem --cert=/etc/nexus-etcd/tls/server-cert.pem --key=/etc/nexus-etcd/tls/server-key.pem"
> $E user add root:"$ROOT_PWD"
> $E auth enable
> ```
> **EXPECTED:** `User root created`; `Authentication Enabled`.
> **VERIFY:** `$E --user root:"$ROOT_PWD" member list` works; without `--user` it's
> denied.

### 5.4 — Bootstrap the Patroni cluster (the exit gate)

> **Step 5.4.1 — Render `patroni.yml` on each PG node**
> **WHERE:** `pg-primary`, `pg-replica-1/2`, root shell.
> **WHY:** the per-node Patroni config — REST API (mTLS), etcd3 connection
> (basic-auth as root), the `bootstrap.dcs` Postgres parameters + `initdb` +
> `pg_hba` + users, and the `postgresql` block (TLS, replication/superuser/rewind
> auth). The `pg_hba` allows replication from **`192.168.0.0/16`** (covers both
> VMnet10 + VMnet11 — replicas pg_basebackup over VMnet11).
> **WHAT (substitute `<host>`/`<vm_ip>` + the 4 passwords from KV):**
> ```bash
> cat > /etc/nexus-patroni/patroni.yml <<'EOF'
> scope: nexus-pg
> namespace: /service/
> name: <host>
>
> restapi:
>   listen: 0.0.0.0:8008
>   connect_address: <vm_ip>:8008
>   certfile: /etc/nexus-patroni/tls/server-cert.pem
>   keyfile: /etc/nexus-patroni/tls/server-key.pem
>   cafile: /etc/nexus-patroni/tls/ca.pem
>   verify_client: optional
>   authentication:
>     username: nexusops
>     password: <PATRONI_REST_PWD>
>
> etcd3:
>   hosts: etcd-1:2379,etcd-2:2379,etcd-3:2379
>   protocol: https
>   cacert: /etc/nexus-patroni/tls/ca.pem
>   cert: /etc/nexus-patroni/tls/server-cert.pem
>   key: /etc/nexus-patroni/tls/server-key.pem
>   username: root
>   password: <ETCD_ROOT_PWD>
>
> bootstrap:
>   dcs:
>     ttl: 30
>     loop_wait: 10
>     retry_timeout: 10
>     maximum_lag_on_failover: 1048576
>     postgresql:
>       use_pg_rewind: true
>       use_slots: true
>       parameters:
>         max_connections: 200
>         shared_buffers: 256MB
>         wal_level: replica
>         hot_standby: "on"
>         wal_log_hints: "on"
>         max_wal_senders: 10
>         max_replication_slots: 10
>         wal_keep_size: 256MB
>         password_encryption: scram-sha-256
>   initdb:
>     - encoding: UTF8
>     - data-checksums
>   pg_hba:
>     - "hostssl replication replicator 192.168.0.0/16 scram-sha-256"
>     - "hostssl all all 192.168.0.0/16 scram-sha-256"
>     - "host all all 127.0.0.1/32 trust"
>     - "host replication replicator 127.0.0.1/32 trust"
>   users:
>     nexusops:
>       password: <POSTGRES_SUPERUSER_PWD>
>       options: [superuser, createrole, createdb]
>
> postgresql:
>   listen: 0.0.0.0:5432
>   connect_address: <vm_ip>:5432
>   data_dir: /var/lib/nexus-patroni/data
>   bin_dir: /usr/lib/postgresql/17/bin
>   pgpass: /var/lib/nexus-patroni/.pgpass_patroni
>   authentication:
>     superuser:    { username: postgres,    password: <POSTGRES_SUPERUSER_PWD> }
>     replication:  { username: replicator,  password: <POSTGRES_REPLICATION_PWD> }
>     rewind:       { username: rewind,      password: <POSTGRES_REPLICATION_PWD> }
>   parameters:
>     unix_socket_directories: /var/run/nexus-patroni
>     ssl: "on"
>     ssl_cert_file: /etc/nexus-patroni/tls/server-cert.pem
>     ssl_key_file: /etc/nexus-patroni/tls/server-key.pem
>     ssl_ca_file: /etc/nexus-patroni/tls/ca.pem
>     password_encryption: scram-sha-256
>
> watchdog:
>   mode: off
>
> tags:
>   nofailover: false
>   noloadbalance: false
>   clonefrom: false
>   nosync: false
> EOF
> chown postgres:postgres /etc/nexus-patroni/patroni.yml ; chmod 640 /etc/nexus-patroni/patroni.yml
>
> # The heredoc above wrote the file with <placeholders> (it used <<'EOF', so they
> # stayed literal). Now fill in THIS node's name/IP + the 4 KV-seeded passwords:
> export VAULT_ADDR=https://192.168.70.121:8200 VAULT_CACERT=$HOME/.nexus/vault-ca-bundle.crt
> HOST=$(hostname) ; VM_IP=$(ip -4 -o addr show nic0 | awk '{print $4}' | cut -d/ -f1)
> REST_PWD=$(vault kv get -field=password nexus/oltp/patroni/patroni-rest-password)
> ETCD_PWD=$(vault kv get -field=password nexus/oltp/patroni/etcd-root-password)
> SUPER_PWD=$(vault kv get -field=password nexus/oltp/patroni/postgres-superuser-password)
> REPL_PWD=$(vault kv get -field=password nexus/oltp/patroni/postgres-replication-password)
> sed -i "s|<host>|$HOST|g; s|<vm_ip>|$VM_IP|g; s|<PATRONI_REST_PWD>|$REST_PWD|g; s|<ETCD_ROOT_PWD>|$ETCD_PWD|g; s|<POSTGRES_SUPERUSER_PWD>|$SUPER_PWD|g; s|<POSTGRES_REPLICATION_PWD>|$REPL_PWD|g" /etc/nexus-patroni/patroni.yml
> ```
> **EXPECTED:** config on each node now has its real name/IP + the secrets filled in
> (no `<...>` placeholders left).
> **VERIFY:** `grep -E '^name:|connect_address' /etc/nexus-patroni/patroni.yml` (real
> values); `grep -c '<' /etc/nexus-patroni/patroni.yml` → `0` (no placeholders remain).

> **Step 5.4.2 — Start Patroni on all 3 in parallel + wait for 1 leader + 2 replicas**
> **WHERE:** the 3 PG nodes (start together), then any one.
> **WHY:** the nodes race for the etcd leader lease; the winner `initdb`'s a fresh
> data dir + becomes leader, the other two see the lease + `pg_basebackup` from
> it. Parallel start is the natural flow.
> **WHAT (from the build host):**
> ```powershell
> '61','62','63' | ForEach-Object -Parallel { ssh nexusadmin@192.168.70.$_ 'sudo systemctl enable --now nexus-patroni' }
> ```
> **EXPECTED:** ~30–60 s later, one Leader + two replicas (`Streaming`).
> **VERIFY:**
> ```bash
> sudo patronictl -c /etc/nexus-patroni/patroni.yml list
> ```
> shows `nexus-pg` with **1 Leader** + **2 Replica** (State `running`, all in sync).

> **Step 5.4.3 — Write/read round-trip (the exit gate)**
> **WHERE:** the leader (write), a replica (read).
> **WHY:** prove streaming replication + mTLS — a write on the leader appears on a
> replica.
> **WHAT:**
> ```bash
> SUPER=$(vault kv get -field=password nexus/oltp/patroni/postgres-superuser-password)
> TOKEN="smoke-$(date +%s)"
> # write to the leader (via the VIP once HAProxy is up, or directly to pg-primary):
> PGPASSWORD="$SUPER" psql "host=192.168.70.61 port=5432 user=nexusops dbname=postgres sslmode=require" \
>   -c "CREATE TABLE IF NOT EXISTS smoke(id int primary key, v text); INSERT INTO smoke VALUES (1,'$TOKEN') ON CONFLICT (id) DO UPDATE SET v=excluded.v;"
> # read from a replica:
> PGPASSWORD="$SUPER" psql "host=192.168.70.62 port=5432 user=nexusops dbname=postgres sslmode=require" \
>   -tAc "SELECT v FROM smoke WHERE id=1;"
> ```
> **EXPECTED:** the replica returns `smoke-<ts>` — streaming replication confirmed.
> **VERIFY:** the read token equals the written token (Patroni cluster exit gate).

### 5.5 — HAProxy (leader-routing) on both nodes

> **Step 5.5.1 — Render `haproxy.cfg` + start (both HAProxy nodes)**
> **WHERE:** `haproxy-pg-1/2` (`.67/.68`), root shell.
> **WHY:** identical config on both. Frontend `:5432` → backend `pg_pool` of the 3
> PG nodes, each probed with `option httpchk GET /leader` against Patroni REST
> `:8008` (over HTTPS) — **only the leader returns 200**, so `:5432` always lands
> on the writer. **`check` must be on `default-server`** or the probe never fires.
> **WHAT:**
> ```bash
> STATS=$(vault kv get -field=password nexus/oltp/patroni/haproxy-stats-password)
> cat > /etc/nexus-haproxy/haproxy.cfg <<EOF
> global
>     log /dev/log local0
>     maxconn 4096
>     stats socket /run/nexus-haproxy/admin.sock mode 660 level admin
> defaults
>     log global
>     mode tcp
>     option tcplog
>     timeout connect 5s
>     timeout client 1m
>     timeout server 1m
>     retries 3
> frontend stats
>     bind *:8404
>     mode http
>     stats enable
>     stats uri /stats
>     stats refresh 10s
>     stats auth nexusops:${STATS}
> frontend pg_write
>     bind *:5432
>     mode tcp
>     default_backend pg_pool
> backend pg_pool
>     mode tcp
>     option httpchk GET /leader
>     http-check expect status 200
>     default-server check inter 2s fall 3 rise 2 on-marked-down shutdown-sessions check-ssl verify required ca-file /etc/nexus-haproxy/tls/ca.pem
>     server pg-primary   192.168.70.61:5432 port 8008
>     server pg-replica-1 192.168.70.62:5432 port 8008
>     server pg-replica-2 192.168.70.63:5432 port 8008
> EOF
> chown haproxy:haproxy /etc/nexus-haproxy/haproxy.cfg
> systemctl enable --now nexus-haproxy
> ```
> **EXPECTED:** HAProxy starts; exactly one backend (the leader) shows UP.
> **VERIFY:** `echo 'show servers state' | sudo socat /run/nexus-haproxy/admin.sock stdio`
> (or the `:8404/stats` UI) shows `pg-primary` UP, replicas DOWN; a
> `psql "host=192.168.70.67 port=5432 …"` connects to the leader.

### 5.6 — keepalived VRRP VIP (`.60`) on both HAProxy nodes

> **Step 5.6.1 — Install the health check + the VRRP config**
> **WHERE:** `haproxy-pg-1` (MASTER, 110) + `haproxy-pg-2` (BACKUP, 100).
> **WHY:** float `192.168.70.60` to the healthy higher-priority HAProxy.
> **Unicast** VRRP (VMnet11 multicast-split-brain gotcha). The health script
> demotes priority if HAProxy stops serving `:5432`.
> **WHAT (on haproxy-pg-1 — for -2 swap `state BACKUP`, `priority 100`, and the
> unicast IPs):**
> ```bash
> VRRP_PWD=$(vault kv get -field=password nexus/oltp/patroni/haproxy-stats-password | cut -c1-8)
> cat > /etc/keepalived/check_haproxy.sh <<'EOF'
> #!/bin/bash
> systemctl is-active --quiet nexus-haproxy || exit 1
> ss -ltn '( sport = :5432 )' | grep -q :5432 || exit 1
> exit 0
> EOF
> chmod 700 /etc/keepalived/check_haproxy.sh
> cat > /etc/keepalived/keepalived.conf <<EOF
> vrrp_script chk_haproxy {
>     script "/etc/keepalived/check_haproxy.sh"
>     interval 2
>     weight -30
>     fall 3
>     rise 2
> }
> vrrp_instance VI_HAPROXY_NEXUS {
>     state MASTER
>     interface nic0
>     virtual_router_id 60
>     priority 110
>     advert_int 1
>     unicast_src_ip 192.168.70.67
>     unicast_peer { 192.168.70.68 }
>     authentication { auth_type AH  auth_pass ${VRRP_PWD} }
>     virtual_ipaddress { 192.168.70.60/24 dev nic0 }
>     track_script { chk_haproxy }
> }
> EOF
> systemctl enable --now keepalived
> ```
> **EXPECTED:** keepalived starts; haproxy-pg-1 claims `.60`.
> **VERIFY:** `ip addr show dev nic0` on haproxy-pg-1 shows `192.168.70.60/24`; not
> on -2. From the host: `psql "host=192.168.70.60 port=5432 user=nexusops dbname=postgres sslmode=require" -c "SELECT pg_is_in_recovery();"`
> → `f` (false — you reached the leader/writer).

---

## 6. Validation — by-hand acceptance smoke

From the **host** (`ssh` + `sudo patronictl` / direct `psql`).

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | etcd 3-member healthy | `ssh …@64 'sudo etcdctl … endpoint status -w table'` | 3 members, 1 leader |
| 2 | Patroni cluster shape | `ssh …@61 'sudo patronictl -c … list'` | 1 Leader + 2 Replica, all running |
| 3 | Streaming replication | write leader, read replica (§5.4.3) | token replicated |
| 4 | mTLS enforced on PG | `psql "host=.61 sslmode=disable"` | rejected (hostssl only) |
| 5 | HAProxy routes to leader | `psql "host=.67 …" -c "SELECT pg_is_in_recovery()"` | `f` |
| 6 | VIP held by MASTER | `ip addr show nic0` on haproxy-pg-1/2 | `.60` on -1 only |
| 7 | **Query via the VIP** | host: `psql "host=192.168.70.60 …"` | connects to the writer |
| 8 | **Patroni auto-failover** | `ssh …@61 'sudo systemctl stop nexus-patroni'`; `patronictl list` from another node | a replica promotes to Leader; #7 still works (then restart node-1 → rejoins via pg_rewind) |
| 9 | **HAProxy VIP failover** | `ssh …@67 'sudo systemctl stop nexus-haproxy'`; re-check #6/#7 | VIP moves to haproxy-pg-2; #7 still works (then restart) |

**1–7 green ⇒ Guide 10 satisfied.** 8 + 9 are the two HA proofs — Patroni promotes
a new leader, and the HAProxy VIP survives a front-door loss.

---

## 7. Teardown / reset

```bash
for ip in 67 68; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now keepalived nexus-haproxy'; done
for ip in 61 62 63; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-patroni'; done
for ip in 64 65 66; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-etcd'; done
# then vmrun stop + deleteVM each of the 8 (Guide 00 §7).
```

> The Patroni cluster state lives in **etcd** (the leader lease) + each PG node's
> `data_dir`. To reset the Postgres cluster but keep etcd: stop Patroni on all 3,
> `sudo rm -rf /var/lib/nexus-patroni/data/*`, `etcdctl del --prefix /service/nexus-pg`,
> then re-run §5.4. Secrets stay in Vault KV (reused on rebuild).

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| etcd never forms quorum (spins in pre-vote) | nodes started sequentially | start all 3 etcd **in parallel** (§5.3.1). |
| Patroni: `etcd … name resolution` / can't connect | used `*.nexus.lab` (dnsmasq serves `nexus.local`); or cert lacks the bare-name SAN | use **bare** `etcd-1/2/3:2379` + ensure the cert SANs include the bare hostname (§5.2.2). |
| Replica fails: `no pg_hba.conf entry for replication connection from host 192.168.70.x` | `pg_hba` only allowed VMnet10, but replicas connect over VMnet11 | allow `192.168.0.0/16` (covers both) — §5.4.1. |
| HAProxy shows all backends `no check` / never fails over | `check` missing from `default-server` | put `check …` on `default-server` (§5.5.1). |
| HAProxy marks the leader DOWN too | `/leader` probe not over HTTPS, or CA wrong | `check-ssl verify required ca-file …/ca.pem`, `port 8008` (§5.5.1). |
| Both HAProxy nodes hold the VIP | multicast VRRP on VMnet11 | use **unicast** (`unicast_src_ip`/`unicast_peer`) — §5.6.1. |
| Old primary won't rejoin after failover | timeline diverged | `use_pg_rewind: true` handles it; if it still fails, `patronictl reinit nexus-pg <node>` (full re-clone). |
| `patronictl` AccessDenied reading the config/cert | TLS material is `0640`/`0700` | `sudo` every `patronictl`/`psql` that reads node-local TLS. |

---

## 9. Production tuning — PostgreSQL 17 under Patroni

> **Everything below is *beyond the lab replica*.** The §5.4.1 `patroni.yml` is what the
> lab actually ships — 4 GB PG VMs sized to *form the cluster and prove streaming
> replication + auto-failover + mTLS*, not to carry production load. This section is what
> you would change on a **production** deployment and why. It does **not** alter the §5
> renders. **Do not paste these onto the 4 GB lab VMs blindly** — `shared_buffers`,
> `effective_cache_size`, and static hugepages all assume production-sized RAM.
>
> **OS layer first.** A production PG host also needs the kernel / ulimit / THP tuning in
> [Guide 00 §9](./00-lab-host-and-base-vm.md#9-production-tuning--the-os-layer-feeds-every-linux-tier)
> — set that once per host, then add the engine overrides below. PostgreSQL is the one
> engine that wants **THP off *and* static `HugePages` on** (§9.3) plus `memlock` unlimited.

### 9.1 How you change these — Patroni owns `postgresql.conf`, so use `patronictl edit-config`

⚠️ **Do not `ALTER SYSTEM` these and do not hand-edit `postgresql.conf`.** Patroni is the
source of truth for the Postgres config: it renders `postgresql.conf` from the cluster
spec in the **DCS (etcd)** on every start, so a manual `ALTER SYSTEM` / file edit is
**overwritten on the next restart or reload**. Production parameters go into the **dynamic
cluster configuration** — `bootstrap.dcs.postgresql.parameters` at bootstrap, and
`patronictl edit-config` thereafter — which Patroni writes to etcd and applies to **every
member** (leader + replicas) uniformly.

```bash
# PRODUCTION — not applied in the lab. Run on any PG node; edits the whole cluster's config.
# Opens the live DCS config in $EDITOR; under `postgresql: parameters:` add/adjust:
sudo patronictl -c /etc/nexus-patroni/patroni.yml edit-config nexus-pg
```

```yaml
# The block you are editing (production values for a 4-vCPU / 16 GB PG node):
postgresql:
  use_pg_rewind: true
  use_slots: true
  parameters:
    # --- memory ---
    shared_buffers: 4GB                 # ≈ 25% of RAM
    effective_cache_size: 12GB          # ≈ 75% of RAM (a planner hint, allocates nothing)
    work_mem: 32MB                      # per sort/hash node — see the formula below
    maintenance_work_mem: 1GB           # VACUUM / CREATE INDEX / bulk load
    huge_pages: "on"                    # ⚠️ requires pre-allocated OS hugepages — §9.3
    # --- WAL / checkpoint ---
    max_wal_size: 8GB
    min_wal_size: 2GB
    checkpoint_completion_target: 0.9
    wal_compression: "on"
    # --- replication ---
    hot_standby_feedback: "on"
    # --- planner / SSD ---
    random_page_cost: 1.1
    effective_io_concurrency: 200
    # --- observability / autovacuum ---
    shared_preload_libraries: pg_stat_statements
    autovacuum_vacuum_cost_limit: 2000
```

> **Reload vs. restart.** After `edit-config`, Patroni marks members "pending restart" for
> parameters that need one. Memory/preload params (`shared_buffers`, `huge_pages`,
> `shared_preload_libraries`, `max_connections`, `max_wal_senders`) are **restart-only** —
> apply them with a rolling `sudo patronictl -c … restart nexus-pg --role replica` first,
> then switchover and restart the old leader, to keep the writer available. The rest
> (`work_mem`, `effective_cache_size`, `checkpoint_completion_target`, `autovacuum_*`,
> `random_page_cost`) take effect on **reload** (`patronictl reload nexus-pg`).

### 9.2 Memory, WAL, planner & autovacuum

| Setting | Production value | Lab value (§5.4.1) | Why it matters |
|---|---|---|---|
| `shared_buffers` | **≈ 25 % of RAM** (e.g. `4GB` on 16 GB) | `256MB` | Postgres's own page cache. ~25 % is the sweet spot — Postgres deliberately relies on the OS page cache too (unlike InnoDB), so pushing it much higher yields double-buffering, not more cache. **Restart-only.** |
| `effective_cache_size` | **≈ 75 % of RAM** (e.g. `12GB`) | unset (default `4GB`) | A **planner hint** — it allocates nothing; it tells the planner how much data is likely cached (in `shared_buffers` + OS cache) so it favours index scans over seq scans. Too low ⇒ needless sequential scans. |
| `work_mem` | **≈ RAM / (`max_connections` × avg parallel sort/hash nodes)** — e.g. 16 GB / (200 × ~2) ≈ `32MB` | unset (default `4MB`) | Per **sort/hash node** (a single query can use several × its parallel workers), so the true ceiling is `work_mem × nodes × connections`. Too high ⇒ OOM under concurrency; too low ⇒ sorts spill to disk. Size against `max_connections`, or lower `max_connections` via pooling first (see note). |
| `maintenance_work_mem` | **`512MB`–`1GB`** | unset (default `64MB`) | Memory for `VACUUM`, `CREATE INDEX`, `ALTER TABLE`, bulk restore. Higher = dramatically faster index builds and vacuum passes; it's per-maintenance-op and there are few concurrent ones, so it's safe to set high. |
| `max_wal_size` / `min_wal_size` | **several GB** (e.g. `8GB` / `2GB`) | unset (`1GB` / `80MB`) | The soft ceiling that triggers a checkpoint. Too small on a write-heavy DB ⇒ frequent forced checkpoints → I/O spikes + WAL write amplification. Larger = fewer, smoother checkpoints (at the cost of longer crash recovery). |
| `checkpoint_completion_target` | **`0.9`** | unset (default `0.9` in PG 17) | Spreads checkpoint page-flushing across 90 % of the interval instead of dumping it at once, smoothing the I/O spike. PG 14+ already defaults to `0.9` — set it explicitly so intent is documented. |
| `wal_compression` | **`on`** (`lz4`/`zstd`) | unset (`off`) | Compresses full-page images in the WAL → less WAL volume → less network to ship to replicas and less disk. Cheap CPU for meaningful WAL reduction on OLTP. |
| `hot_standby_feedback` | **`on`** | unset (`off`) | ⚠️ set on for **read-replica** workloads: the replica tells the primary which rows its running queries still need, so the primary's vacuum won't remove them out from under a long replica query (which would otherwise cause replication conflicts / query cancellations). Costs a little primary bloat — worth it when replicas serve reads. |
| `random_page_cost` | **`1.1`** (SSD/NVMe) | unset (default `4.0`) | The default `4.0` assumes spinning disks where random I/O is ~4× sequential. On SSD random ≈ sequential, so `1.1` stops the planner from irrationally avoiding index scans. |
| `effective_io_concurrency` | **`200`** (SSD/NVMe) | unset (default `1`) | How many concurrent I/Os the planner assumes the storage can service (drives bitmap-heap prefetch). `1` suits one spindle; SSDs handle hundreds. |
| `shared_preload_libraries` | **`pg_stat_statements`** | unset | Loads the query-fingerprint stats extension (the #1 production tuning tool — shows which queries burn time/I/O). **Restart-only**; then `CREATE EXTENSION pg_stat_statements`. |
| `autovacuum_vacuum_cost_limit` | **`2000`** (up from `200`) | unset (default `200`) | Raises autovacuum's I/O budget so it keeps pace with a high write rate on SSD; the default throttles vacuum so hard that dead tuples accumulate → table/index bloat + transaction-ID wraparound risk. Consider also more `autovacuum_max_workers` on many-table clusters. |

> **Connection pooling vs. raw `max_connections`.** The lab sets `max_connections: 200`
> directly. Each Postgres backend is a full OS process (~10 MB+ and a `work_mem` budget),
> so a high raw `max_connections` wastes RAM and increases lock/latch contention. In
> production, keep `max_connections` **moderate** (say `200-400`) and front it with a
> **pooler** — **PgBouncer** in `transaction` mode — so thousands of client connections
> multiplex onto a small backend pool. This is what lets `work_mem` be generous (the
> divisor is the *real* backend count, not the client count). PgBouncer would sit on the
> HAProxy nodes or beside each PG node; it is out of scope for the lab's §5 build.

### 9.3 OS layer — THP off, static HugePages on, memlock (set per [Guide 00 §9](./00-lab-host-and-base-vm.md#9-production-tuning--the-os-layer-feeds-every-linux-tier))

PostgreSQL is the exception that wants **Transparent** Huge Pages **off** *and* **static**
`HugePages` **on** — the static pages back `shared_buffers` without the runtime-compaction
stalls that THP introduces. With `huge_pages: on`, **Postgres refuses to start** unless the
kernel has enough hugepages pre-allocated, so allocate them first (or use `huge_pages: try`
as the softer fallback — it silently falls back to 4K pages if the reservation is short).

```bash
# PRODUCTION — not applied in the lab. Compute the pages Postgres needs, then reserve them.
# PG 15+ reports the exact count it wants for the current shared_buffers:
sudo -u postgres psql -tAc "SHOW shared_memory_size_in_huge_pages;"   # e.g. 2100 (× 2 MB)

cat > /etc/sysctl.d/91-nexus-postgres-hugepages.conf <<'EOF'
vm.nr_hugepages = 2200          # the value above + a little headroom
EOF
sysctl --system
# memlock must be unlimited for the postgres user (Guide 00 §9.3 sets this fleet-wide):
#   postgres  soft/hard  memlock  unlimited
# VERIFY: grep HugePages_Total /proc/meminfo   (matches nr_hugepages, and Free drops when PG starts)
```

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `huge_pages` (postgresql) | **`on`** + `vm.nr_hugepages` sized to `shared_buffers` | unset (`try` default) | ⚠️ Static hugepages back the shared-memory segment with 2 MB pages (fewer TLB misses, no `khugepaged` stalls). `on` **fails startup** if the reservation is short — pre-allocate `vm.nr_hugepages`, or use `try` to fall back gracefully. |
| Transparent Huge Pages | **`never`** | unset | ⚠️ Disable THP (Guide 00 §9.2) — its background compaction stalls Postgres; static HugePages give the win without the stalls. |
| `memlock` ulimit (`postgres`) | **`unlimited`** | unset | Required to lock the hugepage-backed shared memory; without it `huge_pages: on` cannot mlock and Postgres won't start. Guide 00 §9.3 grants it fleet-wide. |
| `vm.swappiness` | `1` | unset (`60`) | Keeps `shared_buffers` + the OS page cache resident instead of swapped. Guide 00 §9.1. |

---

### Cross-references

- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (Patroni `.60`–`.68`, VIP `.60`); ADR-0025 (LB-tier HA)
- **Automated equivalents:** `nexus-infra-oltp/packer/oltp-{patroni,etcd,haproxy}-node/` + `terraform/envs/oltp-patroni/role-overlay-*.tf`
- **Scaffolding pattern reused:** [`04-foundation-vault-pki-ldap.md`](./04-foundation-vault-pki-ldap.md) Part D
- **Previous guide:** [`09-oltp-percona-xtradb-cluster.md`](./09-oltp-percona-xtradb-cluster.md) (the sync-replication counterpart)
- **Next guide:** Guide 11 — OLTP · SQL Server FCI + Always-On AG (WSFC + iSCSI shared storage). See [`INDEX.md`](../INDEX.md).
