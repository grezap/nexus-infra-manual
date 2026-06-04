# Guide 21 — Sharding · Vitess (horizontally-sharded MySQL)

> **Mirrors:** `nexus-infra-vitess` (a **separate repo**) — the 3 Packer templates
> (`vitess-{etcd,gate,tablet}-node`) + the `vitess` Terraform env overlays
> (`…-nftables-backplane`, `…-vault-agents`, `…-tls`, `…-etcd-bootstrap`,
> `…-gate` [vtctld/vtgate/vtorc], `…-tablets`, `…-reparent`, `…-schema`) — Phase 0.O /
> ADR-0041, tier `07-vitess`. The **MySQL sharding** axis of the platform.

> The first of the two **sharding** guides. Contrast: Guide 10 (Patroni) = PG HA by
> *replication*; Guide 08/12 = Mongo replica-set / document sharding; **this** =
> relational **MySQL** horizontal sharding. Self-contained — depends only on the
> foundation (no MinIO).

---

## 1. Overview & purpose

**Vitess** — a database clustering system that shards a MySQL keyspace across many
servers while presenting a **single MySQL endpoint** to clients. **12 nodes, four
roles:**

- **Topology — `vitess-etcd-1/2/3` (`.190/.191/.192`)** — a 3-node **etcd** quorum
  holding the global+local topology (which tablet is where, who's PRIMARY, the
  vschema). Full mTLS (the client cert *is* the access control — no RBAC).
- **Control — `vitess-control-1` (`.193`)** — **vtctld** (the cluster admin/API
  server, web `:15000`) + **VTOrc** (the failure detector that **auto-reparents** a
  shard when its PRIMARY dies).
- **Routers — `vitess-vtgate-1/2` (`.194/.195`)** — **vtgate**, the stateless proxy
  that speaks the **MySQL protocol** (`:15306`) to clients and routes/scatter-gathers
  queries to the right shard(s). Round-robin `vtgate.nexus.lab`, no VIP.
- **Tablets — 2 shards × 3 (`.196–.201`)** — each tablet = **vttablet** (the Vitess
  agent) + a local **Percona Server 8.4** mysqld (managed by **mysqlctld**). Keyspace
  `commerce` is split into shards `-80` (`.196/.197/.198`, aliases `nexus-100/101/102`)
  and `80-` (`.199/.200/.201`, aliases `nexus-200/201/202`); each shard is **1 PRIMARY
  + 2 REPLICA**.

**Why Vitess:** to scale MySQL *writes* horizontally — a single keyspace's data is
split by a **hash vindex** on the sharding key, so each shard holds a slice and the
cluster's write capacity grows with shard count, all behind one MySQL endpoint. **Why
it matters:** it's the relational-sharding showcase — a single `INSERT` stream
physically lands across multiple shards, and VTOrc keeps each shard available.

---

## 2. Component primer

