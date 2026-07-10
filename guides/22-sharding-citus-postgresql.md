# Guide 22 — Sharding · Citus (horizontally-sharded PostgreSQL, full Patroni HA)

> **Mirrors:** `nexus-infra-citus` (a **separate repo**) — the 2 Packer templates
> (`citus-{etcd,pg}-node`) + the `citus` Terraform env overlays
> (`…-nftables-backplane`, `…-vault-agents`, `…-tls`, `…-etcd-bootstrap`,
> `…-patroni-bootstrap`, `…-keepalived`, `…-extension`, `…-distribute`) — Phase 0.P /
> ADR-0042, tier `08-citus`. **The PostgreSQL sharding** axis — and the **final guide**.

> The second sharding guide and the sibling of Guide 21 (Vitess = MySQL sharding).
> Citus shards **PostgreSQL**, and every node-group is itself a **Patroni HA pair**,
> so this is also the most HA-dense tier. Self-contained — depends only on the
> foundation. **This guide closes the 23-guide by-hand build of the platform.**

---

## 1. Overview & purpose

**Citus** — a PostgreSQL extension that turns a coordinator + workers into a single
distributed database: tables are **sharded** (or replicated) across workers, and the
coordinator routes/plans queries. **9 nodes + 3 VRRP VIPs, three node-groups:**

- **DCS — `citus-etcd-1/2/3` (`.202/.203/.204`)** — a 3-node **etcd** quorum that is
  the **Patroni** distributed-configuration store (which PG node is leader in each
  group). Full mTLS, `client-cert-auth` (no RBAC). *Not* a Citus component — it's the
  HA substrate.
- **Coordinator — `citus-coord-1/2` (`.205/.206`)** — a **2-node Patroni** PG pair
  (1 leader + 1 streaming replica). Holds Citus's `pg_dist_*` metadata + plans
  distributed queries. VIP **`coord.citus.nexus.lab` (`.211`)** = the client endpoint.
- **Worker 1 — `citus-worker1-1/2` (`.207/.208`)** — a Patroni PG pair holding a
  slice of every distributed table's shards. VIP **`worker1.citus.nexus.lab` (`.212`)**.
- **Worker 2 — `citus-worker2-1/2` (`.209/.210`)** — a second Patroni pair holding the
  other slice. VIP **`worker2.citus.nexus.lab` (`.213`)**.

Every group's VIP **follows its Patroni leader**: a keepalived `vrrp_script` curls the
local Patroni REST `/leader` (200 only on the leader) → weight +50 → the leader holds
the VIP. The coordinator registers workers in `pg_dist_node` **by VIP**, so a worker
failover moves the VIP to the new leader with **no metadata rewrite**.

**Why Citus + Patroni HA:** to scale PostgreSQL *and* keep every shard
fault-tolerant — distributed tables spread across workers (scale-out) while each
group is a self-healing HA pair (no single point of failure). **Why it matters:** the
PG-sharding showcase — a distributed `events` table physically split across both
worker groups, with auto-failover at every tier.

---

## 2. Component primer

- **Citus.** A PostgreSQL extension (`shared_preload_libraries=citus`) that adds a
  **coordinator/worker** distributed architecture: `create_distributed_table` shards a
  table by a column; `create_reference_table` replicates a small table everywhere;
  the coordinator plans + routes. *Why:* horizontal PG scale-out with standard SQL.
  *Otherwise:* Vitess (that's MySQL — Guide 21), Postgres-XL/Greenplum (heavier
  forks), or app-level sharding.
- **Distributed / reference / colocated tables.** **Distributed** (`events`, sharded
  on `tenant_id`, 32 shards) spreads across workers; **reference** (`tenants`) is
  replicated to every node (joinable with no reshuffle); **colocated** (`event_tags`,
  same shard key as `events`) keeps related rows on the same worker so tenant-key
  joins are worker-local. *Why colocation:* avoids cross-worker repartition.
- **Patroni.** A PostgreSQL HA orchestrator — runs one leader + streaming replicas per
  cluster, stores leader state in a DCS (etcd here), and auto-fails-over. *Why per
  group:* each Citus node (coordinator + each worker) is itself an HA pair. *vs. Guide
  10's Patroni:* same tool, but here there are **three** Patroni clusters sharing one
  etcd under distinct namespaces (`/citus/<scope>`).
- **etcd as DCS.** Holds each Patroni cluster's leader lease + config. Full mTLS;
  Patroni presents its leaf cert (the cert *is* the etcd auth). *Otherwise:* Consul/
  ZooKeeper (etcd is the lab standard for Patroni).
- **keepalived VIP-follows-leader.** Instead of a static MASTER/BACKUP, the VIP floats
  to whichever node passes the `/leader` check (+50 weight). **No `nopreempt`** — the
  newly-promoted leader (priority 150) must preempt the old one (drops to 100), else
  the VIP strands on a demoted replica (T6). *vs. Guides 17/19's keepalived:* those
  promote on `notify_master`; here the VIP simply *follows* Patroni's own election.
- **Register workers by VIP.** `citus_add_node('worker1.citus.nexus.lab', 5432)` — the
  VIP, not a node IP — so a worker failover needs no `pg_dist_node` rewrite (the VIP
  moves; the cert covers the VIP via its SAN). *Why:* failover-stable topology.
- **Full mTLS, `clientcert=verify-ca`.** `pg_hba` requires every connection (clients,
  Citus inter-node, streaming replication) to present a **CA-signed client cert AND a
  scram password**. `citus.node_conninfo` makes the coordinator dial workers
  `verify-full`. The PG client key is **`0600 postgres:postgres`** (libpq accepts it
  as a client key for the coordinator→worker path *and* the server reads it). The
  shared passwords live in `~postgres/.pgpass` (Citus forbids a password in
  `node_conninfo`).

---

## 3. Prerequisites

| # | Requirement | One-command verify |
|---|---|---|
| 1 | **Foundation alive** (Guides 00–04) — Vault PKI + KV; gateway DNS | `vault status` on `vault-1` → `Sealed: false` |
| 2 | **CA bundle** on the build host (`~/.nexus/vault-ca-bundle.crt`) | `Test-Path ~/.nexus/vault-ca-bundle.crt` → `True` |
| 3 | **9 `deb13` nodes** baselined (Guide 00), dual-NIC static `.202–.210` | the 9 answer `:22`; firstboot mapped roles + `NEXUS_PG_ROLE` |
| 4 | **PG 17 + Citus 14.1 + Patroni + etcd artifacts** reachable (or caches) | `curl -sI https://repos.citusdata.com | head -1` |

> **Versions:** **PostgreSQL 17** (Debian trixie native apt) + **Citus 14.1**, Patroni
> **4.0.5** (pip venv `/opt/patroni-venv`), etcd **3.5.16**, keepalived (apt). Database
> `citus`; distributed on `tenant_id` (32 shards). Endpoints: VIPs
> `coord/worker1/worker2.citus.nexus.lab` → `.211/.212/.213`. KV under
> `nexus/citus/{superuser,replication,patroni-restapi,citus-app}-password`.

