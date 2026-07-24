# Guide 23 — Connect & Observe Cookbook

**How to connect to every node and every tier of the lab and actually *see the data*** — the right
GUI tool per engine (SSMS, DataGrip, NoSQLBooster, RedisInsight, Offset Explorer, the web consoles)
**and** the CLI equivalent, with **exact** endpoints, credentials, and commands.

> 🔄 **This is a living document.** Whenever we add a node, tier, credential, or tool, its connection
> details are added here in the same session. If something in the lab isn't listed below, it's a bug
> in this guide — fix it. (Standing rule; see `nexus-platform-plan` canon-docs currency.)

> Companion to the by-hand build guides 00–22: those build each tier; this one is how you log in and
> look around once it's alive. Jump to the tier you want.

---

## §0 · Shared prelude (do these once)

### 0.1 — Credentials: exact Vault path + field per tier

Every secret is a **field inside** a Vault KV entry. Set the session up once, then use the exact
command for the credential you need — **no guessing the field name**:

```powershell
$env:VAULT_ADDR   = 'https://192.168.70.121:8200'
$env:VAULT_CACERT = "$HOME\.nexus\vault-ca-bundle.crt"
$env:VAULT_TOKEN  = (Get-Content "$HOME\.nexus\vault-init.json" -Raw | ConvertFrom-Json).root_token
```

| Tier / use | Vault KV path | Field | One-shot command |
|---|---|---|---|
| **SQL Server** `sa` | `nexus/oltp/sqlserver/sa-password` | `content` | `vault kv get -field=content nexus/oltp/sqlserver/sa-password` |
| SQL Server operator (`nexus-cluster-admin`) | `nexus/oltp/sqlserver/operator-password` | `password` | `vault kv get -field=password nexus/oltp/sqlserver/operator-password` |
| **MongoDB RS** operator | `nexus/oltp/mongo/operator-password` | `content` | `vault kv get -field=content nexus/oltp/mongo/operator-password` |
| **Percona** `root` | `nexus/oltp/percona/root-password` | `content` | `vault kv get -field=content nexus/oltp/percona/root-password` |
| Percona app (`operator`) | `nexus/oltp/percona/operator-password` | `content` | `vault kv get -field=content nexus/oltp/percona/operator-password` |
| ProxySQL admin | `nexus/oltp/percona/proxysql-admin-password` | `content` | `vault kv get -field=content nexus/oltp/percona/proxysql-admin-password` |
| **Patroni PG** `postgres` | `nexus/oltp/patroni/postgres-superuser-password` | `content` | `vault kv get -field=content nexus/oltp/patroni/postgres-superuser-password` |
| Patroni operator | `nexus/oltp/patroni/operator-password` | `content` | `vault kv get -field=content nexus/oltp/patroni/operator-password` |
| **ClickHouse** admin | `nexus/analytics/clickhouse/admin-password` | `password` | `vault kv get -field=password nexus/analytics/clickhouse/admin-password` |
| ClickHouse app | `nexus/analytics/clickhouse/app-password` | `password` | `vault kv get -field=password nexus/analytics/clickhouse/app-password` |
| **StarRocks** `root` | `nexus/analytics/starrocks/root-password` | `password` | `vault kv get -field=password nexus/analytics/starrocks/root-password` |
| StarRocks (shared-data) `root` | `nexus/analytics/starrocks-sd/root-password` | `password` | `vault kv get -field=password nexus/analytics/starrocks-sd/root-password` |
| **Citus** superuser (`postgres`) | `nexus/citus/superuser-password` | `content` | `vault kv get -field=content nexus/citus/superuser-password` |
| Citus operator | `nexus/citus/operator-password` | `content` | `vault kv get -field=content nexus/citus/operator-password` |
| **Vitess** `root` | `nexus/vitess/mysql-root-password` | `content` | `vault kv get -field=content nexus/vitess/mysql-root-password` |
| Vitess app | `nexus/vitess/mysql-app-password` | `content` | `vault kv get -field=content nexus/vitess/mysql-app-password` |
| **Harbor** admin | `nexus/registry/harbor-admin` | `value` | `vault kv get -field=value nexus/registry/harbor-admin` |
| **Marquez** DB (`marquez`) | `nexus/platform-tools/marquez/db-password` | `value` | `vault kv get -field=value nexus/platform-tools/marquez/db-password` |
| Marquez PG replication | `nexus/platform-tools/marquez/replication-password` | `value` | `vault kv get -field=value nexus/platform-tools/marquez/replication-password` |
| Marquez PG superuser (`postgres`) | `nexus/platform-tools/marquez/superuser-password` | `value` | `vault kv get -field=value nexus/platform-tools/marquez/superuser-password` |
| **Portainer** admin | `nexus/portainer/admin-bcrypt` | `plaintext` | `vault kv get -field=plaintext nexus/portainer/admin-bcrypt` |
| **Consul** mgmt token | `nexus/swarm/consul-bootstrap-token` | `management_token` | `vault kv get -field=management_token nexus/swarm/consul-bootstrap-token` |
| **Nomad** mgmt token | `nexus/swarm/nomad-bootstrap-token` | `management_token` | `vault kv get -field=management_token nexus/swarm/nomad-bootstrap-token` |
| **Grafana** admin | `nexus/observability/grafana` | `admin-password` | `vault kv get -field=admin-password nexus/observability/grafana` |
| **MinIO** root | `nexus/lakehouse/minio` | `root-user` / `root-password` | `vault kv get -field=root-password nexus/lakehouse/minio` |
| **AD** DSRM (dc-nexus) | `nexus/foundation/dc-nexus` | `dsrm` | `vault kv get -field=dsrm nexus/foundation/dc-nexus` |
| Windows `nexusadmin` (domain) | *local file* `~/.nexus/nexusadmin-credentials.json` | `password` | `(Get-Content "$HOME\.nexus\nexusadmin-credentials.json" -Raw | ConvertFrom-Json).password` |

> **Redis** and the **sharded-Mongo** wire use **mTLS client certificates only** (no password) — see
> those sections. The `root_token` is the break-glass operator token; day-to-day you'd log in with an
> AD account mapped to a Vault policy (`vault login -method=ldap -username=<you>`).

### 0.2 — Trust the lab CA (so GUI tools + browsers don't warn)

```powershell
Import-Certificate -FilePath "$HOME\.nexus\vault-ca-bundle.crt" -CertStoreLocation Cert:\LocalMachine\Root
```

…or tick each tool's **"Trust server certificate"** box per connection.

### 0.3 — Names vs IPs

The build host is **WORKGROUP** and does not resolve `*.nexus.lab`. Connect by **IP** everywhere
(the tables below list them); engine certs carry IP SANs so TLS still validates. To use the
`*.nexus.lab` round-robin names, set your VMnet11 adapter's DNS to the gateway `192.168.70.1` or add
`C:\Windows\System32\drivers\etc\hosts` entries.