- **Vitess.** A MySQL sharding/clustering layer (vtgate + vttablet + vtctld + VTOrc +
  etcd topo). *Why:* horizontal MySQL scaling + online resharding + transparent
  failover, all behind the MySQL wire protocol. *Otherwise:* ProxySQL+manual sharding
  (no resharding/topology), Citus (that's *PostgreSQL* — Guide 22), or app-level
  sharding (brittle).
- **vtgate.** The stateless query router/proxy — speaks MySQL to clients, parses SQL,
  consults the **vschema**, and routes to the owning shard (or scatters + gathers).
  *Why:* clients see one MySQL; sharding is invisible. *vs. a tablet:* vtgate holds no
  data.
- **vttablet + mysqlctld + Percona.** Each MySQL instance is fronted by a **vttablet**
  (health, query gating, reparenting) and its mysqld is lifecycle-managed by
  **mysqlctld** (init, start/stop, datadir). Percona Server 8.4 LTS is the engine.
  *Why mysqlctld not the apt mysqld:* Vitess owns the my.cnf + datadir layout
  (`$VTDATAROOT/vt_<uid>/`); the stock `mysql.service` is **masked**.
- **vtctld + VTOrc.** **vtctld** is the admin API (every `vtctldclient` call + the
  reparent/schema operations go through it; web `:15000`). **VTOrc** watches the
  keyspace and, when a PRIMARY dies, **automatically promotes a replica** (the HA
  story) via the tablet-manager gRPC. *Otherwise:* manual `PlannedReparentShard` only.
- **etcd topology service.** Stores the cluster's source of truth (cells, shards,
  tablet records, vschema). 3-node Raft quorum. **Full mTLS with `client-cert-auth`** —
  a valid client cert from the `vitess-server` PKI role is the authorization (no etcd
  RBAC). *Otherwise:* ZooKeeper/Consul (etcd is Vitess's default).
- **keyspace / shard / vindex.** A **keyspace** is a logical database (`commerce`); a
  **shard** is a key-range slice (`-80` = hashes `< 0x80…`, `80-` = the rest); a
  **vindex** maps a column to a shard (here a **hash** vindex on `customer_id`).
  *Why hash:* even distribution. *Otherwise:* a lookup vindex (for secondary keys).
- **Durability policy.** `none` (async replication) for the lab — `semi_sync` (a write
  acks only after a replica acks) needs the `semisync_source/replica` plugins loaded
  (the 0.O.1 hardening). VTOrc still reparents on PRIMARY loss.
- **Full mTLS.** Every gRPC channel (vtgate↔tablet, vtctld↔tablet, tablet↔tablet),
  the mysqld wire, *and* the vtgate MySQL listener carry Vault-PKI certs. No
  `--server-name` on most clients — Go verifies against the **dial IP** (every
  per-host cert has its VMnet10/11 IPs in the SANs).

---

## 3. Prerequisites

| # | Requirement | One-command verify |
|---|---|---|
| 1 | **Foundation alive** (Guides 00–04) — Vault PKI + KV; gateway DNS | `vault status` on `vault-1` → `Sealed: false` |
| 2 | **CA bundle** on the build host (`~/.nexus/vault-ca-bundle.crt`) | `Test-Path ~/.nexus/vault-ca-bundle.crt` → `True` |
| 3 | **12 `deb13` nodes** baselined (Guide 00), dual-NIC static `.190–.201` | the 12 answer `:22`; firstboot mapped roles |
| 4 | **Percona 8.4 + Vitess v24 + etcd 3.5 artifacts** reachable (or local caches) | `curl -sI https://github.com/vitessio/vitess/releases | head -1` |

> **Versions:** **Vitess v24.0.1** (vtgate/vttablet/vtctld/vtorc/vtctldclient),
> **Percona Server 8.4 LTS**, **etcd 3.5.16**. Cell `nexus`; keyspace `commerce`
> (2 shards, hash vindex on `customer_id`). Front door: round-robin
> `vtgate.nexus.lab` → `.194/.195`, MySQL `:15306`. KV creds under
> `nexus/vitess/{mysql-root,mysql-app,mysql-allprivs,mysql-repl,vtorc-topo}-password`.

> **By-hand divergence:** read KV with `vault kv get` (no Vault Agent); issue certs
> with the `vault` CLI from the **`vitess-server`** PKI role (CN `<host>.vitess.nexus.lab`,
> IP-SANs for both NICs + `vtgate.nexus.lab`). Vitess uses the `server-cert.pem` /
> `server-key.pem` / `ca.pem` naming (etcd nodes under `/etc/nexus-etcd/tls`, all
> others `/etc/nexus-vitess/tls`). **Vitess v24 flags are the DASH form**
> (`--topo-implementation`, `--grpc-cert`, …).

---

## 4. Target topology

| Node | Role | VMnet11 | VMnet10 | Tablet alias / shard | Ports |
|---|---|---|---|---|---|
| `vitess-etcd-1/2/3` | etcd topo (mTLS quorum) | `.190/.191/.192` | `.10.190–.192` | — | 2379 client · 2380 peer |
| `vitess-control-1` | vtctld + VTOrc | `.193` | `.10.193` | — | 15000 vtctld web · 15999 grpc · 16000 VTOrc |
| `vitess-vtgate-1/2` | vtgate (MySQL router) | `.194/.195` | `.10.194/.195` | — | 15306 MySQL · 15001 web · 15991 grpc |
| `vitess-shard1-tablet-1/2/3` | tablet (shard `-80`) | `.196/.197/.198` | `.10.196–.198` | `nexus-100/101/102` | 3306 mysqld · 15101 web · 16101 grpc |
| `vitess-shard2-tablet-1/2/3` | tablet (shard `80-`) | `.199/.200/.201` | `.10.199–.201` | `nexus-200/201/202` | 3306 · 15101 · 16101 |

> MAC block `:CB–:D6`. **All gRPC + replication + etcd ride the VMnet10 backplane**
> (by IP, matching cert IP-SANs); client MySQL (`:15306`) + web UIs ride VMnet11. VMs
> under `H:\VMS\NexusPlatform\07-vitess\<node>\`. Initial PRIMARY of each shard = its
> lowest-uid tablet (`nexus-100` / `nexus-200`).

---

## 5. Step-by-step build

> **WHERE:** node steps as `nexusadmin` → `sudo -i` (root). `vault` runs on
> **`vault-1`**. The Vitess bring-up is **not linear** — order: etcd → vtctld+cell →
> tablets → reparent → vtgate+VTOrc → schema. `vtctldclient` runs on the **control
> node** via the mTLS-preloaded `/usr/local/sbin/nexus-vtctldclient` wrapper.

### 5.0 — Foundation: KV seeds + the PKI role

> **Step 5.0.1 — Seed the 5 MySQL/VTOrc creds + create the `vitess-server` PKI role**
> **WHERE:** `vault-1` (`.121`), root shell.
> **WHY:** the tablets/routers read these passwords; the PKI role issues every leaf
> (server **and** client EKU — clients dial by IP).
> **WHAT:**
> ```bash
> export VAULT_ADDR=https://127.0.0.1:8200 VAULT_CACERT=$HOME/.nexus/vault-ca-bundle.crt
> for c in mysql-root mysql-app mysql-allprivs mysql-repl vtorc-topo; do
>   vault kv put nexus/vitess/$c-password value="$(openssl rand -hex 20)"
> done
> vault write pki_int/roles/vitess-server \
>   allowed_domains='vitess.nexus.lab,vtgate.nexus.lab,nexus.lab,localhost' \
>   allow_subdomains=true allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
> ```
> **WHAT (gateway — round-robin DNS):**
> ```bash
> ssh nexusadmin@192.168.70.1 'printf "192.168.70.194 vtgate.nexus.lab\n192.168.70.195 vtgate.nexus.lab\n" | sudo tee /etc/dnsmasq-vitess.hosts; echo "addn-hosts=/etc/dnsmasq-vitess.hosts" | sudo tee /etc/dnsmasq.d/vitess-records.conf; sudo systemctl reload dnsmasq'
> ```
> **VERIFY:** `vault read pki_int/roles/vitess-server`; `dig @192.168.70.1 vtgate.nexus.lab +short` → `.194` + `.195`.

> **Step 5.0.2 — Issue + place a leaf cert on ALL 12 nodes**
> **WHERE:** issue on `vault-1`; place on each of the 12 VMs.
> **WHY:** every component validates over mTLS, so **every one of the 12 nodes** gets
> its own leaf cert (CN `<host>.vitess.nexus.lab`). The only per-node differences are
> the IP SANs (that node's two IPs), the destination dir, and the owner — captured in
> the table below. The 3 **etcd** nodes place certs in `/etc/nexus-etcd/tls` (owner
> `etcd`, the user etcd runs as); the other 9 in `/etc/nexus-vitess/tls` (owner
> `root:vitess`). Only the 2 **vtgate** nodes add `vtgate.nexus.lab` to the SANs (it's
> the round-robin name clients hit).
>
> | # | Node | VMnet11 | CN | `ip_sans` | dest dir | owner |
> |---|---|---|---|---|---|---|
> | 1 | `vitess-etcd-1` | `.190` | `vitess-etcd-1.vitess.nexus.lab` | `192.168.10.190,192.168.70.190,127.0.0.1` | `/etc/nexus-etcd/tls` | `etcd:etcd` |
> | 2 | `vitess-etcd-2` | `.191` | `vitess-etcd-2.vitess.nexus.lab` | `192.168.10.191,192.168.70.191,127.0.0.1` | `/etc/nexus-etcd/tls` | `etcd:etcd` |
> | 3 | `vitess-etcd-3` | `.192` | `vitess-etcd-3.vitess.nexus.lab` | `192.168.10.192,192.168.70.192,127.0.0.1` | `/etc/nexus-etcd/tls` | `etcd:etcd` |
> | 4 | `vitess-control-1` | `.193` | `vitess-control-1.vitess.nexus.lab` | `192.168.10.193,192.168.70.193,127.0.0.1` | `/etc/nexus-vitess/tls` | `root:vitess` |
> | 5 | `vitess-vtgate-1` | `.194` | `vitess-vtgate-1.vitess.nexus.lab` | `192.168.10.194,192.168.70.194,127.0.0.1` | `/etc/nexus-vitess/tls` | `root:vitess` |
> | 6 | `vitess-vtgate-2` | `.195` | `vitess-vtgate-2.vitess.nexus.lab` | `192.168.10.195,192.168.70.195,127.0.0.1` | `/etc/nexus-vitess/tls` | `root:vitess` |
> | 7 | `vitess-shard1-tablet-1` | `.196` | `vitess-shard1-tablet-1.vitess.nexus.lab` | `192.168.10.196,192.168.70.196,127.0.0.1` | `/etc/nexus-vitess/tls` | `root:vitess` |
> | 8 | `vitess-shard1-tablet-2` | `.197` | `vitess-shard1-tablet-2.vitess.nexus.lab` | `192.168.10.197,192.168.70.197,127.0.0.1` | `/etc/nexus-vitess/tls` | `root:vitess` |
> | 9 | `vitess-shard1-tablet-3` | `.198` | `vitess-shard1-tablet-3.vitess.nexus.lab` | `192.168.10.198,192.168.70.198,127.0.0.1` | `/etc/nexus-vitess/tls` | `root:vitess` |
> | 10 | `vitess-shard2-tablet-1` | `.199` | `vitess-shard2-tablet-1.vitess.nexus.lab` | `192.168.10.199,192.168.70.199,127.0.0.1` | `/etc/nexus-vitess/tls` | `root:vitess` |
> | 11 | `vitess-shard2-tablet-2` | `.200` | `vitess-shard2-tablet-2.vitess.nexus.lab` | `192.168.10.200,192.168.70.200,127.0.0.1` | `/etc/nexus-vitess/tls` | `root:vitess` |
> | 12 | `vitess-shard2-tablet-3` | `.201` | `vitess-shard2-tablet-3.vitess.nexus.lab` | `192.168.10.201,192.168.70.201,127.0.0.1` | `/etc/nexus-vitess/tls` | `root:vitess` |
>
> ⏱ **Ordering:** the `vault write` (issuance) can run now. The **placement** writes
> into each node's tls dir, which that node's **§5.1 install step creates** — so run a
> node's placement *after* its §5.1 step (or pre-create the dir here). Either way, all
> 12 are done before §5.3 (etcd) needs them.
> **WHAT — issuance, for EACH of the 12 (substitute CN + ip_sans from the table; only vtgate rows add `vtgate.nexus.lab`):**
> ```bash
> # on vault-1 -- example: vitess-shard1-tablet-1 (row 7). Repeat per node.
> vault write -format=json pki_int/issue/vitess-server \
>   common_name=vitess-shard1-tablet-1.vitess.nexus.lab \
>   alt_names='vitess-shard1-tablet-1,vitess-shard1-tablet-1.vitess.nexus.lab,localhost' \
>   ip_sans='192.168.10.196,192.168.70.196,127.0.0.1' ttl=2160h > /tmp/vitess-shard1-tablet-1.json
> # a vtgate row instead would be:
> #   alt_names='vitess-vtgate-1,vitess-vtgate-1.vitess.nexus.lab,vtgate.nexus.lab,localhost'
> vault read -field=certificate pki_int/cert/ca_chain > /tmp/nexus-ca-chain.pem
> ```
> **WHAT — placement, on EACH node (set `D` + `OWN` from the table's dest dir + owner):**
> ```bash
> # copy /tmp/<host>.json + /tmp/nexus-ca-chain.pem to the node, then as root on the node:
> D=/etc/nexus-vitess/tls ; OWN=root:vitess          # etcd nodes: D=/etc/nexus-etcd/tls ; OWN=etcd:etcd
> jq -r '.data.certificate' /tmp/<host>.json > /tmp/leaf.crt
> jq -r '.data.issuing_ca'  /tmp/<host>.json > /tmp/int.crt
> jq -r '.data.private_key' /tmp/<host>.json > /tmp/leaf.key
> cat /tmp/leaf.crt /tmp/int.crt > "$D/server-cert.pem"            # leaf + intermediate
> openssl pkcs8 -topk8 -nocrypt -in /tmp/leaf.key -out "$D/server-key.pem"   # PKCS#8 key
> cp /tmp/nexus-ca-chain.pem "$D/ca.pem"                           # the CA chain
> chown -R $OWN "$D" ; chmod 0644 "$D/server-cert.pem" "$D/ca.pem" ; chmod 0640 "$D/server-key.pem"
> rm -f /tmp/leaf.crt /tmp/int.crt /tmp/leaf.key /tmp/<host>.json
> ```
> **VERIFY (each of the 12 nodes):** `openssl x509 -in <D>/server-cert.pem -noout -subject -ext subjectAltName`
> → the host's CN + its two IP SANs (and `vtgate.nexus.lab` on the 2 vtgate nodes).

### 5.1 — Install the three node types

> **Step 5.1.1 — etcd nodes (`.190–.192`)**
> **WHERE:** `vitess-etcd-1/2/3`, root shell.
> **WHY:** etcd 3.5.16 + an `nexus-etcdctl` mTLS wrapper. ⚠️ the node needs **both** an
> `etcd` group (for the cert dir) **and** a `vitess` group (firstboot chowns the
> node-identity to `vitess`, T3). Unit installed disabled.
> **WHAT (each etcd node):**
> ```bash
> getent group vitess >/dev/null || groupadd --system vitess
> getent group etcd >/dev/null || groupadd --system etcd
> getent passwd etcd >/dev/null || useradd --system --gid etcd --home-dir /var/lib/nexus-etcd --create-home --shell /usr/sbin/nologin etcd
> curl -fSL https://github.com/etcd-io/etcd/releases/download/v3.5.16/etcd-v3.5.16-linux-amd64.tar.gz | tar xz -C /tmp
> install -m0755 /tmp/etcd-v3.5.16-linux-amd64/etcd /tmp/etcd-v3.5.16-linux-amd64/etcdctl /usr/local/bin/
> install -d -o etcd -g etcd -m0750 /var/lib/nexus-etcd /etc/nexus-etcd/tls
> # nexus-etcdctl wrapper (mTLS-preloaded; reads /etc/nexus-etcd/endpoints)
> cat > /usr/local/sbin/nexus-etcdctl <<'EOS'
> #!/bin/bash
> exec /usr/local/bin/etcdctl --endpoints="$(cat /etc/nexus-etcd/endpoints)" \
>   --cacert=/etc/nexus-etcd/tls/ca.pem --cert=/etc/nexus-etcd/tls/server-cert.pem --key=/etc/nexus-etcd/tls/server-key.pem "$@"
> EOS
> chmod 0755 /usr/local/sbin/nexus-etcdctl
> ```
> Install `nexus-etcd.service` (`ExecStart=/usr/local/bin/etcd --config-file=/etc/nexus-etcd/etcd.conf.yml`, `User=etcd`), disabled.
> **VERIFY:** `etcd --version` → `3.5.16`.

> **Step 5.1.2 — control + vtgate nodes (`.193/.194/.195`) — the `gate` engine**
> **WHERE:** `vitess-control-1` + `vitess-vtgate-1/2`, root shell.
> **WHY:** the Vitess binaries with no mysqld (vtgate/vtctld/vtorc/vtctldclient). ⚠️
> extract the Vitess tarball under **`/var/tmp`**, not `/tmp` (tmpfs ENOSPC on the
> ~1.5 GB extract, B2). One template serves both roles; the right units start later.
> **WHAT (each gate node):**
> ```bash
> getent group vitess >/dev/null || groupadd --system vitess
> getent passwd vitess >/dev/null || useradd --system --gid vitess --home-dir /var/lib/nexus-vitess --create-home --shell /usr/sbin/nologin vitess
> cd /var/tmp   # NOT /tmp (B2)
> curl -fSL https://github.com/vitessio/vitess/releases/download/v24.0.1/vitess-24.0.1-*.tar.gz -o vitess.tgz
> tar xzf vitess.tgz && rm vitess.tgz
> install -m0755 vitess-24.0.1*/bin/{vtgate,vtctld,vtorc,vtctldclient} /usr/local/bin/
> install -d -o root -g vitess -m0750 /etc/nexus-vitess /etc/nexus-vitess/tls
> install -d -o vitess -g vitess -m0750 /var/lib/nexus-vitess
> ```
> Install the `nexus-vtgate/vtctld/vtorc.service` units
> (`EnvironmentFile=-/etc/nexus-vitess/<comp>.env`, `ExecStart=/usr/local/bin/<comp> $<COMP>_FLAGS`), all disabled.
> **VERIFY:** `vtgate --version` → `v24.0.1`.

> **Step 5.1.3 — tablet nodes (`.196–.201`) — Percona 8.4 + Vitess**
> **WHERE:** the 6 tablet nodes, root shell.
> **WHY:** Percona Server 8.4 LTS (apt `ps-84-lts`) + the Vitess tablet binaries. ⚠️
> create the `vitess` user with its **primary group only**, then add it to `mysql`
> **after** Percona installs that group (B1); extract Vitess under `/var/tmp` (B2);
> **mask** the stock `mysql.service` (mysqlctld owns mysqld).
> **WHAT (each tablet node):**
> ```bash
> getent group vitess >/dev/null || groupadd --system vitess
> getent passwd vitess >/dev/null || useradd --system --gid vitess --home-dir /var/lib/nexus-vitess --create-home --shell /usr/sbin/nologin vitess
> # Percona 8.4 LTS
> curl -fSL https://repo.percona.com/apt/percona-release_latest.generic_all.deb -o /tmp/pr.deb && dpkg -i /tmp/pr.deb
> percona-release setup ps-84-lts
> apt-get update && apt-get install -y percona-server-server
> systemctl mask mysql.service   # mysqlctld owns mysqld
> usermod -aG mysql vitess       # AFTER Percona created the mysql group (B1)
> # Vitess tablet binaries
> cd /var/tmp
> curl -fSL https://github.com/vitessio/vitess/releases/download/v24.0.1/vitess-24.0.1-*.tar.gz -o vitess.tgz && tar xzf vitess.tgz && rm vitess.tgz
> install -m0755 vitess-24.0.1*/bin/{vttablet,mysqlctl,mysqlctld,vtctldclient} /usr/local/bin/
> install -d -o root -g vitess -m0750 /etc/nexus-vitess /etc/nexus-vitess/tls
> install -d -o vitess -g vitess -m0750 /var/lib/nexus-vitess
> ```
> Install `nexus-mysqlctld.service` + `nexus-vttablet.service` (EnvironmentFile pattern), disabled.
> **VERIFY:** `/usr/sbin/mysqld --version` → `8.4`; `vttablet --version` → `v24.0.1`.

### 5.2 — nftables (all 12: backplane trust + role ports)

> **Step 5.2.1 — Apply the per-cluster ruleset**
> **WHERE:** each node, root shell.
> **WHY:** trust the VMnet10 backplane (all gRPC + etcd + replication); open the
> client/web ports on VMnet11 per role (etcd none on VMnet11; vtgate `15306`+`15001`;
> control `15000`+`16000`; tablet `15101`).
> **WHAT (vtgate example — adapt the VMnet11 ports per role):**
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
>         iifname "nic1" ip saddr 192.168.10.0/24 accept comment "trusted cluster backplane (VMnet10) -- gRPC + etcd + replication"
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport { 15306, 15001 } accept comment "vtgate MySQL listener + web (VMnet11)"
>         counter drop
>     }
>     chain forward { type filter hook forward priority 0; policy drop; }
>     chain output  { type filter hook output priority 0; policy accept; }
> }
> EOF
> nft -f /etc/nftables.conf ; systemctl enable nftables 2>/dev/null || true
> ```
> **VERIFY:** `nft list chain inet filter input | grep '192.168.10.0/24 accept'` (all 12).

### 5.3 — Bring up the etcd topology (3-member, full mTLS)

> **Step 5.3.1 — Render `etcd.conf.yml` on all 3, start in parallel, verify**
> **WHERE:** `vitess-etcd-1/2/3`, root shell.
> **WHY:** Raft needs the 3 members started close together for the first
> `initial-cluster-state: new` bootstrap. `client-cert-auth: true` makes the cert the
> authorization. Each node listens/advertises on its **VMnet10** IP (matching the cert
> IP-SAN — avoids the hostname self-dial trap).
> **WHAT (on each etcd node — set `SELF`/`NAME` per node):**
> ```bash
> NAME=vitess-etcd-1 ; SELF=192.168.10.190 ; CLIENT11=192.168.70.190   # per node
> cat > /etc/nexus-etcd/etcd.conf.yml <<EOF
> name: $NAME
> data-dir: /var/lib/nexus-etcd
> listen-peer-urls: https://$SELF:2380
> listen-client-urls: https://$SELF:2379,https://$CLIENT11:2379,https://127.0.0.1:2379
> initial-advertise-peer-urls: https://$SELF:2380
> advertise-client-urls: https://$SELF:2379
> initial-cluster: vitess-etcd-1=https://192.168.10.190:2380,vitess-etcd-2=https://192.168.10.191:2380,vitess-etcd-3=https://192.168.10.192:2380
> initial-cluster-token: nexus-vitess-etcd
> initial-cluster-state: new
> peer-transport-security:   { cert-file: /etc/nexus-etcd/tls/server-cert.pem, key-file: /etc/nexus-etcd/tls/server-key.pem, trusted-ca-file: /etc/nexus-etcd/tls/ca.pem, client-cert-auth: true }
> client-transport-security: { cert-file: /etc/nexus-etcd/tls/server-cert.pem, key-file: /etc/nexus-etcd/tls/server-key.pem, trusted-ca-file: /etc/nexus-etcd/tls/ca.pem, client-cert-auth: true }
> auto-compaction-mode: periodic
> auto-compaction-retention: "1"
> EOF
> chown root:etcd /etc/nexus-etcd/etcd.conf.yml ; chmod 0640 /etc/nexus-etcd/etcd.conf.yml
> printf 'https://192.168.10.190:2379,https://192.168.10.191:2379,https://192.168.10.192:2379' > /etc/nexus-etcd/endpoints
> systemctl daemon-reload ; systemctl enable nexus-etcd
> ```
> Then **start all 3 close together**: `systemctl reset-failed nexus-etcd; systemctl start nexus-etcd` on each.
> **EXPECTED:** a leader is elected within a minute.
> **VERIFY:** `nexus-etcdctl endpoint status --write-out=table`; `nexus-etcdctl put /nexus/x ok && nexus-etcdctl get /nexus/x` round-trips.

### 5.4 — vtctld + the cell (must precede tablets)

> **Step 5.4.1 — Render `vtctld.env` + the `nexus-vtctldclient` wrapper; start; AddCellInfo**
> **WHERE:** `vitess-control-1` (`.193`), root shell.
> **WHY:** a tablet can't register until the **cell** exists, so vtctld comes up first.
> ⚠️ the wrapper must use `vtctldclient`'s **client** flags `--vtctld-grpc-{ca,cert,key,server-name}`
> (not the server-side `--grpc-*`, T5); and vtctld needs the **tablet-manager** client
> certs so it can mTLS-dial tablets for reparent (T6).
> **WHAT:**
> ```bash
> CFG=/etc/nexus-vitess ; TLS=$CFG/tls
> ETCD='https://192.168.10.190:2379,https://192.168.10.191:2379,https://192.168.10.192:2379'
> cat > $CFG/vtctld.env <<EOF
> VTCTLD_FLAGS=--alsologtostderr --topo-implementation etcd2 --topo-global-server-address $ETCD --topo-global-root /vitess/global \
>   --topo-etcd-tls-ca $TLS/ca.pem --topo-etcd-tls-cert $TLS/server-cert.pem --topo-etcd-tls-key $TLS/server-key.pem \
>   --port 15000 --grpc-port 15999 --service-map grpc-vtctl,grpc-vtctld \
>   --grpc-cert $TLS/server-cert.pem --grpc-key $TLS/server-key.pem --grpc-ca $TLS/ca.pem \
>   --tablet-manager-grpc-ca $TLS/ca.pem --tablet-manager-grpc-cert $TLS/server-cert.pem --tablet-manager-grpc-key $TLS/server-key.pem
> EOF
> chown root:vitess $CFG/vtctld.env ; chmod 0640 $CFG/vtctld.env
> cat > /usr/local/sbin/nexus-vtctldclient <<'EOS'
> #!/bin/bash
> exec /usr/local/bin/vtctldclient --server 192.168.10.193:15999 \
>   --vtctld-grpc-ca /etc/nexus-vitess/tls/ca.pem \
>   --vtctld-grpc-cert /etc/nexus-vitess/tls/server-cert.pem \
>   --vtctld-grpc-key /etc/nexus-vitess/tls/server-key.pem \
>   --vtctld-grpc-server-name vitess-control-1.vitess.nexus.lab "$@"
> EOS
> chmod 0755 /usr/local/sbin/nexus-vtctldclient
> systemctl daemon-reload ; systemctl enable --now nexus-vtctld
> sleep 5
> nexus-vtctldclient AddCellInfo --root /vitess/nexus --server-address "$ETCD" nexus
> ```
> **EXPECTED:** vtctld answers gRPC; cell `nexus` created (idempotent — "already exists" is fine).
> **VERIFY:** `nexus-vtctldclient GetCellInfoNames` → lists `nexus`.

### 5.5 — Tablets: Percona + vttablet (all 6)

> **Step 5.5.1 — Render `init_db.sql` + `ssl.cnf` + the env files; start mysqlctld + vttablet**
> **WHERE:** each tablet node (`.196–.201`), root shell. Set `UID`/`SHARD`/`SELF` per
> node (shard `-80` → uid `100/101/102`; shard `80-` → uid `200/201/202`).
> **WHY:** `init_db.sql` creates the Vitess `vt_*` user set. ⚠️ Vitess's generated
> my.cnf starts mysqld **super-read-only**, so `init_db.sql` must `SET GLOBAL
> super_read_only='OFF'` first or the user DDL aborts with errno 1290 (T7); and the
> `vt_*` users are `IDENTIFIED WITH mysql_native_password` + `mysql_native_password=ON`
> in `ssl.cnf` (8.4 disables it by default — else `caching_sha2` RSA failures on the
> `vt_repl` TCP path, T7). vttablet advertises its **VMnet10** host
> (`--tablet-hostname`) — **don't** set `--db-host 127.0.0.1` (replicas would
> self-replicate, T10).
> **WHAT (per tablet — the load-bearing pieces; passwords from `vault kv get`):**
> ```bash
> CFG=/etc/nexus-vitess ; TLS=$CFG/tls ; DATAROOT=/var/lib/nexus-vitess
> UID=100 ; SHARD='-80' ; SELF=192.168.10.196 ; CELL=nexus ; KS=commerce   # per node
> UIDPAD=$(printf '%010d' $UID)
> ETCD='https://192.168.10.190:2379,https://192.168.10.191:2379,https://192.168.10.192:2379'
> ROOT=$(vault kv get -field=value nexus/vitess/mysql-root-password)
> APP=$(vault kv get -field=value nexus/vitess/mysql-app-password)
> ALL=$(vault kv get -field=value nexus/vitess/mysql-allprivs-password)
> REPL=$(vault kv get -field=value nexus/vitess/mysql-repl-password)
> ORC=$(vault kv get -field=value nexus/vitess/vtorc-topo-password)
> # init_db.sql -- the vt_* users (super_read_only OFF first; mysql_native_password)
> cat > $CFG/init_db.sql <<SQL
> SET sql_log_bin = 0;
> SET GLOBAL super_read_only = 'OFF';
> SET GLOBAL read_only = 'OFF';
> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '$ROOT';
> CREATE USER IF NOT EXISTS 'vt_dba'@'localhost'      IDENTIFIED WITH mysql_native_password BY '$ALL';  GRANT ALL ON *.* TO 'vt_dba'@'localhost' WITH GRANT OPTION;
> CREATE USER IF NOT EXISTS 'vt_app'@'localhost'      IDENTIFIED WITH mysql_native_password BY '$APP';  GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP,INDEX,ALTER,LOCK TABLES,EXECUTE,CREATE VIEW,SHOW VIEW,REPLICATION CLIENT ON *.* TO 'vt_app'@'localhost';
> CREATE USER IF NOT EXISTS 'vt_allprivs'@'localhost' IDENTIFIED WITH mysql_native_password BY '$ALL';  GRANT ALL ON *.* TO 'vt_allprivs'@'localhost';
> CREATE USER IF NOT EXISTS 'vt_repl'@'%'             IDENTIFIED WITH mysql_native_password BY '$REPL'; GRANT REPLICATION SLAVE ON *.* TO 'vt_repl'@'%';
> CREATE USER IF NOT EXISTS 'vt_filtered'@'localhost' IDENTIFIED WITH mysql_native_password BY '$ALL';  GRANT ALL ON *.* TO 'vt_filtered'@'localhost';
> CREATE USER IF NOT EXISTS 'vt_orc'@'%'              IDENTIFIED WITH mysql_native_password BY '$ORC';  GRANT SUPER,PROCESS,REPLICATION SLAVE,REPLICATION CLIENT,RELOAD,SELECT ON *.* TO 'vt_orc'@'%';
> FLUSH PRIVILEGES;
> RESET BINARY LOGS AND GTIDS;
> SQL
> chown root:vitess $CFG/init_db.sql ; chmod 0640 $CFG/init_db.sql
> # ssl.cnf -- mysqld wire TLS + re-enable mysql_native_password
> cat > $CFG/ssl.cnf <<SSL
> [mysqld]
> ssl-ca=$TLS/ca.pem
> ssl-cert=$TLS/server-cert.pem
> ssl-key=$TLS/server-key.pem
> mysql_native_password=ON
> SSL
> chown root:vitess $CFG/ssl.cnf ; chmod 0640 $CFG/ssl.cnf
> # mysqlctld.env (--db-dba-user/-password so it can health-check across restarts, T12)
> cat > $CFG/mysqlctld.env <<ENV
> VTDATAROOT=$DATAROOT
> EXTRA_MY_CNF=$CFG/ssl.cnf
> MYSQLCTLD_FLAGS=--alsologtostderr --tablet-uid=$UID --mysql-port=3306 --db-charset=utf8mb4 --init-db-sql-file=$CFG/init_db.sql --socket-file=$DATAROOT/mysqlctl.sock --db-dba-user vt_dba --db-dba-password $ALL
> ENV
> # vttablet.env (alias/keyspace/shard/topo mTLS/gRPC mTLS/db creds; NO --db-host)
> cat > $CFG/vttablet.env <<ENV
> VTDATAROOT=$DATAROOT
> VTTABLET_FLAGS=--alsologtostderr --tablet-path $CELL-$UID --init-keyspace $KS --init-shard $SHARD --init-tablet-type replica \
>   --tablet-hostname $SELF --topo-implementation etcd2 --topo-global-server-address $ETCD --topo-global-root /vitess/global \
>   --topo-etcd-tls-ca $TLS/ca.pem --topo-etcd-tls-cert $TLS/server-cert.pem --topo-etcd-tls-key $TLS/server-key.pem \
>   --port 15101 --grpc-port 16101 --service-map grpc-queryservice,grpc-tabletmanager,grpc-updatestream \
>   --db-socket $DATAROOT/vt_$UIDPAD/mysql.sock --db-app-user vt_app --db-app-password $APP --db-dba-user vt_dba --db-dba-password $ALL \
>   --db-allprivs-user vt_allprivs --db-allprivs-password $ALL --db-repl-user vt_repl --db-repl-password $REPL --db-filtered-user vt_filtered --db-filtered-password $ALL \
>   --db-ssl-ca $TLS/ca.pem --db-ssl-cert $TLS/server-cert.pem --db-ssl-key $TLS/server-key.pem \
>   --grpc-cert $TLS/server-cert.pem --grpc-key $TLS/server-key.pem --grpc-ca $TLS/ca.pem \
>   --tablet-manager-grpc-ca $TLS/ca.pem --tablet-manager-grpc-cert $TLS/server-cert.pem --tablet-manager-grpc-key $TLS/server-key.pem --restore-from-backup=false
> ENV
> chown root:vitess $CFG/{mysqlctld.env,vttablet.env} ; chmod 0640 $CFG/{mysqlctld.env,vttablet.env}
> mkdir -p $DATAROOT ; chown vitess:vitess $DATAROOT
> systemctl daemon-reload ; systemctl enable nexus-mysqlctld nexus-vttablet
> systemctl restart nexus-mysqlctld   # mysqld comes up + runs init_db.sql
> ```
> **EXPECTED:** mysqld accepts connections (probe accepts **"Access denied"** as up —
> the server responded, T8), then vttablet registers the tablet in the topo.
> **WHAT (after mysqld is up):** `systemctl restart nexus-vttablet`
> **VERIFY:** `mysqladmin --socket=$DATAROOT/vt_$UIDPAD/mysql.sock ping` → up;
> `curl -fsS http://127.0.0.1:15101/debug/vars | head -c1` → `{`; on the control node
> `nexus-vtctldclient GetTablets --keyspace commerce` lists all 6 (REPLICA, no primary yet).

