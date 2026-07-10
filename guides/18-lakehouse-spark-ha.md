# Guide 18 — Lakehouse · Spark HA (2 masters + ZooKeeper quorum + 3 workers)

> **Mirrors:** `nexus-infra-lakehouse` — the `lakehouse-spark-node` +
> `lakehouse-zookeeper-node` Packer templates + the `lakehouse-spark` Terraform env
> overlays (`…-nftables-backplane`, `…-vault-agents`, `…-tls`, `…-zk-ensemble`,
> `…-spark-config`, `…-spark-cluster-bootstrap`) — Phase 0.L.3 / ADR-0035. The
> **third lakehouse-tier guide**: the distributed **compute** engine that reads and
> writes the Iceberg tables.

> 🔗 **Depends on Guide 16 (MinIO) + Guide 17 (Iceberg/Nessie).** Spark is the
> **natural Iceberg client** — it registers Nessie's REST catalog and writes Parquet
> to MinIO's `s3a://warehouse`. This guide's exit gate is the **table→S3 write
> proof that Guide 17 deferred** (Spark → Nessie → MinIO, end to end). Build 16 + 17
> first.

---

## 1. Overview & purpose

A **Spark standalone HA cluster** — the platform's general-purpose distributed
compute. **8 nodes, three roles:**

- **Masters (`spark-master-1/2`, `.140/.153`)** — two Spark standalone masters in
  **ZooKeeper-elected HA**: one is `ALIVE`, the other `STANDBY`. If the active one
  dies, ZooKeeper promotes the standby and the workers re-register — no job state
  lost. Fronted by round-robin DNS `spark-master.nexus.lab`.
- **ZooKeeper ensemble (`zookeeper-1/2/3`, `.155/.156/.157`)** — a 3-node quorum
  that holds the Spark master-HA election state (`/spark` znode) and decides which
  master is active. Backplane-only, plaintext (ADR-0035 — the platform's one
  deliberate Apache-ZK exception; network segmentation is the boundary).
- **Workers (`spark-worker-1/2/3`, `.145/.146/.154`)** — the executors' hosts. Each
  registers with the multi-master URL and runs tasks. Lose one and Spark
  reschedules on the survivors.
- **Iceberg + S3 client.** Spark carries the **Iceberg Spark runtime** + the S3A /
  S3FileIO connectors, so a `spark-sql` job can `CREATE TABLE … USING iceberg`
  against **Nessie** (Guide 17) and land Parquet in **MinIO** `s3a://warehouse`
  (Guide 16).

**Why standalone (not YARN/K8s):** the lab wants the *Spark* concepts —
master/worker, HA election, executors — without dragging in a Hadoop/YARN or
Kubernetes control plane. **Why it matters:** Spark is the engine that *proves* the
whole lakehouse stack works together: catalog (Nessie) + storage (MinIO) + compute
(Spark) writing a real Iceberg table.

---

## 2. Component primer

- **Apache Spark (standalone mode).** A distributed compute engine; "standalone" is
  its built-in cluster manager (a master schedules across workers). *Why:* the
  simplest way to run a real multi-node Spark cluster. *Otherwise:* Spark-on-YARN
  (needs Hadoop), Spark-on-Kubernetes (needs a cluster), or local mode (not
  distributed). *vs. the analytics engines (ClickHouse/StarRocks, Guides 13–15):*
  those are SQL warehouses; Spark is general-purpose ETL/compute that *writes* the
  lake tables they (and Trino) read.
- **Spark master HA via ZooKeeper.** `spark.deploy.recoveryMode=ZOOKEEPER` makes the
  masters store election + cluster state in ZooKeeper; exactly one is `ALIVE`, the
  rest `STANDBY`, and a failover auto-promotes. *Why:* no single point of failure in
  the scheduler. *Otherwise:* `FILESYSTEM` recovery (single-node, shared dir) or no
  recovery (a master crash loses the cluster).