### 0.4 — Connect to ANY node (the pattern)

Every node has a **VMnet11 (service)** IP `192.168.70.X` and a matching **VMnet10 (backplane)** IP
`192.168.10.X` (same last octet). You reach nodes on **VMnet11**.

- **Linux (`deb13`)** — SSH as `nexusadmin` with the lab key (password fallback exists):
  ```bash
  ssh -i ~/.ssh/nexus_gateway_ed25519 nexusadmin@192.168.70.<X>
  # become root:  sudo -i
  ```
- **Windows (`ws2025-desktop`, `win11ent`)** — **RDP** to `192.168.70.<X>`, log in as
  `nexus\nexusadmin` (domain) or `.\Administrator` (local, DSRM/local-administrator creds from Vault):
  ```powershell
  mstsc /v:192.168.70.<X>
  ```

### 0.5 — Every node (pick an IP, apply the pattern above)

*(From `vms.yaml`. `L`=SSH, `W`=RDP. dc-nexus/dc-nexus-2 run at their DHCP-actual `.240`/`.242`, a
known drift from the canonical `.10`/`.11` in `vms.yaml`.)*

| Tier | Nodes (hostname → `192.168.70.X`) | Conn |
|---|---|---|
| **Edge / Foundation** | nexus-gateway `.1` · dc-nexus `.240` · dc-nexus-2 `.242` · vault-1/2/3 `.121/.122/.123` · vault-transit `.124` | L (gateway/vault), W (dc-*) |
| **SQL Server AG** | sql-fci-1/2 `.11/.12` · sql-ag-rep-1/2 `.13/.14` | W |
| **Kafka east** | kafka-east-1/2/3 `.21/.22/.23` | L |
| **Kafka west (DR)** | kafka-west-1/2/3 `.24/.25/.26` | L |
| **Kafka ecosystem** | schema-registry-1/2 `.91/.92` · kafka-connect-1/2 `.95/.96` · ksqldb-1/2 `.97/.98` · kafka-rest-1 `.88` · mm2-1/2 `.85/.86` | L |
| **Redis Cluster** | redis-1/2/3/4/5/6 `.81/.82/.83/.84/.87/.89` | L |
| **MongoDB RS** | mongo-1/2/3 `.71/.72/.73` | L |
| **MongoDB sharded** | mongo-cfg-1/2/3 `.74/.75/.76` · mongo-shard-1-1/2/3 `.77/.78/.79` · mongo-shard-2-1/2/3 `.80/.56/.57` · mongo-mongos-1/2 `.58/.59` | L |
| **Percona (Galera)** | pxc-node-1/2/3 `.51/.52/.53` · proxysql-1/2 `.54/.55` | L |
| **Patroni PostgreSQL** | pg-primary `.61` · pg-replica-1/2 `.62/.63` · etcd-1/2/3 `.64/.65/.66` · haproxy-pg-1/2 `.67/.68` | L |
| **ClickHouse** | ch-keeper-1/2/3 `.41/.42/.43` · ch-shard1-rep1/2 `.44/.45` · ch-shard2-rep1/2 `.46/.47` · ch-shard3-rep1/2 `.48/.49` | L |
| **StarRocks (SN)** | sr-fe-leader `.31` · sr-fe-follower-1/2 `.32/.33` · sr-be-1/2/3 `.34/.35/.36` | L |
| **StarRocks (SD)** | sr-sd-fe-1/2/3 `.37/.38/.39` · sr-sd-cn-1/2 `.30/.40` | L |
| **Vitess** | vitess-etcd-1/2/3 `.190/.191/.192` · vitess-control-1 `.193` · vitess-vtgate-1/2 `.194/.195` · vitess-shard1-tablet-1/2/3 `.196/.197/.198` · vitess-shard2-tablet-1/2/3 `.199/.200/.201` | L |
| **Citus** | citus-etcd-1/2/3 `.202/.203/.204` · citus-coord-1/2 `.205/.206` · citus-worker1-1/2 `.207/.208` · citus-worker2-1/2 `.209/.210` | L |
| **Orchestration** | swarm-manager-1/2/3 `.111/.112/.113` · swarm-worker-1/2/3 `.131/.132/.133` | L |
| **Observability** | prom-1/2 `.170/.171` · loki-1/2/3 `.172/.173/.174` · tempo-1/2/3 `.175/.176/.177` · grafana-1/2 `.178/.179` · grafana-pg-1/2 `.180/.181` · otel-collector-1/2 `.182/.183` | L |
| **Lakehouse** | minio-1/2/3/4 `.141/.142/.143/.144` · spark-master-1/2 `.140/.153` · spark-worker-1/2/3 `.145/.146/.154` · zookeeper-1/2/3 `.155/.156/.157` · iceberg-rest-1/2 `.147/.148` · iceberg-pg-1/2 `.149/.150` | L |
| **Registry** | registry-1/2 `.115/.116` · registry-pg-1/2 `.117/.118` | L |
| **Workstation** | nexusdesk-dev `.160` | W |
| **Platform-tools (Marquez)** | marquez `.127` · marquez-pg-1/2 `.134/.135` (VIP marquez-db `.136`) | L |
| **Platform-tools** *(reserved, not built)* | prefect-server `.125` · unleash-1 `.126` · backstage `.128` | L |

---

## §1 · Vault (secrets + PKI) — **Web UI**

- **Endpoint:** `https://192.168.70.121:8200/ui` (also `.122`/`.123`).  **Login:** *Token* = the
  `root_token` from `~/.nexus/vault-init.json`.
- **GUI:** open → *Method: Token* → paste → **Secrets → `nexus/`** to read any KV; **`pki_int/`** to
  issue/inspect certs.
- **CLI:** `vault kv list nexus/oltp` · `vault kv get nexus/oltp/sqlserver/sa-password`.

## §2 · Active Directory (`dc-nexus` `.240`, `dc-nexus-2` `.242`) — **RSAT / ADUC**

- **Endpoint:** LDAPS `192.168.70.240:636`; domain `nexus.lab`.
- **Creds:** `nexus\nexusadmin` (password in `~/.nexus/nexusadmin-credentials.json`); DSRM/local at
  `nexus/foundation/dc-nexus` (`dsrm`, `local-administrator`).