### 5.6 — Reparent: elect each shard's initial PRIMARY

> **Step 5.6.1 — Set durability + PlannedReparentShard per shard**
> **WHERE:** `vitess-control-1`, root shell (via the wrapper).
> **WHY:** the tablets came up as REPLICA with no primary; elect one per shard. The
> initial PRIMARY = the lowest-uid tablet (`nexus-100` / `nexus-200`).
> **WHAT:**
> ```bash
> nexus-vtctldclient SetKeyspaceDurabilityPolicy --durability-policy=none commerce
> nexus-vtctldclient PlannedReparentShard commerce/-80 --new-primary nexus-100
> nexus-vtctldclient PlannedReparentShard commerce/80- --new-primary nexus-200
> ```
> **EXPECTED:** each shard converges to 1 PRIMARY + 2 REPLICA.
> **VERIFY:** `nexus-vtctldclient GetTablets --keyspace commerce --shard -80` → one
> `primary` + two `replica` (and the same for `80-`).

### 5.7 — vtgate routers + VTOrc

> **Step 5.7.1 — Render `vtgate.env` + the static-auth creds; start both vtgates**
> **WHERE:** `vitess-vtgate-1/2` (`.194/.195`), root shell.
> **WHY:** vtgate's MySQL listener (`:15306`) uses **static auth** (user `nexus` /
> the app password) + **mTLS** (server cert; clients present a cert). It routes to
> tablets over mTLS gRPC.
> **WHAT (each vtgate):**
> ```bash
> CFG=/etc/nexus-vitess ; TLS=$CFG/tls
> ETCD='https://192.168.10.190:2379,https://192.168.10.191:2379,https://192.168.10.192:2379'
> APP=$(vault kv get -field=value nexus/vitess/mysql-app-password)
> printf '{ "nexus": [ { "Password": "%s", "UserData": "nexus" } ] }' "$APP" > $CFG/vtgate_creds.json
> chown root:vitess $CFG/vtgate_creds.json ; chmod 0640 $CFG/vtgate_creds.json
> cat > $CFG/vtgate.env <<EOF
> VTGATE_FLAGS=--alsologtostderr --cell nexus --cells-to-watch nexus --topo-implementation etcd2 --topo-global-server-address $ETCD --topo-global-root /vitess/global \
>   --topo-etcd-tls-ca $TLS/ca.pem --topo-etcd-tls-cert $TLS/server-cert.pem --topo-etcd-tls-key $TLS/server-key.pem \
>   --port 15001 --grpc-port 15991 --mysql-server-port 15306 --mysql-server-bind-address 0.0.0.0 --mysql-server-version 8.4.0-Percona-Server \
>   --mysql-auth-server-impl static --mysql-auth-server-static-file $CFG/vtgate_creds.json \
>   --mysql-server-ssl-cert $TLS/server-cert.pem --mysql-server-ssl-key $TLS/server-key.pem --mysql-server-ssl-ca $TLS/ca.pem \
>   --service-map grpc-vtgateservice --grpc-cert $TLS/server-cert.pem --grpc-key $TLS/server-key.pem --grpc-ca $TLS/ca.pem \
>   --tablet-grpc-ca $TLS/ca.pem --tablet-grpc-cert $TLS/server-cert.pem --tablet-grpc-key $TLS/server-key.pem --tablet-types-to-wait PRIMARY,REPLICA
> EOF
> chown root:vitess $CFG/vtgate.env ; chmod 0640 $CFG/vtgate.env
> systemctl daemon-reload ; systemctl enable --now nexus-vtgate
> ```
> **VERIFY:** `systemctl is-active nexus-vtgate`; `curl -fsS http://127.0.0.1:15001/debug/vars | head -c1` → `{`; `:15306` listening.

