# Guide 16 — Lakehouse · MinIO (4-node distributed erasure-coded object store)

> **Mirrors:** `nexus-infra-lakehouse` — the `lakehouse-minio-node` Packer template
> + the `lakehouse-minio` Terraform env overlays (`…-nftables-backplane`,
> `…-vault-agents`, `…-tls`, `…-config`, `…-bucket-bootstrap`) — Phase 0.L.1 /
> ADR-0033. The **first lakehouse-tier guide** and the foundation of the whole
> data-lake layer.

> 🔓 **This guide UNBLOCKS Guide 15.** StarRocks shared-data (Guide 15) stores
> every tablet in MinIO and is blocked on the `starrocks` bucket + S3 creds this
> guide creates. It is also the object backend for **Iceberg/Nessie (Guide 17)**,
> **Spark history (Guide 18)**, **Harbor blobs (Guide 19)**, and **Loki/Tempo
> (Guide 20)**. Build MinIO first; everything downstream that needs S3 depends on
> it. (If you followed the INDEX top-to-bottom and skipped Guide 15's forward dep,
> come back to Guide 15 now — it's ready once this guide is green.)

---

## 1. Overview & purpose

A genuinely **distributed** object store: **4 nodes** (`minio-1..4` at `.141–.144`),
each with a **dedicated data drive**, presenting one S3-compatible namespace that is
**erasure-coded across the whole set**. Lose a node (or a drive) and reads + writes
keep working; the missing shards are reconstructed from parity.

- **4 nodes, one erasure set.** Each node runs the same `minio server` process and
  is handed the *same* `MINIO_VOLUMES` list — `https://192.168.10.{141...144}:9000/mnt/minio/data`
  — so all four agree on the topology and form a single distributed pool. MinIO
  splits every object into data + parity shards and spreads them across the 4
  drives (default `EC:2` at this set size → survives the loss of any single node).
- **Two planes.** Inter-node **erasure / heal / distributed-lock** traffic rides
  the isolated **VMnet10 backplane** (`https://192.168.10.x:9000`); **client S3 +
  Console** ride **VMnet11** (`minio.nexus.lab:9000` / `minio-N.nexus.lab:9001`).
  The per-node TLS cert covers both.
- **Front door = round-robin DNS** `minio.nexus.lab` → the 4 node IPs. **No VIP**
  (ADR-0031): any node can serve any request, so DNS round-robin spreads the load
  and survives a node loss without a floating address.
- **mTLS everywhere.** Each node gets a Vault-PKI leaf; peers validate each other
  over the backplane against the same CA chain.

**Why object storage at all:** the lakehouse pattern separates **storage** (cheap,
durable, S3) from **compute** (StarRocks CN, Spark, Trino). MinIO is the durable
floor every compute engine reads/writes through. **Why distributed + erasure-coded**
rather than a single-node bucket: durability and availability — the *point* of the
tier is that a node can die and the data survives.

---

## 2. Component primer

- **MinIO.** A single-binary, S3-API-compatible object store (written in Go, no
  JVM, no external DB). *Why here:* it's the de-facto self-hosted S3 — every tool
  downstream (StarRocks, Iceberg, Spark, Harbor, Loki, Tempo) speaks S3, so one
  MinIO backs them all. *Otherwise:* **Ceph RADOS Gateway** (far heavier — needs a
  full Ceph cluster), **SeaweedFS**, or a cloud **AWS S3 / GCS** bucket (defeats
  the on-prem-lab goal). MinIO gives real S3 semantics with a 100 MB binary.
- **Distributed erasure coding.** MinIO shards each object into *k* data + *m*
  parity blocks (Reed-Solomon) striped across the drives in the set; any *k* blocks
  rebuild the object. *Why:* survives drive/node loss at a fraction of the disk
  cost of full replication (N-way copies). *vs. replication:* 3× replication wastes
  200% to tolerate 1 loss; erasure coding tolerates the same with far less
  overhead. *vs. RAID:* erasure coding is **distributed** across nodes, not a
  single box — a whole node can die. At this 4-drive set MinIO defaults to `EC:2`
  (2 parity), so the pool stays **read-write with 1 node down** and read-only logic
  kicks in only past quorum.
- **`mc` (MinIO Client).** The admin/ops CLI (`mc mb`, `mc admin info`, `mc admin
  user add`, `mc cp/cat/ls`). *Why:* bucket + user + policy management and health
  checks by hand. *Otherwise:* the AWS CLI (data-plane only — no `mc admin`).
- **Dedicated data disk.** Each node attaches a **second VMDK** (100 GB, xfs, label
  `minio-data`, mounted `/mnt/minio/data`) so erasure coding runs on a dedicated
  drive, not the root filesystem. *Why a second disk:* MinIO wants a clean drive
  per node; mixing it with `/` invites ENOSPC and noisy-neighbour I/O. **This is
  the source of the build's one structural gotcha** — see §8 (the Debian preseed
  multi-disk stall).
- **Round-robin DNS (no VIP).** `minio.nexus.lab` resolves to all 4 node IPs;
  clients pick one per lookup. *Why not a VIP/keepalived (like Patroni/PXC):*
  MinIO is symmetric — every node serves the full namespace, so there's no
  "leader" to float an address to. DNS round-robin is simpler and load-spreads.
  *Otherwise:* an HAProxy/keepalived VIP (unnecessary overhead here).
- **`MINIO_SERVER_URL`.** Tells MinIO its public address (`https://minio.nexus.lab:9000`)
  so presigned URLs + the Console point clients at the round-robin name, not a raw
  node IP.

---

## 3. Prerequisites

The 6-VM foundation must be alive, plus the 4 MinIO nodes created + baselined.

| # | Requirement | One-command verify |
|---|---|---|
| 1 | **Foundation alive** — gateway + DC + Vault HA (Guides 00–04) | `vault status` on `vault-1` → `Sealed: false`; `dig @192.168.70.1 nexus.lab` answers |
| 2 | **Vault PKI** usable (intermediate CA signs leaves) — Guide 04 | `vault read pki_int/cert/ca` on `vault-1` returns the intermediate |
| 3 | **CA bundle** on the build host (`~/.nexus/vault-ca-bundle.crt`) — Guide 04 | `Test-Path ~/.nexus/vault-ca-bundle.crt` → `True` |
| 4 | **4 `deb13` nodes** created per Guide 00 **each with a 2nd 100 GB data VMDK**, baselined, dual-NIC static `.141–.144` / `.10.141–.144` | the 4 answer `:22`; `findmnt /mnt/minio/data` → xfs (after §5.1) |
| 5 | **MinIO + mc binaries** reachable (dl.min.io, or a local cache) | `curl -sI https://dl.min.io/server/minio/release/linux-amd64/minio \| head -1` → `200` |