> **By-hand divergence:** read KV with `vault kv get` (no Vault Agent); issue certs
> with the `vault` CLI from the **`citus-server`** PKI role. Certs in
> `/etc/nexus-citus/tls/` (`server-cert.pem`/`server-key.pem`/`ca.pem`); the **PG
> client key is `0600 postgres:postgres`** (T5). The etcd DCS uses the same full-mTLS
> pattern as Vitess's etcd (Guide 21) and Patroni is the same orchestrator as Guide
> 10 — but **every command is written out in full here**; you don't need to open
> those guides.

---

## 4. Target topology

| Node | Role | VMnet11 | VMnet10 | Patroni scope | VIP |
|---|---|---|---|---|---|
| `citus-etcd-1/2/3` | etcd DCS (Patroni store) | `.202/.203/.204` | `.10.202–.204` | — | — |
| `citus-coord-1/2` | coordinator (Patroni pair) | `.205/.206` | `.10.205/.206` | `citus-coord` | **`.211`** `coord.citus.nexus.lab` |
| `citus-worker1-1/2` | worker 1 (Patroni pair) | `.207/.208` | `.10.207/.208` | `citus-worker1` | **`.212`** `worker1.citus.nexus.lab` |
| `citus-worker2-1/2` | worker 2 (Patroni pair) | `.209/.210` | `.10.209/.210` | `citus-worker2` | **`.213`** `worker2.citus.nexus.lab` |