> **Step 5.7.2 — Start VTOrc (control node)**
> **WHERE:** `vitess-control-1`, root shell.
> **WHY:** the auto-reparent watcher for keyspace `commerce` (via tablet-manager gRPC mTLS).
> **WHAT:**
> ```bash
> CFG=/etc/nexus-vitess ; TLS=$CFG/tls
> ETCD='https://192.168.10.190:2379,https://192.168.10.191:2379,https://192.168.10.192:2379'
> cat > $CFG/vtorc.env <<EOF
> VTORC_FLAGS=--alsologtostderr --topo-implementation etcd2 --topo-global-server-address $ETCD --topo-global-root /vitess/global \
>   --topo-etcd-tls-ca $TLS/ca.pem --topo-etcd-tls-cert $TLS/server-cert.pem --topo-etcd-tls-key $TLS/server-key.pem \
>   --port 16000 --clusters-to-watch commerce \
>   --tablet-manager-grpc-ca $TLS/ca.pem --tablet-manager-grpc-cert $TLS/server-cert.pem --tablet-manager-grpc-key $TLS/server-key.pem
> EOF
> chown root:vitess $CFG/vtorc.env ; chmod 0640 $CFG/vtorc.env
> systemctl daemon-reload ; systemctl enable --now nexus-vtorc
> ```
> **VERIFY:** `systemctl is-active nexus-vtorc`; `curl -fsS http://127.0.0.1:16000/debug/health`.