> **MinIO** = the official single-binary stable release. Identity dir
> `/etc/nexus-minio`; certs `/etc/nexus-minio/certs`; data `/mnt/minio/data`
> (xfs). PKI role **`minio-server`**. Front door: round-robin DNS
> `minio.nexus.lab` → `.141–.144` (no VIP). Vault KV creds under
> `nexus/lakehouse/minio/{root-user,root-password,app-access-key,app-secret-key}`.

> **By-hand divergence:** the automated path renders certs + reads KV via a
> per-node Vault Agent (`role-overlay-minio-{vault-agents,tls,config}.tf`). The
> manual path issues each leaf with `vault write pki_int/issue/minio-server`
> directly and reads KV with `vault kv get` from your operator session — no Vault
> Agent. The on-disk result (`public.crt` / `private.key` / `CAs/nexus-ca.crt` +
> `minio.conf`) is byte-identical.

---

## 4. Target topology

| Node | Role | VMnet11 | VMnet10 | MAC (primary / secondary) | vCPU/RAM | Disks | Ports |
|---|---|---|---|---|---|---|---|
| `minio-1` | erasure node (mc alias host) | `.141` | `.10.141` | `…3F:00:9A` / `…3F:01:9A` | 2 / 2 GB | 20 GB OS + **100 GB data** | 9000 S3 · 9001 Console · 9100 |
| `minio-2` | erasure node | `.142` | `.10.142` | `…3F:00:9B` / `…3F:01:9B` | 2 / 2 GB | 20 GB OS + **100 GB data** | 9000 · 9001 · 9100 |
| `minio-3` | erasure node | `.143` | `.10.143` | `…3F:00:9C` / `…3F:01:9C` | 2 / 2 GB | 20 GB OS + **100 GB data** | 9000 · 9001 · 9100 |
| `minio-4` | erasure node | `.144` | `.10.144` | `…3F:00:9D` / `…3F:01:9D` | 2 / 2 GB | 20 GB OS + **100 GB data** | 9000 · 9001 · 9100 |

