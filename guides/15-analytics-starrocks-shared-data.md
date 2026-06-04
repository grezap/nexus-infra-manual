# Guide 15 — Analytics · StarRocks shared-data (3 FE + 2 CN over MinIO)

> **Mirrors:** `nexus-infra-analytics` — the `analytics-starrocks-sd` env overlays
> (`starrocks-sd-fe-bootstrap`, `starrocks-sd-cn-join`,
> `starrocks-sd-storage-volume`, `starrocks-sd-schema-bootstrap`, `…-tls`,
> `…-nftables-backplane`) — Phase 0.L.5 / ADR-0037. A **delta** on Guide 14: the
> FE quorum + install are the same; what changes is `run_mode = shared_data`, the
> **MinIO storage volume**, and **stateless CN** in place of local-storage BE.

> ⚠️ **Ordering / forward dependency:** shared-data StarRocks stores **all table
> data in MinIO object storage**. **Build Guide 16 (MinIO) first** — this guide
> needs the `starrocks` bucket + the S3 access/secret keys in Vault KV, and
> `minio.nexus.lab:9000` reachable + TLS-trusted. If you're following the INDEX
> top-to-bottom, do 16 before 15 (the INDEX order is build-dependency for *most*
> tiers; this is the one cross-tier exception).

---

## 1. Overview & purpose

A second StarRocks cluster — this one in **shared-data** (cloud-native) mode:
the compute layer is **stateless** and **all durable data lives in MinIO**
(`s3://starrocks/`). **5 nodes:**

- **Frontends (`sr-sd-fe-1/2/3`, `.37/.38/.39`)** — same BDB-JE metadata quorum
  as Guide 14 (1 leader + 2 followers), but configured with **`run_mode =
  shared_data`**. The FE owns the metadata + the **storage-volume** binding to
  MinIO.
- **Compute nodes (`sr-sd-cn-1/2`, `.30/.40`)** — **CN** replaces BE. A CN is a
  *stateless* query executor: it reads/writes tablets to MinIO and keeps only a
  **local cache** (not durable storage). You can add/remove CN freely for
  elastic compute without rebalancing data.
- **Object storage** — MinIO (Guide 16) holds every tablet. Durability +
  replication are the object store's job (erasure coding), so shared-data tables
  **don't** carry `replication_num` the way shared-nothing tables do.
- **Front door** = round-robin DNS `starrocks-sd-fe.nexus.lab` → the 3 FE.