### 5.8 — Schema + vschema + the sharding proof (exit gate)

> **Step 5.8.1 — ApplySchema + ApplyVSchema; seed via vtgate; confirm both shards**
> **WHERE:** schema on `vitess-control-1`; seed on a tablet node (it has the `mysql` client).
> **WHY:** create the `customer` table on both shards, mark `commerce` sharded with a
> **hash vindex** on `customer_id`, then INSERT 100 rows **through vtgate** — the hash
> vindex splits them across both shards. ⚠️ vtgate's MySQL listener requires a **client
> cert** (mTLS); present the node's leaf (sudo — the key is `0640 root:vitess`, T9).
> **WHAT (on the control node — DDL):**
> ```bash
> cat > /tmp/customer.sql <<'SQL'
> CREATE TABLE IF NOT EXISTS customer (customer_id BIGINT NOT NULL, email VARCHAR(128), PRIMARY KEY (customer_id)) ENGINE=InnoDB;
> SQL
> cat > /tmp/vschema.json <<'JSON'
> { "sharded": true, "vindexes": { "hash": { "type": "hash" } },
>   "tables": { "customer": { "column_vindexes": [ { "column": "customer_id", "name": "hash" } ] } } }
> JSON
> nexus-vtctldclient ApplySchema --sql-file /tmp/customer.sql commerce
> nexus-vtctldclient ApplyVSchema --vschema-file /tmp/vschema.json commerce
> ```
> **WHAT (on a tablet node — seed via vtgate `.194` + per-shard count):**
> ```bash
> CFG=/etc/nexus-vitess ; TLS=$CFG/tls ; APP=$(sudo cat $CFG/mysql-app-password)
> MYSQL="sudo mysql --host=192.168.70.194 --port=15306 --user=nexus --password=$APP --ssl-mode=REQUIRED --ssl-cert=$TLS/server-cert.pem --ssl-key=$TLS/server-key.pem --ssl-ca=$TLS/ca.pem --batch --skip-column-names"
> for i in $(seq 1 100); do echo "INSERT IGNORE INTO customer(customer_id,email) VALUES($i,'user$i@nexus.lab');"; done | $MYSQL commerce
> echo "shard -80: $($MYSQL commerce/-80 -e 'SELECT COUNT(*) FROM customer')"
> echo "shard 80-: $($MYSQL commerce/80- -e 'SELECT COUNT(*) FROM customer')"
> ```
> **EXPECTED:** **both** shard counts > 0 (e.g. ~53 / ~47), summing to 100.
> **VERIFY:** that's the sharding proof — a single INSERT stream physically split
> across both shards by the hash vindex. **➡ The sharded MySQL cluster is live.**

