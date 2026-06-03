# Guide 06 — Kafka ecosystem (2 KRaft clusters + MirrorMaker 2 DR)

> **Mirrors:** `nexus-infra-kafka` — the `kafka-node` Packer template (Debian 13 +
> Temurin JDK 21 + Apache Kafka 3.8.1 + Confluent Community 7.7.1, all six role
> units baked **disabled**) + the `kafka` env overlays (`broker-config`,
> `kraft-format`, `kafka-tls`, `ecosystem-tls`, `schema-registry`, `rest`,
> `connect`, `ksqldb`, `mm2`). Where the automated lab clones one template 15×
> and renders configs over SSH via per-node Vault Agents, this guide installs the
> stack by hand and **issues certs directly with the `vault` CLI**.

---

## 1. Overview & purpose

This is the lab's **streaming data backbone** — and its largest tier: **15
nodes**. Two independent 3-node **KRaft** Kafka clusters plus a 9-node ecosystem:

- **`kafka-east`** (`.21/.22/.23`) — the **primary** cluster.
- **`kafka-west`** (`.24/.25/.26`) — the **DR** cluster.
- **Schema Registry** HA pair (`.91/.92`), **REST Proxy** (`.88`), **Kafka
  Connect** + Debezium (`.95/.96`), **ksqlDB** (`.97/.98`), **MirrorMaker 2**
  (`.85/.86`).

Built in five sub-phases: **KRaft bring-up (PLAINTEXT) → broker mTLS → Schema
Registry + REST → Connect + ksqlDB → MirrorMaker 2 DR**. The **exit gate** is the
cross-cluster round-trip: produce to `kafka-east` → the record appears on the
mirrored `east.*` topic on `kafka-west` (and the reverse).

**Why two clusters + MM2:** to exercise real cross-datacenter **disaster-recovery
replication**. MM2 mirrors topics bidirectionally with a source-alias prefix
(`east.orders` on west) so the pair is loop-safe.

**Dependency:**
- **Guides 00–04** — foundation alive; specifically Guide 04's **PKI**
  (`pki_int/issue/...`) + the **Part D scaffolding pattern** (this guide creates
  the `kafka-broker` PKI role + AppRoles + KV per that recipe).
- 15 `deb13` nodes baselined per Guide 00 §5.B, dual-NIC static (`.21–.26`,
  `.85/.86/.88/.91/.92/.95/.96/.97/.98`).