- **GUI:** install **RSAT AD DS Tools**, then (host isn't domain-joined):
  `runas /netonly /user:nexus\nexusadmin "mmc dsa.msc"` → *Change Domain* → `nexus.lab` (server
  `192.168.70.240`). Or just **RDP to `.240`** and use the built-in ADUC.
- **CLI:** `Get-ADUser -Server 192.168.70.240 -Credential (Get-Credential nexus\nexusadmin) -Filter *`

## §3 · SQL Server AG — OLTP (`OltpDb`, `nexus_demo`) — **SSMS**

- **Endpoint:** `192.168.70.16,1433` (FCI virtual) or `192.168.70.17,1433` (AG Listener). Nodes:
  sql-fci-1/2 `.11/.12`, sql-ag-rep-1/2 `.13/.14`.
- **Creds:** `sa` / `vault kv get -field=content nexus/oltp/sqlserver/sa-password`.
- **GUI (SSMS):** *Connect → Database Engine* → Server `192.168.70.16,1433` → *SQL Server
  Authentication* → `sa` + password → **Options → Encryption: Mandatory, ✅ Trust server certificate**.
  Expand `OltpDb → Tables` → right-click → *Select Top 1000 Rows*.
- **CLI:** `sqlcmd -S 192.168.70.16,1433 -U sa -P '<pw>' -N -C -Q "SELECT TOP 5 * FROM OltpDb.dbo.Customers"`
- **Pipeline walkthrough:** [`dataflow-studio/docs/demos/watch-the-pipeline.md`](https://github.com/grezap/dataflow-studio/blob/main/docs/demos/watch-the-pipeline.md).

## §4 · Redis Cluster — **RedisInsight**

- **Endpoint:** any of redis-1..6 `192.168.70.81/82/83/84/87/89:6379` (TLS-only).
- **Auth:** **mTLS client cert** (issue: `vault write -field=certificate pki_int/issue/… common_name=localhost`); no password.
- **GUI (RedisInsight):** *Add DB* → Host `192.168.70.81` Port `6379` → **TLS on**, import client
  cert+key + CA (`~/.nexus/vault-ca-bundle.crt`). Follows cluster redirects.
- **CLI (on a node):** `redis-cli -c --tls --cert /etc/nexus-redis/tls/server.crt --key /etc/nexus-redis/tls/server.key --cacert /etc/nexus-redis/tls/ca.crt -h 192.168.10.81 -p 6379 CLUSTER INFO`

## §5 · MongoDB replica set (`nexus-rs`) — **NoSQLBooster / Compass**

- **Endpoint:** mongo-1/2/3 `192.168.70.71/72/73:27017`, RS `nexus-rs` (mTLS + keyFile).
- **Creds:** `vault kv get -field=content nexus/oltp/mongo/operator-password`.
- **GUI (NoSQLBooster):** *Connect → From URI*
  `mongodb://192.168.70.71:27017,192.168.70.72:27017,192.168.70.73:27017/?replicaSet=nexus-rs&tls=true`
  → **TLS tab:** CA + client cert (PEM); auth SCRAM with the operator user.
- **CLI:** `mongosh --tls --tlsCAFile ca.pem --tlsCertificateKeyFile client.pem "mongodb://192.168.70.71/?replicaSet=nexus-rs" -u <user> -p <pw> --eval "rs.status().members.map(m=>m.name+':'+m.stateStr)"`

## §6 · MongoDB sharded cluster — **NoSQLBooster** (via `mongos`)

- **Endpoint:** routers mongo-mongos-1/2 `192.168.70.58/59:27017`.
- **Auth:** `nexus-sharded-admin`@`admin` over mTLS (the sharded env seeds its own admin at apply time;
  if `nexus/oltp/mongo-sharded/*` is empty, the wire is **cert-auth** — present the client x509 cert
  and auth against `$external`, or use the RS operator on a shard directly).
- **GUI:** connect to `mongodb://192.168.70.58:27017/?tls=true` (a router) with TLS → `sh.status()`.
- **CLI:** `mongosh --tls … "mongodb://192.168.70.58/admin" --eval "sh.status()"`

## §7 · Percona XtraDB Cluster (+ ProxySQL) — **DataGrip (MySQL)**

- **Endpoint:** VRRP VIP `192.168.70.50` — ProxySQL client `:6033`, admin `:6032`; PXC nodes `.51/.52/.53:3306`.
- **Creds:** `root` `vault kv get -field=content nexus/oltp/percona/root-password`; ProxySQL admin
  `-field=content nexus/oltp/percona/proxysql-admin-password`.
- **GUI (DataGrip):** *New → MySQL* → Host `192.168.70.50` Port `6033` → user+pw → **SSL: require, CA
  = vault-ca-bundle.crt**.
- **CLI:** `mysql -h 192.168.70.50 -P 6033 -u root -p --ssl-ca=ca.pem -e "SELECT @@hostname; SHOW DATABASES"`
  · cluster: `mysql -h 192.168.70.50 -P 6032 -u admin -p -e "SELECT hostname,status FROM runtime_mysql_servers"`

## §8 · PostgreSQL Patroni HA — **DataGrip (PostgreSQL)**

- **Endpoint:** VRRP VIP `192.168.70.60:5432` (HAProxy → current leader); nodes pg-primary `.61`,
  pg-replica-1/2 `.62/.63`.
- **Creds:** `postgres` `vault kv get -field=content nexus/oltp/patroni/postgres-superuser-password`.
- **GUI (DataGrip):** *New → PostgreSQL* → Host `192.168.70.60` Port `5432` User `postgres` + pw →
  **SSL: require, CA bundle**.
- **CLI:** `psql "host=192.168.70.60 port=5432 dbname=postgres user=postgres sslmode=verify-ca sslrootcert=ca.pem" -c "SELECT inet_server_addr(), pg_is_in_recovery()"`
  · leader: `curl -sk https://192.168.70.61:8008/cluster`

## §9 · ClickHouse — **DataGrip** / `clickhouse-client`

- **Endpoint:** data nodes ch-shard*-rep* `192.168.70.44–49`, HTTPS `:8443` / secure native `:9440`
  (keeper `.41/.42/.43`).
- **Creds:** `admin` `vault kv get -field=password nexus/analytics/clickhouse/admin-password`.
- **GUI (DataGrip):** *New → ClickHouse* → Host `192.168.70.44` Port `8443` → user+pw → **SSL on**.
  Query `system.clusters`, then `analytics.*`.
- **CLI (on a node):** `clickhouse-client --host 192.168.10.44 --port 9440 --secure --user admin --password <p> -q "SELECT * FROM system.clusters WHERE cluster='nexus_analytics'"`
  (add `--accept-invalid-certificate` when connecting to `localhost`, whose name may not match the node leaf).
- **⚠️ External HTTPS clients need root + INTERMEDIATE CA.** `~/.nexus/vault-ca-bundle.crt` is **root-only**,
  and CH presents just its leaf (issued by the Intermediate CA) → chain = *PartialChain*. Build a combined
  bundle: `cat ~/.nexus/vault-ca-bundle.crt <(vault read -field=certificate pki_int/cert/ca) > ca-chain.crt`
  and trust that. **No client cert** is required on `:8443` (password auth over server-TLS; mTLS is not enforced).
- **App schema:** `dataflow-studio` owns the **`analytics`** telemetry DB (`pipeline_events` [local + Distributed],
  `pipeline_latency_by_hour` MV, `cdc_lag_seconds`, `error_events`) — migrated by its DbUp runner (ADR-0005).

### §9.1 · ClickHouse as a **Kafka consumer** — native telemetry ingestion (dataflow-studio ADR-0008)

Since dataflow-studio Session 3D the six data nodes are also **Kafka clients**: they consume the
pipeline's own telemetry directly, with no .NET consumer on the path.

- **What was added on each data node `.44–.49`:**
  - `/etc/clickhouse-server/kafka/{client.crt,client.key,ca.crt}` — `root:clickhouse`, dir `0750`, files `0640`.
    `ca.crt` is the **root-only** `vault-ca-bundle.crt` (the brokers send their full chain, so root alone
    validates them — unlike CH's own HTTPS listener above, which needs root+intermediate).
  - `/etc/clickhouse-server/config.d/kafka-telemetry.xml` — the `<kafka>` block
    (`security_protocol=ssl` + the three paths). **TLS lives here, never in DDL.**
  - Requires `sudo systemctl restart nexus-clickhouse-server` to take effect.
- **Identity:** its own principal **`CN=clickhouse-telemetry`**, issued from the dedicated role
  `pki_int/roles/kafka-clickhouse-client` (client-auth only, 168 h).
  ⚠️ The shared `kafka-broker` role **cannot** issue this CN (`allow_any_name=false`) — and must not be
  patched, since a partial `vault write` resets a role's other fields.
  ```bash
  vault write -format=json pki_int/issue/kafka-clickhouse-client common_name=clickhouse-telemetry ttl=168h
  ```
- **ACLs** (grant from a broker with `--command-config /etc/nexus-kafka/client-ssl.properties`):
  topic-prefix `dfs.telemetry` **READ + DESCRIBE**; group-prefix `dfs-clickhouse` **READ**.
- **Topics consumed:** `dfs.telemetry.pipeline_events` · `dfs.telemetry.cdc_lag` ·
  `dfs.telemetry.error_events` (JSON / `JSONEachRow`, RF=3, 1 partition each).
- **Objects:** three `Kafka`-engine source tables + three `MaterializedView`s
  (`*_kafka` / `*_kafka_mv`) feeding `pipeline_events_local`, `cdc_lag_seconds`, `error_events`.
  ```sql
  SELECT name, engine FROM system.tables
  WHERE database='analytics' AND (name LIKE '%_kafka' OR name LIKE '%_kafka_mv');
  ```
- **See the telemetry:**
  ```sql
  SELECT pipeline, stage, count(), round(avg(duration_ms),1) FROM analytics.pipeline_events GROUP BY pipeline, stage;
  SELECT source, count(), round(min(lag_seconds),1), round(max(lag_seconds),1) FROM analytics.cdc_lag_seconds GROUP BY source;
  -- the MV has no Distributed wrapper -> read it with cluster()
  SELECT stage, countMerge(events_state) AS events,
         round(quantilesMerge(0.5,0.95,0.99)(p_state)[1],1) AS p50
  FROM cluster('nexus_analytics', analytics.pipeline_latency_by_hour) GROUP BY stage ORDER BY events DESC;
  ```
- **Troubleshooting:** if a `*_kafka` table ingests nothing, check
  `sudo journalctl -u nexus-clickhouse-server | grep -i kafka` — an ACL/TLS problem shows up there, not
  in the query. With one partition per topic only **one** node in each consumer group holds the
  assignment; that is expected, and its rows replicate within the shard.

### §9.2 · DataFlow Studio OTLP → Tempo (traces) + Prometheus (metrics) (dataflow-studio ADR-0010, E16)

Same pipeline, seen as OpenTelemetry. The curation + warehouse-sink engines export **spans** and the
emit **counter** over OTLP to the Phase-0.I collector; each run's OTel trace id equals the ClickHouse
`pipeline_events.trace_id`, so §9.1 and this section are the **same run** two ways.

- **Run it:** `dataflow-studio/scripts/dfs-otel-demo.ps1` (curation; `-IncludeWarehouseSink` adds the DWH load).
- **Endpoint:** `DFS_OTLP_ENDPOINT=https://192.168.70.182:4318` — the **OTel collector** HTTP/protobuf
  receiver (`otel-collector-1/2` `.182`/`.183`, RR-DNS `otel.nexus.lab`; use the **IP** from a WORKGROUP
  host — the leaf has an IP SAN, `otel.nexus.lab` won't resolve). **Server-TLS only, no client cert.**
- **Trust:** `DFS_OTLP_CACERT=~/.nexus/vault-ca-bundle.crt` (the NexusPlatform **root**; the collector
  serves its own intermediate, so the root alone completes the chain).
- **Traces (Tempo):** Grafana (`https://192.168.70.184:3000`, admin / `nexus/observability/grafana` field
  `admin-password`) → Explore → Tempo → Service Name `dfs-curation`. Or SSH-local on a tempo node:
  ```bash
  sudo curl -s --cacert /etc/nexus-tempo/tls/ca.crt \
    'https://127.0.0.1:3200/api/search?q=%7B%20resource.service.name%3D%22dfs-curation%22%20%7D&limit=5'
  ```
- **Metrics (Prometheus):** `dfs_telemetry_emitted_records_total` (by `stream`). **Query BOTH proms**
  (`.170` + `.171`): the collector RR-DNS-writes to one of two independent Prometheus instances, so a
  remote-written metric lands on one node. Prometheus must run with `--web.enable-remote-write-receiver`
  (enabled on `prom-1/2` 2026-07-24; baked into the obs Packer image).
  ```bash
  sudo curl -s --cacert /etc/nexus-prometheus/tls/ca.crt \
    'https://127.0.0.1:9090/api/v1/query?query=dfs_telemetry_emitted_records_total'
  ```

### §9.3 · DataFlow Studio OpenLineage → Marquez (data lineage) (dataflow-studio ADR-0011, E16)

The curation + warehouse-sink runs POST OpenLineage run events to **Marquez** (Phase 0.Q), so the
`oltp.* → dfs.* → dwh.*` dataset graph is queryable — the "if I change this source, what breaks
downstream?" view. Each run's OpenLineage runId is its OTel trace id, so a run is the same entity as its
Tempo trace (§9.2) and its ClickHouse `pipeline_events` (§9.1).

- **Run it:** `dataflow-studio/scripts/dfs-lineage-demo.ps1` (`-SkipWarehouseSink` for the raw→curated leg only).
- **Endpoint:** `DFS_MARQUEZ_ENDPOINT=https://192.168.70.127` — the **Marquez** nginx TLS front door
  (`marquez` `.127`; the emitter appends `/api/v1/lineage`). Use the **IP** from a WORKGROUP host (the
  leaf carries an IP SAN; `marquez.nexus.lab` won't resolve). **Server-TLS only, no client cert.**
- **Trust:** `DFS_MARQUEZ_CACERT=~/.nexus/vault-ca-bundle.crt` (the NexusPlatform **root**; the front door
  serves its own intermediate, so the root completes the chain).
- **Read the graph back** (SSH-local-curl on the marquez node; the `/etc/nexus-marquez/tls/ca.crt` is
  root-only, so use the world-readable copy):
  ```bash
  CA=/etc/ssl/certs/platform-tools-ca.pem
  CURL="curl -sS --cacert $CA --resolve marquez.nexus.lab:443:127.0.0.1"; API=https://marquez.nexus.lab/api/v1
  $CURL "$API/namespaces/dataflow-studio/jobs"     | grep -o '"name":"[^"]*"' | sort -u   # curation, warehouse-sink
  $CURL "$API/lineage?nodeId=dataset:dataflow-studio:oltp.OltpDb.dbo.Customers&depth=10"   # downstream graph
  ```
- **Browser:** `https://192.168.70.127` (namespace `dataflow-studio`) — 2 jobs + 29 datasets.
- **Gotcha:** if `:443` times out right after power-on, the marquez VM was **suspended** (not stopped) —
  the containers are stale + Docker's DNAT is gone; `sudo systemctl restart docker` on `.127` fixes it.

## §10 · StarRocks — **DataGrip (MySQL protocol)**

- **Endpoint:** FE MySQL `192.168.70.31:9030` (shared-nothing; shared-data FE `.37:9030`), HTTP `:8030`.
- **Creds:** `root` `vault kv get -field=password nexus/analytics/starrocks/root-password`.
  MySQL wire is **TLS-off** — clients must set `--skip-ssl` (mysql CLI) / `SslMode=None` (MySqlConnector).
- **App schema:** `dataflow-studio` owns the **`dwh`** Kimball star (5 dims + 4 facts + `bridge_customer_seg`,
  SCD2 on `dim_customer`/`dim_product`) + **`analytics`** serving (view `dim_customer_current`) — DbUp (ADR-0005).
- **GUI (DataGrip):** *New → MySQL* → Host `192.168.70.31` Port `9030` → `root` + pw. Run
  `SHOW FRONTENDS; SHOW BACKENDS;`.
- **CLI:** `mysql -h 192.168.70.31 -P 9030 -u root -p -e "SHOW BACKENDS\G"` · Web `http://192.168.70.31:8030`.

## §11 · Kafka ecosystem — **Offset Explorer** (+ console tools)

- **Endpoints:** brokers `192.168.10.21/22/23:9092` (mTLS, backplane); Schema Registry `192.168.70.91/92:8081`;
  Connect `192.168.70.95/96:8083`; ksqlDB `:8088`; REST Proxy `.88:8082`.
- **Auth:** mTLS cert from `pki_int/issue/kafka-broker` (CN `localhost` allowed) + **ACLs** (grant with
  `kafka-acls.sh` from a broker — only broker principals are super-users).
- **GUI (Offset Explorer):** *Add Cluster* → Bootstrap `192.168.10.21:9092` → **Security: SSL**, import
  the client keystore (cert+key) + truststore (CA). Browse topics → *Data*.
- **CLI (on a broker, `/tmp/client.properties` = SSL + PEM keystore):** `kafka-topics.sh --bootstrap-server 192.168.10.21:9092 --command-config /tmp/client.properties --list`
- **Schema Registry:** `curl -sk https://192.168.70.91:8081/subjects` · **Connect:** `curl -sk https://192.168.70.95:8083/connectors`

## §11a · Kafka Connect + Debezium (observe CDC connectors)

Debezium runs as a **connector inside Kafka Connect** (kafka-connect-1/2 `192.168.70.95/96:8083`,
HTTPS, **server-TLS only — `client.auth=none`**, so you read/write it with a plain `curl -sk`).
There is no native Connect GUI in the lab; you observe Debezium three ways: the **REST API**, a **GUI
that speaks the Connect API** (Conduktor / Redpanda Console), and by **consuming the topics** it writes.

### Reach the API

Windows `curl` + schannel mishandles the IP-SAN cert, so **SSH to a connect node and curl
`localhost`** (or open the browser after trusting the CA, §0.2):

```bash
ssh -i ~/.ssh/nexus_gateway_ed25519 nexusadmin@192.168.70.95
```

### REST API — the exact calls

| See | Command (on the node; or browse `https://192.168.70.95:8083/…`) |
|---|---|
| all connectors | `curl -sk https://localhost:8083/connectors` |
| **connector + task health** | `curl -sk https://localhost:8083/connectors/oltp-cdc/status` → `"state":"RUNNING"` (or `FAILED` + a `trace`) |
| its config | `curl -sk https://localhost:8083/connectors/oltp-cdc/config` |
| topics it writes to | `curl -sk https://localhost:8083/connectors/oltp-cdc/topics` |
| its tasks | `curl -sk https://localhost:8083/connectors/oltp-cdc/tasks` |
| is Debezium loaded | `curl -sk https://localhost:8083/connector-plugins \| grep -o SqlServerConnector` |
| pause / resume | `curl -sk -X PUT https://localhost:8083/connectors/oltp-cdc/pause` (or `/resume`) |
| restart (+ tasks) | `curl -sk -X POST 'https://localhost:8083/connectors/oltp-cdc/restart?includeTasks=true'` |

Pretty-print any of these with `| python3 -m json.tool` or `| jq`. **Live-tail** the worker log to
watch snapshot → streaming and any errors as they happen:

```bash
sudo journalctl -u connect-distributed.service -f | grep -iE 'oltp-cdc|sqlserver|snapshot|streaming|ERROR'
```

### What Debezium is producing (consume the topics)

From a broker node with the mTLS `/tmp/client.properties` (see §11):

| Topic | What it is |
|---|---|
| `oltp.OltpDb.dbo.*` (all 10 order-flow tables) | the **raw CDC events** — JSON envelope `{before, after, op, source}`. The `oltp-cdc` connector captures Customers, ProductCategories, Products, Warehouses, CustomerAddresses, Orders, OrderLines, Transactions, Shipments, ProductInventory (`decimal.handling.mode=string`, `time.precision.mode=connect`). |
| `dfs.<entity>.changed.v1` (10 topics) | the **curated Avro** events produced by the dataflow-studio curation worker (`dfs.customers.changed.v1`, `dfs.orders.changed.v1`, …). Consumed by the StarRocks DWH sink. |
| `dfs.telemetry.{pipeline_events,cdc_lag,error_events}` | the pipeline's **own telemetry** — **JSON, not Avro** (stage latency, CDC lag, structured errors). Consumed **natively by ClickHouse** via Kafka-engine tables, not by any .NET worker (see §9.1). |
| `schemahistory.oltp` | Debezium's internal **schema history** (DDL) |
| `oltp` | the connector's schema-change topic |

> **dataflow-studio pipeline tools:** `scripts/dfs-seed.ps1` seeds a representative order-flow dataset
> into OltpDb; `scripts/dfs-curate.ps1` drains the raw CDC into the curated Avro topics;
> `scripts/dfs-warehouse-sink.ps1` loads the StarRocks `dwh` Kimball star (SCD2 dims + facts) from the
> curated topics; `scripts/dfs-trace.ps1` follows one record across all faces. The curation consumer
> group is `dfs-curation*` (Kafka ACL: group-prefix `dfs-curation` READ for `User:CN=localhost`,
> granted with `kafka-acls.sh --command-config /etc/nexus-kafka/client-ssl.properties`); the DWH sink
> reuses that authorized group prefix. **Kafka client PEM from Vault:** extract with pwsh
> `ConvertFrom-Json` + `Set-Content -NoNewline` (a `grep`/`sed` extraction of `vault -format=json`
> silently yields empty files → the broker replies `tlsv13 alert certificate required`).

```bash
sudo /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server 192.168.10.21:9092 \
  --consumer.config /tmp/client.properties --topic oltp.OltpDb.dbo.Customers --from-beginning
```

Insert a row in SSMS (§3) and watch it appear here within a few seconds — that's the CDC → Debezium
→ Kafka hop, live.

### GUI options (they speak the Connect REST API)

- **Conduktor** — the *Kafka Connect* view: add a Connect cluster URL `https://192.168.70.95:8083`
  (trust the CA) → see every connector, its status/config, and restart / pause buttons.
- **Redpanda Console** (kowl) or **kafka-connect-ui** (lensesio) — web UIs that list connectors +
  tasks and let you browse the produced topics + messages; run either as a container pointed at the
  brokers + Connect (`:8083`) + Schema Registry (`:8081`).

### The SQL side of CDC

Debezium reads SQL Server's CDC change tables — see them directly in **SSMS** (§3):
`SELECT * FROM OltpDb.cdc.dbo_Customers_CT ORDER BY __$start_lsn DESC;`. Full end-to-end walkthrough:
[`dataflow-studio/docs/demos/watch-the-pipeline.md`](https://github.com/grezap/dataflow-studio/blob/main/docs/demos/watch-the-pipeline.md).

## §11b · Schema Registry · ksqlDB · REST Proxy · MirrorMaker 2 · consumer lag

The rest of the Kafka ecosystem (Guide 06) and how to watch it.

**Schema Registry** (schema-registry-1/2 `192.168.70.91/92:8081`, HTTPS):
```bash
curl -sk https://192.168.70.91:8081/subjects                                         # all subjects
curl -sk https://192.168.70.91:8081/subjects/dfs.customers.changed.v1-value/versions/latest
curl -sk https://192.168.70.91:8081/config                                           # global compatibility
```
GUI: Conduktor / Redpanda Console include a Schema Registry browser.

**ksqlDB** (ksqldb-1/2 `192.168.70.97/98:8088`) — streaming SQL. On the node:
```bash
sudo ksql --config-file /etc/nexus-ksqldb/ksql-cli.properties https://localhost:8088
#   ksql> SHOW STREAMS;  SHOW TABLES;  PRINT 'oltp.OltpDb.dbo.Customers' FROM BEGINNING;
```
REST: `curl -sk https://192.168.70.97:8088/info` · `/ksql` (statements) · `/query-stream`.

**REST Proxy** (kafka-rest-1 `192.168.70.88:8082`) — HTTP gateway to Kafka:
```bash
curl -sk https://192.168.70.88:8082/topics
curl -sk https://192.168.70.88:8082/topics/oltp.OltpDb.dbo.Customers
```

**MirrorMaker 2** (mm2-1 east→west `.85`, mm2-2 west→east `.86`) — cross-cluster DR. Watch the process
log + the mirrored topics on the DR cluster:
```bash
ssh …@192.168.70.85 'sudo journalctl -u nexus-mm2 -f'
# east topics appear as east.<topic> on kafka-west (bootstrap 192.168.10.24:9092):
sudo /opt/kafka/bin/kafka-topics.sh --bootstrap-server 192.168.10.24:9092 \
  --command-config /tmp/client.properties --list | grep '^east\.'
```

**Consumer groups + lag** (how far behind a consumer is — the key pipeline-health signal):
```bash
sudo /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server 192.168.10.21:9092 \
  --command-config /tmp/client.properties --describe --all-groups     # LAG per partition
```

**kafka-west (DR cluster)** — same tools, bootstrap `192.168.10.24/25/26:9092`.

## §12 · Vitess (sharded MySQL) — **DataGrip (via `vtgate`)**

- **Endpoint:** vtgate MySQL `192.168.70.194:15306` (also `.195`); keyspace `commerce`.
- **Creds:** `vault kv get -field=content nexus/vitess/mysql-app-password` (root: `mysql-root-password`).
- **GUI (DataGrip):** *New → MySQL* → Host `192.168.70.194` Port `15306` → user+pw → SSL. `SHOW VITESS_SHARDS;`
- **CLI:** `mysql -h 192.168.70.194 -P 15306 -u <user> -p -e "SHOW VITESS_TABLETS"` · admin (on control
  node `.193`): `vtctldclient --server 192.168.10.193:15999 GetTablets`
- **Web:** vtctld admin UI `http://192.168.70.193:15000` · VTOrc (auto-reparent) UI `http://192.168.70.193:16000`.

## §13 · Citus (sharded PostgreSQL) — **DataGrip (coordinator VIP)**

- **Endpoint:** coordinator VRRP VIP `192.168.70.211:5432` (nodes citus-coord-1/2 `.205/.206`; workers
  behind VIPs `.212/.213`).
- **Creds:** `postgres` `vault kv get -field=content nexus/citus/superuser-password`.
- **GUI (DataGrip):** *New → PostgreSQL* → Host `192.168.70.211` Port `5432` → `postgres` + pw → SSL.
  `SELECT * FROM pg_dist_node; SELECT * FROM citus_shards;`
- **CLI:** `psql "host=192.168.70.211 port=5432 user=postgres sslmode=verify-ca sslrootcert=ca.pem" -c "SELECT nodename,nodeport FROM pg_dist_node"`

## §14 · Swarm / Nomad / Consul / Portainer — **Web UIs**

- **Portainer:** `https://192.168.70.111:9443` (also `.112/.113`). Login `admin` /
  `vault kv get -field=plaintext nexus/portainer/admin-bcrypt`.
- **Consul UI:** `https://192.168.70.111:8501/ui` → token `vault kv get -field=management_token nexus/swarm/consul-bootstrap-token`.
- **Nomad UI:** `https://192.168.70.111:4646/ui` → token `vault kv get -field=management_token nexus/swarm/nomad-bootstrap-token`.
- **CLI (on a manager):** `docker node ls` · `consul members` · `nomad server members`.

## §15 · Observability (Grafana LGTM) — **Grafana Web UI**

- **Endpoint:** `https://192.168.70.184:3000` (VRRP VIP; nodes grafana-1/2 `.178/.179`).
- **Creds:** `admin` / `vault kv get -field=admin-password nexus/observability/grafana`.
- **GUI:** log in → **Explore** → **Prometheus** (`up`, `node_load1`), **Loki** (`{fleet="nexusplatform"}`),
  **Tempo** (traces). Dashboards under **Dashboards** — this is how you watch the whole fleet.
- **CLI:** `curl -sk https://192.168.70.170:9090/api/v1/query?query=up` (Prometheus, on prom-1).
- **Direct web UIs (bypass Grafana):** Prometheus `https://192.168.70.170:9090` (also `.171`) →
  *Status → Targets* to see every scraped node; **Alertmanager** `https://192.168.70.170:9093` →
  firing/silenced alerts.

## §16 · Lakehouse (MinIO · Spark · Iceberg) — **Web consoles**

- **MinIO console:** `https://192.168.70.141:9001` (S3 API `:9000`; nodes minio-1..4 `.141–.144`). Login
  `vault kv get -field=root-user nexus/lakehouse/minio` / `-field=root-password`. Browse buckets
  (`warehouse`, `loki`, `tempo`, `harbor`, `starrocks`).
- **Spark master UI:** `http://192.168.70.140:8080` (ZK-elected active; standby `.153`) — running apps,
  workers, completed jobs. **Worker UIs:** `http://192.168.70.145:8081` (also `.146/.154`). The ZK
  ensemble backing master HA is in §18.
- **Iceberg/Nessie:** REST `https://192.168.70.147:19120` (also `.148`); catalog DB VIP `.151`.
- **CLI (on a node):** `mc alias set lab https://192.168.10.141:9000 <ak> <sk>` then `mc ls lab/warehouse`.

## §17 · Registry (Harbor) — **Harbor Web UI**

- **Endpoint:** `https://192.168.70.115` (also `.116`; `:443`). Datastore registry-pg-1/2 `.117/.118`.
- **Creds:** `admin` / `vault kv get -field=value nexus/registry/harbor-admin` (or AD SSO via Vault OIDC).
- **GUI:** log in → **Projects** → a project → **Repositories** (images, Trivy scans, cosign signatures).
- **CLI:** `docker login registry.nexus.lab -u admin` then `docker pull registry.nexus.lab/<proj>/<img>:<tag>`.

## §17a · Marquez (OpenLineage lineage) — **Web UI + REST** (0.Q.1)

The platform's **OpenLineage** backend (Phase 0.Q.1, ADR-0043): a Marquez app node (`marquez` `.127`,
Docker CE + docker-compose) behind an nginx TLS terminator, backed by a **dedicated PG 17 HA pair**
(marquez-pg-1/2 `.134/.135`) with a keepalived VRRP VIP `marquez-db.nexus.lab` `.136` that Marquez
reaches `sslmode=verify-full`. It answers *"if I change `raw.orders`, what breaks downstream?"*

- **Endpoint (web UI):** `https://marquez.nexus.lab` — nginx `:443` → web `:3000`, api `:5000`, admin `:5001`.
  From the WORKGROUP build host, browse by name via a `hosts` entry (`192.168.70.127 marquez.nexus.lab`)
  or point VMnet11 DNS at the gateway (§0.3); the front-door leaf carries the `marquez.nexus.lab` SAN.
- **Creds:** the API is unauthenticated on the lab; the **datastore** login is
  `vault kv get -field=value nexus/platform-tools/marquez/db-password` (user `marquez`, db `marquez`).
- **GUI — the lineage graph (browser):** open `https://marquez.nexus.lab` → pick namespace
  `nexus-lineage-demo` (or `nexus-lineage`) → the graph view shows jobs (`curate-orders`, `load-dwh`) and
  datasets (`raw.orders` → `curated.orders` → `dwh.fact_order`) with the edges between them.
- **GUI — the datastore (DataGrip):** *New → PostgreSQL* → Host `192.168.70.136` (VIP) Port `5432` →
  user `marquez` + pw (above) → SSL, **Mode `verify-full`** (CA = the lab bundle) → database `marquez`.
- **CLI — see the data (REST read model, SSH-local-curl on `.127`):**
  ```bash
  ssh -i ~/.ssh/nexus_gateway_ed25519 nexusadmin@192.168.70.127
  CA=/etc/ssl/certs/platform-tools-ca.pem ; R='--resolve marquez.nexus.lab:443:192.168.70.127'
  curl -sS --cacert $CA $R https://marquez.nexus.lab/api/v1/namespaces
  curl -sS --cacert $CA $R https://marquez.nexus.lab/api/v1/namespaces/nexus-lineage-demo/jobs
  curl -sS --cacert $CA $R https://marquez.nexus.lab/api/v1/namespaces/nexus-lineage-demo/datasets
  # downstream impact from a dataset:
  curl -sS --cacert $CA $R 'https://marquez.nexus.lab/api/v1/lineage?nodeId=dataset:nexus-lineage-demo:raw.orders&depth=10'
  ```
- **CLI — emit a run graph:** POST OpenLineage events to `POST /api/v1/lineage` (an OpenLineage client,
  or dataflow-studio/Prefect/Spark with `OPENLINEAGE_URL=https://marquez.nexus.lab`). The committed
  end-to-end data-flow demo is
  [`nexus-infra-platform-tools/scripts/marquez-lineage-demo.ps1`](https://github.com/grezap/nexus-infra-platform-tools/blob/main/scripts/marquez-lineage-demo.ps1)
  (emits the 2-job / 3-dataset graph, every POST `201`, then reads it back).
- **CLI — the datastore (psql):** `sudo -u postgres psql -h /var/run/postgresql -d marquez` on `.134`,
  then inspect the lineage tables (`\dt`, e.g. `SELECT name FROM jobs;` · `SELECT name FROM datasets;`).
- **Observe:** on `.127` — `docker compose -f /etc/nexus-marquez/docker-compose.yml ps` (api/web/nginx Up)
  and `docker compose … logs -f marquez` (ingest activity). On `.134` — PG streaming replication
  `sudo -u postgres psql -d marquez -c 'SELECT state,sync_state FROM pg_stat_replication'` → `streaming`.
  VIP holder (either PG node): `ip -brief addr | grep 136`.

---

## §18 · Coordination layer — etcd · ZooKeeper · ClickHouse Keeper · Patroni · HAProxy

The consensus / DCS / LB pieces the HA tiers stand on — normally invisible; here's how to look.

**Patroni REST** (PG HA state; `:8008` on each PG node) — the clearest HA view:
```bash
curl -sk https://192.168.70.61:8008/cluster     # leader + replicas + replication lag (JSON)
curl -sk https://192.168.70.61:8008/patroni      # this node's role/state
```
Citus uses the same on the coordinator/worker Patronis (`.205` / `.207` / `.209`).

**HAProxy stats** (Patroni LB; browser): `https://192.168.70.67:8404` — basic-auth `admin` /
`vault kv get -field=content nexus/oltp/patroni/haproxy-stats-password`. Shows which backend is UP (the leader).

**etcd** (DCS for Patroni `.64–.66`, Citus `.202–.204`, Vitess `.190–.192`) — on an etcd node:
```bash
sudo etcdctl --endpoints=https://192.168.10.64:2379 \
  --cacert=/etc/nexus-etcd/tls/ca.crt --cert=/etc/nexus-etcd/tls/server.crt --key=/etc/nexus-etcd/tls/server.key \
  --user root:$(vault kv get -field=content nexus/oltp/patroni/etcd-root-password) \
  endpoint status --write-out=table                 # leader, raft term, db size
sudo etcdctl … get --prefix /service/nexus-pg        # Patroni's stored cluster state
```

**ClickHouse Keeper** (`.41/.42/.43:9181` — CH's RAFT, not ZooKeeper):
```bash
echo mntr | ncat --ssl 192.168.10.41 9181            # 4-letter-word metrics (leader/followers, znodes)
# or from clickhouse-client:  SELECT * FROM system.zookeeper WHERE path = '/'
```

**ZooKeeper** (Spark master-HA ensemble `.155/.156/.157:2181`):
```bash
echo stat | ncat 192.168.10.155 2181                 # mode: leader/follower + client count
echo mntr | ncat 192.168.10.155 2181
```

**StarRocks FE quorum:** `SHOW FRONTENDS\G` (§10). **Vitess topo:** the etcd above + `vtctldclient GetTablets`.

## §19 · Cross-cutting — services, metrics, logs, and VIPs on any node

**Is a service up? What's it logging?** (every Linux node)
```bash
ssh -i ~/.ssh/nexus_gateway_ed25519 nexusadmin@192.168.70.<X>
sudo systemctl status <service>       # e.g. redis-server · mongod · clickhouse-server · patroni · connect-distributed
sudo journalctl -u <service> -f       # live tail;  -n 200 --no-pager for the last 200 lines
```
Windows nodes (RDP): `Get-Service`, `Get-WinEvent -LogName Application -MaxEvents 50`; SQL via SSMS.

**Per-node metrics** — every Linux node runs **node_exporter** `:9100`; ws2025 runs
**windows_exporter** `:9182`:
```bash
curl -s http://192.168.70.<X>:9100/metrics | grep -E 'node_load1|node_memory_MemAvailable|node_filesystem_avail'
```

**The fleet lens = Grafana (§15).** Prometheus scrapes every node's exporter; **Vector** ships every
node's journald + `/var/log` to **Loki**; apps push traces to **Tempo**. In Grafana → *Explore*:
- metrics: `up{instance=~".*<node>.*"}`, `node_load1`
- logs: `{fleet="nexusplatform", host="<node>"}`
- traces: search by service / trace-id.

**Which node holds a VRRP VIP?** Only the current MASTER has it bound — on the candidates:
```bash
ip -brief addr | grep <vip> ;  sudo journalctl -u keepalived -n 20
```
VIPs: `.50` percona · `.60` patroni · `.119` registry-db · `.136` marquez-db · `.151` iceberg-db ·
`.184` grafana · `.185` grafana-db · `.211/.212/.213` citus coord/worker1/worker2.

**Cert expiry on a node:** `sudo openssl x509 -enddate -noout -in /etc/nexus-<svc>/tls/*.crt`.

## §20 · nexus-gateway — DNS · DHCP · NAT · NFS · iSCSI · NTP

The gateway (`192.168.70.1`, SSH) underpins the whole lab (Guide 01). To inspect it:
```bash
ssh -i ~/.ssh/nexus_gateway_ed25519 nexusadmin@192.168.70.1
sudo systemctl status dnsmasq                      # DNS + DHCP
cat /var/lib/misc/dnsmasq.leases                   # current DHCP leases (MAC → IP → hostname)
sudo nft list ruleset | less                       # NAT + firewall
sudo exportfs -v ;  showmount -e 192.168.70.1      # NFS exports (Portainer state, analytics backups)
sudo tgtadm --lld iscsi --op show --mode target    # iSCSI target (SQL FCI shared LUN)
chronyc sources ;  chronyc clients                 # NTP
```

## Appendix — tool ↔ tier quick map

| Tool | Tiers |
|---|---|
| **SSMS** | SQL Server AG (§3) |
| **DataGrip** | Percona (§7), Patroni (§8), ClickHouse (§9), StarRocks (§10), Vitess (§12), Citus (§13), Marquez datastore (§17a) |
| **NoSQLBooster / Compass** | Mongo RS (§5), Mongo sharded (§6) |
| **RedisInsight** | Redis (§4) |
| **Offset Explorer / Conduktor** | Kafka (§11), Kafka Connect + Debezium (§11a) |
| **Connect REST / Conduktor / Redpanda Console** | Kafka Connect + Debezium CDC connectors (§11a) |
| **Web browser** | Vault (§1), Portainer/Consul/Nomad (§14), Grafana + Prometheus + Alertmanager (§15), MinIO + Spark master/workers (§16), Harbor (§17), Marquez lineage graph (§17a), vtctld/VTOrc (§12), HAProxy stats (§18) |
| **curl / OpenLineage client** | Marquez lineage REST — emit run events + read the graph (§17a) |
| **Schema Registry / ksqlDB / REST Proxy / MM2** | Kafka ecosystem (§11b) — Conduktor / Redpanda Console + REST + `ksql` |
| **RSAT / ADUC** | Active Directory (§2) |
| **SSH / RDP** | every node (§0.4–0.5) |
| **systemctl · journalctl · node_exporter** | any node's services, logs, metrics (§19) |
| **etcdctl · zkCli · keeper-client · Patroni REST** | coordination / DCS layer (§18) |
| **CLI clients** | `sqlcmd`, `redis-cli`, `mongosh`, `mysql`, `psql`, `clickhouse-client`, `kafka-*.sh`, `kafka-consumer-groups.sh`, `vtctldclient`, `mc`, `vault`, `etcdctl`, `nft`, `chronyc` |