> MAC block `:9A–:9D` (contiguous, right after StarRocks `:98`). Erasure/heal/lock
> traffic on the **VMnet10 backplane** (`MINIO_VOLUMES` uses the `.10.x` IPs);
> client S3 + Console on **VMnet11**. VMs live under
> `H:\VMS\NexusPlatform\08-spark\minio-N\` (the lakehouse tier shares the
> `08-spark` folder + the `.14x` decade across MinIO/Spark/Iceberg). Default RAM
> 2 GB ([[feedback_prefer_less_memory]]); MinIO is light at lab data volumes.

---

## 5. Step-by-step build

> **WHERE:** node steps as `nexusadmin` → `sudo -i` (root). `vault` runs on
> **`vault-1`** (or any host with `VAULT_ADDR`/`VAULT_TOKEN` + the CA). The 4
> nodes are symmetric — only IP / MAC / hostname differ — so the per-node values
> are spelled out in full where they matter (certs, identity), and the identical
> install steps run once **per node** (all 4 enumerated, no shorthand).

### 5.0 — Seed the MinIO secrets in Vault KV (once)

> **Step 5.0.1 — Write the root + app credentials to Vault KV**
> **WHERE:** `vault-1` (`.121`), root shell with an operator `VAULT_TOKEN`.
> **WHY:** the root creds (the MinIO superuser) and the least-priv `lakehouse-app`
> service account (consumed later by Spark + Iceberg + Harbor) must exist in KV
> **before** the nodes render `minio.conf`. Each node reads its root creds from KV
> so the secret never lives in a Packer image or transits the build host in the
> clear. Generate strong random values in-step.
> **WHAT:**
> ```bash
> export VAULT_ADDR=https://127.0.0.1:8200 VAULT_CACERT=$HOME/.nexus/vault-ca-bundle.crt
> vault kv put nexus/lakehouse/minio/root-user      value="nexus-minio-root"
> vault kv put nexus/lakehouse/minio/root-password  value="$(openssl rand -base64 24)"
> vault kv put nexus/lakehouse/minio/app-access-key value="nexus-lakehouse-app"
> vault kv put nexus/lakehouse/minio/app-secret-key value="$(openssl rand -base64 24)"
> ```
> **EXPECTED:** 4 KV entries written (`version 1`).
> **VERIFY:** `vault kv get -field=value nexus/lakehouse/minio/root-user` → `nexus-minio-root`;
> the password/secret fields return non-empty (don't echo them to a shared terminal).

### 5.1 — Create the 4 VMs (two disks each) + format the data drive

> **Step 5.1.1 — Create each VM with a dedicated 2nd data disk, install, baseline**
> **WHERE:** VMware Workstation GUI on the build host, then each node as root.
> **WHY:** MinIO erasure coding wants a clean dedicated drive per node. Each VM
> gets the standard Guide 00 shape **plus a second 100 GB VMDK**. ⚠️ **The 2nd
> disk is the build's one structural trap** — a stock Debian netinst **stalls at
> partitioning** when a blank 2nd disk is present (it waits for a "which disk?"
> answer that never comes). You **must** pin `/dev/sda` and zap the 2nd disk's
> label in the preseed *before* `partman` runs (§8, T1). Doing the install by hand
> from the netinst menu: at **"Partition disks"** explicitly select **`/dev/sda`**
> (the 20 GB OS disk) for *Guided – use entire disk*, and **leave `/dev/sdb`
> untouched** (do not let the installer auto-include it).
> **WHAT (per VM, in the GUI — repeat for minio-1, minio-2, minio-3, minio-4):**
> - New VM → Debian 13 base shape from Guide 00 §5.A: 2 vCPU, 2 GB RAM, **20 GB**
>   primary disk (SCSI `0:0` → `/dev/sda`), NIC0 = VMnet11, NIC1 = VMnet10.
> - **Add a second hard disk**: 100 GB, SCSI `0:1` → appears as **`/dev/sdb`**,
>   *Store as a single file*, **do not** pre-allocate (growable).
> - Pin the per-node MACs from §4 (primary `…3F:00:9A..9D`, secondary
>   `…3F:01:9A..9D`).
> - Install Debian 13 (Guide 00 §5.B), selecting **`/dev/sda` only** for
>   partitioning; baseline (hostname via firstboot, dual-NIC, SSH, nftables,
>   chrony, node_exporter).
> **EXPECTED:** each node boots, gets its `.141–.144` DHCP lease on nic0, and the
> static `.10.141–.144` on nic1. `lsblk` shows **`sda`** (20 G, partitioned) +
> **`sdb`** (100 G, raw, no partitions).
> **VERIFY:** `ssh nexusadmin@192.168.70.141 hostname` → `minio-1` (and `.142/.143/.144`
> → `minio-2/3/4`); `lsblk -o NAME,SIZE,TYPE` shows `sdb 100G disk` with no children.

> **Step 5.1.2 — Format + mount the data disk as xfs (label `minio-data`) — all 4**
> **WHERE:** each node (`.141`, `.142`, `.143`, `.144`), root shell.
> **WHY:** the erasure drive is xfs (MinIO's recommended FS), mounted by **LABEL**
> (`nofail`) so device reordering / full-clone copies stay robust. The installer
> never touched `/dev/sdb`, so it's a clean raw disk here.
> **WHAT (run on each of the 4 nodes):**
> ```bash
> apt-get install -y xfsprogs
> blkid -s TYPE -o value /dev/sdb || mkfs.xfs -L minio-data /dev/sdb
> mkdir -p /mnt/minio/data
> grep -q ' /mnt/minio/data ' /etc/fstab || \
>   echo 'LABEL=minio-data /mnt/minio/data xfs defaults,noatime,nofail 0 0' >> /etc/fstab
> mountpoint -q /mnt/minio/data || mount /mnt/minio/data
> ```
> **EXPECTED:** `/mnt/minio/data` is an xfs mountpoint on `/dev/sdb`.
> **VERIFY:** `findmnt -no FSTYPE /mnt/minio/data` → `xfs`; `blkid -L minio-data` → `/dev/sdb`.

### 5.2 — Install MinIO + mc; create the `minio` account (all 4)

> **Step 5.2.1 — Install the MinIO server + mc client binaries + the system account**
> **WHERE:** each node (`.141–.144`), root shell.
> **WHY:** MinIO + mc are single Go binaries (no JVM, no deps). The service runs as
> an unprivileged `minio` system user owning the data dir + identity dir.
> **WHAT (run on each of the 4 nodes):**
> ```bash
> # system account
> getent group minio  >/dev/null || groupadd --system minio
> getent passwd minio >/dev/null || useradd --system --gid minio --no-create-home \
>   --shell /usr/sbin/nologin --home-dir /var/lib/minio minio
> install -d -o minio -g minio -m 0750 /var/lib/minio /mnt/minio/data
>
> # binaries
> curl -fsSL https://dl.min.io/server/minio/release/linux-amd64/minio -o /usr/local/bin/minio
> curl -fsSL https://dl.min.io/client/mc/release/linux-amd64/mc       -o /usr/local/bin/mc
> chmod 0755 /usr/local/bin/minio /usr/local/bin/mc
>
> # per-node identity + cert dirs (root:minio 0750)
> install -d -o root -g minio -m 0750 /etc/nexus-minio /etc/nexus-minio/certs /etc/nexus-minio/certs/CAs
> ```
> **EXPECTED:** `/usr/local/bin/minio` + `/usr/local/bin/mc` executable; `minio`
> user exists; dirs created.
> **VERIFY:** `/usr/local/bin/minio --version` prints a `RELEASE.*` tag; `id minio`
> shows the system user.

> **Step 5.2.2 — Install the `nexus-minio.service` unit (DISABLED) — all 4**
> **WHERE:** each node, root shell.
> **WHY:** the unit is `Type=notify` (MinIO signals readiness) and reads its env
> from `/etc/nexus-minio/minio.conf` (rendered in §5.5). Install it **disabled** —
> all 4 nodes are enabled + started **together** in §5.5 so they form the erasure
> set in one shot (a node started alone waits for its peers).
> **WHAT (run on each of the 4 nodes):**
> ```bash
> cat > /etc/systemd/system/nexus-minio.service <<'EOF'
> [Unit]
> Description=Nexus MinIO (S3-compatible object storage, distributed erasure-coded)
> Documentation=https://min.io/docs/minio/linux/index.html
> After=network-online.target
> Wants=network-online.target
> AssertFileIsExecutable=/usr/local/bin/minio
> ConditionPathExists=/etc/nexus-minio/minio.conf
> [Service]
> Type=notify
> NotifyAccess=main
> WorkingDirectory=/var/lib/minio
> User=minio
> Group=minio
> EnvironmentFile=-/etc/nexus-minio/minio.conf
> ExecStartPre=/bin/bash -c "if [ -z \"${MINIO_VOLUMES}\" ]; then echo 'MINIO_VOLUMES not set'; exit 1; fi"
> ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES
> Restart=always
> RestartSec=5
> LimitNOFILE=1048576
> TasksMax=infinity
> TimeoutStopSec=30
> SendSIGKILL=no
> StandardOutput=journal
> StandardError=journal
> [Install]
> WantedBy=multi-user.target
> EOF
> systemctl daemon-reload
> systemctl disable nexus-minio.service 2>/dev/null || true
> ```
> **EXPECTED:** the unit is installed, parsed, and **disabled**.
> **VERIFY:** `systemctl is-enabled nexus-minio.service` → `disabled`;
> `systemctl cat nexus-minio.service` shows the `ExecStart` above.

### 5.3 — nftables: backplane trust + S3/Console on VMnet11 (all 4)

> **Step 5.3.1 — Apply the per-cluster ruleset**
> **WHERE:** each node (`.141–.144`), root shell.
> **WHY:** the distributed peer plane (erasure read/write, heal, distributed locks)
> rides VMnet10 — trust the **whole** backplane segment ([[feedback_cluster_template_nftables_backplane]]).
> Open S3 `9000` + Console `9001` on VMnet11 for clients. Apply atomically with
> `nft -f` ([[feedback_nftables_runtime_add_after_drop]]); a runtime `nft add` lands
> *after* the drop and is unreachable.
> **WHAT (run on each of the 4 nodes — single canonical ruleset):**
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
>         ip protocol icmp   accept
>         ip6 nexthdr icmpv6 accept
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 22   accept comment "SSH from VMnet11"
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 9100 accept comment "node_exporter from VMnet11"
>         iifname "nic1" ip saddr 192.168.10.0/24 accept comment "trusted cluster backplane (VMnet10)"
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport { 9000, 9001 } accept comment "MinIO S3 API + Console from VMnet11"
>         counter drop
>     }
>     chain forward { type filter hook forward priority 0; policy drop; }
>     chain output  { type filter hook output priority 0; policy accept; }
> }
> EOF
> nft -f /etc/nftables.conf
> systemctl enable nftables 2>/dev/null || true
> ```
> **EXPECTED:** the ruleset loads with no error.
> **VERIFY:** `nft list chain inet filter input | grep -E '192.168.10.0/24 accept'`
> (backplane trust present); `grep -E 'dport \{ 9000, 9001 \}'` (client ports open).

### 5.4 — Per-node mTLS certs from Vault PKI