- **Apache ZooKeeper.** A replicated coordination service (a quorum of an odd number
  of nodes; majority = quorum). Spark uses it purely for master-HA election here.
  *Why 3 nodes:* tolerates 1 failure while keeping majority (2/3). *Otherwise:* etcd
  / Consul (Spark's standalone recovery speaks ZK specifically); Raft-based stores
  elsewhere in the platform (Patroni's etcd, Guide 10).
- **Iceberg Spark runtime + S3FileIO.** The `iceberg-spark-runtime` JAR makes Spark
  an Iceberg client (REST catalog support); `iceberg-aws-bundle` (AWS SDK **v2**)
  is what `S3FileIO` uses to write tablet/Parquet files to MinIO. `hadoop-aws` +
  `aws-java-sdk-bundle` (**must** match Spark's bundled Hadoop 3.3.4) provide the
  `s3a://` filesystem. *Why both:* the catalog is REST (Nessie), the data IO is
  S3FileIO → MinIO. *Otherwise:* a Hive catalog + plain `s3a` (no Iceberg
  semantics).
- **`spark.authenticate` + `spark.network.crypto`.** Cluster RPC is **authenticated**
  with a shared secret (from Vault KV) and **AES-encrypted** — so no TLS certs are
  needed on the RPC path. The Web UIs are plain HTTP on the nftables-restricted
  VMnet11 (UI TLS deferred, ADR-0035). Outbound HTTPS (→ Nessie + MinIO) validates
  against the Vault CA imported into the JVM truststore. *Why:* a pragmatic security
  model that secures the wire without a per-node cert dance on every executor port.

---

## 3. Prerequisites

| # | Requirement | One-command verify |
|---|---|---|
| 1 | **Guide 16 (MinIO)** — `warehouse` + `spark-events` buckets; app S3 creds in KV | `mc ls nexuslocal/warehouse` on `minio-1`; `vault kv get nexus/lakehouse/minio/app-access-key` |
| 2 | **Guide 17 (Iceberg/Nessie)** — `iceberg.nexus.lab:19120/iceberg/` reachable + TLS-trusted | `curl -sk -o /dev/null -w '%{http_code}' https://iceberg.nexus.lab:19120/iceberg/v1/config` → `200` |
| 3 | **Foundation alive** (Guides 00–04) — Vault PKI + KV; gateway DNS | `vault status` on `vault-1` → `Sealed: false` |
| 4 | **8 `deb13` nodes** baselined (Guide 00), dual-NIC static `.140/.153` + `.145/.146/.154` + `.155–.157` | the 8 answer `:22`; firstboot mapped `NEXUS_ROLE` / `NEXUS_ZK_ID` |
| 5 | **Spark + ZK + JAR artifacts** reachable (archive.apache.org + Maven Central, or a local cache) | `curl -sI https://archive.apache.org/dist/spark/spark-3.5.3/spark-3.5.3-bin-hadoop3.tgz \| head -1` |

> **Versions:** Apache **Spark 3.5.3** (bin-hadoop3), Apache **ZooKeeper 3.9.3**,
> **JDK 21**, **Iceberg 1.7.1** runtime + aws-bundle, `hadoop-aws 3.3.4` +
> `aws-java-sdk-bundle 1.12.262`. Front door: round-robin DNS
> `spark-master.nexus.lab` → `.140/.153`. Catalog: `nexus` → Nessie
> `https://iceberg.nexus.lab:19120/iceberg/`; warehouse `s3a://warehouse` on MinIO.

> **By-hand divergence:** the automated path uses a per-node Vault Agent to read KV;
> the manual path reads with `vault kv get` directly. The load-bearing trust steps
> done by hand: the **shared `spark.authenticate` secret** + **AES RPC encryption**,
> and the **Vault CA imported into the JVM truststore** (for outbound S3/Nessie TLS).
> A `spark-server` PKI leaf is issued per node for parity, but cluster RPC security
> is shared-secret + AES (no cert on the RPC path), and the UIs are HTTP (ADR-0035).

---

## 4. Target topology

| Node | Role | VMnet11 | VMnet10 | MAC (primary / secondary) | RAM | Ports |
|---|---|---|---|---|---|---|
| `spark-master-1` | Spark master (HA) | `.140` | `.10.140` | `…3F:00:99` / `…3F:01:99` | 4 GB | 7077 RPC · 8080 UI |
| `spark-master-2` | Spark master (HA) | `.153` | `.10.153` | `…3F:00:AA` / `…3F:01:AA` | 4 GB | 7077 · 8080 |
| `spark-worker-1` | Spark worker | `.145` | `.10.145` | `…3F:00:9E` / `…3F:01:9E` | 4 GB | 8081 UI · dynamic exec |
| `spark-worker-2` | Spark worker | `.146` | `.10.146` | `…3F:00:9F` / `…3F:01:9F` | 4 GB | 8081 · dynamic exec |
| `spark-worker-3` | Spark worker | `.154` | `.10.154` | `…3F:00:AB` / `…3F:01:AB` | 4 GB | 8081 · dynamic exec |
| `zookeeper-1` | ZooKeeper (myid 1) | `.155` | `.10.155` | `…3F:00:AC` / `…3F:01:AC` | 2 GB | 2181 client · 2888/3888 quorum (backplane-only) |
| `zookeeper-2` | ZooKeeper (myid 2) | `.156` | `.10.156` | `…3F:00:AD` / `…3F:01:AD` | 2 GB | 2181 · 2888/3888 |
| `zookeeper-3` | ZooKeeper (myid 3) | `.157` | `.10.157` | `…3F:00:AE` / `…3F:01:AE` | 2 GB | 2181 · 2888/3888 |

> MAC blocks: Spark reuses `:99/:9E/:9F` + the 0.L.3 expansion `:AA/:AB`; ZK uses
> `:AC–:AE`. **ZooKeeper + Spark master-HA election + master↔worker + executor RPC
> ride the VMnet10 backplane**; the Spark RPC `:7077` + Web UIs ride VMnet11. ZK
> exposes **nothing** on VMnet11 (backplane-only). VMs under
> `H:\VMS\NexusPlatform\08-spark\<node>\`.

---

## 5. Step-by-step build

> **WHERE:** node steps as `nexusadmin` → `sudo -i` (root). `vault` runs on
> **`vault-1`**. Order: install all 8 → nftables → **ZooKeeper ensemble first**
> (Spark's HA needs it) → Spark (masters then workers) → DNS → exit gate. The 3 ZK
> nodes differ by `myid` (spelled out); the 5 Spark nodes differ by role + local IP.

### 5.0 — Seed the Spark RPC secret in Vault KV (once)

> **Step 5.0.1 — Write the shared `spark.authenticate` secret**
> **WHERE:** `vault-1` (`.121`), root shell with an operator `VAULT_TOKEN`.
> **WHY:** every Spark node shares one secret that authenticates + keys the
> AES-encrypted RPC. The S3 warehouse creds are reused from Guide 16.
> **WHAT:**
> ```bash
> export VAULT_ADDR=https://127.0.0.1:8200 VAULT_CACERT=$HOME/.nexus/vault-ca-bundle.crt
> vault kv put nexus/lakehouse/spark/auth-secret value="$(openssl rand -hex 32)"
> ```
> **VERIFY:** `vault kv get -field=value nexus/lakehouse/spark/auth-secret` returns 64 hex chars.

### 5.1 — Create the 8 VMs + install packages

> **Step 5.1.1 — Create the 8 VMs + baseline**
> **WHERE:** VMware GUI on the build host.
> **WHY:** standard Guide 00 deb13 shape (no extra data disk). Spark nodes 4 GB
> (JVM + executors), ZK nodes 2 GB.
> **WHAT:** create `spark-master-1/2` + `spark-worker-1/2/3` (4 GB) +
> `zookeeper-1/2/3` (2 GB) per Guide 00 §5.A; pin the §4 MACs; install Debian 13 +
> baseline. firstboot maps each `NEXUS_ROLE` (`spark-master`/`spark-worker`/
> `zookeeper`) and emits `NEXUS_ZK_ID` (1/2/3) on the ZK nodes.
> **EXPECTED:** the 8 boot with their leases + `.10.x` backplane.
> **VERIFY:** `ssh nexusadmin@192.168.70.155 'sudo grep NEXUS_ZK_ID /etc/nexus-zookeeper/node-identity.env'`
> → `NEXUS_ZK_ID=1` (and `.156/.157` → `2`/`3`); `…@140 'sudo grep NEXUS_ROLE /etc/nexus-spark/node-identity.env'` → `spark-master`.

> **Step 5.1.2 — Install ZooKeeper 3.9.3 + JDK 21 on the ZK nodes (`.155–.157`)**
> **WHERE:** `zookeeper-1`, `zookeeper-2`, `zookeeper-3`, root shell.
> **WHY:** ZK is a tarball on JDK 21, run as a `zookeeper` system user. The service
> is installed **disabled** — §5.3 renders `zoo.cfg`/`myid` and starts it. ⚠️ Seed
> `logback.xml` into the config dir — `zkServer.sh` looks for it alongside `zoo.cfg`
> in `ZOOCFGDIR` (T9).
> **WHAT (run on all 3 ZK nodes):**
> ```bash
> apt-get update && apt-get install -y openjdk-21-jre-headless curl
> getent group zookeeper >/dev/null || groupadd --system zookeeper
> getent passwd zookeeper >/dev/null || useradd --system --gid zookeeper --home-dir /var/lib/zookeeper --create-home --shell /usr/sbin/nologin zookeeper
> curl -fSL https://archive.apache.org/dist/zookeeper/zookeeper-3.9.3/apache-zookeeper-3.9.3-bin.tar.gz -o /tmp/zk.tgz
> mkdir -p /opt/apache-zookeeper-3.9.3-bin
> tar xzf /tmp/zk.tgz -C /opt/apache-zookeeper-3.9.3-bin --strip-components=1 && rm /tmp/zk.tgz
> ln -sfn /opt/apache-zookeeper-3.9.3-bin /opt/zookeeper
> install -d -o zookeeper -g zookeeper -m 0750 /var/lib/zookeeper /var/log/zookeeper
> install -d -o root -g zookeeper -m 0755 /etc/nexus-zookeeper
> install -m 0644 -g zookeeper /opt/zookeeper/conf/logback.xml /etc/nexus-zookeeper/logback.xml
> ```
> Then install the systemd unit (disabled):
> ```bash
> cat > /etc/systemd/system/nexus-zookeeper.service <<'EOF'
> [Unit]
> Description=Nexus Apache ZooKeeper (Spark master-HA coordination quorum)
> After=network-online.target
> Wants=network-online.target
> ConditionPathExists=/etc/nexus-zookeeper/zoo.cfg
> [Service]
> Type=simple
> User=zookeeper
> Group=zookeeper
> Environment=JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
> Environment=ZOOCFGDIR=/etc/nexus-zookeeper
> Environment=ZOO_LOG_DIR=/var/log/zookeeper
> ExecStart=/opt/zookeeper/bin/zkServer.sh start-foreground /etc/nexus-zookeeper/zoo.cfg
> Restart=on-failure
> RestartSec=10
> LimitNOFILE=65536
> [Install]
> WantedBy=multi-user.target
> EOF
> systemctl daemon-reload ; systemctl disable nexus-zookeeper.service 2>/dev/null || true
> ```
> **VERIFY:** `test -x /opt/zookeeper/bin/zkServer.sh && echo ok`;
> `systemctl is-enabled nexus-zookeeper.service` → `disabled`.

> **Step 5.1.3 — Install Spark 3.5.3 + Iceberg/S3 JARs + JDK 21 on the Spark nodes (`.140/.153/.145/.146/.154`)**
> **WHERE:** all 5 Spark nodes, root shell.
> **WHY:** Spark is a tarball on JDK 21, run as a `spark` system user. The S3A +
> Iceberg JARs go into `$SPARK_HOME/jars`: `hadoop-aws` + `aws-java-sdk-bundle`
> **must** match Spark's bundled Hadoop 3.3.4 (T7); `iceberg-aws-bundle` (AWS SDK
> v2) is what `S3FileIO` needs (T6). Both role units are installed **disabled**;
> §5.4 renders config + enables exactly one per node.
> **WHAT (run on all 5 Spark nodes):**
> ```bash
> apt-get update && apt-get install -y openjdk-21-jre-headless curl
> getent group spark >/dev/null || groupadd --system spark
> getent passwd spark >/dev/null || useradd --system --gid spark --home-dir /var/lib/spark --create-home --shell /usr/sbin/nologin spark
> curl -fSL https://archive.apache.org/dist/spark/spark-3.5.3/spark-3.5.3-bin-hadoop3.tgz -o /tmp/spark.tgz
> mkdir -p /opt/spark-3.5.3-bin-hadoop3
> tar xzf /tmp/spark.tgz -C /opt/spark-3.5.3-bin-hadoop3 --strip-components=1 && rm /tmp/spark.tgz
> ln -sfn /opt/spark-3.5.3-bin-hadoop3 /opt/spark
> # S3A + Iceberg runtime + Iceberg AWS (SDK v2) JARs
> for u in \
>   https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-aws/3.3.4/hadoop-aws-3.3.4.jar \
>   https://repo1.maven.org/maven2/com/amazonaws/aws-java-sdk-bundle/1.12.262/aws-java-sdk-bundle-1.12.262.jar \
>   https://repo1.maven.org/maven2/org/apache/iceberg/iceberg-spark-runtime-3.5_2.12/1.7.1/iceberg-spark-runtime-3.5_2.12-1.7.1.jar \
>   https://repo1.maven.org/maven2/org/apache/iceberg/iceberg-aws-bundle/1.7.1/iceberg-aws-bundle-1.7.1.jar ; do
>   curl -fSL "$u" -o "/opt/spark/jars/$(basename "$u")"
> done
> install -d -o spark -g spark -m 0750 /var/lib/spark /var/lib/spark/work /var/log/spark
> install -d -o root  -g spark -m 0750 /etc/nexus-spark /etc/nexus-spark/tls
> ```
> Then install **both** role units (disabled):
> ```bash
> cat > /etc/systemd/system/nexus-spark-master.service <<'EOF'
> [Unit]
> Description=Nexus Apache Spark standalone Master (HA, ZooKeeper-elected)
> After=network-online.target
> Wants=network-online.target
> ConditionPathExists=/etc/nexus-spark/spark-cluster.env
> [Service]
> Type=simple
> User=spark
> Group=spark
> Environment=JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
> Environment=SPARK_HOME=/opt/spark
> Environment=SPARK_LOG_DIR=/var/log/spark
> EnvironmentFile=-/etc/nexus-spark/spark-cluster.env
> ExecStart=/opt/spark/bin/spark-class org.apache.spark.deploy.master.Master --host ${SPARK_MASTER_HOST} --port 7077 --webui-port 8080
> Restart=on-failure
> RestartSec=10
> LimitNOFILE=65536
> [Install]
> WantedBy=multi-user.target
> EOF
> cat > /etc/systemd/system/nexus-spark-worker.service <<'EOF'
> [Unit]
> Description=Nexus Apache Spark standalone Worker
> After=network-online.target
> Wants=network-online.target
> ConditionPathExists=/etc/nexus-spark/spark-cluster.env
> [Service]
> Type=simple
> User=spark
> Group=spark
> Environment=JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
> Environment=SPARK_HOME=/opt/spark
> Environment=SPARK_LOG_DIR=/var/log/spark
> EnvironmentFile=-/etc/nexus-spark/spark-cluster.env
> ExecStart=/opt/spark/bin/spark-class org.apache.spark.deploy.worker.Worker ${SPARK_MASTER_URL} --webui-port 8081
> Restart=on-failure
> RestartSec=10
> LimitNOFILE=65536
> [Install]
> WantedBy=multi-user.target
> EOF
> systemctl daemon-reload ; systemctl disable nexus-spark-master.service nexus-spark-worker.service 2>/dev/null || true
> ```
> **VERIFY:** `test -x /opt/spark/bin/spark-class && ls /opt/spark/jars/iceberg-aws-bundle-1.7.1.jar`;
> both units `disabled`.

### 5.2 — nftables (all 8: backplane trust + ports + the executor-RPC gap)

> **Step 5.2.1 — Apply the combined ruleset**
> **WHERE:** each of the 8 nodes, root shell.
> **WHY:** trust the VMnet10 backplane (Spark↔ZK election, master↔worker, ZK quorum
> `2888/3888` + client `2181`); open Spark RPC `7077` + UIs `8080/8081` on VMnet11.
> ⚠️ **The executor-RPC gap (T1):** Spark executors talk back to the driver +
> block-manager on **dynamic** ports — without an explicit accept for the 5 Spark
> node IPs, that cluster-peer RPC hits the `counter drop` and executors never
> register. Add the `saddr { 5 Spark IPs } accept` rule. Atomic `nft -f`.
> **WHAT (run on all 8 nodes — opening a port a node doesn't listen on is harmless):**
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
>         iifname "nic1" ip saddr 192.168.10.0/24 accept comment "trusted cluster backplane (VMnet10) -- Spark<->ZK + ZK quorum/client"
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport { 7077, 8080, 8081 } accept comment "Spark RPC + master/worker Web UI from VMnet11"
>         iifname "nic0" ip saddr { 192.168.70.140, 192.168.70.153, 192.168.70.145, 192.168.70.146, 192.168.70.154 } accept comment "Spark cluster-peer RPC (dynamic driver/blockManager ports, 5 Spark nodes)"
>         counter drop
>     }
>     chain forward { type filter hook forward priority 0; policy drop; }
>     chain output  { type filter hook output priority 0; policy accept; }
> }
> EOF
> nft -f /etc/nftables.conf ; systemctl enable nftables 2>/dev/null || true
> ```
> **EXPECTED:** ruleset loads clean on all 8.
> **VERIFY:** `nft list chain inet filter input | grep '192.168.10.0/24 accept'` (all 8);
> Spark nodes show the 5-IP `saddr { … } accept` cluster-peer rule.

### 5.3 — Form the ZooKeeper ensemble (do this first)

> **Step 5.3.1 — Render `zoo.cfg` + `myid` on `zookeeper-1` (myid 1)**
> **WHERE:** `zookeeper-1` (`.155`), root shell.
> **WHY:** `zoo.cfg` is **identical** on all 3 (the `server.N` lines use the
> backplane IPs); only `myid` differs per node. The ensemble talks quorum
> (`2888`) + election (`3888`) on the backplane.
> **WHAT:**
> ```bash
> cat > /etc/nexus-zookeeper/zoo.cfg <<'EOF'
> tickTime=2000
> initLimit=10
> syncLimit=5
> dataDir=/var/lib/zookeeper
> clientPort=2181
> maxClientCnxns=60
> admin.enableServer=false
> 4lw.commands.whitelist=ruok,srvr,stat,mntr,conf,isro
> autopurge.snapRetainCount=3
> autopurge.purgeInterval=24
> server.1=192.168.10.155:2888:3888
> server.2=192.168.10.156:2888:3888
> server.3=192.168.10.157:2888:3888
> EOF
> chown root:zookeeper /etc/nexus-zookeeper/zoo.cfg ; chmod 0640 /etc/nexus-zookeeper/zoo.cfg
> echo 1 > /var/lib/zookeeper/myid
> chown zookeeper:zookeeper /var/lib/zookeeper/myid ; chmod 0644 /var/lib/zookeeper/myid
> systemctl daemon-reload ; systemctl enable --now nexus-zookeeper.service
> ```
> **VERIFY:** `systemctl is-active nexus-zookeeper.service` → `active` (it will log
> elections until the quorum forms).

> **Step 5.3.2 — Render `zoo.cfg` + `myid` on `zookeeper-2` (myid 2)**
> **WHERE:** `zookeeper-2` (`.156`), root shell.
> **WHY:** identical `zoo.cfg`, `myid=2`.
> **WHAT:** write the **same `zoo.cfg`** as §5.3.1, then:
> ```bash
> echo 2 > /var/lib/zookeeper/myid
> chown zookeeper:zookeeper /var/lib/zookeeper/myid ; chmod 0644 /var/lib/zookeeper/myid
> systemctl daemon-reload ; systemctl enable --now nexus-zookeeper.service
> ```
> **VERIFY:** `systemctl is-active nexus-zookeeper.service` → `active`.

> **Step 5.3.3 — Render `zoo.cfg` + `myid` on `zookeeper-3` (myid 3); verify quorum**
> **WHERE:** `zookeeper-3` (`.157`), then any ZK node.
> **WHY:** identical `zoo.cfg`, `myid=3`. With all 3 up, the ensemble elects 1
> leader + 2 followers.
> **WHAT:** write the **same `zoo.cfg`**, then:
> ```bash
> echo 3 > /var/lib/zookeeper/myid
> chown zookeeper:zookeeper /var/lib/zookeeper/myid ; chmod 0644 /var/lib/zookeeper/myid
> systemctl daemon-reload ; systemctl enable --now nexus-zookeeper.service
> ```
> **EXPECTED:** within ~10–20 s the ensemble reaches quorum.
> **VERIFY (on each ZK node):**
> ```bash
> JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 ZOOCFGDIR=/etc/nexus-zookeeper \
>   /opt/zookeeper/bin/zkServer.sh status /etc/nexus-zookeeper/zoo.cfg 2>/dev/null | grep -i Mode
> ```
> across the 3 nodes → **exactly 1 `leader` + 2 `follower`**.

### 5.4 — Configure + start Spark (masters first, then workers)

> **Step 5.4.1 — Render Spark config on each node (CA import + 3 config files)**
> **WHERE:** each of the 5 Spark nodes, root shell with `VAULT_ADDR`/`VAULT_TOKEN` + CA.
> **WHY:** every node needs (a) the Vault CA in the **JVM truststore** so S3A
> (MinIO) + Iceberg REST (Nessie) TLS validate; (b) `spark-env.sh` with
> `recoveryMode=ZOOKEEPER` + `SPARK_LOCAL_IP` + the spark-writable work dir (T2);
> (c) `spark-cluster.env` (the systemd `EnvironmentFile`); (d) `spark-defaults.conf`
> with authenticated+AES RPC + the Iceberg `nexus` catalog → S3FileIO → MinIO.
> **Set `LOCALIP` to *this node's* VMnet11 IP** before running.
> **WHAT (run on each Spark node — substitute `LOCALIP` per node: `.140/.153/.145/.146/.154`):**
> ```bash
> export VAULT_ADDR=https://192.168.70.121:8200 VAULT_CACERT=$HOME/.nexus/vault-ca-bundle.crt
> LOCALIP=192.168.70.140    # <-- THIS node's VMnet11 IP
> JH=/usr/lib/jvm/java-21-openjdk-amd64
> AUTHSEC=$(vault kv get -field=value nexus/lakehouse/spark/auth-secret)
> S3AK=$(vault kv get -field=value nexus/lakehouse/minio/app-access-key)
> S3SK=$(vault kv get -field=value nexus/lakehouse/minio/app-secret-key)
>
> # (a) Trust the Vault CA in the JVM so S3A(MinIO) + Iceberg REST(Nessie) TLS validate
> keytool -delete -alias nexus-ca -keystore "$JH/lib/security/cacerts" -storepass changeit 2>/dev/null || true
> keytool -importcert -noprompt -alias nexus-ca -file /etc/ssl/certs/ca-certificates.crt \
>   -keystore "$JH/lib/security/cacerts" -storepass changeit 2>/dev/null || \
> keytool -importcert -noprompt -alias nexus-ca -file "$VAULT_CACERT" \
>   -keystore "$JH/lib/security/cacerts" -storepass changeit
>
> # (b) spark-env.sh -- recoveryMode=ZOOKEEPER + local bind + spark-writable work dir (T2)
> cat > /opt/spark/conf/spark-env.sh <<EOF
> #!/usr/bin/env bash
> export JAVA_HOME=$JH
> export SPARK_LOG_DIR=/var/log/spark
> export SPARK_WORKER_DIR=/var/lib/spark/work
> export SPARK_LOCAL_IP=$LOCALIP
> export SPARK_PUBLIC_DNS=$LOCALIP
> export SPARK_DAEMON_JAVA_OPTS="-Dspark.deploy.recoveryMode=ZOOKEEPER -Dspark.deploy.zookeeper.url=192.168.10.155:2181,192.168.10.156:2181,192.168.10.157:2181 -Dspark.deploy.zookeeper.dir=/spark"
> EOF
> chown root:spark /opt/spark/conf/spark-env.sh ; chmod 0644 /opt/spark/conf/spark-env.sh
>
> # (c) spark-cluster.env -- systemd EnvironmentFile (master host + multi-master URL)
> cat > /etc/nexus-spark/spark-cluster.env <<EOF
> SPARK_MASTER_HOST=$LOCALIP
> SPARK_MASTER_URL=spark://192.168.70.140:7077,192.168.70.153:7077
> EOF
> chown root:spark /etc/nexus-spark/spark-cluster.env ; chmod 0640 /etc/nexus-spark/spark-cluster.env
>
> # (d) spark-defaults.conf -- authed+AES RPC + Iceberg(Nessie) -> S3FileIO(MinIO)
> cat > /opt/spark/conf/spark-defaults.conf <<EOF
> spark.authenticate                            true
> spark.authenticate.secret                     $AUTHSEC
> spark.network.crypto.enabled                  true
> spark.network.crypto.keyLength                128
> spark.driver.host                             $LOCALIP
> spark.driver.bindAddress                      $LOCALIP
> spark.sql.catalogImplementation               in-memory
> spark.sql.extensions                          org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions
> spark.sql.catalog.nexus                       org.apache.iceberg.spark.SparkCatalog
> spark.sql.catalog.nexus.type                  rest
> spark.sql.catalog.nexus.uri                   https://iceberg.nexus.lab:19120/iceberg/
> spark.sql.catalog.nexus.warehouse             warehouse
> spark.sql.catalog.nexus.io-impl               org.apache.iceberg.aws.s3.S3FileIO
> spark.sql.catalog.nexus.s3.endpoint           https://minio.nexus.lab:9000
> spark.sql.catalog.nexus.s3.path-style-access  true
> spark.sql.catalog.nexus.s3.access-key-id      $S3AK
> spark.sql.catalog.nexus.s3.secret-access-key  $S3SK
> spark.sql.catalog.nexus.client.region         us-east-1
> spark.executorEnv.AWS_EC2_METADATA_DISABLED   true
> spark.eventLog.enabled                        false
> EOF
> chown root:spark /opt/spark/conf/spark-defaults.conf ; chmod 0640 /opt/spark/conf/spark-defaults.conf
> systemctl daemon-reload
> ```
> **EXPECTED:** CA imported; the 3 config files rendered on each node.
> **VERIFY:** `keytool -list -alias nexus-ca -keystore "$JH/lib/security/cacerts" -storepass changeit`;
> `grep -E 'spark.driver.host|warehouse' /opt/spark/conf/spark-defaults.conf` shows
> the node IP + `warehouse` **by name** (not a URI — T5).

> **Step 5.4.2 — Start the masters, then the workers**
> **WHERE:** the 2 masters first (`.140/.153`), then the 3 workers (`.145/.146/.154`).
> **WHY:** the masters register their HA election in ZooKeeper (one ALIVE, one
> STANDBY); the workers connect to the multi-master URL and register with the active
> one. Start the masters first so workers register on the first try.
> **WHAT (on each master `.140/.153`):**
> ```bash
> systemctl enable --now nexus-spark-master.service
> ```
> **WHAT (then on each worker `.145/.146/.154`):**
> ```bash
> systemctl enable --now nexus-spark-worker.service
> ```
> **EXPECTED:** masters elect (1 ALIVE + 1 STANDBY); 3 workers register ALIVE.
> **VERIFY:** on a master, `curl -fsS http://$(hostname -I | awk '{print $1}'):8080/json/ | grep -E '"status"|aliveworkers'`
> → one node `"ALIVE"` with `"aliveworkers" : 3`, the other `"STANDBY"`. (The UI
> binds to the node's VMnet11 IP, not localhost — query by IP, T8.)

### 5.5 — Round-robin DNS `spark-master.nexus.lab` (gateway)

> **Step 5.5.1 — Publish `spark-master.nexus.lab` → the 2 master IPs**
> **WHERE:** `nexus-gateway` (`.70.1`), root shell.
> **WHY:** a convenience name for clients submitting jobs (the cluster URL itself
> uses explicit IPs for robustness — no DNS dependency in the HA path).
> **WHAT:**
> ```bash
> cat > /etc/dnsmasq-spark.hosts <<'EOF'
> 192.168.70.140 spark-master.nexus.lab
> 192.168.70.153 spark-master.nexus.lab
> EOF
> echo 'addn-hosts=/etc/dnsmasq-spark.hosts' > /etc/dnsmasq.d/spark-records.conf
> dnsmasq --test && systemctl reload dnsmasq
> ```
> **VERIFY:** `dig @192.168.70.1 spark-master.nexus.lab +short` → `.140` + `.153`.

### 5.6 — Cluster bootstrap (the exit gate — the lakehouse write proof)

> **Step 5.6.1 — Spark → Iceberg(Nessie) → MinIO write round-trip**
> **WHERE:** the **active** master (the one reporting `ALIVE`), root shell.
> **WHY:** the deterministic exit gate — the full lakehouse write path that Guide 17
> deferred. A `spark-sql` job creates an Iceberg namespace + table **via the Nessie
> REST catalog**, INSERTs rows (Parquet written to `s3a://warehouse` on MinIO), and
> reads the count back.
> **WHAT (on the active master):**
> ```bash
> SQL="CREATE NAMESPACE IF NOT EXISTS nexus.lakehouse_demo;
> CREATE TABLE IF NOT EXISTS nexus.lakehouse_demo.smoke (id bigint, msg string) USING iceberg;
> DELETE FROM nexus.lakehouse_demo.smoke;
> INSERT INTO nexus.lakehouse_demo.smoke VALUES (1,'hello'),(2,'lakehouse');
> SELECT concat('SMOKECOUNT=', count(*)) FROM nexus.lakehouse_demo.smoke;"
> JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 timeout 300 /opt/spark/bin/spark-sql \
>   --master 'spark://192.168.70.140:7077,192.168.70.153:7077' \
>   --name nexus-0L3-smoke -e "$SQL" 2>&1 | grep -E 'SMOKECOUNT=|Exception|ERROR' | head -40
> ```
> **EXPECTED:** the output contains `SMOKECOUNT=2`.
> **VERIFY:** on `minio-1`, `sudo mc ls --recursive nexuslocal/warehouse/lakehouse_demo/`
> shows Parquet + Iceberg metadata objects; via Nessie,
> `curl -sk --cacert /etc/ssl/certs/iceberg-ca.pem 'https://iceberg.nexus.lab:19120/iceberg/v1/main%7Cwarehouse/namespaces'`
> lists `lakehouse_demo` (the namespace prefix is `main%7Cwarehouse` = `{ref}|{warehouse}` — T10).
> **➡ The lakehouse compute + write path is proven end-to-end.**

---

## 6. Validation — by-hand acceptance smoke (demo / playbook)

Condensed from `smoke-0.L.3.ps1`. Per-node SSH probes from the **build host**.

- **Input:** the 8 nodes up; ZK quorum formed; Spark HA elected; exit gate green.
- **Where observed:** SSH to each node / `zkServer.sh` on ZK / master `:8080/json/`
  by IP / `mc` on `minio-1` / `dig` on the gateway.
- **Proves:** an HA Spark cluster (ZK-elected) that writes real Iceberg tables to
  MinIO through Nessie.
- **Prerequisites:** Guides 00–04 + 16 + 17 alive; §5 complete.

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | 8 nodes reachable | `ssh …@140/153/145/146/154/155/156/157 'echo ok'` | all `ok` |
| 2 | Identity | `grep NEXUS_ROLE …/node-identity.env` / `NEXUS_ZK_ID` | roles + ZK ids 1/2/3 correct |
| 3 | nftables backplane trust | `nft list chain inet filter input` (all 8) | `192.168.10.0/24 accept` present |
| 4 | nftables executor-RPC gap | `nft … input` (Spark nodes) | 5-IP `saddr { … } accept` present |
| 5 | ZK quorum | `zkServer.sh status` (each ZK) | 1 `leader` + 2 `follower` |
| 6 | Spark masters HA | `curl http://<master-ip>:8080/json/` (both) | one `ALIVE`, one `STANDBY` |
| 7 | Workers registered | active master `/json/` `aliveworkers` | `>= 3` |
| 8 | Worker units active | `systemctl is-active nexus-spark-worker` (3) | `active` |
| 9 | RPC security | `grep -E 'authenticate \|crypto.enabled' spark-defaults.conf` | both `true` |
| 10 | Election state in ZK | `zkCli.sh ls /spark` (zookeeper-1) | `master_status`/`leader_election` |
| 11 | Round-robin DNS | `dig spark-master.nexus.lab +short` | `.140` + `.153` |
| 12 | **Iceberg write path** | the §5.6 `spark-sql` job | `SMOKECOUNT=2`; Parquet in `s3a://warehouse` |
| 13 | **Master failover** (chaos) | stop the active master; standby promotes to `ALIVE`, workers stay | catalog/cluster survives (then restart old master → STANDBY) |

**1–12 green ⇒ Guide 18 satisfied** (and Guide 17's deferred table→S3 write proven).
13 is the HA payoff. **This completes the Lakehouse compute story: MinIO (16) +
Nessie (17) + Spark (18) writing a real Iceberg table end-to-end.**

---

## 7. Teardown / reset

```bash
for ip in 145 146 154; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-spark-worker.service'; done
for ip in 140 153;     do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-spark-master.service; sudo rm -f /etc/nexus-spark/spark-cluster.env'; done
for ip in 155 156 157; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-zookeeper.service; sudo rm -f /etc/nexus-zookeeper/zoo.cfg /var/lib/zookeeper/myid'; done
# gateway: rm /etc/dnsmasq.d/spark-records.conf /etc/dnsmasq-spark.hosts ; systemctl reload dnsmasq
# then vmrun stop + deleteVM each of the 8 (Guide 00 §7).
```

> **The Iceberg table data persists in MinIO `s3a://warehouse` + the catalog
> pointer in Nessie's PG (Guide 17).** Tearing down Spark removes only the compute;
> the lakehouse_demo table survives and a fresh Spark cluster re-reads it. To wipe
> the demo data: `mc rm --recursive --force nexuslocal/warehouse/lakehouse_demo/`
> *and* drop the Nessie namespace.

---

## 8. Troubleshooting

| # | Symptom | Cause | Fix |
|---|---|---|---|
| **T1** | Workers register but jobs hang / executors never start | the **executor-RPC firewall gap** — executors use dynamic driver/blockManager ports that hit `counter drop` | add `iifname "nic0" ip saddr { <5 Spark node IPs> } accept` to nftables (§5.2.1). |
| **T2** | Worker fails: `AccessDenied` on the work dir | default `$SPARK_HOME/work` under root-owned `/opt/spark` isn't spark-writable | set `SPARK_WORKER_DIR=/var/lib/spark/work` (spark-owned) in `spark-env.sh` (§5.4.1). |
| **T3** | Executors register with the wrong master / never attach | `spark.driver.host` defaulted to reverse DNS → the round-robin `spark-master.nexus.lab` | set `spark.driver.host` + `spark.driver.bindAddress` to the node's own IP (§5.4.1). |
| **T4** | Spark tries to boot an embedded Derby Hive metastore (slow/locks) | default `spark.sql.catalogImplementation=hive` | set it to `in-memory` — every table lives in the Iceberg REST catalog `nexus` (§5.4.1). |
| **T5** | Catalog ops fail: `Warehouse not known` | the warehouse was given as a URI | Nessie resolves the warehouse **by name** — `spark.sql.catalog.nexus.warehouse=warehouse` (location lives in Nessie) (§5.4.1). |
| **T6** | `NoClassDefFoundError software/amazon/awssdk/.../S3Exception` | `iceberg-aws-bundle` (AWS SDK v2) missing — `S3FileIO` needs it | add `iceberg-aws-bundle-1.7.1.jar` to `$SPARK_HOME/jars` (§5.1.3). |
| **T7** | S3A `NoSuchMethodError` | `hadoop-aws` / `aws-java-sdk-bundle` version mismatch with Spark's bundled Hadoop | use `hadoop-aws-3.3.4` + `aws-java-sdk-bundle-1.12.262` (match Hadoop 3.3.4) (§5.1.3). |
| **T8** | Master `/json/` returns nothing on `localhost` | the master UI binds to `SPARK_LOCAL_IP` (the node's VMnet11 IP), not localhost | query `http://<node-vmnet11-ip>:8080/json/` (§5.4.2). |
| **T9** | ZooKeeper starts then exits / no logging | `zkServer.sh` can't find `logback.xml` in `ZOOCFGDIR` | seed `logback.xml` into `/etc/nexus-zookeeper/` alongside `zoo.cfg` (§5.1.2). |
| **T10** | Nessie `/v1/namespaces` returns empty after a write | Nessie scopes namespaces under `{ref}\|{warehouse}` | query the prefixed path `…/iceberg/v1/main%7Cwarehouse/namespaces` (discover the prefix from `/v1/config?warehouse=warehouse` if it changes) (§5.6.1). |
| **—** | Backplane down (ZK can't form quorum / Spark can't reach ZK) | VMware left `ethernet1`/nic1 NO-CARRIER at power-on | reconnect the 2nd NIC + `systemctl restart systemd-networkd`; confirm `ip addr show nic1` has `192.168.10.1XX` (same as Guides 16/17). |

---

## 9. Production tuning — Apache Spark 3.5 + ZooKeeper 3.9

> **Everything below is *beyond the lab replica*.** §5 ships the verbatim lab configs — 5
> Spark nodes at 4 GB + 3 ZK nodes at 2 GB, `spark-defaults.conf` carrying only the
> security/catalog wiring (§5.4.1) with **no memory or parallelism sizing**, and ZK on its
> default JVM heap. This section is what you would change for a **production** Spark cluster +
> ZooKeeper ensemble, and *why*; it never alters the §5 values. **Do not paste these onto the
> 4 GB/2 GB lab VMs blindly.** The **OS-layer** knobs (swappiness, THP, ulimits, I/O scheduler)
> live once in **[Guide 00 §9](./00-lab-host-and-base-vm.md#9-production-tuning--the-os-layer-feeds-every-linux-tier)** —
> both engines want them; only the engine-specific overrides are restated here. This guide has
> **two** engines, tuned separately below: **Spark** (§9.1–9.3) and **ZooKeeper** (§9.4–9.5).

---

### Spark (masters + workers + executors)

The lab leaves every sizing knob at the Spark default (driver/executor `1g`, executor cores =
all on the worker, `shuffle.partitions=200`, Java serializer) — enough to run the §5.6 write
proof, nowhere near tuned for real ETL. In standalone mode you size the **worker** (how much of
the box it offers) and then the **per-application** executor/driver requests that carve it up.

### 9.1 Executor & driver sizing (`spark-defaults.conf` / submit args)

```properties
# PRODUCTION — append to /opt/spark/conf/spark-defaults.conf (or pass per spark-submit).
spark.driver.memory              4g
spark.executor.memory            8g
spark.executor.cores             4
spark.executor.instances         6
spark.executor.memoryOverhead    2g
```

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `spark.executor.memory` | **`8g`** (heap for a real working set; size to fit the worker) | unset (`1g`) | The JVM heap each executor gets for shuffle/aggregation/cache. `1g` spills to disk almost immediately on real data → jobs crawl. Leave headroom on the worker for `memoryOverhead` + the OS. |
| `spark.executor.cores` | **`4`–`5`** (the classic "5 cores per executor" sweet spot) | unset (all worker cores → one fat executor) | Tasks per executor. Too many cores per executor causes HDFS/S3 client and GC contention; `≤5` balances throughput vs. contention. Cores × instances must fit the worker's vCPU. |
| `spark.executor.instances` | **explicit** (e.g. `6`), or use dynamic allocation (§9.3) | unset (one executor per worker) | How many executors the app gets. Static sizing gives predictable capacity; pair with cores/memory so `instances × (cores, memory)` fits the cluster. |
| `spark.executor.memoryOverhead` | **`2g`** (≈ 10–20 % of executor memory; more for PySpark/S3) | unset (`max(384m, 0.1×executor.memory)`) | Off-heap memory for the JVM's native buffers, the S3A/Netty buffers, and PySpark worker processes. The default `~384m` is the **#1 cause of `Container killed … exceeds memory limits`** — raise it for S3/Iceberg workloads. |
| `spark.driver.memory` | **`4g`** (more for broadcast-heavy / large result-collecting jobs) | unset (`1g`) | The driver holds the DAG, broadcast tables, and any `collect()`ed results. `1g` OOMs the driver on a big broadcast join or a large collect, killing the whole application. |

### 9.2 Memory management & executor GC

```properties
# PRODUCTION — append to spark-defaults.conf.
spark.memory.fraction            0.6
spark.memory.storageFraction     0.5
spark.executor.extraJavaOptions  -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:InitiatingHeapOccupancyPercent=35
```

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `spark.memory.fraction` | **`0.6`** (raise to `0.75` for shuffle-heavy, no caching) | unset (`0.6`) | Fraction of (heap − 300 MB) for the **unified** execution+storage region; the rest is user data structures. Too low → more spilling; too high → risk of user-code OOM. |
| `spark.memory.storageFraction` | **`0.5`** | unset (`0.5`) | Within the unified region, the floor reserved for **cached** blocks that execution can't evict. Lower it if you cache little and shuffle a lot (execution borrows more). |
| Executor GC | **G1** `MaxGCPauseMillis=200` | unset (JVM default) | Large executor heaps under shuffle produce long pauses on the default collector; G1 with a bounded pause target keeps stragglers from stalling a whole stage. Set via `spark.executor.extraJavaOptions`. |

### 9.3 Shuffle, serialization, AQE & shuffle-spill disk

```properties
# PRODUCTION — append to spark-defaults.conf.
spark.serializer                 org.apache.spark.serializer.KryoSerializer
spark.sql.shuffle.partitions     400
spark.sql.adaptive.enabled       true
spark.local.dir                  /mnt/nvme/spark-local
spark.network.timeout            300s
# Optional — let the cluster grow/shrink executors with demand (needs an external shuffle service):
spark.dynamicAllocation.enabled  true
spark.dynamicAllocation.shuffleTracking.enabled true
```

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `spark.serializer` | **`org.apache.spark.serializer.KryoSerializer`** | unset (`JavaSerializer`) | Kryo is far faster and more compact than Java serialization for shuffle + cache payloads — a broad, low-risk win. The Java default is slow and bloats shuffle bytes. |
| `spark.sql.shuffle.partitions` | **`400`** (rule of thumb: ≈ total executor cores × 2–3; or let AQE coalesce) | unset (`200`) | Post-shuffle partition count. `200` under-parallelises big joins/aggregations (huge partitions, spill) and over-parallelises tiny ones. AQE (below) coalesces at runtime, but the starting point still matters. |
| `spark.sql.adaptive.enabled` (AQE) | **`true`** (confirm — on by default since 3.2) | unset (`true` by default) | Adaptive Query Execution re-optimises at runtime: coalesces shuffle partitions, converts sort-merge → broadcast joins, and splits skewed partitions. Confirm it's on; it's the biggest free SQL win. |
| `spark.local.dir` | **a dedicated fast disk** (`/mnt/nvme/…`, or several comma-separated) | unset (`/tmp`) | Where shuffle blocks + RDD spill land. On `/tmp` (often small or on the root/OS disk) a big shuffle fills the disk and fails the job; point it at fast, roomy, dedicated storage — ideally striped across disks. |
| `spark.network.timeout` | **`300s`** | unset (`120s`) | The umbrella timeout for block fetches/heartbeats. Under GC pauses or heavy I/O the `120s` default triggers spurious executor-lost/task-retry churn; raising it steadies large jobs. |
| `spark.dynamicAllocation.enabled` | **`true`** on a shared cluster (requires the external shuffle service or shuffle tracking) | unset (`false`) | Lets an application add executors when tasks queue and release them when idle, so many jobs share the cluster fairly instead of one pinning all workers. Off by default; needs the shuffle service so released executors don't lose shuffle data. |

---

### ZooKeeper (the Spark master-HA election quorum)

ZooKeeper is small and **latency-critical** — Spark's master HA election lives in it. It does
not need much RAM, but it must **never swap** and must **never run out of disk**, or the
ensemble stalls and Spark can't elect a master. Note ZK here is the platform's **only
plaintext-backplane service** (ADR-0035): quorum/election ride VMnet10 unencrypted by design,
so production hardening (`sslQuorum=true` + X.509) is an *architecture* decision recorded in
the ADR, not a tuning knob.

### 9.4 ZooKeeper JVM heap (⚠️ never swap)

The lab runs ZK on the tarball default heap (≈1 GB via `zkServer.sh`). That is actually the
right *ballpark* for production too — ZK holds the entire data tree in memory but its dataset
(here: a handful of Spark election znodes) is tiny, so a **small, fixed** heap is correct.

```bash
# PRODUCTION — pin a fixed small heap on each ZK node via the systemd unit (Environment=).
# ZK is latency-sensitive: a fixed heap avoids GC growth pauses, and it must fit in RAM so it
# NEVER swaps (a swapped ZK misses heartbeats -> false leader elections).
Environment=JVMFLAGS=-Xms1g -Xmx1g -XX:+UseG1GC
```

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| ZK heap `-Xmx`/`-Xms` | **`1g`** fixed (`2g` only for large datasets; rarely more) | unset (≈`1g` default) | ZK keeps the whole znode tree + watches on-heap. Small **fixed** heap = short, predictable GC. Oversizing invites long pauses; **the heap + OS must fit in RAM with room to spare** so ZK never touches swap. |
| `vm.swappiness` / no swap for ZK | **`1`** (or `memlock` the process) | unset (`60`) | **⚠️ the critical ZK rule.** A ZK server swapped out under memory pressure misses session/quorum heartbeats → the ensemble declares it dead and re-elects, which can bounce Spark's active master. Keep ZK resident (Guide 00 §9 `swappiness=1`). |

### 9.5 Ensemble timing, autopurge (⚠️) & disk layout (`zoo.cfg`)

```properties
# PRODUCTION — zoo.cfg deltas from §5.3.1 (autopurge is ALREADY set in the lab — keep it).
tickTime=2000
initLimit=10
syncLimit=5
maxClientCnxns=200
autopurge.snapRetainCount=3
autopurge.purgeInterval=12
dataDir=/var/lib/zookeeper          # snapshots
dataLogDir=/mnt/nvme/zookeeper-txnlog   # transaction log on a SEPARATE fast disk
# forceSync=yes                     # KEEP the default; never set 'no' on a durable ensemble
```

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `autopurge.purgeInterval` / `autopurge.snapRetainCount` | **`12` h / `3`** | **PRESENT** — `24` h / `3` (§5.3.1) | **⚠️ without autopurge the transaction logs + snapshots grow forever and fill the disk**, at which point ZK stops writing and the ensemble freezes. The lab already enables it (every 24 h, keep 3) — keep it on; a busy ensemble purges more often (12 h). |
| `tickTime` | **`2000` ms** | **PRESENT** — `2000` (§5.3.1) | The base time unit; session timeouts and `initLimit`/`syncLimit` are counted in ticks. Lower for faster failure detection (more sensitive to jitter), higher on a noisy network. |
| `initLimit` / `syncLimit` | **`10` / `5`** ticks | **PRESENT** — `10` / `5` (§5.3.1) | How long a follower may take to initially connect+sync (`initLimit`) and to stay in sync (`syncLimit`) before the leader drops it. Too tight → healthy followers evicted on a slow disk/net; too loose → slow failure detection. |
| `maxClientCnxns` | **`200`+** (per client IP) | **PRESENT** — `60` (§5.3.1) | Per-IP connection cap. `60` is fine for 5 Spark nodes; raise it if many clients (or a proxy) connect from one address, or legitimate connections get refused. |
| `dataLogDir` (txn log on its own fast disk) | **separate NVMe/SSD**, distinct from `dataDir` | unset (txn log shares `dataDir=/var/lib/zookeeper`) | ZK `fsync`s **every write to the transaction log before ack** — this is the latency-critical path. Putting the txn log on its own dedicated fast disk (away from snapshot writes + the OS) is the single biggest ZK performance win. |
| `forceSync` | **`yes`** (the default — keep it) | unset (`yes`) | Forces an `fsync` of the txn log before acking. **Never set `no`** on a durable ensemble — it trades away crash-safety and can lose committed election state on power loss. Listed only to warn against "tuning" it off. |

> **Where these build on the OS layer:** both engines want the Guide 00 §9 base — THP `never`,
> `vm.swappiness=1` (**critical for ZK**, §9.4), the systemd `DefaultLimitNOFILE`/`nproc`
> ceilings (Spark's JVM thread count; the lab already sets `LimitNOFILE=65536` in both units,
> §5.1), and `mq-deadline`/`none` on the shuffle-spill (Spark) and txn-log (ZK) disks. Set the
> OS layer once per Guide 00 §9, then these sections on top.

---

### Cross-references

- **0.L.3 architecture:** memory `project_nexus_infra_lakehouse_phase`; ADR-0035
  (Spark standalone HA + ZooKeeper; plaintext-ZK exception; authenticate+AES RPC)
- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (Spark masters
  `.140/.153`, workers `.145/.146/.154`, ZK `.155–.157`); `vms.yaml` (`cluster: spark`)
- **Automated equivalents:** `nexus-infra-lakehouse/packer/lakehouse-{spark,zookeeper}-node/`
  + `terraform/envs/lakehouse-spark/role-overlay-{zk-ensemble,spark-config,spark-cluster-bootstrap,spark-nftables-backplane}.tf`
- **Smoke mirror:** `nexus-infra-lakehouse/scripts/smoke-0.L.3.ps1`
- **Depends on:** [`16-lakehouse-minio.md`](./16-lakehouse-minio.md) (warehouse `s3a://warehouse`) + [`17-lakehouse-iceberg-nessie.md`](./17-lakehouse-iceberg-nessie.md) (the Iceberg REST catalog)
- **Proves:** the table→S3 write Guide 17 deferred (Spark → Nessie → MinIO)
- **Related ZK use:** Kafka (Guide 06) is KRaft (no ZK); this is the platform's only ZooKeeper
- **Transients:** [[feedback_cluster_template_nftables_backplane]] · [[feedback_nftables_runtime_add_after_drop]]
- **Previous guide:** [`17-lakehouse-iceberg-nessie.md`](./17-lakehouse-iceberg-nessie.md)
- **Next guide:** Guide 19 — Registry · Harbor HA (2 stateless app nodes + PG/Redis HA + MinIO blobs + Vault OIDC SSO). See [`INDEX.md`](../INDEX.md). **(Lakehouse tier 16–18 complete.)**