> MAC block `:D7–:DF`. **PG wire + etcd + replication ride the VMnet10 backplane**
> (the `pg_hba` `192.168.0.0/16` spans both NICs); client/VIP/VRRP ride VMnet11. VMs
> under `H:\VMS\NexusPlatform\08-citus\<node>\`. PG socket dir `/var/run/nexus-citus`;
> data dir `/var/lib/nexus-citus/pgdata`.

---

## 5. Step-by-step build

> **WHERE:** node steps as `nexusadmin` → `sudo -i` (root). `vault` runs on
> **`vault-1`**. Order: etcd DCS → Patroni (3 groups) → keepalived (VIPs) → Citus
> extension (wire the cluster) → distribute (shard the demo). `patronictl` runs via
> `/usr/local/sbin/nexus-patronictl` (preloaded with the etcd endpoints + certs).

### 5.0 — Foundation: KV seeds, PKI role, VIP DNS

> **Step 5.0.1 — Seed the 4 creds + the `citus-server` PKI role + VIP DNS**
> **WHERE:** `vault-1` (KV + PKI); `nexus-gateway` (DNS).
> **WHY:** the PG/Patroni passwords + the PKI role (server **and** client EKU — the
> coordinator dials workers as a client; certs validate the VIP via its SAN). The 3
> VIP DNS host-records front the groups.
> **WHAT (on vault-1):**
> ```bash
> export VAULT_ADDR=https://127.0.0.1:8200 VAULT_CACERT=$HOME/.nexus/vault-ca-bundle.crt
> for c in superuser replication patroni-restapi citus-app; do
>   vault kv put nexus/citus/$c-password value="$(openssl rand -hex 20)"
> done
> vault write pki_int/roles/citus-server \
>   allowed_domains='citus.nexus.lab,coord.citus.nexus.lab,worker1.citus.nexus.lab,worker2.citus.nexus.lab,nexus.lab,localhost' \
>   allow_subdomains=true allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
> ```
> **WHAT (on the gateway — VIP host-records):**
> ```bash
> cat > /etc/dnsmasq-citus.hosts <<'EOF'
> 192.168.70.211 coord.citus.nexus.lab
> 192.168.70.212 worker1.citus.nexus.lab
> 192.168.70.213 worker2.citus.nexus.lab
> EOF
> echo 'addn-hosts=/etc/dnsmasq-citus.hosts' > /etc/dnsmasq.d/citus-records.conf
> dnsmasq --test && systemctl reload dnsmasq
> ```
> **VERIFY:** `vault read pki_int/roles/citus-server`; `dig @192.168.70.1 coord.citus.nexus.lab +short` → `.211`.

> **Step 5.0.2 — Issue + place a leaf cert on ALL 9 nodes**
> **WHERE:** issue on `vault-1`; place on each of the 9 VMs.
> **WHY:** every PG wire + etcd + Patroni REST connection is mTLS. The per-node
> differences are in the table below. The 3 **etcd** nodes go to `/etc/nexus-etcd/tls`
> (owner `etcd`); the 6 **PG** nodes go to `/etc/nexus-citus/tls` (owner `postgres`,
> **key `0600`** — PG rejects a db-user key that's group/world-readable, T5). Each PG
> node adds its **group VIP name + VIP IP** to the SANs (so the cert validates when a
> client hits the floating VIP).
>
> | # | Node | VMnet11 | CN | extra `alt_names` / `ip_sans` VIP | cert dir | owner / key |
> |---|---|---|---|---|---|---|
> | 1 | `citus-etcd-1` | `.202` | `citus-etcd-1.citus.nexus.lab` | — / `192.168.10.202,192.168.70.202,127.0.0.1` | `/etc/nexus-etcd/tls` | `etcd:etcd` / `0640` |
> | 2 | `citus-etcd-2` | `.203` | `citus-etcd-2.citus.nexus.lab` | — / `192.168.10.203,192.168.70.203,127.0.0.1` | `/etc/nexus-etcd/tls` | `etcd:etcd` / `0640` |
> | 3 | `citus-etcd-3` | `.204` | `citus-etcd-3.citus.nexus.lab` | — / `192.168.10.204,192.168.70.204,127.0.0.1` | `/etc/nexus-etcd/tls` | `etcd:etcd` / `0640` |
> | 4 | `citus-coord-1` | `.205` | `citus-coord-1.citus.nexus.lab` | `coord.citus.nexus.lab` / `192.168.10.205,192.168.70.205,192.168.70.211,127.0.0.1` | `/etc/nexus-citus/tls` | `postgres:postgres` / **`0600`** |
> | 5 | `citus-coord-2` | `.206` | `citus-coord-2.citus.nexus.lab` | `coord.citus.nexus.lab` / `192.168.10.206,192.168.70.206,192.168.70.211,127.0.0.1` | `/etc/nexus-citus/tls` | `postgres:postgres` / **`0600`** |
> | 6 | `citus-worker1-1` | `.207` | `citus-worker1-1.citus.nexus.lab` | `worker1.citus.nexus.lab` / `192.168.10.207,192.168.70.207,192.168.70.212,127.0.0.1` | `/etc/nexus-citus/tls` | `postgres:postgres` / **`0600`** |
> | 7 | `citus-worker1-2` | `.208` | `citus-worker1-2.citus.nexus.lab` | `worker1.citus.nexus.lab` / `192.168.10.208,192.168.70.208,192.168.70.212,127.0.0.1` | `/etc/nexus-citus/tls` | `postgres:postgres` / **`0600`** |
> | 8 | `citus-worker2-1` | `.209` | `citus-worker2-1.citus.nexus.lab` | `worker2.citus.nexus.lab` / `192.168.10.209,192.168.70.209,192.168.70.213,127.0.0.1` | `/etc/nexus-citus/tls` | `postgres:postgres` / **`0600`** |
> | 9 | `citus-worker2-2` | `.210` | `citus-worker2-2.citus.nexus.lab` | `worker2.citus.nexus.lab` / `192.168.10.210,192.168.70.210,192.168.70.213,127.0.0.1` | `/etc/nexus-citus/tls` | `postgres:postgres` / **`0600`** |
>
> **WHAT — issuance, for EACH of the 9 (substitute CN + alt_names + ip_sans from the table):**
> ```bash
> # on vault-1 -- example: citus-worker1-1 (row 6). etcd rows have no VIP alt_name.
> vault write -format=json pki_int/issue/citus-server \
>   common_name=citus-worker1-1.citus.nexus.lab \
>   alt_names='citus-worker1-1,citus-worker1-1.citus.nexus.lab,worker1.citus.nexus.lab,localhost' \
>   ip_sans='192.168.10.207,192.168.70.207,192.168.70.212,127.0.0.1' ttl=2160h > /tmp/citus-worker1-1.json
> # an etcd row instead: alt_names='citus-etcd-1,citus-etcd-1.citus.nexus.lab,localhost' (no VIP)
> vault read -field=certificate pki_int/cert/ca_chain > /tmp/nexus-ca-chain.pem
> ```
> **WHAT — placement, on EACH node (set `D` + `OWN` + `KMODE` from the table):**
> ```bash
> # copy /tmp/<host>.json + /tmp/nexus-ca-chain.pem to the node, then as root:
> D=/etc/nexus-citus/tls ; OWN=postgres:postgres ; KMODE=0600    # etcd nodes: D=/etc/nexus-etcd/tls OWN=etcd:etcd KMODE=0640
> install -d -o ${OWN%:*} -g ${OWN#*:} -m 0750 "$D"
> jq -r '.data.certificate' /tmp/<host>.json > /tmp/leaf.crt
> jq -r '.data.issuing_ca'  /tmp/<host>.json > /tmp/int.crt
> jq -r '.data.private_key' /tmp/<host>.json > /tmp/leaf.key
> cat /tmp/leaf.crt /tmp/int.crt > "$D/server-cert.pem"
> openssl pkcs8 -topk8 -nocrypt -in /tmp/leaf.key -out "$D/server-key.pem"
> cp /tmp/nexus-ca-chain.pem "$D/ca.pem"
> chown -R "$OWN" "$D" ; chmod 0644 "$D/server-cert.pem" "$D/ca.pem" ; chmod "$KMODE" "$D/server-key.pem"
> rm -f /tmp/leaf.crt /tmp/int.crt /tmp/leaf.key /tmp/<host>.json
> ```
> **VERIFY (each node):** `sudo openssl x509 -in <D>/server-cert.pem -noout -ext subjectAltName`
> → the CN (+ the group VIP name + VIP IP on PG nodes); on a PG node
> `stat -c '%a %U' /etc/nexus-citus/tls/server-key.pem` → `600 postgres`.

### 5.1 — Install the two node types

> **Step 5.1.1 — etcd DCS nodes (`.202–.204`)**
> **WHERE:** `citus-etcd-1/2/3`, root shell.
> **WHY:** etcd 3.5.16 is the Patroni store. The node needs **both** an `etcd` group
> (owns the cert dir + runs the daemon) and a `citus` group (firstboot chowns the
> node-identity to `citus`). Install the binary + an mTLS-preloaded `nexus-etcdctl`
> wrapper; the unit stays disabled until §5.2.2 renders the config.
> **WHAT (on each etcd node):**
> ```bash
> getent group citus >/dev/null || groupadd --system citus
> getent group etcd >/dev/null  || groupadd --system etcd
> getent passwd etcd >/dev/null || useradd --system --gid etcd --home-dir /var/lib/nexus-etcd --create-home --shell /usr/sbin/nologin etcd
> curl -fSL https://github.com/etcd-io/etcd/releases/download/v3.5.16/etcd-v3.5.16-linux-amd64.tar.gz | tar xz -C /tmp
> install -m0755 /tmp/etcd-v3.5.16-linux-amd64/etcd /tmp/etcd-v3.5.16-linux-amd64/etcdctl /usr/local/bin/
> install -d -o etcd -g etcd -m0750 /var/lib/nexus-etcd /etc/nexus-etcd/tls
> # mTLS-preloaded wrapper (reads /etc/nexus-etcd/endpoints, written in §5.2.2)
> cat > /usr/local/sbin/nexus-etcdctl <<'EOS'
> #!/bin/bash
> exec /usr/local/bin/etcdctl --endpoints="$(cat /etc/nexus-etcd/endpoints)" \
>   --cacert=/etc/nexus-etcd/tls/ca.pem --cert=/etc/nexus-etcd/tls/server-cert.pem --key=/etc/nexus-etcd/tls/server-key.pem "$@"
> EOS
> chmod 0755 /usr/local/sbin/nexus-etcdctl
> # systemd unit (config-gated, disabled until §5.2.2)
> cat > /etc/systemd/system/nexus-etcd.service <<'EOF'
> [Unit]
> Description=Nexus etcd (Citus Patroni DCS)
> After=network-online.target
> Wants=network-online.target
> ConditionPathExists=/etc/nexus-etcd/etcd.conf.yml
> [Service]
> Type=notify
> User=etcd
> ExecStart=/usr/local/bin/etcd --config-file=/etc/nexus-etcd/etcd.conf.yml
> Restart=on-failure
> RestartSec=5
> LimitNOFILE=65536
> [Install]
> WantedBy=multi-user.target
> EOF
> systemctl daemon-reload ; systemctl disable nexus-etcd 2>/dev/null || true
> ```
> **VERIFY:** `etcd --version` → `3.5.16`; `id etcd` + `getent group citus`.

> **Step 5.1.2 — PG nodes (`.205–.210`): PostgreSQL 17 + Citus 14.1 + Patroni + keepalived**
> **WHERE:** the 6 PG nodes, root shell.
> **WHY:** PG 17 (native trixie apt) + Citus 14.1 from the Citus apt repo + Patroni in
> a pip venv + keepalived. ⚠️ the Citus apt codename probe must **follow redirects**
> (packagecloud answers `Release` with a 302; a HEAD+`==200` check wrongly falls back
> to bookworm — and `13.0` is bookworm-only; trixie publishes `14.1`, T1). The stock
> Debian `main` cluster is dropped + the `postgresql*` units masked (Patroni owns the
> datadir).
> **WHAT (each PG node):**
> ```bash
> getent group citus >/dev/null || groupadd --system citus
> apt-get install -y postgresql-17 postgresql-client-17 keepalived python3-venv python3-psycopg2 curl
> # Citus 14.1 (follow redirects to land the native trixie repo, T1)
> curl -fsSL https://install.citusdata.com/community/deb.sh | bash    # adds the repo (trixie)
> apt-get install -y postgresql-17-citus-14.1
> # drop the stock cluster + mask the units (Patroni owns PGDATA)
> pg_dropcluster --stop 17 main 2>/dev/null || true
> systemctl mask postgresql postgresql@17-main
> # Patroni 4.0.5 in a venv
> python3 -m venv /opt/patroni-venv
> /opt/patroni-venv/bin/pip install 'patroni[etcd3]==4.0.5' psycopg2-binary
> install -d -o postgres -g postgres -m0750 /var/lib/nexus-citus /var/lib/nexus-citus/pgdata /etc/patroni
> install -d -o root -g postgres -m0750 /etc/nexus-citus /etc/nexus-citus/tls
> # patronictl wrapper
> cat > /usr/local/sbin/nexus-patronictl <<'EOS'
> #!/bin/bash
> exec /opt/patroni-venv/bin/patronictl -c /etc/patroni/patroni.yml "$@"
> EOS
> chmod 0755 /usr/local/sbin/nexus-patronictl
> ```
> Install `nexus-patroni.service`
> (`ExecStart=/opt/patroni-venv/bin/patroni /etc/patroni/patroni.yml`, `User=postgres`,
> **`RuntimeDirectory=nexus-citus`** so `/var/run/nexus-citus` exists on every start —
> `/run` is tmpfs + root-owned, T4) + `nexus-keepalived.service`, both disabled.
> **VERIFY:** `/usr/lib/postgresql/17/bin/postgres --version` → `17.x`;
> `/opt/patroni-venv/bin/patroni --version` → `4.0.5`; `ls /usr/lib/postgresql/17/lib/citus.so`.

### 5.2 — nftables + etcd DCS

> **Step 5.2.1 — nftables (all 9: backplane trust + PG/Patroni/VRRP ports)**
> **WHERE:** each node, root shell.
> **WHY:** trust the VMnet10 backplane (PG wire + etcd + replication); open PG `5432`
> + Patroni REST `8008` + VRRP on VMnet11 (PG nodes); etcd `2379/2380` is backplane-only.
> **WHAT (PG node — adapt for etcd nodes: drop 5432/8008, the backplane rule covers 2379/2380):**
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
>         ip protocol ah accept comment "keepalived VRRP AH auth"
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 22   accept comment "SSH"
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 9100 accept comment "node_exporter"
>         iifname "nic1" ip saddr 192.168.10.0/24 accept comment "trusted cluster backplane (VMnet10) -- PG + etcd + replication"
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport { 5432, 8008 } accept comment "PostgreSQL + Patroni REST (VMnet11)"
>         counter drop
>     }
>     chain forward { type filter hook forward priority 0; policy drop; }
>     chain output  { type filter hook output priority 0; policy accept; }
> }
> EOF
> nft -f /etc/nftables.conf ; systemctl enable nftables 2>/dev/null || true
> ```
> **VERIFY:** `nft list chain inet filter input | grep '192.168.10.0/24 accept'` (all 9).

> **Step 5.2.2 — Bring up the etcd DCS (3-member, full mTLS)**
> **WHERE:** `citus-etcd-1/2/3`, root shell.
> **WHY:** render `etcd.conf.yml` on all 3 (each listens/advertises on its **VMnet10**
> IP — matching the cert IP-SAN — so Patroni dials it by IP and avoids the hostname
> self-dial trap), `client-cert-auth: true` (the cert is the authorization, no RBAC),
> then start all 3 **close together** (Raft needs simultaneous start for the first
> `initial-cluster-state: new` bootstrap) and verify a leader + an mTLS round-trip.
> **WHAT (on each etcd node — set `NAME`/`SELF`/`CLIENT11` per node: `.202/.203/.204`):**
> ```bash
> NAME=citus-etcd-1 ; SELF=192.168.10.202 ; CLIENT11=192.168.70.202   # per node
> cat > /etc/nexus-etcd/etcd.conf.yml <<EOF
> name: $NAME
> data-dir: /var/lib/nexus-etcd
> listen-peer-urls: https://$SELF:2380
> listen-client-urls: https://$SELF:2379,https://$CLIENT11:2379,https://127.0.0.1:2379
> initial-advertise-peer-urls: https://$SELF:2380
> advertise-client-urls: https://$SELF:2379
> initial-cluster: citus-etcd-1=https://192.168.10.202:2380,citus-etcd-2=https://192.168.10.203:2380,citus-etcd-3=https://192.168.10.204:2380
> initial-cluster-token: nexus-citus-etcd
> initial-cluster-state: new
> peer-transport-security:   { cert-file: /etc/nexus-etcd/tls/server-cert.pem, key-file: /etc/nexus-etcd/tls/server-key.pem, trusted-ca-file: /etc/nexus-etcd/tls/ca.pem, client-cert-auth: true }
> client-transport-security: { cert-file: /etc/nexus-etcd/tls/server-cert.pem, key-file: /etc/nexus-etcd/tls/server-key.pem, trusted-ca-file: /etc/nexus-etcd/tls/ca.pem, client-cert-auth: true }
> auto-compaction-mode: periodic
> auto-compaction-retention: "1"
> EOF
> chown root:etcd /etc/nexus-etcd/etcd.conf.yml ; chmod 0640 /etc/nexus-etcd/etcd.conf.yml
> printf 'https://192.168.10.202:2379,https://192.168.10.203:2379,https://192.168.10.204:2379' > /etc/nexus-etcd/endpoints
> systemctl daemon-reload ; systemctl enable nexus-etcd
> ```
> Then **start all 3 close together**: on each node run
> `systemctl reset-failed nexus-etcd; systemctl start nexus-etcd`.
> **EXPECTED:** a leader is elected within a minute.
> **VERIFY:** `nexus-etcdctl endpoint status --write-out=table` shows a leader;
> `nexus-etcdctl put /nexus/x ok && nexus-etcdctl get /nexus/x` round-trips.

### 5.3 — Patroni: 3 HA groups over the shared etcd

> **Step 5.3.1 — Render `patroni.yml` + `.pgpass` per node; start each group in parallel**
> **WHERE:** each PG node, root shell. Set `SCOPE`/`SELF` per node (coord →
> `citus-coord`, worker1 → `citus-worker1`, worker2 → `citus-worker2`).
> **WHY:** three 2-node Patroni clusters sharing one etcd under `/citus/<scope>`. The
> PG params carry `shared_preload_libraries=citus` + `max_prepared_transactions=100`
> (Citus needs both) + `citus.node_conninfo` (verify-full mTLS to workers). `pg_hba`
> is `clientcert=verify-ca` everywhere. ⚠️ Patroni 4 formats `username:password`
> literally at load — substitute the KV passwords into the config (T from 0.G.4); the
> superuser/replication/rewind auth blocks **must** carry `sslmode/sslcert/sslkey/
> sslrootcert` or the replica's `pg_basebackup` is rejected (no client cert, T5); and
> `~postgres/.pgpass` carries the password half of inter-node auth.
> **WHAT (per node — the load-bearing `patroni.yml`; passwords from `vault kv get`):**
> ```bash
> SCOPE=citus-coord ; SELF=192.168.70.205   # per node
> SUPER=$(vault kv get -field=value nexus/citus/superuser-password)
> REPL=$(vault kv get -field=value nexus/citus/replication-password)
> REST=$(vault kv get -field=value nexus/citus/patroni-restapi-password)
> cat > /etc/patroni/patroni.yml <<EOF
> scope: $SCOPE
> namespace: /citus/
> name: $(hostname)
> restapi:
>   listen: 0.0.0.0:8008
>   connect_address: $SELF:8008
>   certfile: /etc/nexus-citus/tls/server-cert.pem
>   keyfile: /etc/nexus-citus/tls/server-key.pem
>   cafile: /etc/nexus-citus/tls/ca.pem
>   authentication: { username: nexusops, password: $REST }
> etcd3:
>   hosts: citus-etcd-1:2379,citus-etcd-2:2379,citus-etcd-3:2379
>   protocol: https
>   cacert: /etc/nexus-citus/tls/ca.pem
>   cert: /etc/nexus-citus/tls/server-cert.pem
>   key: /etc/nexus-citus/tls/server-key.pem
> bootstrap:
>   dcs:
>     ttl: 30
>     loop_wait: 10
>     retry_timeout: 10
>     postgresql:
>       use_pg_rewind: true
>       use_slots: true
>       parameters:
>         wal_level: replica
>         hot_standby: "on"
>         max_wal_senders: 10
>         max_replication_slots: 10
>         password_encryption: scram-sha-256
>         shared_preload_libraries: citus
>         max_prepared_transactions: 100
>         citus.node_conninfo: "sslmode=verify-full sslrootcert=/etc/nexus-citus/tls/ca.pem sslcert=/etc/nexus-citus/tls/server-cert.pem sslkey=/etc/nexus-citus/tls/server-key.pem"
>   initdb: [ { encoding: UTF8 }, data-checksums ]
>   pg_hba:
>     - "hostssl replication replicator 192.168.0.0/16 scram-sha-256 clientcert=verify-ca"
>     - "hostssl all all 192.168.0.0/16 scram-sha-256 clientcert=verify-ca"
>     - "host all all 127.0.0.1/32 trust"
>     - "host replication replicator 127.0.0.1/32 trust"
> postgresql:
>   listen: 0.0.0.0:5432
>   connect_address: $SELF:5432
>   data_dir: /var/lib/nexus-citus/pgdata
>   bin_dir: /usr/lib/postgresql/17/bin
>   authentication:
>     superuser:   { username: postgres,   password: $SUPER, sslmode: verify-ca, sslrootcert: /etc/nexus-citus/tls/ca.pem, sslcert: /etc/nexus-citus/tls/server-cert.pem, sslkey: /etc/nexus-citus/tls/server-key.pem }
>     replication: { username: replicator, password: $REPL,  sslmode: verify-ca, sslrootcert: /etc/nexus-citus/tls/ca.pem, sslcert: /etc/nexus-citus/tls/server-cert.pem, sslkey: /etc/nexus-citus/tls/server-key.pem }
>     rewind:      { username: rewind,      password: $REPL,  sslmode: verify-ca, sslrootcert: /etc/nexus-citus/tls/ca.pem, sslcert: /etc/nexus-citus/tls/server-cert.pem, sslkey: /etc/nexus-citus/tls/server-key.pem }
>   parameters:
>     unix_socket_directories: /var/run/nexus-citus
>     ssl: "on"
>     ssl_cert_file: /etc/nexus-citus/tls/server-cert.pem
>     ssl_key_file: /etc/nexus-citus/tls/server-key.pem
>     ssl_ca_file: /etc/nexus-citus/tls/ca.pem
> watchdog: { mode: off }
> EOF
> chown postgres:postgres /etc/patroni/patroni.yml ; chmod 0640 /etc/patroni/patroni.yml
> # the replicator/postgres passwords for inter-node + replication libpq (Citus forbids password in node_conninfo)
> printf '*:5432:*:postgres:%s\n*:5432:*:replicator:%s\n' "$SUPER" "$REPL" > ~postgres/.pgpass
> chown postgres:postgres ~postgres/.pgpass ; chmod 0600 ~postgres/.pgpass
> systemctl daemon-reload ; systemctl reset-failed nexus-patroni 2>/dev/null || true ; systemctl enable nexus-patroni
> ```
> Then **start both nodes of each group together** (race for the etcd lease):
> `systemctl restart nexus-patroni` on both `citus-coord-*`, then both `citus-worker1-*`,
> then both `citus-worker2-*`.
> **EXPECTED:** each scope converges to **1 Leader + 1 streaming Replica**.
> **VERIFY:** `nexus-patronictl list` (on any node) → per scope, one `Leader` + one
> `Replica` (`running`/`streaming`).

### 5.4 — keepalived: a VIP per group, following the Patroni leader

> **Step 5.4.1 — Render the leader-check + `keepalived.conf` on all 6 PG nodes**
> **WHERE:** each PG node (`.205–.210`), root shell. Set `VIP`/`VRID`/`INSTANCE`/`PEER`
> per group (coord `.211`/`211`/`VI_CITUS_COORD`; worker1 `.212`/`212`; worker2
> `.213`/`213`).
> **WHY:** the VIP **follows** the Patroni leader — the `vrrp_script` curls the local
> Patroni REST `/leader` (200 only on the leader) and adds **weight +50** so the leader
> (priority 150) holds the VIP. ⚠️ **No `nopreempt`** (else the VIP strands on a
> demoted replica after failover, T6). Unicast VRRP (VMware multicast is unreliable),
> AH auth from the first 8 chars of the restapi password.
> **WHAT (per node):**
> ```bash
> VIP=192.168.70.211 ; VRID=211 ; INSTANCE=VI_CITUS_COORD ; SELF=192.168.70.205 ; PEER=192.168.70.206   # per node
> AUTH=$(sudo cat /etc/nexus-citus/patroni-restapi-password | cut -c1-8)
> cat > /etc/keepalived/check_citus_leader.sh <<'EOS'
> #!/bin/bash
> code=$(curl -s -o /dev/null -w '%{http_code}' --cacert /etc/nexus-citus/tls/ca.pem https://127.0.0.1:8008/leader 2>/dev/null || echo 000)
> [ "$code" = "200" ] && exit 0 || exit 1
> EOS
> chmod 0700 /etc/keepalived/check_citus_leader.sh
> cat > /etc/keepalived/keepalived.conf <<EOF
> global_defs { router_id citus_$(hostname) ; enable_script_security ; script_user root }
> vrrp_script chk_citus_leader { script "/etc/keepalived/check_citus_leader.sh" ; interval 2 ; timeout 3 ; rise 1 ; fall 2 ; weight 50 }
> vrrp_instance $INSTANCE {
>   state BACKUP
>   interface nic0
>   virtual_router_id $VRID
>   priority 100
>   advert_int 1
>   unicast_src_ip $SELF
>   unicast_peer { $PEER }
>   authentication { auth_type AH ; auth_pass $AUTH }
>   virtual_ipaddress { $VIP/24 dev nic0 label nic0:vip }
>   track_script { chk_citus_leader }
> }
> EOF
> chmod 0640 /etc/keepalived/keepalived.conf
> systemctl daemon-reload ; systemctl enable --now nexus-keepalived
> ```
> **EXPECTED:** each VIP binds on its group's Patroni **leader**.
> **VERIFY:** `ip -4 -o addr show nic0 | grep 192.168.70.211` on exactly the
> coordinator leader; `dig @192.168.70.1 coord.citus.nexus.lab +short` → `.211`.

### 5.5 — Wire the Citus distributed cluster (extension + add workers)

> **Step 5.5.1 — CREATE EXTENSION + register coordinator + add workers (by VIP)**
> **WHERE:** the **coordinator leader** (find it with `nexus-patronictl list`), root shell.
> **WHY:** create the `citus` database + extension on the coordinator **and** each
> worker (reached via its VIP over verify-full mTLS), set the coordinator host (by its
> VIP), and `citus_add_node` each worker **by VIP** (so a worker failover needs no
> metadata rewrite). ⚠️ keepalived just started — a VIP can briefly sit on a read-only
> replica; **gate on `pg_is_in_recovery()=f` (`wait_rw`) before CREATE EXTENSION** or
> it fails "cannot execute … in a read-only transaction" (T7).
> **WHAT (on the coordinator leader):**
> ```bash
> SOCK=/var/run/nexus-citus ; DB=citus
> TLS="sslmode=verify-full sslrootcert=/etc/nexus-citus/tls/ca.pem sslcert=/etc/nexus-citus/tls/server-cert.pem sslkey=/etc/nexus-citus/tls/server-key.pem"
> APP=$(sudo cat /etc/nexus-citus/citus-app-password)
> wait_rw() { for i in $(seq 1 40); do [ "$(sudo -u postgres psql "host=$1 port=5432 dbname=postgres user=postgres $TLS" -tA -c 'SELECT pg_is_in_recovery()' 2>/dev/null)" = "f" ] && return 0; sleep 3; done; return 1; }
> # coordinator: db + extension
> sudo -u postgres psql -h "$SOCK" -tA -c "SELECT 1 FROM pg_database WHERE datname='$DB'" | grep -q 1 || sudo -u postgres psql -h "$SOCK" -c "CREATE DATABASE $DB"
> sudo -u postgres psql -h "$SOCK" -d "$DB" -c "CREATE EXTENSION IF NOT EXISTS citus"
> # each worker (via VIP, mTLS): db + extension
> for w in worker1.citus.nexus.lab worker2.citus.nexus.lab; do
>   wait_rw "$w"
>   sudo -u postgres psql "host=$w port=5432 dbname=postgres user=postgres $TLS" -tA -c "SELECT 1 FROM pg_database WHERE datname='$DB'" | grep -q 1 || sudo -u postgres psql "host=$w port=5432 dbname=postgres user=postgres $TLS" -c "CREATE DATABASE $DB"
>   sudo -u postgres psql "host=$w port=5432 dbname=$DB user=postgres $TLS" -c "CREATE EXTENSION IF NOT EXISTS citus"
> done
> # register coordinator (by VIP) + add workers (by VIP) + the citus_app role
> sudo -u postgres psql -h "$SOCK" -d "$DB" -c "SELECT citus_set_coordinator_host('coord.citus.nexus.lab', 5432)"
> for w in worker1.citus.nexus.lab worker2.citus.nexus.lab; do
>   sudo -u postgres psql -h "$SOCK" -d "$DB" -tA -c "SELECT 1 FROM pg_dist_node WHERE nodename='$w'" | grep -q 1 || sudo -u postgres psql -h "$SOCK" -d "$DB" -c "SELECT citus_add_node('$w', 5432)"
> done
> sudo -u postgres psql -h "$SOCK" -d "$DB" -c "DO \$\$ BEGIN IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname='citus_app') THEN CREATE ROLE citus_app LOGIN PASSWORD '$APP'; END IF; END \$\$;"
> sudo -u postgres psql -h "$SOCK" -d "$DB" -c "GRANT ALL ON DATABASE $DB TO citus_app; GRANT ALL ON SCHEMA public TO citus_app"
> ```
> **EXPECTED:** `pg_dist_node` shows the coordinator + 2 active worker primaries.
> **VERIFY:** `sudo -u postgres psql -h "$SOCK" -d citus -c "SELECT nodename,noderole,isactive FROM pg_dist_node ORDER BY groupid"`
> → coordinator + `worker1/worker2.citus.nexus.lab` active.

### 5.6 — Distribute the demo + the sharding proof (exit gate)

> **Step 5.6.1 — Create reference + distributed + colocated tables; seed; prove**
> **WHERE:** the coordinator leader, root shell.
> **WHY:** the deterministic exit gate — a **reference** table (`tenants`, replicated
> everywhere), a **distributed** table (`events`, hash-sharded on `tenant_id`, 32
> shards across both workers), and a **colocated** table (`event_tags`). Seed, then
> prove the shards span **both** worker groups.
> **WHAT (on the coordinator leader, in the `citus` DB):**
> ```bash
> SOCK=/var/run/nexus-citus
> PQ() { sudo -u postgres psql -h "$SOCK" -U postgres -d citus -v ON_ERROR_STOP=1 -tA -c "$1"; }
> PQ "CREATE TABLE IF NOT EXISTS tenants (tenant_id int PRIMARY KEY, name text NOT NULL)"
> PQ "SELECT create_reference_table('tenants')"
> PQ "SET citus.shard_count = 32; CREATE TABLE IF NOT EXISTS events (event_id bigint, tenant_id int NOT NULL, payload text, created_at timestamptz DEFAULT now(), PRIMARY KEY (tenant_id, event_id))"
> PQ "SELECT create_distributed_table('events', 'tenant_id')"
> PQ "CREATE TABLE IF NOT EXISTS event_tags (event_id bigint, tenant_id int NOT NULL, tag text NOT NULL, PRIMARY KEY (tenant_id, event_id, tag))"
> PQ "SELECT create_distributed_table('event_tags', 'tenant_id', colocate_with => 'events')"
> # seed
> PQ "INSERT INTO tenants SELECT g, 'tenant-'||g FROM generate_series(1,8) g ON CONFLICT DO NOTHING"
> PQ "INSERT INTO events (event_id,tenant_id,payload) SELECT g,(g%8)+1,'evt-'||g FROM generate_series(1,800) g ON CONFLICT DO NOTHING"
> PQ "INSERT INTO event_tags (event_id,tenant_id,tag) SELECT g,(g%8)+1,'tag-'||(g%5) FROM generate_series(1,800) g ON CONFLICT DO NOTHING"
> # PROOF: shards span both worker groups + cross-shard aggregate + colocated join
> PQ "SELECT count(DISTINCT nodename) FROM citus_shards WHERE table_name='events'::regclass"   # expect 2
> PQ "SELECT count(*) FROM events"                                                              # expect 800
> PQ "SELECT count(*) FROM events e JOIN event_tags t USING (tenant_id, event_id)"
> PQ "SELECT nodename, count(*) shards FROM citus_shards WHERE table_name='events'::regclass GROUP BY nodename ORDER BY nodename"
> ```
> **EXPECTED:** `count(DISTINCT nodename)` = **2** (both worker groups); `count(*)` =
> 800; the per-node shard summary lists both worker VIPs.
> **VERIFY:** that's the sharding proof — a distributed table physically split across
> both worker groups, with a coordinator-merged cross-shard aggregate + a worker-local
> colocated join. **➡ The sharded PostgreSQL cluster is live — and the platform's
> 23-guide by-hand build is complete.**

---

## 6. Validation — by-hand acceptance smoke (demo / playbook)

Condensed from `smoke-0.P.ps1`. Per-node SSH probes from the **build host**.

- **Input:** the 9 nodes up; etcd quorum; 3 Patroni groups converged; 3 VIPs bound;
  Citus wired; demo distributed.
- **Where observed:** SSH / `nexus-etcdctl` / `nexus-patronictl` / `psql` on the
  coordinator / VIP `ip addr`.
- **Proves:** a sharded PostgreSQL with per-group Patroni HA + VIP-follows-leader.
- **Prerequisites:** Guides 00–04 alive; §5 complete.

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | 9 nodes reachable | `ssh …@202..210 'echo ok'` | all `ok` |
| 2 | etcd quorum | `nexus-etcdctl endpoint status` | a leader; healthy |
| 3 | 3 Patroni groups | `nexus-patronictl list` (each scope) | each: 1 Leader + 1 Replica streaming |
| 4 | 3 VIPs bound | `ip addr show nic0 \| grep .211/.212/.213` | each on its group's leader |
| 5 | VIP DNS | `dig coord/worker1/worker2.citus.nexus.lab +short` | `.211/.212/.213` |
| 6 | Citus topology | `SELECT … FROM pg_dist_node` | coordinator + 2 active workers |
| 7 | mTLS | a no-client-cert `psql` to a worker VIP | rejected (`clientcert=verify-ca`) |
| 8 | **Sharding proof** | `count(DISTINCT nodename) FROM citus_shards` (events) | **2** worker groups; `count(*)` = 800 |
| 9 | **Worker Patroni failover** (chaos) | kill the worker1 leader's Patroni; VIP `.212` follows; distributed query still works | new leader + VIP within ~30 s; `SELECT count(*) FROM events` unaffected |

**1–8 green ⇒ Guide 22 satisfied.** 9 is the HA payoff — a worker group survives a
leader loss (Patroni promotes, the VIP follows, `pg_dist_node` needs no rewrite). **8
also closes the platform: every tier from the host network to PostgreSQL sharding is
now reproducible entirely by hand.**

---

## 7. Teardown / reset

```bash
for ip in 205 206 207 208 209 210; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-keepalived nexus-patroni; sudo rm -rf /var/lib/nexus-citus/pgdata/*'; done
for ip in 202 203 204; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-etcd'; done
# gateway: rm /etc/dnsmasq.d/citus-records.conf /etc/dnsmasq-citus.hosts ; systemctl reload dnsmasq
# then vmrun stop + deleteVM each of the 9 (Guide 00 §7).
```

> **The distributed data lives in the workers' PGDATA** (`/var/lib/nexus-citus/pgdata`).
> A fresh apply re-inits empty Patroni clusters + re-wires Citus. The Vault PKI role +
> KV creds + the VIP DNS survive (sticky). To wipe: delete the VMs.

---

## 8. Troubleshooting (the 0.P 7-transient gauntlet)

| # | Symptom | Cause | Fix |
|---|---|---|---|
| **T1** | Citus apt install fails / wrong codename (bookworm) / `13.0` not found | the codename probe used HEAD+`==200`, but packagecloud answers `Release` with a **302** → fell back to bookworm; `13.0` is bookworm-only | follow redirects (`GET`, `follow_redirects: all`) to land the **native trixie** repo; install `citus-14.1` (§5.1.2). |
| **T2** | `patronictl --version` errors | only the `patroni` binary has `--version`, not `patronictl` | check the binary with `stat`/`patroni --version` (§5.1.2). |
| **T3** | keepalived version probe blows up | `keepalived --version` writes to **stderr**; an empty stdout sequence indexing fails | version-check `postgres`/`patroni`, `stat` keepalived (§5.1.2). |
| **T4** | Patroni (postgres) can't create `/run/nexus-citus` | `/run` is tmpfs + root-owned; Patroni runs as `postgres` | `RuntimeDirectory=nexus-citus` in `nexus-patroni.service` (auto-creates per start) (§5.1.2). |
| **T5** | replica `pg_basebackup` rejected | `clientcert=verify-ca` requires a client cert, but the Patroni auth blocks had none | add `sslmode/sslcert/sslkey/sslrootcert` to the superuser+replication+rewind auth blocks (§5.3.1). |
| **T6** | after a failover the VIP strands on a demoted replica | keepalived `nopreempt` pinned the VIP to whoever was MASTER first | **remove `nopreempt`** — the VIP follows the leader (the +50 weight makes the new leader preempt) (§5.4.1). |
| **T7** | `CREATE EXTENSION` fails "cannot execute … in a read-only transaction" | keepalived just started; the VIP was briefly on a read-only replica | gate on `pg_is_in_recovery()=f` (`wait_rw`) before wiring (§5.5.1). |
| **—** | Backplane down (etcd/replication can't connect) | VMware left nic1 NO-CARRIER at power-on | reconnect the 2nd NIC + `systemctl restart systemd-networkd` (as Guides 16–21). |

---

## 9. Production tuning — Citus on PostgreSQL

> **Everything below is *beyond the lab replica*.** The §5 configs run coordinator + workers
> at lab-scale on 2 GB VMs with default Citus sizing (`shard_count` unset → `32`,
> `shard_replication_factor` unset → `1`) — enough to prove distribution + HA, not to serve a
> real sharded workload. This section is what you would change for **production**, and *why*.
> **Do not paste these onto the 2 GB lab VMs blindly.**

### 9.1 The base PostgreSQL layer — tune it via Patroni

Citus **is** PostgreSQL 17 + Patroni (the *same* stack as Guide 10, just three Patroni groups
instead of one), so the base engine tuning — `shared_buffers`, `effective_cache_size`,
`work_mem`, `maintenance_work_mem`, checkpoint/WAL, `max_connections` — is covered by
**[Guide 10 §9](./10-oltp-patroni-postgresql-ha.md)** and the Guide 00 §9 OS layer (THP-off,
`nofile`, `vm.overcommit_memory`, huge pages). This guide does not restate it.

> **One difference in *how* you apply it:** these nodes are Patroni-managed, so PG parameters
> go through **`patronictl edit-config`** (which writes them into the DCS and rolls them out),
> **not** a hand-edited `postgresql.conf` — Patroni owns that file and will overwrite it. Edit
> the `postgresql.parameters:` block (the §5.3.1 Patroni YAML already sets
> `shared_preload_libraries`, `max_prepared_transactions`, `citus.node_conninfo` there):
>
> ```bash
> # PRODUCTION — run on any node; Patroni propagates to all members of that scope.
> sudo /usr/local/sbin/nexus-patronictl edit-config citus-coord \
>   -s 'postgresql.parameters.shared_buffers=4GB' \
>   -s 'postgresql.parameters.effective_cache_size=12GB'
> # repeat for scopes citus-worker1 / citus-worker2 (workers get the larger share — see 9.3)
> ```

### 9.2 Citus distribution knobs (GUCs — set via `patronictl edit-config` or `ALTER SYSTEM`)

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `citus.shard_count` | **2–4× total worker cores** (e.g. 8-core × 2 workers → `48`–`64`) | default `32`; §5.4 demo sets `SET citus.shard_count = 32` session-local | Fixes the shard grid at `create_distributed_table` time and **cannot be changed for an existing table** without a re-distribute. Too few shards → hot workers + no room to rebalance when you add nodes; too many → per-shard planning/connection overhead. Size to cores so shards spread evenly and leave headroom for scale-out. |
| `citus.shard_replication_factor` | `1` (rely on **Patroni streaming HA** per group) | default `1` (unset) | With per-worker-group Patroni (this lab's design), each shard is already HA via streaming replication — so `1` is correct. Raise to `2` **only** if you use Citus statement-based shard replication *instead* of streaming HA (this lab does not). Leave at `1` here. |
| `citus.max_adaptive_executor_pool_size` | `16`–`32` per worker (bounded by worker cores) | default `16` | Max connections the coordinator opens **per worker** for one distributed query. Higher = more intra-query parallelism, but multiplied across concurrent queries it can exhaust worker `max_connections`. Balance against 9.3. |
| `citus.max_shared_pool_size` | ≈ worker `max_connections` − headroom (e.g. `max_connections=200` → `160`) | default (follows `max_connections`) | **Global** cap on coordinator→worker connections across *all* queries — the guard that stops a query storm from exhausting every worker's connection slots. Set it explicitly below worker `max_connections` so PG always keeps slots for replication/admin. |
| `citus.node_conninfo` | `sslmode=verify-full …` | **PRESENT** — `sslmode=verify-full` + ca/cert/key (§5.3.1, line 415) | The coordinator↔worker libpq TLS string. The lab already ships full mutual-TLS verification; noted so it's not mistaken for a gap. Production keeps `verify-full`. |
| `max_prepared_transactions` | `≥ max_connections` (e.g. `200`+) | **PRESENT** — `100` (§5.3.1, line 414) | Citus uses 2PC for cross-shard writes, so this **must** be non-zero (a hard Citus requirement, already satisfied). Raise it in step with `max_connections` under high write concurrency, or distributed writes fail with `maximum number of prepared transactions reached`. |
| `citus.enable_repartition_joins` | `on` | default `off` | Lets Citus execute joins **not** aligned on the distribution column by repartitioning shards on the fly. Essential for ad-hoc analytical joins across differently-sharded tables; off means those joins error (`complex joins are only supported…`). Costs network/IO, so enable knowingly. |

```sql
-- PRODUCTION — on the coordinator leader (also settable via patronictl edit-config so it
-- survives failover). Not applied in the lab.
ALTER SYSTEM SET citus.max_adaptive_executor_pool_size = 24;
ALTER SYSTEM SET citus.max_shared_pool_size            = 160;
ALTER SYSTEM SET citus.enable_repartition_joins        = on;
SELECT pg_reload_conf();
-- shard_count is per-table at creation, not a reloadable GUC for existing tables:
-- SET citus.shard_count = 64; CREATE TABLE events (...) ... ; SELECT create_distributed_table('events','tenant_id');
```

### 9.3 Coordinator vs. worker sizing — they are not the same shape

Citus's two roles have **opposite** resource profiles; size their VMs and PG params
accordingly rather than cloning one config across all six nodes:

| Dimension | Coordinator (`citus-coord-1/2`, VIP `.211`) | Workers (`citus-worker1/2-*`, VIPs `.212/.213`) |
|---|---|---|
| Primary pressure | **Connection + planning heavy** — terminates every client connection, plans + routes distributed queries, runs 2PC coordination | **Memory + I/O heavy** — hold the actual shard data, execute the per-shard scans/joins/aggregates |
| RAM / `shared_buffers` | Moderate — it stores *no* shard data; enough for the catalog + planner | **Large** — this is where `shared_buffers`/`effective_cache_size` matter; workers cache the real data pages |
| `max_connections` | **High** — every app connection lands here; pair with a pooler (PgBouncer) in front | Moderate — sized to `citus.max_adaptive_executor_pool_size × concurrent-coordinator-queries` + headroom |
| CPU | Planning + result merging | Parallel shard execution — more cores = more shards scanned at once |
| Scale axis | Vertical (bigger box) + an HA standby; usually **1 active** coordinator | **Horizontal** — add worker groups + `rebalance_table_shards()` to grow |

> **Rule of thumb:** give workers the RAM/IO/cores and the big `shared_buffers`; give the
> coordinator connection capacity + a front-end pooler. A coordinator starved of connections
> bottlenecks the whole cluster; a worker starved of RAM turns every shard scan into disk I/O.

### 9.4 OS layer

All six Citus nodes are Debian PG hosts — apply **[Guide 00 §9](./00-lab-host-and-base-vm.md)**
(THP-off, `vm.overcommit_memory=1`, `nofile`, huge pages for `shared_buffers`, low readahead)
exactly as for the Guide 10 OLTP nodes, before the Citus knobs above.

---

### Cross-references

- **0.P architecture:** memory `project_nexus_infra_0p_phase`; ADR-0042 (Citus + full
  Patroni HA); the `nexus-infra-citus` handbook §3 (the T1–T7 gauntlet)
- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (citus `.202–.210` +
  VIPs `.211/.212/.213`, MAC `:D7–:DF`); `vms.yaml` (tier `08-citus`)
- **Automated equivalents:** `nexus-infra-citus/packer/citus-{etcd,pg}-node/` +
  `terraform/envs/citus/role-overlay-citus-{etcd-bootstrap,patroni-bootstrap,keepalived,extension,distribute,tls}.tf`
- **Smoke mirror:** `nexus-infra-citus/scripts/smoke-0.P.ps1`
- **Sibling sharding tier:** [`21-sharding-vitess-mysql.md`](./21-sharding-vitess-mysql.md) (Vitess = the MySQL sharding axis)
- **Patroni precedent:** [`10-oltp-patroni-postgresql-ha.md`](./10-oltp-patroni-postgresql-ha.md) (single Patroni cluster; here there are three) · etcd mTLS pattern = [`21-…`](./21-sharding-vitess-mysql.md) §5.3
- **Transients:** [[feedback_cluster_template_nftables_backplane]] · [[systemd-runtime-directory-tmpfs]] · [[keepalived-check-versioned-binary]]
- **Previous guide:** [`21-sharding-vitess-mysql.md`](./21-sharding-vitess-mysql.md)
- **Next guide:** — **none. This is the final guide (22 of 22).** The entire NexusPlatform
  infrastructure layer — host networking through PostgreSQL sharding — is now documented
  for a complete by-hand rebuild. See [`INDEX.md`](../INDEX.md).