> **Step 5.4.1 — Create the `minio-server` PKI role (once)**
> **WHERE:** `vault-1` (`.121`), root shell with `VAULT_TOKEN`.
> **WHY:** one role issues all 4 leaves. Each leaf's SANs must cover the node short
> name + FQDN, the round-robin `minio.nexus.lab`, and **IP SANs for both the
> VMnet11 service IP and the VMnet10 backplane IP** (peers connect by backplane IP,
> so the cert must validate against it). 90-day leaf TTL.
> **WHAT (on vault-1):**
> ```bash
> vault write pki_int/roles/minio-server \
>   allowed_domains='nexus.lab,minio.nexus.lab,minio-1,minio-2,minio-3,minio-4,localhost' \
>   allow_subdomains=true allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
> ```
> **EXPECTED:** `Success! Data written to: pki_int/roles/minio-server`.
> **VERIFY:** `vault read pki_int/roles/minio-server` shows the allowed domains + flags.

> **Step 5.4.2 — Issue + place `minio-1`'s cert**
> **WHERE:** issue on `vault-1`; place on `minio-1` (`.141`).
> **WHY:** MinIO reads `public.crt` (leaf **+** intermediate, the Go default server
> cert), `private.key` (**PKCS#8** — MinIO/Go rejects PKCS#1 `RSA PRIVATE KEY`), and
> `CAs/nexus-ca.crt` (the full chain so this node **trusts its 3 peers** in
> distributed mode). Also drop the chain at `/etc/ssl/certs/minio-ca.pem` for `mc`
> + `curl`.
> **WHAT (on vault-1 — issue):**
> ```bash
> vault write -format=json pki_int/issue/minio-server \
>   common_name=minio-1.nexus.lab \
>   alt_names='minio-1,minio-1.nexus.lab,minio.nexus.lab,localhost' \
>   ip_sans='192.168.10.141,192.168.70.141,127.0.0.1' ttl=2160h > /tmp/minio-1.json
> # also fetch the CA chain (intermediate + root)
> vault read -field=certificate pki_int/cert/ca_chain > /tmp/nexus-ca-chain.pem
> ```
> **WHAT (copy `/tmp/minio-1.json` + `/tmp/nexus-ca-chain.pem` to `minio-1`, then as root on `minio-1`):**
> ```bash
> jq -r '.data.certificate'      /tmp/minio-1.json > /tmp/leaf.crt
> jq -r '.data.issuing_ca'       /tmp/minio-1.json > /tmp/int.crt
> jq -r '.data.private_key'      /tmp/minio-1.json > /tmp/leaf.key
> # public.crt = leaf + intermediate ; key -> PKCS#8 ; CAs = full chain
> cat /tmp/leaf.crt /tmp/int.crt > /etc/nexus-minio/certs/public.crt
> openssl pkcs8 -topk8 -nocrypt -in /tmp/leaf.key -out /etc/nexus-minio/certs/private.key
> cp /tmp/nexus-ca-chain.pem /etc/nexus-minio/certs/CAs/nexus-ca.crt
> cp /tmp/nexus-ca-chain.pem /etc/ssl/certs/minio-ca.pem
> chown -R root:minio /etc/nexus-minio/certs
> chmod 0644 /etc/nexus-minio/certs/public.crt /etc/nexus-minio/certs/CAs/nexus-ca.crt
> chmod 0640 /etc/nexus-minio/certs/private.key
> rm -f /tmp/leaf.crt /tmp/int.crt /tmp/leaf.key /tmp/minio-1.json
> ```
> **EXPECTED:** the 3 cert files in `/etc/nexus-minio/certs/`.
> **VERIFY:** `openssl x509 -in /etc/nexus-minio/certs/public.crt -noout -subject -ext subjectAltName`
> → CN `minio-1.nexus.lab`, SAN includes `minio.nexus.lab` + IP `192.168.10.141`;
> `head -1 /etc/nexus-minio/certs/private.key` → `-----BEGIN PRIVATE KEY-----` (PKCS#8).

> **Step 5.4.3 — Issue + place `minio-2`'s cert**
> **WHERE:** issue on `vault-1`; place on `minio-2` (`.142`).
> **WHY:** same as 5.4.2 with `minio-2`'s names + IPs.
> **WHAT (on vault-1):**
> ```bash
> vault write -format=json pki_int/issue/minio-server \
>   common_name=minio-2.nexus.lab \
>   alt_names='minio-2,minio-2.nexus.lab,minio.nexus.lab,localhost' \
>   ip_sans='192.168.10.142,192.168.70.142,127.0.0.1' ttl=2160h > /tmp/minio-2.json
> ```
> Then place on `minio-2` exactly as 5.4.2 (split `public.crt`/`private.key`/`CAs`,
> PKCS#8 the key, copy the CA chain to `/etc/ssl/certs/minio-ca.pem`, fix
> ownership/perms).
> **EXPECTED / VERIFY:** as 5.4.2 — CN `minio-2.nexus.lab`, SAN IP `192.168.10.142`.

> **Step 5.4.4 — Issue + place `minio-3`'s cert**
> **WHERE:** issue on `vault-1`; place on `minio-3` (`.143`).
> **WHY:** same with `minio-3`'s names + IPs.
> **WHAT (on vault-1):**
> ```bash
> vault write -format=json pki_int/issue/minio-server \
>   common_name=minio-3.nexus.lab \
>   alt_names='minio-3,minio-3.nexus.lab,minio.nexus.lab,localhost' \
>   ip_sans='192.168.10.143,192.168.70.143,127.0.0.1' ttl=2160h > /tmp/minio-3.json
> ```
> Then place on `minio-3` as 5.4.2.
> **EXPECTED / VERIFY:** CN `minio-3.nexus.lab`, SAN IP `192.168.10.143`.

> **Step 5.4.5 — Issue + place `minio-4`'s cert**
> **WHERE:** issue on `vault-1`; place on `minio-4` (`.144`).
> **WHY:** same with `minio-4`'s names + IPs.
> **WHAT (on vault-1):**
> ```bash
> vault write -format=json pki_int/issue/minio-server \
>   common_name=minio-4.nexus.lab \
>   alt_names='minio-4,minio-4.nexus.lab,minio.nexus.lab,localhost' \
>   ip_sans='192.168.10.144,192.168.70.144,127.0.0.1' ttl=2160h > /tmp/minio-4.json
> ```
> Then place on `minio-4` as 5.4.2.
> **EXPECTED / VERIFY:** CN `minio-4.nexus.lab`, SAN IP `192.168.10.144`.

### 5.5 — Render `minio.conf` + start all 4 as one erasure set

> **Step 5.5.1 — Confirm the VMnet10 backplane is up on every node (before start)**
> **WHERE:** each node (`.141–.144`), root shell.
> **WHY:** MinIO identifies its **local** position in the distributed grid by
> matching a `MINIO_VOLUMES` backplane IP to a local interface. If nic1 has no
> backplane IP, MinIO can't place itself and the set won't form. (VMware
> occasionally leaves the 2nd NIC NO-CARRIER at power-on — §8, T2.)
> **WHAT (on each node):**
> ```bash
> ip -4 -o addr show nic1 | grep -q "192.168.10.${HOSTNAME##*-}\b" || \
>   { systemctl restart systemd-networkd; sleep 3; }
> ip -4 -o addr show nic1
> ```
> **EXPECTED:** nic1 carries `192.168.10.14X/24` (matching the node).
> **VERIFY:** `ping -c1 -W2 192.168.10.141` from `minio-2` succeeds (backplane reachable).

> **Step 5.5.2 — Render `/etc/nexus-minio/minio.conf` on every node**
> **WHERE:** each node (`.141–.144`), root shell with `VAULT_ADDR`/`VAULT_TOKEN` + CA.
> **WHY:** every node gets the **identical** `MINIO_VOLUMES` (the backplane glob
> across all 4) so they agree on the erasure set; each reads its own root creds
> from Vault KV (§5.0) so the secret never sits in a template. `MINIO_SERVER_URL`
> points clients at the round-robin name; `MINIO_OPTS` carries the certs dir.
> **WHAT (run on each of the 4 nodes — identical file):**
> ```bash
> export VAULT_ADDR=https://192.168.70.121:8200 VAULT_CACERT=/etc/ssl/certs/minio-ca.pem
> RU=$(vault kv get -field=value nexus/lakehouse/minio/root-user)
> RP=$(vault kv get -field=value nexus/lakehouse/minio/root-password)
> [ -n "$RU" ] && [ -n "$RP" ] || { echo "empty MinIO root creds from Vault KV"; exit 1; }
> cat > /etc/nexus-minio/minio.conf <<EOF
> # Rendered by hand (Guide 16) -- mirrors role-overlay-minio-config.tf.
> MINIO_VOLUMES="https://192.168.10.{141...144}:9000/mnt/minio/data"
> MINIO_OPTS="--address :9000 --console-address :9001 --certs-dir /etc/nexus-minio/certs"
> MINIO_ROOT_USER=$RU
> MINIO_ROOT_PASSWORD=$RP
> MINIO_SERVER_URL=https://minio.nexus.lab:9000
> MINIO_PROMETHEUS_AUTH_TYPE=public
> EOF
> chown root:minio /etc/nexus-minio/minio.conf
> chmod 0640 /etc/nexus-minio/minio.conf
> systemctl daemon-reload
> systemctl enable nexus-minio.service
> ```
> **EXPECTED:** `minio.conf` present (`0640 root:minio`) with `MINIO_VOLUMES` set,
> service enabled (not yet started).
> **VERIFY:** `grep -c '^MINIO_VOLUMES=' /etc/nexus-minio/minio.conf` → `1`;
> `systemctl is-enabled nexus-minio.service` → `enabled`.

> **Step 5.5.3 — Start MinIO on all 4 nodes together; wait for cluster health**
> **WHERE:** each node (start), then `minio-1` (health), root shell.
> **WHY:** the 4 peers wait for each other at start to form the distributed pool.
> Start them in quick succession, then poll the cluster-health endpoint (returns
> `200` only once quorum is reached + the erasure set is online).
> **WHAT (start on each node):**
> ```bash
> systemctl start nexus-minio.service
> ```
> **WHAT (poll from minio-1):**
> ```bash
> for i in $(seq 1 30); do
>   code=$(curl -fsS -k -o /dev/null -w '%{http_code}' https://localhost:9000/minio/health/cluster 2>/dev/null || true)
>   [ "$code" = "200" ] && { echo "cluster healthy"; break; }
>   sleep 10
> done
> ```
> **EXPECTED:** `cluster healthy` within a few minutes; `journalctl -u nexus-minio`
> on each node shows the API + Console listening with TLS.
> **VERIFY:** `systemctl is-active nexus-minio.service` → `active` on all 4;
> `curl -fsS -k -o /dev/null -w '%{http_code}' https://localhost:9000/minio/health/cluster` → `200`.

### 5.6 — Round-robin DNS `minio.nexus.lab` (gateway)

> **Step 5.6.1 — Publish `minio.nexus.lab` → the 4 node IPs**
> **WHERE:** `nexus-gateway` (`.70.1`), root shell.
> **WHY:** the no-VIP front door (ADR-0031) — one name, 4 A-records, round-robin.
> Use the `addn-hosts` hosts-file form (host-file round-robin, not `host-record`)
> per [[feedback_smoke_gate_probe_robustness]].
> **WHAT:**
> ```bash
> cat > /etc/dnsmasq-minio.hosts <<'EOF'
> 192.168.70.141 minio.nexus.lab
> 192.168.70.142 minio.nexus.lab
> 192.168.70.143 minio.nexus.lab
> 192.168.70.144 minio.nexus.lab
> EOF
> echo 'addn-hosts=/etc/dnsmasq-minio.hosts' > /etc/dnsmasq.d/minio-records.conf
> dnsmasq --test && systemctl reload dnsmasq
> ```
> **EXPECTED:** dnsmasq reloads clean.
> **VERIFY:** `dig @192.168.70.1 minio.nexus.lab +short` → the 4 IPs (order rotates
> per query).

### 5.7 — Bucket bootstrap (the exit gate)

> **Step 5.7.1 — mc alias + buckets + app user + erasure health + round-trip**
> **WHERE:** `minio-1` (`.141`), root shell.
> **WHY:** the deterministic exit gate — prove the cluster is **usable**, not just
> running: configure `mc` against the cluster (root creds from KV), create the
> warehouse + auxiliary buckets, create the least-priv `lakehouse-app` service
> account (the key Spark + Iceberg + Harbor will use), confirm ≥4 online drives,
> and do an object write/read round-trip.
> **WHAT (on minio-1):**
> ```bash
> export VAULT_ADDR=https://192.168.70.121:8200 VAULT_CACERT=/etc/ssl/certs/minio-ca.pem
> RU=$(vault kv get -field=value nexus/lakehouse/minio/root-user)
> RP=$(vault kv get -field=value nexus/lakehouse/minio/root-password)
> APP_AK=$(vault kv get -field=value nexus/lakehouse/minio/app-access-key)
> APP_SK=$(vault kv get -field=value nexus/lakehouse/minio/app-secret-key)
>
> # Trust the cluster CA for mc (mc config dir /root/.mc)
> install -d -m 0700 /root/.mc/certs/CAs
> cp /etc/ssl/certs/minio-ca.pem /root/.mc/certs/CAs/nexus-ca.crt
> mc alias set nexuslocal https://localhost:9000 "$RU" "$RP"
>
> # Erasure-set health: count online drives (expect >= 4)
> ONLINE=$(mc admin info nexuslocal --json | jq '[.info.servers[].drives[] | select(.state=="ok")] | length')
> echo "online drives: $ONLINE"; [ "$ONLINE" -ge 4 ] || { mc admin info nexuslocal; exit 1; }
>
> # Buckets (idempotent): warehouse (Iceberg), spark-events (Spark history), lakehouse (medallion root)
> for b in warehouse spark-events lakehouse; do mc mb --ignore-existing "nexuslocal/$b"; done
>
> # Least-priv app service account (consumed by Iceberg/Spark/Harbor)
> mc admin user info nexuslocal "$APP_AK" >/dev/null 2>&1 || mc admin user add nexuslocal "$APP_AK" "$APP_SK"
> mc admin policy attach nexuslocal readwrite --user "$APP_AK" 2>/dev/null || true
>
> # Object write/read round-trip on warehouse
> T=$(mktemp); echo "nexus-lakehouse-0.L.1-$(date -u +%FT%TZ)" > "$T"
> mc cp "$T" nexuslocal/warehouse/.nexus-bootstrap-probe
> mc cat nexuslocal/warehouse/.nexus-bootstrap-probe | grep -q 'nexus-lakehouse-0.L.1'
> mc rm nexuslocal/warehouse/.nexus-bootstrap-probe; rm -f "$T"
> echo BOOTSTRAP_OK
> ```
> **EXPECTED:** `online drives: 4` (or more), the 3 buckets created, the app user
> added + `readwrite` attached, round-trip prints `BOOTSTRAP_OK`.
> **VERIFY:** `mc ls nexuslocal` → `warehouse/ spark-events/ lakehouse/`;
> `mc admin user list nexuslocal` lists `nexus-lakehouse-app`.

> **Step 5.7.2 — (For Guide 15) create the StarRocks tenant bucket + creds**
> **WHERE:** `minio-1` (`.141`) + `vault-1`.
> **WHY:** Guide 15 (StarRocks shared-data) needs a **dedicated** tenant —
> bucket `starrocks` + a scoped service account (`s3:*` on the `starrocks` bucket
> only, ADR-0037), with its keys in Vault KV. This is the step that **unblocks
> Guide 15**.
> **WHAT (on vault-1 — seed the SR tenant creds):**
> ```bash
> vault kv put nexus/analytics/starrocks-sd/s3-access-key value="nexus-starrocks-app"
> vault kv put nexus/analytics/starrocks-sd/s3-secret-key value="$(openssl rand -base64 30)"
> ```
> **WHAT (on minio-1 — bucket + scoped policy + user):**
> ```bash
> SR_AK=$(vault kv get -field=value nexus/analytics/starrocks-sd/s3-access-key)
> SR_SK=$(vault kv get -field=value nexus/analytics/starrocks-sd/s3-secret-key)
> mc mb --ignore-existing nexuslocal/starrocks
> cat > /tmp/starrocks-tenant.json <<'EOF'
> { "Version": "2012-10-17", "Statement": [
>   { "Effect": "Allow", "Action": ["s3:*"],
>     "Resource": ["arn:aws:s3:::starrocks", "arn:aws:s3:::starrocks/*"] } ] }
> EOF
> mc admin policy create nexuslocal starrocks-tenant /tmp/starrocks-tenant.json 2>/dev/null || true
> mc admin user info nexuslocal "$SR_AK" >/dev/null 2>&1 || mc admin user add nexuslocal "$SR_AK" "$SR_SK"
> mc admin policy attach nexuslocal starrocks-tenant --user "$SR_AK" 2>/dev/null || true
> rm -f /tmp/starrocks-tenant.json
> ```
> **EXPECTED:** bucket `starrocks` + policy `starrocks-tenant` + user `nexus-starrocks-app`.
> **VERIFY:** `mc ls nexuslocal | grep starrocks`; cross-bucket deny holds — an
> `mc cp` to `nexuslocal/warehouse/` *as the SR key* must **fail** (scoped policy).
> **➡ Guide 15 is now unblocked.**

---

## 6. Validation — by-hand acceptance smoke (demo / playbook)

Condensed from `smoke-0.L.1.ps1`. Run the per-node SSH probes from the **build
host**; `mc` ops run on `minio-1` against the `nexuslocal` alias.

- **Input:** the 4 nodes up; `minio.nexus.lab` published; the exit gate (§5.7) green.
- **Where observed:** SSH to each node / `mc` on `minio-1` / `dig` on the gateway.
- **Proves:** a genuine distributed erasure-coded S3 store, mTLS, round-robin
  fronted, single-node-loss tolerant.
- **Prerequisites:** Guides 00–04 alive; §5 complete.

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | 4 nodes reachable | `ssh nexusadmin@.141..144 'echo ok'` | all `ok` |
| 2 | Data disk is xfs | `findmnt -no FSTYPE /mnt/minio/data` (each) | `xfs` ×4 |
| 3 | TLS material | `openssl x509 -in …/certs/public.crt -noout -ext subjectAltName` | SAN has `minio.nexus.lab` + backplane IP; key PKCS#8 |
| 4 | nftables backplane trust | `nft list chain inet filter input` (each) | `192.168.10.0/24 accept` present |
| 5 | Config rendered | `grep -c '^MINIO_VOLUMES=' /etc/nexus-minio/minio.conf` | `1` ×4 |
| 6 | Service active | `systemctl is-active nexus-minio` (each) | `active` ×4 |
| 7 | Cluster health (quorum) | `curl -k -w '%{http_code}' https://localhost:9000/minio/health/cluster` (each) | `200` ×4 |
| 8 | Erasure set ≥4 drives | `mc admin info nexuslocal --json \| jq '[…drives…select(state==ok)]\|length'` | `>= 4` |
| 9 | Buckets present | `mc ls nexuslocal` | `warehouse` + `spark-events` + `lakehouse` |
| 10 | App service account | `mc admin user list nexuslocal` | `nexus-lakehouse-app` |
| 11 | Round-robin DNS | `dig @192.168.70.1 minio.nexus.lab +short` | the 4 IPs |
| 12 | Object round-trip | `mc cp` + `mc cat` + `mc rm` on `warehouse` | content matches |
| 13 | **Node-loss tolerance** (chaos) | stop `nexus-minio` on `minio-4`; `mc cp/cat/rm` still succeeds; restart | write succeeds at 3/4 (write quorum) |

**1–12 green ⇒ Guide 16 satisfied** (and Guide 15 unblocked once §5.7.2 ran). 13 is
the erasure-coding payoff — the cluster stays **read-write with a node down**.

---

## 7. Teardown / reset

```bash
for ip in 141 142 143 144; do
  ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-minio.service; sudo rm -f /etc/nexus-minio/minio.conf'
done
# gateway: rm /etc/dnsmasq.d/minio-records.conf /etc/dnsmasq-minio.hosts ; systemctl reload dnsmasq
# then vmrun stop + deleteVM each of the 4 (Guide 00 §7).
```

> **The data lives on the dedicated `/dev/sdb` (xfs `minio-data`).** Deleting the
> VMs discards it. To keep an erasure set across a rebuild, preserve the 4 data
> VMDKs and re-point the new VMs at them — but the lab treats MinIO as
> rebuildable (downstream Iceberg/Spark/StarRocks re-create their objects). The
> gateway DNS record + the Vault KV creds belong to Guides 01/04.

---

## 8. Troubleshooting

| # | Symptom | Cause | Fix |
|---|---|---|---|
| **T1** | **Debian install hangs at "Partition disks"** (or by-hand netinst stalls / a Packer build times out waiting for SSH at ~30 min) | a **blank 2nd disk** (`/dev/sdb`) present — the installer waits for a "which disk?" answer that never comes ([[feedback_debian_preseed_multidisk_stall]]) | **pin `/dev/sda` + zap `/dev/sdb` before `partman`**: preseed `partman/early_command` = `debconf-set partman-auto/disk /dev/sda; … dd if=/dev/zero of=/dev/sdb bs=1M count=16 …`, plus `partman-auto/disk /dev/sda` + `grub-installer/bootdev /dev/sda`. By hand: select **`/dev/sda` only** for guided partitioning, leave `sdb` untouched. (Packer also bumps `boot_wait` 15→20s, `ssh_timeout` 30→45m on a loaded host.) |
| **T2** | MinIO won't form the set / a node logs "unable to find my IP" or peers unreachable | VMware left **ethernet1 (nic1) NO-CARRIER** at power-on, so the backplane IP is missing and MinIO can't place itself in `MINIO_VOLUMES` | bring nic1 up before start: in VMware reconnect the 2nd NIC (or `vmrun connectNamedDevice <vmx> ethernet1`), then `systemctl restart systemd-networkd`; confirm `ip -4 -o addr show nic1` carries `192.168.10.14X` (§5.5.1). |
| **T3** | MinIO refuses to start: "found a key encoded in PKCS#1" / TLS key error | the leaf key was placed as PKCS#1 (`RSA PRIVATE KEY`) — MinIO/Go wants **PKCS#8** | `openssl pkcs8 -topk8 -nocrypt -in leaf.key -out private.key` (§5.4.2); `head -1` must read `BEGIN PRIVATE KEY`. |
| **T4** | Peers reject each other's TLS over the backplane | `CAs/nexus-ca.crt` missing the full chain, or the leaf lacks the **backplane IP SAN** | place the **intermediate+root** chain at `/etc/nexus-minio/certs/CAs/nexus-ca.crt`; issue leaves with `ip_sans` including the `192.168.10.x` backplane IP (§5.4). |
| **T5** | `mc` / `curl` → `x509: certificate signed by unknown authority` | the client doesn't trust the Vault CA | drop the chain at `/etc/ssl/certs/minio-ca.pem` and `/root/.mc/certs/CAs/nexus-ca.crt` (§5.4.2 / §5.7.1); never use `mc --insecure` in the lab. |
| **T6** | `MINIO_VOLUMES not set` on start (ExecStartPre fails) | `minio.conf` absent or empty (KV read returned blank) | confirm §5.0 seeded KV; re-render §5.5.2 with a valid `VAULT_TOKEN`; check `grep MINIO_VOLUMES /etc/nexus-minio/minio.conf`. |
| **T7** | Cluster health never returns `200` | one node started long before its peers, or a drive failed to mount | start all 4 close together (§5.5.3); confirm `findmnt /mnt/minio/data` xfs on every node (§5.1.2); inspect `journalctl -u nexus-minio -n 40`. |
| **T8** | StarRocks (Guide 15) can't reach the bucket | §5.7.2 not run, wrong endpoint, or virtual-host addressing | run §5.7.2 (bucket + creds); Guide 15 uses `aws.s3.enable_path_style_access=true` + endpoint `https://minio.nexus.lab:9000`. |

---

## 9. Production tuning — MinIO (distributed erasure-coded)

> **Everything below is *beyond the lab replica*.** §5 ships the verbatim lab configs — 4
> nodes at 2 GB, one 100 GB data VMDK each, `minio.conf` with only `MINIO_VOLUMES`/`OPTS`/
> creds/`SERVER_URL` set (§5.5.2), everything else at the MinIO default. This section is what
> you would change for a **production** object store, and *why*; it never alters the §5
> values. The **OS-layer** knobs (swappiness, THP, I/O scheduler) live once in
> **[Guide 00 §9](./00-lab-host-and-base-vm.md#9-production-tuning--the-os-layer-feeds-every-linux-tier)** —
> only the MinIO-specific overrides are restated here.
>
> **There is no heap to size.** MinIO is a single self-contained **Go** binary (§5.2.1) — no
> JVM, no GC tuning, no `-Xmx`. It manages its own memory and leans hard on the OS page cache,
> so the tuning surface is (a) a few server env vars in `minio.conf`, (b) the OS/filesystem/
> network layer, and (c) the **erasure-set geometry**, which is an *architecture* decision, not
> a runtime knob. That makes this §9 deliberately short.

### 9.0 ⚠️ OS, filesystem & FD requirements

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `nofile` (open files) | **`1048576`** | **PRESENT** — `LimitNOFILE=1048576` in `nexus-minio.service` (§5.2.2) | **⚠️ required.** An erasure node keeps an FD open per object part it is reading/writing/healing plus every peer + client connection; at scale this reaches into the hundreds of thousands. The default `1024` yields `Too many open files` and failed reads. The lab already ships the high limit in the unit — **not** deferred. |
| XFS mount options | **`noatime`** (XFS is MinIO's required FS) | **PRESENT** — `xfs defaults,noatime,nofail` (§5.1.2) | XFS is what MinIO recommends for the data drive; `noatime` stops every read from issuing a metadata write (an atime update), which on an object store's many-small-file access pattern is pure wasted I/O. The lab already sets both — **not** deferred. |
| Transparent Huge Pages / `vm.swappiness` | THP `never`, `swappiness=1` | unset | THP compaction stalls and swapping hurt the page-cache-heavy read path; set fleet-wide in **[Guide 00 §9](./00-lab-host-and-base-vm.md#9-production-tuning--the-os-layer-feeds-every-linux-tier)**. |
| Dedicated data disk | one clean drive per node (no RAID under it) | **PRESENT** — dedicated 2nd VMDK, xfs (§5.1) | MinIO wants a raw dedicated drive per erasure set member and does its own redundancy — **do not** put it on a RAID volume (double-redundancy wastes capacity) or share it with `/`. The lab already gives each node a dedicated `/dev/sdb`. |

### 9.1 Erasure-set geometry — an **architecture** choice, not a tunable

`MINIO_STORAGE_CLASS_STANDARD=EC:N` sets the STANDARD class **parity** (`N` = parity shards per
object; the set tolerates the loss of `N` drives/nodes). **The catch on this topology:** the
erasure set is **fixed at 4 drives** (the 4 nodes, one drive each — see §2 / §5.5), so valid
parity is only `EC:0`–`EC:2`, and MinIO already defaults to **`EC:2`** here (survives 1 node
down read-write; the guide's §2 note spells this out). You **cannot** raise redundancy past
`EC:2` by editing a config value — you get there by **adding nodes/drives** so a larger set
supports `EC:3`/`EC:4`. That is a rebuild of the tier's topology (`vms.yaml` + more VMs), not a
knob. Treat the row below accordingly.

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `MINIO_STORAGE_CLASS_STANDARD` | **`EC:4`** on a ≥8-drive set for enterprise durability; **`EC:2` is the ceiling on this fixed 4-drive set** | unset (defaults to `EC:2`) | Parity = how many simultaneous drive/node losses the pool survives while staying readable. Higher parity = more durability, less usable capacity. On a 4-drive set EC:2 is both the default and the max; higher parity needs **more drives** (architecture change), not a config edit. |
| Erasure **set size** (# drives) | 8–16 per set for higher parity + parallelism | **4 (fixed)** — an architecture constraint (§2, §5.5) | The set size caps the achievable parity and the healing parallelism. Changing it means adding nodes/drives and re-forming the pool — out of scope for a config tune; see the §2 note that this is a design constraint of the 4-node lab. |
| `MINIO_STORAGE_CLASS_RRS` (reduced-redundancy class) | `EC:2` for non-critical objects to reclaim capacity | unset | Lets you mark less-important buckets/objects with lower parity to trade durability for usable space. Only meaningful once the set is large enough to offer a *range* of parity. |

### 9.2 Request concurrency & background scanner (`minio.conf`)

The lab sets neither — the defaults are auto-derived from RAM, which is right at lab volume but
worth pinning on a busy, larger node.

```bash
# PRODUCTION — append to /etc/nexus-minio/minio.conf on each node (re-render + restart to apply).
MINIO_API_REQUESTS_MAX=1600           # cap concurrent S3 requests (0 = auto from RAM)
MINIO_API_REQUESTS_DEADLINE=10s       # how long a request waits in the queue before 503
MINIO_SCANNER_SPEED=default           # background scanner pace: slowest|slow|default|fast|fastest
```

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `MINIO_API_REQUESTS_MAX` | **explicit** (e.g. `1600`) sized to node RAM/cores | unset (`0` = auto ≈ RAM-derived) | Bounds concurrent S3 requests so a client stampede can't exhaust memory/FDs; excess requests queue rather than OOM the node. Auto-sizing off a small lab RAM figure would set an artificially low cap in production. |
| `MINIO_API_REQUESTS_DEADLINE` | **`10s`** (raise for large-object/slow clients) | unset (`10s`) | How long a queued request waits before MinIO returns `503 SlowDown` — back-pressure that sheds load instead of piling latency. Too short → spurious 503s under normal bursts. |
| `MINIO_SCANNER_SPEED` | **`default`**; `slow`/`slowest` on I/O-constrained nodes, `fast` when heal/ILM must catch up | unset (`default`) | Paces the background scanner that drives **healing**, lifecycle (ILM), and usage accounting. Faster clears a heal backlog sooner but competes with client I/O; slower protects foreground latency at the cost of slower self-healing. |

### 9.3 Backplane network — jumbo frames for erasure & heal traffic

Erasure write/read spreads shards across all 4 nodes and **healing streams whole objects
between peers**, so the VMnet10 backplane (§4) carries heavy east-west traffic. On a real 10 GbE
backplane, **jumbo frames (MTU 9000)** cut per-packet overhead materially for these large
transfers — but MTU must be raised **end to end** (every node's `nic1` **and** the switch/VMnet10
segment), or path-MTU mismatch causes silent fragmentation/black-holing.

```bash
# PRODUCTION — on each node, set nic1 (VMnet10 backplane) to MTU 9000 via systemd-networkd.
# Requires the switch / VMnet10 to also carry 9000 end-to-end, or connectivity breaks.
mkdir -p /etc/systemd/network
cat >> /etc/systemd/network/20-nic1.network <<'EOF'
[Link]
MTUBytes=9000
EOF
systemctl restart systemd-networkd
# VERIFY: ip link show nic1 | grep -o 'mtu 9000'  ; and a large ping must NOT fragment:
#   ping -M do -s 8972 -c1 192.168.10.142   (from minio-1 -> minio-2)
```

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| Backplane MTU (`nic1`, VMnet10) | **`9000`** (jumbo), set end-to-end | unset (`1500`) | Erasure/heal moves large object streams between nodes; jumbo frames reduce packet count and CPU per byte on the backplane. **Must** match on every node + the switch or PMTU mismatch black-holes traffic — an all-or-nothing change. |

> **Where these build on the OS layer:** a production MinIO node wants the Guide 00 §9 base —
> THP `never`, `vm.swappiness=1`, and the `mq-deadline`/`none` I/O scheduler on the data drive —
> on top of the XFS+`noatime`+high-`nofile`+dedicated-disk facts the lab already ships (§9.0).
> There is no engine heap to size; MinIO's performance is the page cache + the disks + the
> backplane, so tune those.

---

### Cross-references

- **0.L.1 architecture:** memory `project_nexus_infra_lakehouse_phase`; ADR-0033
  (MinIO distributed erasure-coded), ADR-0031 (round-robin DNS, no VIP)
- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (MinIO `.141–.144`,
  MAC block `:9A–:9D`); `vms.yaml` (`cluster: minio`)
- **Automated equivalents:** `nexus-infra-lakehouse/packer/lakehouse-minio-node/` +
  `terraform/envs/lakehouse-minio/role-overlay-minio-*.tf`
- **Smoke mirror:** `nexus-infra-lakehouse/scripts/smoke-0.L.1.ps1`
- **Unblocks:** [`15-analytics-starrocks-shared-data.md`](./15-analytics-starrocks-shared-data.md)
  (StarRocks shared-data — the forward dep) + downstream Iceberg/Spark/Harbor/Loki/Tempo
- **Transients:** [[feedback_debian_preseed_multidisk_stall]] · [[feedback_cluster_template_nftables_backplane]] · [[feedback_nftables_runtime_add_after_drop]]
- **Previous guide:** [`15-analytics-starrocks-shared-data.md`](./15-analytics-starrocks-shared-data.md)
- **Next guide:** Guide 17 — Lakehouse · Iceberg / Nessie (REST catalog + PG HA). See [`INDEX.md`](../INDEX.md).