---

## 6. Validation — by-hand acceptance smoke (demo / playbook)

Condensed from `smoke-0.O.ps1`. Per-node SSH probes from the **build host**.

- **Input:** the 12 nodes up; etcd quorum; each shard 1P+2R; vtgate routing; schema applied.
- **Where observed:** SSH / `nexus-etcdctl` / `nexus-vtctldclient` on control / `mysql`
  via vtgate / web UIs (vtctld `:15000`, VTOrc `:16000`, vtgate `:15001`).
- **Proves:** a sharded MySQL cluster with full mTLS + auto-failover.
- **Prerequisites:** Guides 00–04 alive; §5 complete.

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | 12 nodes reachable | `ssh …@190..201 'echo ok'` | all `ok` |
| 2 | etcd quorum | `nexus-etcdctl endpoint status` (each etcd) | a leader; healthy |
| 3 | Cell exists | `nexus-vtctldclient GetCellInfoNames` | `nexus` |
| 4 | Per-shard 1P+2R | `GetTablets --keyspace commerce --shard -80` / `80-` | one `primary` + two `replica` each |
| 5 | vtctld + VTOrc | `systemctl is-active nexus-vtctld nexus-vtorc` | `active` |
| 6 | vtgate routing | `mysql -h vtgate… :15306 -e 'SELECT 1'` (mTLS) | `1` |
| 7 | **Sharding proof** | per-shard `SELECT COUNT(*) FROM customer` | both shards > 0 |
| 8 | mTLS | etcd `client-cert-auth`; vtgate listener requires client cert | a no-cert connect fails |
| 9 | **VTOrc auto-reparent** (chaos) | kill the PRIMARY's mysqld (`nexus-100`); VTOrc promotes a replica | a new PRIMARY within ~15–30 s; cluster still writable |

