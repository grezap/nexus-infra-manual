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
| **Platform-tools** *(reserved, not built)* | prefect-server `.125` · unleash-1 `.126` · marquez `.127` · backstage `.128` | L |

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

## §10 · StarRocks — **DataGrip (MySQL protocol)**

- **Endpoint:** FE MySQL `192.168.70.31:9030` (shared-nothing; shared-data FE `.37:9030`), HTTP `:8030`.
- **Creds:** `root` `vault kv get -field=password nexus/analytics/starrocks/root-password`.
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
| `oltp.OltpDb.dbo.Customers` (+ the other `oltp.OltpDb.dbo.*`) | the **raw CDC events** — JSON envelope `{before, after, op, source}` |
| `schemahistory.oltp` | Debezium's internal **schema history** (DDL) |
| `oltp` | the connector's schema-change topic |

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

## §12 · Vitess (sharded MySQL) — **DataGrip (via `vtgate`)**

- **Endpoint:** vtgate MySQL `192.168.70.194:15306` (also `.195`); keyspace `commerce`.
- **Creds:** `vault kv get -field=content nexus/vitess/mysql-app-password` (root: `mysql-root-password`).
- **GUI (DataGrip):** *New → MySQL* → Host `192.168.70.194` Port `15306` → user+pw → SSL. `SHOW VITESS_SHARDS;`
- **CLI:** `mysql -h 192.168.70.194 -P 15306 -u <user> -p -e "SHOW VITESS_TABLETS"` · admin (on control
  node `.193`): `vtctldclient --server 192.168.10.193:15999 GetTablets`

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

## §16 · Lakehouse (MinIO · Spark · Iceberg) — **Web consoles**

- **MinIO console:** `https://192.168.70.141:9001` (S3 API `:9000`; nodes minio-1..4 `.141–.144`). Login
  `vault kv get -field=root-user nexus/lakehouse/minio` / `-field=root-password`. Browse buckets
  (`warehouse`, `loki`, `tempo`, `harbor`, `starrocks`).
- **Spark master UI:** `http://192.168.70.140:8080` (ZK-elected active; standby `.153`).
- **Iceberg/Nessie:** REST `https://192.168.70.147:19120` (also `.148`); catalog DB VIP `.151`.
- **CLI (on a node):** `mc alias set lab https://192.168.10.141:9000 <ak> <sk>` then `mc ls lab/warehouse`.

## §17 · Registry (Harbor) — **Harbor Web UI**

- **Endpoint:** `https://192.168.70.115` (also `.116`; `:443`). Datastore registry-pg-1/2 `.117/.118`.
- **Creds:** `admin` / `vault kv get -field=value nexus/registry/harbor-admin` (or AD SSO via Vault OIDC).
- **GUI:** log in → **Projects** → a project → **Repositories** (images, Trivy scans, cosign signatures).
- **CLI:** `docker login registry.nexus.lab -u admin` then `docker pull registry.nexus.lab/<proj>/<img>:<tag>`.

---

## Appendix — tool ↔ tier quick map

| Tool | Tiers |
|---|---|
| **SSMS** | SQL Server AG (§3) |
| **DataGrip** | Percona (§7), Patroni (§8), ClickHouse (§9), StarRocks (§10), Vitess (§12), Citus (§13) |
| **NoSQLBooster / Compass** | Mongo RS (§5), Mongo sharded (§6) |
| **RedisInsight** | Redis (§4) |
| **Offset Explorer / Conduktor** | Kafka (§11), Kafka Connect + Debezium (§11a) |
| **Connect REST / Conduktor / Redpanda Console** | Kafka Connect + Debezium CDC connectors (§11a) |
| **Web browser** | Vault (§1), Portainer/Consul/Nomad (§14), Grafana (§15), MinIO/Spark (§16), Harbor (§17) |
| **RSAT / ADUC** | Active Directory (§2) |
| **SSH / RDP** | every node (§0.4–0.5) |
| **CLI clients** | `sqlcmd`, `redis-cli`, `mongosh`, `mysql`, `psql`, `clickhouse-client`, `kafka-*.sh`, `vtctldclient`, `mc`, `vault` |