**Why a second StarRocks** (alongside Guide 14's shared-nothing): to demonstrate
the **disaggregated compute/storage** model (ADR-0037) — the modern cloud-native
analytics shape, where compute scales independently of storage. It's a *separate
parallel cluster*, not a replacement.

**Dependency:**
- **Guide 16 (MinIO)** — the `starrocks` bucket + S3 creds in Vault KV +
  `minio.nexus.lab:9000` TLS-reachable. **(The forward dep — build it first.)**
- **Guides 00 + 04** — foundation alive; Vault PKI + KV.
- 5 `deb13` nodes baselined per Guide 00 §5.B, dual-NIC static.

> **By-hand divergence:** same as Guide 14 — issue certs with the `vault` CLI,
> run `mysql`/`start_fe.sh` directly, no Vault Agent. The big shared-data-specific
> by-hand step is importing the Vault CA into the FE's **JDK cacerts** + the CN's
> system trust store (so the AWS SDK validates MinIO's TLS).

---

## 2. Component primer

- **Shared-data (cloud-native) mode.** `run_mode = shared_data` makes StarRocks
  store tablets in **object storage** (S3/MinIO) instead of on the BE's local
  disks. *Why:* compute (CN) and storage (MinIO) scale independently; CN are
  disposable. *vs. shared-nothing (Guide 14):* there, BE held data locally with
  `replication_num=3`; here, durability is MinIO's erasure coding. *Otherwise:*
  shared-nothing (Guide 14) — the classic MPP shape.
- **CN (Compute Node) vs. BE.** A **CN** is a *stateless* executor — it pulls
  tablets from MinIO, computes, and caches locally (the `storage_root_path` is a
  **cache**, not durable). Registered with `ALTER SYSTEM ADD COMPUTE NODE`
  (not `ADD BACKEND`); listed by `SHOW COMPUTE NODES`. *Why:* add/remove compute
  without moving data.
- **Storage volume.** A first-class StarRocks object that binds the cluster to an
  S3 location + credentials: `CREATE STORAGE VOLUME … TYPE=S3 LOCATIONS=('s3://
  starrocks/') PROPERTIES(endpoint, access/secret key, path-style)`. Set as the
  **default** so new tables land there. *Why path-style:* MinIO needs
  `aws.s3.enable_path_style_access = true` (vs. AWS's virtual-host style).
- **`cloud_native_meta_port`.** An extra FE port (`6090`) used in shared-data
  mode for cloud-native metadata coordination. (FE keeps its usual `9030`/`8030`/
  `9020`/`9010`.)
- **The TLS-to-MinIO trust step (the shared-data-specific gotcha).** MinIO serves
  HTTPS with a Vault-PKI cert. The FE's **AWS Java SDK** validates it against the
  **JDK truststore** (`java-21 cacerts`), and the CN's **aws-sdk-cpp** validates
  it against the **system trust store**. So you must import the Vault CA into
  *both* — into `cacerts` via `keytool` on each FE (then restart FE), and via
  `update-ca-certificates` on each CN — or `CREATE STORAGE VOLUME` / data I/O
  fails TLS validation.
- **Inherited from Guide 14.** Same install (JDK 21 + the StarRocks 3.5.17 tarball
  from the download portal in `/var/tmp`), the same FE `--helper` join mechanics,
  `priority_networks` on the backplane, `mysql --skip-ssl`, round-robin DNS.

---

## 3. Prerequisites

| # | Requirement | One-command verify |
|---|---|---|
| 1 | **Guide 16 (MinIO) built**; `starrocks` bucket + S3 creds in Vault KV | `vault kv get nexus/analytics/starrocks-sd/s3-access-key`; `mc ls nexus/starrocks` (or the bucket exists) |
| 2 | `minio.nexus.lab:9000` reachable + Vault-CA-signed TLS | `ssh …@37 'curl -sI https://minio.nexus.lab:9000/minio/health/live --cacert /etc/vault-agent/ca-bundle.crt \| head -1'` → `200` |
| 3 | Foundation alive (Guides 00 + 04); Vault PKI/KV | `vault read pki_int/cert/ca` on vault-1 |
| 4 | 5 `deb13` nodes baselined, dual-NIC static `.37/.38/.39` + `.30/.40` | those 5 answer `:22` |
| 5 | StarRocks 3.5.17 tarball reachable (download-portal API) | `curl -fsSL 'https://releases.starrocks.io/api/v2/download?version=3.5.17&os=ubuntu&arch=amd64' \| jq -r '.fileUrl'` returns a URL (§5.1.1 uses it) |

> StarRocks **3.5.17**, **JDK 21**, `run_mode = shared_data`. Front door:
> round-robin DNS `starrocks-sd-fe.nexus.lab` → `.37/.38/.39`. Storage volume
> `nexus_minio_starrocks` → `s3://starrocks/` at `https://minio.nexus.lab:9000`.

---

## 4. Target topology

| Node | Role | VMnet11 | VMnet10 | RAM |
|---|---|---|---|---|
| `sr-sd-fe-1` | FE leader (shared_data) | `.37` | `.10.37` | 4 GB |
| `sr-sd-fe-2` | FE follower | `.38` | `.10.38` | 4 GB |
| `sr-sd-fe-3` | FE follower | `.39` | `.10.39` | 4 GB |
| `sr-sd-cn-1` | Compute Node (stateless) | `.30` | `.10.30` | 6 GB |
| `sr-sd-cn-2` | Compute Node (stateless) | `.40` | `.10.40` | 6 GB |

> The CN IPs `.30`/`.40` are a documented decade-spill (the SR `.3x` decade was
> nearly full). FE ports add `cloud_native_meta_port 6090` to Guide 14's set;
> CN heartbeat `9050`, RPC `9060`. Durable data: MinIO `s3://starrocks/`.

---

## 5. Step-by-step build

> **WHERE:** node steps as `nexusadmin` → `sudo -i` (root). `mysql` against an FE
> always uses **`--skip-ssl`** (MariaDB 11.8 client requires TLS for a password,
> which the FE doesn't offer — S5). `vault` runs on **`vault-1`**. Every step is
> written out in full below — you do **not** need to open Guide 14. The 5 nodes are
> `sr-sd-fe-1/2/3` (`.37/.38/.39`) + `sr-sd-cn-1/2` (`.30/.40`).

### 5.1 — Base install + FE quorum (fully self-contained)

> **Step 5.1.1 — Install JDK 21 + the StarRocks 3.5.17 tarball (all 5 nodes)**
> **WHERE:** **each of the 5 nodes** (`.37`, `.38`, `.39`, `.30`, `.40`), root shell.
> **WHY:** StarRocks needs JDK 21 (Debian 13 ships no openjdk-17 — S2). Download the
> tarball from the **download-portal `fileUrl`** (the old public CDN 403s — S1) into
> **`/var/tmp`** (the root `/tmp` is a small tmpfs that ENOSPCs on the 2.2 GB tarball
> — S3). The one tarball holds both the FE binaries (`fe/`) and the BE/CN binaries
> (`be/`). Run this identical block on all 5.
> **WHAT (on each of the 5 nodes):**
> ```bash
> apt-get update -qq && apt-get install -y openjdk-21-jdk curl openssl jq
> # JAVA_HOME (the FE needs it; bake it into the machine environment)
> echo 'JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64' >> /etc/environment
> # Get the StarRocks 3.5.17 download URL from the portal API, then download to /var/tmp
> # (NOT /tmp -- tmpfs ENOSPC). The portal returns a JSON with a 'fileUrl' field:
> URL=$(curl -fsSL 'https://releases.starrocks.io/api/v2/download?version=3.5.17&os=ubuntu&arch=amd64' | jq -r '.fileUrl')
> cd /var/tmp
> curl -fSL "$URL" -o starrocks.tar.gz
> tar xzf starrocks.tar.gz && rm starrocks.tar.gz
> mv StarRocks-3.5.17-ubuntu-amd64 /opt/starrocks            # contains fe/ + be/
> # create the unprivileged service user that owns the install + runs the daemons
> id starrocks >/dev/null 2>&1 || useradd --system -m -d /opt/starrocks -s /usr/sbin/nologin starrocks
> chown -R starrocks:starrocks /opt/starrocks
> ```
> **EXPECTED:** `/opt/starrocks/fe` + `/opt/starrocks/be` exist.
> **VERIFY:** `java -version` → `21`; `ls /opt/starrocks/fe/bin/start_fe.sh /opt/starrocks/be/bin/start_cn.sh` (both exist).

> **Step 5.1.2 — Install the systemd units (FE unit on `.37/.38/.39`; CN unit on `.30/.40`)**
> **WHERE:** the 3 FE nodes get the FE unit; the 2 CN nodes get the CN unit. Root shell.
> **WHY:** wrap `start_fe.sh`/`start_cn.sh --daemon` in systemd, run as `starrocks`
> with `JAVA_HOME` set. (Shared-data uses **CN**, the stateless compute node — its
> `start_cn.sh` lives in `/opt/starrocks/be/bin/`, the same dir as a shared-nothing BE.)
> **WHAT (on each FE node `.37/.38/.39` — the FE unit):**
> ```bash
> cat > /etc/systemd/system/nexus-starrocks-sd-fe.service <<'EOF'
> [Unit]
> Description=Nexus StarRocks FE (shared-data)
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
> **WHAT (on each CN node `.30/.40` — the CN unit):**
> ```bash
> cat > /etc/systemd/system/nexus-starrocks-sd-cn.service <<'EOF'
> [Unit]
> Description=Nexus StarRocks CN (shared-data compute node)
> After=network-online.target
> Wants=network-online.target
> [Service]
> Type=forking
> User=starrocks
> Group=starrocks
> ExecStart=/opt/starrocks/be/bin/start_cn.sh --daemon
> ExecStop=/opt/starrocks/be/bin/stop_cn.sh
> Restart=on-failure
> LimitNOFILE=655350
> [Install]
> WantedBy=multi-user.target
> EOF
> systemctl daemon-reload
> ```
> **VERIFY:** `systemctl cat nexus-starrocks-sd-fe.service` (FE nodes) /
> `nexus-starrocks-sd-cn.service` (CN nodes) shows the right `ExecStart`. (Don't
> start anything yet — config comes first.)

> **Step 5.1.3 — nftables: trust the backplane + open the FE/CN ports (all 5 nodes)**
> **WHERE:** **each of the 5 nodes**, root shell.
> **WHY:** FE↔FE (BDB-JE) + FE↔CN traffic rides the VMnet10 backplane (trust the
> whole segment); the MySQL query port `:9030` + FE HTTP `:8030` are reachable on
> VMnet11. Apply the full ruleset atomically with `nft -f`.
> **WHAT (on each of the 5 nodes):**
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
>         iifname "nic1" ip saddr 192.168.10.0/24 accept comment "trusted cluster backplane (VMnet10) -- FE BDB-JE + FE<->CN"
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport { 9030, 8030, 8040 } accept comment "FE MySQL/HTTP + CN web (VMnet11)"
>         counter drop
>     }
>     chain forward { type filter hook forward priority 0; policy drop; }
>     chain output  { type filter hook output priority 0; policy accept; }
> }
> EOF
> nft -f /etc/nftables.conf ; systemctl enable nftables 2>/dev/null || true
> ```
> Also install the MySQL client somewhere you'll run queries (the build host or any
> node): `apt-get install -y mariadb-client`.
> **VERIFY:** `nft list chain inet filter input | grep '192.168.10.0/24 accept'` (all 5).

> **Step 5.1.4 — Per-node PKI certs from Vault (all 5 nodes)**
> **WHERE:** issue on `vault-1`; place on each node, root shell.
> **WHY:** per-node certs for the FE/CN HTTP endpoints. (Cluster-internal FE/CN
> security here is the trusted backplane + nftables — StarRocks' internal protocol
> isn't full mTLS, and the MySQL endpoint uses `--skip-ssl`. The certs that *are*
> load-bearing are MinIO's, trusted in §5.3.1.) Create one PKI role, then issue +
> split a leaf per node into `/opt/starrocks/<fe|be>/conf/tls/`.
> **WHAT (once, on `vault-1`):**
> ```bash
> export VAULT_ADDR=https://127.0.0.1:8200 VAULT_CACERT=$HOME/.nexus/vault-ca-bundle.crt
> vault write pki_int/roles/starrocks-sd-server \
>   allowed_domains='nexus.lab,starrocks-sd-fe.nexus.lab,sr-sd-fe-1,sr-sd-fe-2,sr-sd-fe-3,sr-sd-cn-1,sr-sd-cn-2,localhost' \
>   allow_subdomains=true allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
> ```
> Per-node values (FE nodes add the round-robin name `starrocks-sd-fe.nexus.lab` to
> the SANs; CN nodes don't):
>
> | Node | VMnet11 | CN | `ip_sans` | cert dir |
> |---|---|---|---|---|
> | `sr-sd-fe-1` | `.37` | `sr-sd-fe-1.nexus.lab` | `192.168.10.37,192.168.70.37,127.0.0.1` | `/opt/starrocks/fe/conf/tls` |
> | `sr-sd-fe-2` | `.38` | `sr-sd-fe-2.nexus.lab` | `192.168.10.38,192.168.70.38,127.0.0.1` | `/opt/starrocks/fe/conf/tls` |
> | `sr-sd-fe-3` | `.39` | `sr-sd-fe-3.nexus.lab` | `192.168.10.39,192.168.70.39,127.0.0.1` | `/opt/starrocks/fe/conf/tls` |
> | `sr-sd-cn-1` | `.30` | `sr-sd-cn-1.nexus.lab` | `192.168.10.30,192.168.70.30,127.0.0.1` | `/opt/starrocks/be/conf/tls` |
> | `sr-sd-cn-2` | `.40` | `sr-sd-cn-2.nexus.lab` | `192.168.10.40,192.168.70.40,127.0.0.1` | `/opt/starrocks/be/conf/tls` |
>
> **WHAT (issue on `vault-1` — example `sr-sd-fe-1`; FE rows add the RR name):**
> ```bash
> vault write -format=json pki_int/issue/starrocks-sd-server \
>   common_name=sr-sd-fe-1.nexus.lab \
>   alt_names='sr-sd-fe-1,sr-sd-fe-1.nexus.lab,starrocks-sd-fe.nexus.lab,localhost' \
>   ip_sans='192.168.10.37,192.168.70.37,127.0.0.1' ttl=2160h > /tmp/sr-sd-fe-1.json
> # a CN row instead: alt_names='sr-sd-cn-1,sr-sd-cn-1.nexus.lab,localhost' (no RR name)
> vault read -field=certificate pki_int/cert/ca_chain > /tmp/nexus-ca-chain.pem
> ```
> **WHAT (place on each node — set `D` from the table's cert dir):**
> ```bash
> # copy /tmp/<host>.json + /tmp/nexus-ca-chain.pem to the node, then as root:
> D=/opt/starrocks/fe/conf/tls          # CN nodes: D=/opt/starrocks/be/conf/tls
> install -d -o starrocks -g starrocks -m0750 "$D"
> jq -r '.data.certificate' /tmp/<host>.json > /tmp/leaf.crt
> jq -r '.data.issuing_ca'  /tmp/<host>.json > /tmp/int.crt
> jq -r '.data.private_key' /tmp/<host>.json > /tmp/leaf.key
> cat /tmp/leaf.crt /tmp/int.crt > "$D/server.crt"
> openssl pkcs8 -topk8 -nocrypt -in /tmp/leaf.key -out "$D/server.key"
> cp /tmp/nexus-ca-chain.pem "$D/ca.crt"
> chown -R starrocks:starrocks "$D" ; chmod 0644 "$D/server.crt" "$D/ca.crt" ; chmod 0640 "$D/server.key"
> rm -f /tmp/leaf.crt /tmp/int.crt /tmp/leaf.key /tmp/<host>.json
> ```
> **VERIFY (each node):** `sudo openssl x509 -in $D/server.crt -noout -subject` → the node CN.

> **Step 5.1.5 — Append `run_mode = shared_data` to `fe.conf` + bring up the FE quorum**
> **WHERE:** the 3 FE nodes (`.37/.38/.39`); start the **leader `.37` first**. Root shell.
> **WHY:** the shared-data delta is `run_mode = shared_data` + `cloud_native_meta_port`.
> The first FE started with empty metadata **becomes the leader**; the other two
> register on the leader (`ALTER SYSTEM ADD FOLLOWER`) then do a one-shot `--helper`
> start to pull the BDB-JE metadata before handing off to systemd.
> **WHAT (on all 3 FE nodes — append to `fe.conf`, idempotent):**
> ```bash
> CONF=/opt/starrocks/fe/conf/fe.conf
> grep -q '^run_mode' "$CONF" || cat >> "$CONF" <<'EOF'
>
> # --- nexus shared-data per-host settings ---
> run_mode = shared_data
> cloud_native_meta_port = 6090
> priority_networks = 192.168.10.0/24
> JAVA_OPTS="-Xmx2g -XX:+UseG1GC"
> EOF
> ```
> **WHAT (on the leader `sr-sd-fe-1` `.37` only — start + wait for the query port):**
> ```bash
> systemctl enable --now nexus-starrocks-sd-fe.service
> until mysql --skip-ssl -h 127.0.0.1 -P 9030 -u root -N -e 'SHOW FRONTENDS' >/dev/null 2>&1; do sleep 8; done
> mysql --skip-ssl -h 127.0.0.1 -P 9030 -u root -e 'SHOW FRONTENDS'    # one row, LEADER
> ```
> **WHAT (register + join `sr-sd-fe-2` `.38`, then `sr-sd-fe-3` `.39`):**
> ```bash
> # On the leader (.37): register the follower's backplane edit-log endpoint (port 9010):
> mysql --skip-ssl -h 127.0.0.1 -P 9030 -u root -e "ALTER SYSTEM ADD FOLLOWER '192.168.10.38:9010'"
> ```
> ```bash
> # On the follower (.38): one-shot --helper join (persists BDB-JE meta), then hand to systemd:
> export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
> if [ ! -d /opt/starrocks/fe/meta/bdb ]; then
>   sudo -u starrocks JAVA_HOME="$JAVA_HOME" /opt/starrocks/fe/bin/start_fe.sh --helper 192.168.10.37:9010 --daemon
>   sleep 15
>   sudo -u starrocks JAVA_HOME="$JAVA_HOME" /opt/starrocks/fe/bin/stop_fe.sh || true
>   sleep 3
> fi
> systemctl enable --now nexus-starrocks-sd-fe.service
> ```
> Repeat the register-then-join for `sr-sd-fe-3` `.39` (on `.37`:
> `ALTER SYSTEM ADD FOLLOWER '192.168.10.39:9010'`; on `.39`: the same `--helper`
> block with `--helper 192.168.10.37:9010`).
> **EXPECTED:** 3 FE — 1 LEADER + 2 FOLLOWER, all Alive, all in `shared_data` mode.
> **VERIFY (on the leader):** `mysql --skip-ssl -h 127.0.0.1 -P 9030 -u root -e "SHOW FRONTENDS"`
> → 3 FE alive; `… -e "ADMIN SHOW FRONTEND CONFIG LIKE 'run_mode'"` → `shared_data`.

### 5.2 — Join the Compute Nodes (CN)

> **Step 5.2.1 — `cn.conf` + start; register as COMPUTE NODE**
> **WHERE:** CN nodes (`.30/.40`), then the FE leader.
> **WHY:** CN is the stateless data plane. Its `storage_root_path` is a **local
> cache** (durable data is in MinIO). Register with `ALTER SYSTEM ADD COMPUTE
> NODE` (not `ADD BACKEND`).
> **WHAT (on each CN — `cn.conf` lives in `/opt/starrocks/be/conf/`):**
> ```bash
> CONF=/opt/starrocks/be/conf/cn.conf
> grep -q '^priority_networks' "$CONF" || cat >> "$CONF" <<'EOF'
>
> # --- nexus shared-data per-host settings ---
> priority_networks = 192.168.10.0/24
> storage_root_path = /opt/starrocks/be/storage
> EOF
> install -d -o starrocks -g starrocks /opt/starrocks/be/storage
> systemctl enable --now nexus-starrocks-sd-cn.service
> ```
> ```bash
> # On the FE leader (.37): register each CN by its backplane heartbeat endpoint:
> for c in 30 40; do
>   mysql --skip-ssl -h 127.0.0.1 -P 9030 -u root -e "ALTER SYSTEM ADD COMPUTE NODE '192.168.10.$c:9050'"
> done
> ```
> **EXPECTED:** 2 CN come up + register.
> **VERIFY (on the leader):** `mysql --skip-ssl … -e "SHOW COMPUTE NODES"` → **2 CN**,
> all Alive `true`.

### 5.3 — Trust MinIO's TLS + create the storage volume (the exit gate)

> **Step 5.3.1 — Import the Vault CA into the FE JDK cacerts + the CN system trust**
> **WHERE:** each FE (cacerts) + each CN (system trust), root shell.
> **WHY:** MinIO serves HTTPS with a Vault-PKI cert; the FE's **AWS Java SDK**
> validates it against the **JDK truststore**, the CN's **aws-sdk-cpp** against the
> **system trust store**. Import the Vault CA into both or `CREATE STORAGE VOLUME`
> + data I/O fail TLS validation. Restart the engine after importing.
> **WHAT (on each FE — `.37/.38/.39`):**
> ```bash
> JH=/usr/lib/jvm/java-21-openjdk-amd64
> sudo keytool -import -trustcacerts -noprompt -alias nexus-vault-ca \
>   -file /etc/vault-agent/ca-bundle.crt \
>   -keystore "$JH/lib/security/cacerts" -storepass changeit 2>/dev/null || true
> systemctl restart nexus-starrocks-sd-fe.service ; sleep 8
> ```
> **WHAT (on each CN — `.30/.40`):**
> ```bash
> install -m 0644 /etc/vault-agent/ca-bundle.crt /usr/local/share/ca-certificates/nexus-vault-ca.crt
> update-ca-certificates
> systemctl restart nexus-starrocks-sd-cn.service ; sleep 5
> ```
> **EXPECTED:** the CA is trusted; FE leader comes back on `:9030`.
> **VERIFY:** `keytool -list -alias nexus-vault-ca -keystore "$JH/lib/security/cacerts" -storepass changeit`
> shows the cert; `mysql --skip-ssl … -e 'SELECT 1'` works after the FE restart.

> **Step 5.3.2 — `CREATE STORAGE VOLUME` (MinIO) + set as default**
> **WHERE:** the FE leader (`.37`), root shell.
> **WHY:** bind the cluster to `s3://starrocks/` in MinIO with path-style access +
> the S3 creds from Vault KV; make it the default so new tables land there. This
> is the shared-data cluster's exit gate.
> **WHAT (read the creds from Vault KV — never echo them):**
> ```bash
> AK=$(vault kv get -field=access_key nexus/analytics/starrocks-sd/s3-access-key)
> SK=$(vault kv get -field=secret_key nexus/analytics/starrocks-sd/s3-secret-key)
> mysql --skip-ssl -h 127.0.0.1 -P 9030 -u root <<SQL
> CREATE STORAGE VOLUME nexus_minio_starrocks
> TYPE = S3
> LOCATIONS = ('s3://starrocks/')
> PROPERTIES (
>   "enabled"                             = "true",
>   "aws.s3.endpoint"                     = "https://minio.nexus.lab:9000",
>   "aws.s3.region"                       = "us-east-1",
>   "aws.s3.access_key"                   = "$AK",
>   "aws.s3.secret_key"                   = "$SK",
>   "aws.s3.enable_path_style_access"     = "true",
>   "aws.s3.use_aws_sdk_default_behavior" = "false",
>   "aws.s3.use_instance_profile"         = "false"
> );
> SET nexus_minio_starrocks AS DEFAULT STORAGE VOLUME;
> SQL
> ```
> **EXPECTED:** the storage volume is created + set default.
> **VERIFY:** `mysql --skip-ssl … -e "SHOW STORAGE VOLUMES"` lists
> `nexus_minio_starrocks` with `IsDefault = true`.

### 5.4 — Round-robin DNS + schema (data lands in MinIO)

> **Step 5.4.1 — Publish `starrocks-sd-fe.nexus.lab` → the 3 FE (gateway)**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** same round-robin pattern as Guides 13/14 (`addn-hosts` hosts-file form).
> **WHAT:**
> ```bash
> cat > /etc/dnsmasq-starrocks-sd.hosts <<'EOF'
> 192.168.70.37 starrocks-sd-fe.nexus.lab
> 192.168.70.38 starrocks-sd-fe.nexus.lab
> 192.168.70.39 starrocks-sd-fe.nexus.lab
> EOF
> echo 'addn-hosts=/etc/dnsmasq-starrocks-sd.hosts' > /etc/dnsmasq.d/starrocks-sd-records.conf
> dnsmasq --test && systemctl reload dnsmasq
> ```
> **VERIFY:** `dig @192.168.70.1 starrocks-sd-fe.nexus.lab +short` → the 3 FE IPs.

> **Step 5.4.2 — Create a table + insert; confirm data lives in MinIO**
> **WHERE:** the FE leader, root shell.
> **WHY:** prove the shared-data path end-to-end — a cloud-native table whose
> tablets are written to `s3://starrocks/`. Shared-data tables **don't** take
> `replication_num` (durability is MinIO's job).
> **WHAT:**
> ```bash
> RP="mysql --skip-ssl -h 127.0.0.1 -P 9030 -u root"
> $RP -e "CREATE DATABASE IF NOT EXISTS nexus_sd"
> $RP -e "CREATE TABLE nexus_sd.events (event_id BIGINT, ts DATETIME, bucket INT, payload VARCHAR(64)) \
>         DUPLICATE KEY(event_id) DISTRIBUTED BY HASH(event_id) BUCKETS 6"
> # build a 200-row VALUES list with the shell, then INSERT it in one statement:
> VALUES=$(for i in $(seq 1 200); do printf "(%d,'2026-06-04 00:00:00',%d,'demo-%d')," "$i" "$((i % 6))" "$i"; done | sed 's/,$//')
> $RP -e "INSERT INTO nexus_sd.events VALUES $VALUES"
> $RP -N -e "SELECT count(*) FROM nexus_sd.events"
> ```
> **EXPECTED:** the `count(*)` matches inserted.
> **VERIFY:** the data is in MinIO — `mc ls --recursive nexus/starrocks/` (or the
> MinIO console) shows tablet objects under the `starrocks` bucket; querying via a
> CN returns the rows (CN read tablets from MinIO).

---

## 6. Validation — by-hand acceptance smoke

From the **host** (`mysql --skip-ssl`). Condensed from `smoke-0.L.5.ps1`.

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | 5 nodes reachable | `Test-NetConnection` (FE `:9030`, CN `:8040`) | all `True` |
| 2 | FE quorum, shared_data | `SHOW FRONTENDS` + `ADMIN SHOW FRONTEND CONFIG LIKE 'run_mode'` | 3 FE alive; `shared_data` |
| 3 | CN cluster | `SHOW COMPUTE NODES` | 2 CN, all Alive |
| 4 | Storage volume default | `SHOW STORAGE VOLUMES` | `nexus_minio_starrocks`, `IsDefault=true` |
| 5 | Round-robin DNS | `dig @192.168.70.1 starrocks-sd-fe.nexus.lab +short` | 3 FE IPs |
| 6 | Table writes to MinIO | insert + `mc ls nexus/starrocks/` | tablet objects appear in the bucket |
| 7 | Row round-trip via CN | `SELECT count(*)` (routed to a CN) | matches inserted |
| 8 | TLS to MinIO validates | `CREATE STORAGE VOLUME` succeeded (no TLS error) | no SSL handshake error in `fe.log`/`cn.out` |
| 9 | **CN elastic add/remove** | `ALTER SYSTEM DROP COMPUTE NODE '<b10>:9050'`; query still works (data in MinIO); re-add | queries unaffected (stateless CN) |
| 10 | **FE follower loss** | stop an FE follower; quorum holds (2/3) | leader still serves (then restart) |

**1–8 green ⇒ Guide 15 satisfied.** 9 is the shared-data payoff (compute is
disposable — data survives in MinIO); 10 is FE-quorum HA.

---

## 7. Teardown / reset

```bash
for ip in 30 40; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-starrocks-sd-cn'; done
for ip in 37 38 39; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-starrocks-sd-fe'; done
# then vmrun stop + deleteVM each of the 5 (Guide 00 §7).
```

> **The data outlives the cluster** — it's in MinIO's `starrocks` bucket. To wipe
> it, empty the bucket (`mc rm --recursive --force nexus/starrocks/`) *after* the
> cluster is down. A fresh cluster + the same `CREATE STORAGE VOLUME` re-attaches
> to existing data (if you keep the bucket). The gateway DNS record + the MinIO
> bucket/creds belong to Guides 01/16.

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `CREATE STORAGE VOLUME` fails with a TLS / cert error | the FE's JDK / CN's system trust doesn't have the Vault CA | import the CA into `cacerts` (FE, via `keytool`) + the system trust (CN, `update-ca-certificates`) + restart the engines (§5.3.1). |
| Storage volume created but tables fail to write | MinIO unreachable, wrong endpoint, or virtual-host addressing | confirm `minio.nexus.lab:9000` reachable; set `aws.s3.enable_path_style_access = true` (§5.3.2). |
| FE won't enter shared-data mode | `run_mode` not set, or set after first start (metadata already shared-nothing) | set `run_mode = shared_data` **before** the first FE start; a fresh meta dir is required to change mode. |
| `ADD COMPUTE NODE` then CN shows Dead | wrong port (used `9060` not heartbeat `9050`), or backplane blocked | register on `<b10>:9050`; confirm nftables trusts VMnet10 (§5.2.1). |
| Need MinIO but it isn't built | Guide 16 hasn't run | build Guide 16 (MinIO) first — the forward dependency called out at the top. |
| `mysql` auth/TLS errors | MariaDB 11.8 client requires TLS for password | `--skip-ssl` (inherited from Guide 14). |
| Tarball download 403s / JDK missing / tar ENOSPC | old public CDN 403s (S1); Debian 13 has no openjdk-17 (S2); root `/tmp` tmpfs too small for the 2.2 GB tarball (S3) | get the URL from the download-portal API (`…/api/v2/download?version=3.5.17…` → `.fileUrl`); install `openjdk-21-jdk`; extract under `/var/tmp` (all in §5.1.1). |

---

### Cross-references

- **0.L.5 architecture:** memory `project_nexus_infra_analytics_phase` + `project_nexus_infra_lakehouse_phase`; ADR-0037 (StarRocks shared-data over MinIO)
- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (SR shared-data `.37`–`.39` + CN `.30`/`.40`)
- **Automated equivalents:** `nexus-infra-analytics/terraform/envs/analytics-starrocks-sd/role-overlay-starrocks-sd-*.tf`
- **Object storage dependency:** [`16-lakehouse-minio.md`](./16-lakehouse-minio.md) (MinIO — **build first**)
- **Inherits from:** [`14-analytics-starrocks-shared-nothing.md`](./14-analytics-starrocks-shared-nothing.md) (install + FE quorum mechanics)
- **Previous guide:** [`14-analytics-starrocks-shared-nothing.md`](./14-analytics-starrocks-shared-nothing.md)
- **Next guide:** Guide 16 — Lakehouse · MinIO (4-node distributed erasure-coded object store). See [`INDEX.md`](../INDEX.md). **(Analytics tier 13–15 complete; the guides move into the Lakehouse.)**