**1–8 green ⇒ Guide 21 satisfied.** 9 is the HA payoff — VTOrc promotes a replica when
a shard PRIMARY dies, with no manual intervention.

---

## 7. Teardown / reset

```bash
for ip in 194 195; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-vtgate'; done
ssh nexusadmin@192.168.70.193 'sudo systemctl disable --now nexus-vtorc nexus-vtctld'
for ip in 196 197 198 199 200 201; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-vttablet nexus-mysqlctld; sudo rm -rf /var/lib/nexus-vitess/vt_*'; done
for ip in 190 191 192; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-etcd'; done
# gateway: rm /etc/dnsmasq.d/vitess-records.conf /etc/dnsmasq-vitess.hosts ; systemctl reload dnsmasq
# then vmrun stop + deleteVM each of the 12 (Guide 00 §7).
```

> **The data lives in each tablet's mysqld datadir** (`$VTDATAROOT/vt_<uid>/`). A fresh
> apply re-clones + re-inits empty tablets. The Vault PKI role + KV creds survive
> (sticky). To wipe: delete the VMs (the datadirs go with them).

---

## 8. Troubleshooting (the 0.O 15-transient gauntlet — representative)

| # | Symptom | Cause | Fix |
|---|---|---|---|
| **B1** | tablet build `Group mysql does not exist` | the `vitess` user was added to `mysql` before Percona created that group | add `vitess` to `mysql` **after** the Percona install (§5.1.3). |
| **B2** | Vitess tarball unpack `No space left on device` | Debian 13 mounts `/tmp` as tmpfs (~½ RAM); the ~1.5 GB extract overflows | download + extract under `/var/tmp` (§5.1.2/5.1.3). |
| **T3** | etcd firstboot `chown: invalid group 'root:vitess'` | the etcd node had only the `etcd` group; firstboot chowns identity to `vitess` | create **both** `etcd` + `vitess` groups (§5.1.1). |
| **T5** | `vtctldclient`: `unknown flag: --grpc-ca` | the wrapper used server-side `--grpc-*` flags | use the **client** flags `--vtctld-grpc-{ca,cert,key,server-name}` (§5.4.1). |
| **T6** | reparent `error reading server preface: EOF` | vtctld dials tablets' tablet-manager over mTLS but had no client cert | add `--tablet-manager-grpc-{ca,cert,key}` to vtctld (§5.4.1). |
| **T7** | tablets have **no `vt_*` users** / later `unsupported auth method` | `init_db.sql`'s first write hit `super_read_only` (errno 1290) → aborted; and 8.4 disables `mysql_native_password` | `init_db.sql` sets `super_read_only=OFF` first; users `IDENTIFIED WITH mysql_native_password` + `mysql_native_password=ON` in `ssl.cnf` (§5.5.1). |
| **T8** | tablet overlay hangs on mysqld-readiness | `mysqladmin ping` returns "Access denied" once root has a password | accept **"Access denied"** as up (the server responded) (§5.5.1). |
| **T9** | seed to vtgate `Lost connection … reading authorization packet` | the vtgate MySQL listener requires a client cert (mTLS); the client presented none | connect with `--ssl-cert/--ssl-key/--ssl-ca` (+ `sudo` for the `0640` key) (§5.8.1). |
| **T10** | seed write times out; replica IO `equal server ids` / `Source_Host 127.0.0.1` | `--db-host 127.0.0.1` made Vitess advertise 127.0.0.1 → replicas self-replicate; `semi_sync` blocked writes | drop `--db-host`/`--db-port` (advertise the VMnet10 host via `--tablet-hostname`); durability `none` (§5.5.1/5.6.1). |
| **T12** | mysqld looks unstable across restarts | mysqlctld couldn't health-check mysqld (vt_dba retry loop) | give mysqlctld `--db-dba-user/-password` (§5.5.1). |
| **—** | Backplane down (etcd/gRPC can't connect) | VMware left nic1 NO-CARRIER at power-on | reconnect the 2nd NIC + `systemctl restart systemd-networkd` (as Guides 16–20). |

---

### Cross-references

- **0.O architecture:** memory `project_nexus_infra_0o_phase`; ADR-0041 (Vitess
  topology); the `nexus-infra-vitess` handbook §3 (the B1–B3/T0–T12 gauntlet)
- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (vitess `.190–.201`,
  MAC `:CB–:D6`); `vms.yaml` (tier `07-vitess`)
- **Automated equivalents:** `nexus-infra-vitess/packer/vitess-{etcd,gate,tablet}-node/`
  + `terraform/envs/vitess/role-overlay-vitess-{etcd-bootstrap,gate,tablets,reparent,schema,tls}.tf`
- **Smoke mirror:** `nexus-infra-vitess/scripts/smoke-0.O.ps1`
- **Sibling sharding tier:** [`22-sharding-citus-postgresql.md`](./22-sharding-citus-postgresql.md) (Citus = the PostgreSQL sharding axis)
- **Contrast:** [`10-oltp-patroni-postgresql-ha.md`](./10-oltp-patroni-postgresql-ha.md) (PG HA by replication, not sharding)
- **Transients:** [[feedback_cluster_template_nftables_backplane]] · [[terraform-heredoc-powershell]]
- **Previous guide:** [`20-observability-grafana-lgtm.md`](./20-observability-grafana-lgtm.md)
- **Next guide:** Guide 22 — Sharding · Citus (PostgreSQL). See [`INDEX.md`](../INDEX.md). **(The final guide.)**