- Guide 05 is **not** a dependency (Kafka doesn't run on Swarm) — build order is
  just sequential.

> **By-hand divergence:** the automated lab runs a per-node Vault Agent that
> renders + splits the broker/ecosystem certs. By hand we **issue each cert with
> the `vault` CLI on `vault-1` and scp it**, doing the PEM→PKCS#8 and `.p12`
> conversions on the node — same idiom as Guides 03–05.

---

## 2. Component primer

- **Apache Kafka + KRaft.** Kafka is a distributed append-only log (topics →
  partitions → replicas). **KRaft** is Kafka's built-in Raft consensus — it
  replaces the old ZooKeeper dependency, so each broker runs in **combined mode**
  (`process.roles=broker,controller`): the 3 nodes form both the data plane and
  the controller quorum. *Why KRaft:* no separate ZooKeeper cluster to run.
  *Otherwise:* ZooKeeper-mode Kafka (deprecated) or Redpanda (different engine).
- **Replication factor / ISR.** Each partition is replicated to `RF=3` brokers;
  `min.insync.replicas=2` means a write needs 2 in-sync replicas to ack —
  surviving one broker loss without data loss. *Why:* durability + availability
  on a 3-node cluster.
- **Broker mTLS.** All three listeners (client `9092`, inter-broker, controller
  `9093`) go SSL with `ssl.client.auth=required` — **mutual** TLS everywhere
  (inter-broker replication, the controller quorum, and external clients all
  present a cert). *Why no "API without client cert" carve-out:* every Kafka
  client in the lab runs *from* a broker and reuses that broker's keystore as its
  client identity. *PEM keystore caveat:* Kafka 3.8 reads native `PEM` keystores,
  but Vault PKI issues a **PKCS#1** key while Kafka's parser needs **PKCS#8** —
  so a split step runs `openssl pkcs8 -topk8`.
- **Schema Registry.** Stores/validates Avro/Protobuf/JSON schemas so producers
  and consumers agree on shape; an HA pair (one primary-eligible, both serving
  reads) backed by a Kafka topic. *Otherwise:* schemas drift, consumers break.
- **REST Proxy.** An HTTP front to Kafka for clients that can't speak the native
  protocol. **Kafka Connect** runs source/sink connectors (here **Debezium** for
  change-data-capture); **ksqlDB** is streaming SQL over Kafka topics.
- **The PEM-vs-PKCS#12 split (a real gotcha).** Brokers, MM2, and the CLI use
  **`PEM`** keystores. But Connect's and ksqlDB's **embedded REST servers reject
  `ssl.keystore.type=PEM`** — they need **`.p12`** keystores, and the truststore
  `.p12` must be built with **`keytool -importcert`** (an `openssl pkcs12 -export
  -nokeys` truststore has an empty cert bag → `trustAnchors must be non-empty`).
- **MirrorMaker 2 (MM2).** Kafka's cross-cluster replicator (built on Connect).
  Each DR node runs `connect-mirror-maker.sh` in **dedicated mode** with exactly
  **one flow** (mm2-1 east→west, mm2-2 west→east). `DefaultReplicationPolicy`
  prefixes mirrored topics with the source alias (`east.orders`) — mandatory for
  a loop-safe bidirectional pair. *Otherwise:* `IdentityReplicationPolicy` would
  create an infinite mirror loop.
- **The `sudo` rule.** `/etc/nexus-kafka/` is `0750 root:kafka` and `nexusadmin`
  isn't in `kafka` — so **every** Kafka CLI invocation that reads
  `client-ssl.properties` must be `sudo` (else `AccessDeniedException`). Same
  lesson as Consul's `/etc/consul.d/`.

---

## 3. Prerequisites

| # | Requirement | One-command verify |
|---|---|---|
| 1 | Foundation alive (Guides 00–04); Vault PKI usable | `vault read pki_int/cert/ca` on vault-1 returns the intermediate |
| 2 | 15 `deb13` nodes baselined, dual-NIC static | brokers `.21–.26` + ecosystem `.85/.86/.88/.91/.92/.95/.96/.97/.98` all answer `:22` |
| 3 | Vault root token on build host | `Test-Path ~/.nexus/secrets/vault-cluster-init.json` |
| 4 | Internet egress on nodes (JDK/Kafka/Confluent downloads) | `ssh …@21 'curl -sI https://dlcdn.apache.org \| head -1'` → `200` |

> Versions: **Temurin JDK 21**, **Apache Kafka 3.8.1**, **Confluent Community
> 7.7.1**. Stable KRaft cluster UUIDs: `kafka-east` `ZD-HhB5fQfioHydHQWiHkw`,
> `kafka-west` `FezdEIKlRCWP6nCSrHNJww`.

---

## 4. Target topology

| Node | Role | VMnet11 | VMnet10 | KRaft node.id | vCPU/RAM/disk |
|---|---|---|---|:--:|---|
| `kafka-east-1/2/3` | east KRaft broker+controller | `.21/.22/.23` | `.10.21/.22/.23` | 1/2/3 | 4 / 8 GB / 200 GB |
| `kafka-west-1/2/3` | west KRaft broker+controller | `.24/.25/.26` | `.10.24/.25/.26` | 1/2/3 | 4 / 8 GB / 200 GB |
| `schema-registry-1/2` | Schema Registry HA | `.91/.92` | `.10.91/.92` | — | 2 / 8 GB / 40 GB |
| `kafka-rest-1` | REST Proxy | `.88` | `.10.88` | — | 2 / 8 GB / 30 GB |
| `kafka-connect-1/2` | Connect + Debezium | `.95/.96` | `.10.95/.96` | — | 2 / 8 GB / 40 GB |
| `ksqldb-1/2` | ksqlDB | `.97/.98` | `.10.97/.98` | — | 2 / 8 GB / 60 GB |
| `mm2-1` | MirrorMaker 2 east→west | `.85` | `.10.85` | — | 2 / 8 GB / 40 GB |
| `mm2-2` | MirrorMaker 2 west→east | `.86` | `.10.86` | — | 2 / 8 GB / 40 GB |

All cluster traffic on the **VMnet10 backplane**: client/inter-broker `9092`,
controller `9093`. Each cluster's `controller.quorum.voters` lists all three of
its nodes as `node_id@vmnet10:9093`. All 15 nodes run at **8 GB** (template-baked;
documented in `vms.yaml`).

---

## 5. Step-by-step build

> **WHERE:** node steps as `nexusadmin`→`sudo -i`; Kafka CLI under `sudo`. `vault`
> commands on **`vault-1`** (root token). "east" cluster = `.21/.22/.23`; do every
> cluster-wide action on east, then repeat for west (`.24/.25/.26`) verbatim.

### 5.1 — Per-node base install (all 15)

> **Step 5.1.1 — Install JDK 21 + Apache Kafka 3.8.1 + Confluent 7.7.1**
> **WHERE:** each node, root shell.
> **WHY:** the JVM + the Kafka brokers/MM2 (Apache) + the ecosystem services
> (Confluent Community). Create the `kafka` user + the `0750 root:kafka` config
> dir.
> **WHAT:**
> ```bash
> apt-get update -qq && apt-get install -y wget gnupg openssl
> # Temurin JDK 21
> mkdir -p /etc/apt/keyrings
> wget -qO - https://packages.adoptium.net/artifactory/api/gpg/key/public | gpg --dearmor -o /etc/apt/keyrings/adoptium.gpg
> echo "deb [signed-by=/etc/apt/keyrings/adoptium.gpg] https://packages.adoptium.net/artifactory/deb trixie main" > /etc/apt/sources.list.d/adoptium.list
> apt-get update -qq && apt-get install -y temurin-21-jdk
> # users + dirs
> groupadd --system kafka 2>/dev/null || true
> id kafka >/dev/null 2>&1 || useradd --system -g kafka -s /usr/sbin/nologin -d /opt/kafka -M kafka
> install -d -o kafka -g kafka -m0755 /var/lib/kafka/data
> install -d -o root  -g kafka -m0750 /etc/nexus-kafka /etc/nexus-kafka/tls
> # Apache Kafka 3.8.1 (brokers + connect-mirror-maker)
> cd /opt && wget -q https://archive.apache.org/dist/kafka/3.8.1/kafka_2.13-3.8.1.tgz
> tar xzf kafka_2.13-3.8.1.tgz && ln -sfn kafka_2.13-3.8.1 kafka && rm kafka_2.13-3.8.1.tgz
> chown -R kafka:kafka /opt/kafka_2.13-3.8.1
> # Confluent Community 7.7.1 (Schema Registry / Connect / ksqlDB / REST)
> cd /opt && wget -q https://packages.confluent.io/archive/7.7/confluent-community-7.7.1.tar.gz
> tar xzf confluent-community-7.7.1.tar.gz && ln -sfn confluent-7.7.1 confluent && rm confluent-community-7.7.1.tar.gz
> chown -R kafka:kafka /opt/confluent-7.7.1
> ```
> **EXPECTED:** all install cleanly.
> **VERIFY:** `/opt/kafka/bin/kafka-topics.sh --version` → `3.8.1`; `java -version`
> → `21`.

> **Step 5.1.2 — Install the backplane-trust nftables ruleset**
> **WHERE:** each node, root shell.
> **WHY:** the deb13 baseline only opens SSH. The Kafka cluster has too many
> backplane ports to enumerate, so trust the whole **VMnet10** segment on `nic1`
> (per `feedback_cluster_template_nftables_backplane`), and open the operator/UI
> ports from VMnet11 per role.
> **WHAT:** add to `/etc/nftables.conf` `chain input` (before `counter drop`):
> ```
> iifname "nic1" ip saddr 192.168.10.0/24 accept comment "trusted cluster backplane (VMnet10)"
> ```
> Plus per-role VMnet11 UI ports as needed (Schema Registry `8081`, REST `8082`,
> Connect `8083`, ksqlDB `8088`), then `nft -c -f /etc/nftables.conf && systemctl reload nftables`.
> **EXPECTED:** ruleset valid.
> **VERIFY:** `nft list chain inet filter input | grep nic1` shows the backplane accept.

### 5.2 — KRaft bring-up (PLAINTEXT), per cluster

> **Step 5.2.1 — Render `server.properties` on each broker (PLAINTEXT)**
> **WHERE:** each of the 6 brokers, root shell.
> **WHY:** combined-mode KRaft, RF=3, all listeners on the backplane. Each broker
> gets its own `node.id` + its cluster's `controller.quorum.voters`. PLAINTEXT
> first; mTLS flips in §5.3.
> **WHAT (on `kafka-east-1` — `node.id=1`, voters = all 3 east nodes; substitute per node/cluster):**
> ```bash
> cat > /etc/nexus-kafka/server.properties <<'EOF'
> process.roles=broker,controller
> node.id=1
> controller.quorum.voters=1@192.168.10.21:9093,2@192.168.10.22:9093,3@192.168.10.23:9093
> listeners=PLAINTEXT://0.0.0.0:9092,CONTROLLER://192.168.10.21:9093
> advertised.listeners=PLAINTEXT://192.168.10.21:9092
> controller.listener.names=CONTROLLER
> inter.broker.listener.name=PLAINTEXT
> listener.security.protocol.map=PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
> log.dirs=/var/lib/kafka/data
> num.partitions=3
> default.replication.factor=3
> offsets.topic.replication.factor=3
> transaction.state.log.replication.factor=3
> transaction.state.log.min.isr=2
> min.insync.replicas=2
> auto.create.topics.enable=false
> EOF
> chown root:kafka /etc/nexus-kafka/server.properties ; chmod 640 /etc/nexus-kafka/server.properties
> ```
> For `kafka-west-*`, the voters are `1@192.168.10.24:9093,2@.25:9093,3@.26:9093`.
> **EXPECTED:** config on each broker with its real node.id/IP.
> **VERIFY:** `grep -E 'node.id|quorum.voters' /etc/nexus-kafka/server.properties`.

> **Step 5.2.2 — Format KRaft storage with the per-cluster UUID + create the systemd unit**
> **WHERE:** each broker, root shell.
> **WHY:** `kafka-storage format` writes `meta.properties` (the cluster UUID +
> node identity) into the log dir. All 3 nodes of a cluster format with the **same**
> UUID. `--ignore-formatted` makes re-runs idempotent.
> **WHAT (east uses `ZD-HhB5fQfioHydHQWiHkw`; west uses `FezdEIKlRCWP6nCSrHNJww`):**
> ```bash
> sudo -u kafka /opt/kafka/bin/kafka-storage.sh format \
>   --cluster-id ZD-HhB5fQfioHydHQWiHkw \
>   --config /etc/nexus-kafka/server.properties --ignore-formatted
>
> cat > /etc/systemd/system/kafka.service <<'EOF'
> [Unit]
> Description=Apache Kafka (KRaft)
> After=network-online.target
> Wants=network-online.target
> [Service]
> User=kafka
> Group=kafka
> Environment=KAFKA_HEAP_OPTS=-Xmx2g -Xms2g
> ExecStart=/opt/kafka/bin/kafka-server-start.sh /etc/nexus-kafka/server.properties
> Restart=on-failure
> LimitNOFILE=1048576
> [Install]
> WantedBy=multi-user.target
> EOF
> systemctl daemon-reload
> ```
> **EXPECTED:** `Formatting ... with metadata.version ...`.
> **VERIFY:** `cat /var/lib/kafka/data/meta.properties | grep cluster.id` → the UUID.

> **Step 5.2.3 — Big-bang start each cluster + verify RF=3 round-trip**
> **WHERE:** the 3 brokers of a cluster (start together), then any one.
> **WHY:** the controller quorum (a Raft group) needs all 3 to elect — start them
> together. Then prove a replicated produce/consume.
> **WHAT (east — from the build host, start all 3 together):**
> ```powershell
> '21','22','23' | ForEach-Object -Parallel { ssh nexusadmin@192.168.70.$_ 'sudo systemctl enable --now kafka' }
> ```
> ```bash
> # On kafka-east-1, after ~20s:
> K=/opt/kafka/bin ; BS=192.168.10.21:9092
> sudo $K/kafka-metadata-quorum.sh --bootstrap-server $BS describe --status   # leader + 3 voters
> sudo $K/kafka-topics.sh --bootstrap-server $BS --create --topic smoke --partitions 3 --replication-factor 3
> echo hello | sudo $K/kafka-console-producer.sh --bootstrap-server $BS --topic smoke
> sudo $K/kafka-console-consumer.sh --bootstrap-server $BS --topic smoke --from-beginning --max-messages 1 --timeout-ms 10000
> ```
> **EXPECTED:** quorum shows 1 leader + 3 voters; the consumer prints `hello`.
> **VERIFY:** repeat the whole §5.2 for **west** (`.24/.25/.26`, west UUID). Both
> clusters independently green.

### 5.3 — Broker mTLS (per cluster, parallel restart)

> **Step 5.3.1 — Create the `kafka-broker` PKI role (Guide 04 Part D)**
> **WHERE:** `vault-1`, root shell.
> **WHY:** server **and** client EKU (brokers are also clients of each other), all
> 15 tier hostnames allowed, 90-day leaves.
> **WHAT:**
> ```bash
> vault write pki_int/roles/kafka-broker \
>   allowed_domains='nexus.lab,kafka-east-1,kafka-east-2,kafka-east-3,kafka-west-1,kafka-west-2,kafka-west-3,schema-registry-1,schema-registry-2,kafka-rest-1,kafka-connect-1,kafka-connect-2,ksqldb-1,ksqldb-2,mm2-1,mm2-2,localhost' \
>   allow_subdomains=true allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
> ```
> **EXPECTED:** role written.
> **VERIFY:** `vault read pki_int/roles/kafka-broker | grep client_flag` → `true`.

> **Step 5.3.2 — Issue + place each broker's PEM keystore (PKCS#8 conversion)**
> **WHERE:** issue on `vault-1`; place on each broker.
> **WHY:** Kafka 3.8 reads `PEM` keystores, but Vault issues a **PKCS#1** key and
> Kafka needs **PKCS#8** — convert with `openssl pkcs8 -topk8`. The keystore is
> key + leaf + intermediate (full chain on the wire); the truststore is the
> intermediate alone.
> **WHAT (per broker — issue on vault-1, substitute host/IPs):**
> ```bash
> ISSUED=$(vault write -format=json pki_int/issue/kafka-broker \
>   common_name="<host>.nexus.lab" alt_names="<host>,localhost" \
>   ip_sans="<vmnet10>,<vmnet11>,127.0.0.1" ttl=2160h)
> echo "$ISSUED" | jq -r '.data.private_key' | openssl pkcs8 -topk8 -nocrypt -out /tmp/key.pk8.pem
> { echo "$ISSUED" | jq -r '.data.certificate'; echo "$ISSUED" | jq -r '.data.issuing_ca'; cat /tmp/key.pk8.pem; } > /tmp/keystore.pem
> echo "$ISSUED" | jq -r '.data.issuing_ca' > /tmp/truststore.pem
> # scp keystore.pem + truststore.pem to <host>:/etc/nexus-kafka/tls/ (owned root:kafka, 0640)
> ```
> **EXPECTED:** `keystore.pem` contains a `PRIVATE KEY` (not `RSA PRIVATE KEY`) +
> two certs.
> **VERIFY:** `grep -c 'BEGIN CERTIFICATE' /etc/nexus-kafka/tls/keystore.pem` →
> `2`; `grep -c 'BEGIN PRIVATE KEY' …` → `1` (PKCS#8).

> **Step 5.3.3 — Flip `server.properties` to mTLS + client-ssl, then parallel-restart each cluster**
> **WHERE:** each broker (config), then all 3 of a cluster together.
> **WHY:** all listeners → SSL, `ssl.client.auth=required`. **The restart must be
> parallel per cluster** — a TLS flip makes a node reject plain peers, so a
> sequential restart isolates it from the controller quorum (same lesson as
> Consul/Nomad TLS).
> **WHAT (rewrite `/etc/nexus-kafka/server.properties` on each broker):**
> ```bash
> # Same as §5.2.1 but the listener block becomes:
> #   listeners=SSL://0.0.0.0:9092,CONTROLLER://<vmnet10>:9093
> #   advertised.listeners=SSL://<vmnet10>:9092
> #   inter.broker.listener.name=SSL
> #   listener.security.protocol.map=SSL:SSL,CONTROLLER:SSL
> # plus the SSL block:
> cat >> /etc/nexus-kafka/server.properties <<'EOF'
> ssl.keystore.type=PEM
> ssl.keystore.location=/etc/nexus-kafka/tls/keystore.pem
> ssl.truststore.type=PEM
> ssl.truststore.location=/etc/nexus-kafka/tls/truststore.pem
> ssl.client.auth=required
> ssl.endpoint.identification.algorithm=https
> EOF
> # The CLI client config (so `sudo kafka-*.sh --command-config` works):
> cat > /etc/nexus-kafka/client-ssl.properties <<'EOF'
> security.protocol=SSL
> ssl.keystore.type=PEM
> ssl.keystore.location=/etc/nexus-kafka/tls/keystore.pem
> ssl.truststore.type=PEM
> ssl.truststore.location=/etc/nexus-kafka/tls/truststore.pem
> ssl.endpoint.identification.algorithm=https
> EOF
> chmod 640 /etc/nexus-kafka/client-ssl.properties
> ```
> (Edit the `listeners`/`advertised.listeners`/`inter.broker`/`map` lines in place
> to the SSL forms above.) Then, from the build host:
> ```powershell
> '21','22','23' | ForEach-Object -Parallel { ssh nexusadmin@192.168.70.$_ 'sudo systemctl restart kafka' }
> ```
> **EXPECTED:** the east quorum re-forms over SSL; `:9092` now SSL-only.
> **VERIFY:** `sudo $K/kafka-metadata-quorum.sh --bootstrap-server SSL://192.168.10.21:9092 --command-config /etc/nexus-kafka/client-ssl.properties describe --status`
> → leader + 3 voters. Repeat for **west**.

### 5.4 — Schema Registry HA + REST Proxy (0.H.3)

> **Step 5.4.1 — Issue ecosystem certs (PEM + `.p12`) for the ecosystem nodes**
> **WHERE:** issue on `vault-1`; place on each ecosystem node.
> **WHY:** ecosystem nodes are Kafka **clients** of the brokers (PEM keystore,
> §5.3.2 idiom). **But** Connect's + ksqlDB's embedded REST servers reject PEM —
> they additionally need a **`.p12`** keystore, and the truststore `.p12` must be
> built with **`keytool -importcert`** (an `openssl … -nokeys` truststore yields
> `trustAnchors must be non-empty`).
> **WHAT (on each ecosystem node, after the PEM pair is in place):**
> ```bash
> P='NexusKafkaP12!1'   # lab keystore password (production -> Vault KV)
> # keystore.p12 from the PEM key+leaf:
> openssl pkcs12 -export -in /etc/nexus-kafka/tls/keystore.pem -inkey /etc/nexus-kafka/tls/keystore.pem \
>   -name node -passout "pass:$P" -out /etc/nexus-kafka/tls/keystore.p12
> # truststore.p12 — MUST use keytool -importcert (not openssl -nokeys):
> keytool -importcert -noprompt -alias nexus-ca -file /etc/nexus-kafka/tls/truststore.pem \
>   -keystore /etc/nexus-kafka/tls/truststore.p12 -storetype PKCS12 -storepass "$P"
> chown root:kafka /etc/nexus-kafka/tls/*.p12 ; chmod 640 /etc/nexus-kafka/tls/*.p12
> ```
> **EXPECTED:** both `.p12` files created.
> **VERIFY:** `keytool -list -keystore /etc/nexus-kafka/tls/truststore.p12 -storepass "$P"`
> lists the `nexus-ca` cert (non-empty).

> **Step 5.4.2 — Configure + start the Schema Registry HA pair**
> **WHERE:** `schema-registry-1/2` (`.91/.92`), root shell.
> **WHY:** both nodes back onto the **east** brokers (SSL), share a `_schemas`
> topic, and elect a primary among themselves. The REST listener is HTTPS.
> **WHAT (on each — `host.name` is this node's IP):**
> ```bash
> cat > /etc/nexus-kafka/schema-registry.properties <<'EOF'
> listeners=https://0.0.0.0:8081
> kafkastore.bootstrap.servers=SSL://192.168.10.21:9092,SSL://192.168.10.22:9092,SSL://192.168.10.23:9092
> kafkastore.security.protocol=SSL
> kafkastore.ssl.keystore.type=PEM
> kafkastore.ssl.keystore.location=/etc/nexus-kafka/tls/keystore.pem
> kafkastore.ssl.truststore.type=PEM
> kafkastore.ssl.truststore.location=/etc/nexus-kafka/tls/truststore.pem
> kafkastore.topic=_schemas
> ssl.keystore.location=/etc/nexus-kafka/tls/keystore.p12
> ssl.keystore.type=PKCS12
> ssl.keystore.password=NexusKafkaP12!1
> ssl.truststore.location=/etc/nexus-kafka/tls/truststore.p12
> ssl.truststore.type=PKCS12
> ssl.truststore.password=NexusKafkaP12!1
> schema.registry.group.id=nexus-schema-registry
> EOF
> systemctl enable --now schema-registry   # unit baked disabled by the template; enable here
> ```
> **EXPECTED:** both nodes start, one becomes primary.
> **VERIFY:** `curl -sk https://127.0.0.1:8081/subjects` → `[]` (empty list, 200).

> **Step 5.4.3 — Configure + start the REST Proxy**
> **WHERE:** `kafka-rest-1` (`.88`), root shell.
> **WHY:** an HTTP front to the east brokers + Schema Registry.
> **WHAT:**
> ```bash
> cat > /etc/nexus-kafka/kafka-rest.properties <<'EOF'
> listeners=https://0.0.0.0:8082
> bootstrap.servers=SSL://192.168.10.21:9092,SSL://192.168.10.22:9092,SSL://192.168.10.23:9092
> client.security.protocol=SSL
> client.ssl.keystore.type=PEM
> client.ssl.keystore.location=/etc/nexus-kafka/tls/keystore.pem
> client.ssl.truststore.type=PEM
> client.ssl.truststore.location=/etc/nexus-kafka/tls/truststore.pem
> schema.registry.url=https://192.168.10.91:8081,https://192.168.10.92:8081
> ssl.keystore.location=/etc/nexus-kafka/tls/keystore.p12
> ssl.keystore.type=PKCS12
> ssl.keystore.password=NexusKafkaP12!1
> EOF
> systemctl enable --now kafka-rest
> ```
> **EXPECTED:** REST Proxy starts.
> **VERIFY:** `curl -sk https://127.0.0.1:8082/topics` → a JSON topic list.

### 5.5 — Kafka Connect + Debezium + ksqlDB (0.H.4)

> **Step 5.5.1 — Configure + start the Connect distributed cluster (+ Debezium)**
> **WHERE:** `kafka-connect-1/2` (`.95/.96`), root shell.
> **WHY:** a 2-node distributed Connect cluster (same `group.id`) for CDC via
> Debezium. The REST listener needs the **`.p12`** keystore.
> **WHAT (on each — note the `.p12` for the REST listener, PEM for the Kafka client):**
> ```bash
> cat > /etc/nexus-kafka/connect-distributed.properties <<'EOF'
> bootstrap.servers=SSL://192.168.10.21:9092,SSL://192.168.10.22:9092,SSL://192.168.10.23:9092
> group.id=nexus-connect
> key.converter=org.apache.kafka.connect.json.JsonConverter
> value.converter=org.apache.kafka.connect.json.JsonConverter
> offset.storage.topic=connect-offsets
> config.storage.topic=connect-configs
> status.storage.topic=connect-status
> offset.storage.replication.factor=3
> config.storage.replication.factor=3
> status.storage.replication.factor=3
> security.protocol=SSL
> ssl.keystore.type=PEM
> ssl.keystore.location=/etc/nexus-kafka/tls/keystore.pem
> ssl.truststore.type=PEM
> ssl.truststore.location=/etc/nexus-kafka/tls/truststore.pem
> # producer./consumer. inherit the top-level SSL via Connect's worker config
> listeners=https://0.0.0.0:8083
> listeners.https.ssl.keystore.location=/etc/nexus-kafka/tls/keystore.p12
> listeners.https.ssl.keystore.type=PKCS12
> listeners.https.ssl.keystore.password=NexusKafkaP12!1
> plugin.path=/opt/confluent/share/java,/opt/connect-plugins
> EOF
> # (drop the Debezium connector jars under /opt/connect-plugins)
> systemctl enable --now connect-distributed
> ```
> **EXPECTED:** both workers join the `nexus-connect` group.
> **VERIFY:** `curl -sk https://127.0.0.1:8083/connector-plugins | grep -i debezium`
> lists the Debezium connectors.

> **Step 5.5.2 — Configure + start ksqlDB (Java-21 security-manager flag)**
> **WHERE:** `ksqldb-1/2` (`.97/.98`), root shell.
> **WHY:** streaming SQL over the east brokers. **Java 21 removed
> `System.setSecurityManager()`**, so ksqlDB crashes on start unless
> `ksql.udf.enable.security.manager=false`.
> **WHAT:**
> ```bash
> cat > /etc/nexus-kafka/ksqldb-server.properties <<'EOF'
> listeners=https://0.0.0.0:8088
> bootstrap.servers=SSL://192.168.10.21:9092,SSL://192.168.10.22:9092,SSL://192.168.10.23:9092
> ksql.service.id=nexus_ksql_
> security.protocol=SSL
> ssl.keystore.type=PEM
> ssl.keystore.location=/etc/nexus-kafka/tls/keystore.pem
> ssl.truststore.type=PEM
> ssl.truststore.location=/etc/nexus-kafka/tls/truststore.pem
> ssl.keystore.location=/etc/nexus-kafka/tls/keystore.p12
> ssl.keystore.type=PKCS12
> ssl.keystore.password=NexusKafkaP12!1
> ksql.schema.registry.url=https://192.168.10.91:8081,https://192.168.10.92:8081
> ksql.udf.enable.security.manager=false
> EOF
> systemctl enable --now ksqldb-server
> ```
> **EXPECTED:** ksqlDB starts (no `SecurityManager is deprecated` crash).
> **VERIFY:** `curl -sk https://127.0.0.1:8088/info | jq .KsqlServerInfo.serverStatus`
> → `RUNNING`.

### 5.6 — MirrorMaker 2 DR pair + the exit gate (0.H.5)

> **Step 5.6.1 — Render `mm2.properties` + the `--clusters` drop-in on each MM2 node**
> **WHERE:** `mm2-1` (`.85`, east→west) and `mm2-2` (`.86`, west→east), root shell.
> **WHY:** both nodes share the same base (both clusters registered, per-cluster
> mTLS) but enable **exactly one flow**. In dedicated mode, `<alias>.ssl.*`
> auto-cascades to the producer/consumer/admin clients (write it once per alias).
> The systemd `--clusters <target>` flag pins where the Connect-internal topics
> land.
> **WHAT (on mm2-1 — `east->west.enabled=true`; on mm2-2 flip the two `enabled` lines):**
> ```bash
> cat > /etc/nexus-kafka/mm2.properties <<'EOF'
> clusters = east, west
> east.bootstrap.servers = 192.168.10.21:9092,192.168.10.22:9092,192.168.10.23:9092
> west.bootstrap.servers = 192.168.10.24:9092,192.168.10.25:9092,192.168.10.26:9092
> east->west.enabled = true
> east->west.topics = .*
> west->east.enabled = false
> west->east.topics = .*
> replication.factor = 3
> tasks.max = 4
> checkpoints.topic.replication.factor = 3
> heartbeats.topic.replication.factor = 3
> offset-syncs.topic.replication.factor = 3
> east.offset.storage.replication.factor = 3
> east.config.storage.replication.factor = 3
> east.status.storage.replication.factor = 3
> west.offset.storage.replication.factor = 3
> west.config.storage.replication.factor = 3
> west.status.storage.replication.factor = 3
> refresh.topics.interval.seconds = 10
> emit.heartbeats.interval.seconds = 5
> emit.checkpoints.interval.seconds = 5
> replication.policy.class = org.apache.kafka.connect.mirror.DefaultReplicationPolicy
> replication.policy.separator = .
> east.security.protocol = SSL
> east.ssl.keystore.type = PEM
> east.ssl.keystore.location = /etc/nexus-kafka/tls/keystore.pem
> east.ssl.truststore.type = PEM
> east.ssl.truststore.location = /etc/nexus-kafka/tls/truststore.pem
> east.ssl.endpoint.identification.algorithm = https
> west.security.protocol = SSL
> west.ssl.keystore.type = PEM
> west.ssl.keystore.location = /etc/nexus-kafka/tls/keystore.pem
> west.ssl.truststore.type = PEM
> west.ssl.truststore.location = /etc/nexus-kafka/tls/truststore.pem
> west.ssl.endpoint.identification.algorithm = https
> EOF
>
> install -d /etc/systemd/system/mm2.service.d
> # mm2-1 produces INTO west; mm2-2 -> --clusters east
> cat > /etc/systemd/system/mm2.service.d/10-clusters.conf <<'EOF'
> [Service]
> ExecStart=
> ExecStart=/opt/kafka/bin/connect-mirror-maker.sh /etc/nexus-kafka/mm2.properties --clusters west
> EOF
> systemctl daemon-reload
> ```
> **EXPECTED:** config + drop-in written (one flow enabled per node).
> **VERIFY:** `grep enabled /etc/nexus-kafka/mm2.properties` → exactly one `true`.

> **Step 5.6.2 — Start MM2 (sequential) + assert the exit-gate round-trip**
> **WHERE:** mm2-1 then mm2-2; assert from any broker.
> **WHY:** start mm2-1 first (clean journal diagnosis), then mm2-2. The exit gate:
> a record on `kafka-east` appears as `east.<topic>` on `kafka-west` (and the
> reverse). The `heartbeats` topic on the target cluster is the strongest single
> liveness signal.
> **WHAT:**
> ```bash
> # mm2-1, then mm2-2:
> systemctl enable --now mm2 ; sleep 20
> ```
> Exit-gate round-trip (CLI under `sudo`, mTLS):
> ```bash
> K=/opt/kafka/bin ; CC=/etc/nexus-kafka/client-ssl.properties
> # produce to east:
> echo "dr-test-$(date +%s)" | sudo $K/kafka-console-producer.sh \
>   --bootstrap-server SSL://192.168.10.21:9092 --producer.config $CC --topic smoke
> # consume the mirrored topic on west (~15-20s for MM2 to discover + mirror):
> sleep 20
> sudo $K/kafka-console-consumer.sh --bootstrap-server SSL://192.168.10.24:9092 \
>   --consumer.config $CC --topic east.smoke --from-beginning --max-messages 1 --timeout-ms 20000
> ```
> **EXPECTED:** mm2 services active; the west consumer prints the record produced
> to east, from topic **`east.smoke`** (source-alias prefix).
> **VERIFY:** `sudo $K/kafka-topics.sh --bootstrap-server SSL://192.168.10.24:9092 --command-config $CC --list | grep -E 'heartbeats|east\.'`
> shows `heartbeats` + the mirrored `east.*` topics.

---

## 6. Validation — by-hand acceptance smoke

From a broker (CLI under `sudo`, mTLS) and the build host.

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | East KRaft quorum (mTLS) | `sudo …kafka-metadata-quorum.sh --bootstrap-server SSL://192.168.10.21:9092 --command-config … describe --status` | leader + 3 voters |
| 2 | West KRaft quorum (mTLS) | same on `192.168.10.24:9092` | leader + 3 voters |
| 3 | Broker mTLS enforced | `curl http://192.168.10.21:9092` from off-broker | refused/closed (no PLAINTEXT) |
| 4 | RF=3 round-trip (each cluster) | produce→consume on `smoke` | record returns |
| 5 | Schema Registry HA | `curl -sk https://192.168.10.91:8081/subjects` + `…92:8081` | both `200` |
| 6 | REST Proxy | `curl -sk https://192.168.10.88:8082/topics` | JSON topic list |
| 7 | Connect + Debezium | `curl -sk https://192.168.10.95:8083/connector-plugins \| grep -i debezium` | Debezium listed |
| 8 | ksqlDB running | `curl -sk https://192.168.10.97:8088/info \| jq …serverStatus` | `RUNNING` |
| 9 | MM2 both flows | `sudo journalctl -u mm2 \| grep -i MirrorSourceConnector` on each | connector present, no SSLHandshakeException |
| 10 | **Exit gate: east→west mirror** | produce to `east:smoke`, consume `west:east.smoke` | record appears (DR proven) |
| 11 | **Exit gate: west→east mirror** | produce to `west:smoke`, consume `east:west.smoke` | record appears |

**1–9 green ⇒ the tier is up; 10–11 are the Phase 0.H exit gate** (bidirectional
DR replication proven).

---

## 7. Teardown / reset

```bash
# Stop role services, then delete VMs (Guide 00 §7 pattern).
for ip in 85 86 88 91 92 95 96 97 98; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now mm2 schema-registry kafka-rest connect-distributed ksqldb-server 2>/dev/null'; done
for ip in 21 22 23 24 25 26; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now kafka'; done
# then vmrun stop + deleteVM each of the 15.
```

> **Cold rebuild has no stale-KV prerequisite** (unlike Guide 05): the Kafka
> tier's Vault state is per-host AppRoles (re-issue regenerates secret-ids) + the
> `kafka-broker` PKI role (an upsert). A fresh apply **mints new KRaft UUIDs** —
> the old VMs (and their `meta.properties`) are gone — so use the same UUIDs only
> if you intend continuity; otherwise generate fresh with `kafka-storage random-uuid`.

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `java.nio.file.AccessDeniedException: /etc/nexus-kafka/...` | Kafka CLI run without `sudo` — can't traverse `0750 root:kafka` | Prefix the CLI with `sudo` (the Kafka edition of the Consul `/etc/consul.d/` lesson). |
| Broker crash-loop: `InvalidKeyException: algid parse error, not a sequence` | Vault issued a PKCS#1 key; Kafka's PEM parser needs PKCS#8 | Convert with `openssl pkcs8 -topk8` before building `keystore.pem` (§5.3.2). |
| Connect/ksqlDB REST won't start: `PEM KeyStore not available` / `Invalid value PEM` | their embedded REST servers reject `ssl.keystore.type=PEM` | Use the `.p12` keystore for the REST listener (§5.4.1/5.5). |
| `trustAnchors parameter must be non-empty` (Connect/ksqlDB) | `truststore.p12` built with `openssl … -nokeys` (empty cert bag) | Build it with `keytool -importcert` (§5.4.1). |
| ksqlDB won't start: `UnsupportedOperationException: The Security Manager is deprecated` | Java 21 removed `System.setSecurityManager()` | `ksql.udf.enable.security.manager=false` (§5.5.2). |
| KRaft controller can't elect after the TLS flip | sequential restart isolated the first SSL node from plain peers | Restart all 3 of a cluster **in parallel** (§5.3.3). |
| MM2 active but no `east.*` topics on west | MM2 discovers on a refresh interval | wait ~15–20s (`refresh.topics.interval.seconds=10`); check `sudo journalctl -u mm2`. |
| Mirrored topic is `orders` not `east.orders` | `replication.policy.class` isn't `DefaultReplicationPolicy` | use `DefaultReplicationPolicy` — never `IdentityReplicationPolicy` (loop risk). |
| Both flows running on one MM2 node | `mm2.properties` enabled both directions | exactly one `<src>->​<dst>.enabled = true` per node; the reverse `false`. |
| "heartbeats topic never appeared" but MM2 looks healthy | the probe ran the Kafka CLI without `sudo` | `sudo` the probe; MM2 itself is fine. |
| Vault HA reboot → node Vault Agents crashloop | `vault-transit` sealed after host reboot | recover Vault HA first (Guide 03 §8 boot-race); by-hand certs don't depend on the agent. |

---

### Cross-references

- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (kafka `.21`–`.26` + ecosystem decade)
- **Automated equivalents:** `nexus-infra-kafka/packer/kafka-node/` + `terraform/envs/kafka/role-overlay-*.tf`
- **Scaffolding pattern reused:** [`04-foundation-vault-pki-ldap.md`](./04-foundation-vault-pki-ldap.md) Part D
- **Previous guide:** [`05-orchestration-swarm-nomad-consul-portainer.md`](./05-orchestration-swarm-nomad-consul-portainer.md)
- **Next guide:** Guide 07 — OLTP · Redis Cluster (6-node, 3 shards × 2 replicas, mTLS). See [`INDEX.md`](../INDEX.md).
