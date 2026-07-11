# Guide 23 — Connect & Observe Cookbook

**How to connect to every tier of the lab and actually *see the data*** — with the right GUI tool
for each engine (SSMS, DataGrip, NoSQLBooster, RedisInsight, Offset Explorer, and the web consoles)
**and** the CLI equivalent. This is the operator's companion to Guides 00–22: those build each tier;
this one shows you how to log in and look around once it's alive.

> Unlike Guides 00–22, this is a **cross-tier reference**, not a build guide — jump to the section
> for the tier you want. Every entry lists the **endpoint**, the **Vault path to the credential**,
> the **GUI tool + steps**, and a **CLI + see-the-data** example.

---

## §0 · Shared prelude (do these once)

### 0.1 — Get any credential from Vault

Every password/token lives in Vault KV under `nexus/`. From **PowerShell** on the build host:

```powershell
$env:VAULT_ADDR   = 'https://192.168.70.121:8200'
$env:VAULT_CACERT = "$HOME\.nexus\vault-ca-bundle.crt"
$env:VAULT_TOKEN  = (Get-Content "$HOME\.nexus\vault-init.json" -Raw | ConvertFrom-Json).root_token
# then, e.g.:
vault kv get -field=content  nexus/oltp/sqlserver/sa-password
vault kv get -field=password nexus/analytics/clickhouse/admin-password   # (field name varies; see below)
vault kv get -format=json    nexus/oltp/patroni/postgres-superuser-password   # shows the field names
```

> The `root_token` is the break-glass operator token. Each tier's password is a **field inside** the
> KV secret — if `-field=content`/`-field=password` returns nothing, run `-format=json` to see the
> actual field name (they were sticky-seeded with varying field names). This cookbook notes the path;
> confirm the field with `-format=json` the first time.

### 0.2 — Trust the lab CA (so GUI tools don't warn)

The lab uses a private PKI. Either **import the root once** into the Windows trust store (GUI tools
then validate cleanly):

```powershell
Import-Certificate -FilePath "$HOME\.nexus\vault-ca-bundle.crt" -CertStoreLocation Cert:\LocalMachine\Root
```

…or, per-connection, tick the tool's **"Trust server certificate" / "Use SSL, don't verify CA"** box.

### 0.3 — Names vs IPs

The build host is **WORKGROUP** and does not resolve `*.nexus.lab`. Two options:
- **Simplest:** connect by **IP** everywhere (all listed below). Engine certs carry IP SANs, so TLS
  still validates (or tick "trust server cert").
- **To use the `*.nexus.lab` round-robin names** (e.g. `clickhouse.nexus.lab`): set your VMnet11
  adapter's DNS to the gateway `192.168.70.1`, or add `C:\Windows\System32\drivers\etc\hosts` entries.

### 0.4 — SSH to Linux nodes

```bash
ssh -i ~/.ssh/nexus_gateway_ed25519 nexusadmin@<node-ip>
```

The exact per-tier node IPs are each guide's **Target topology** table (and `vms.yaml`).

---

## §1 · Vault (secrets + PKI) — **Web UI**

- **Endpoint:** `https://192.168.70.121:8200/ui`  ·  **Login:** token = the `root_token` from
  `~/.nexus/vault-init.json` (or `Username` LDAP with an AD account mapped to a Vault policy).
- **GUI:** open the URL → *Method: Token* → paste the root token → browse **Secrets → `nexus/`** to
  read any KV secret, or **Secrets → `pki_int/`** to issue/inspect certs.
- **CLI:** `vault kv list nexus/oltp` · `vault kv get nexus/oltp/sqlserver/sa-password`.

## §2 · Active Directory (`dc-nexus`) — **RSAT / ADUC**

- **Endpoint:** `dc-nexus` `192.168.70.240` (domain `nexus.lab`); LDAPS `:636`.
- **Creds:** `nexus\nexusadmin` — password at `nexus/foundation/dc-nexus/nexusadmin` (`-format=json`).
- **GUI:** install **RSAT** (`Add Windows Features → RSAT: AD DS Tools`). Since the host isn't
  domain-joined, launch ADUC with domain creds:
  `runas /netonly /user:nexus\nexusadmin "mmc dsa.msc"` → *Change Domain* → `nexus.lab` (server
  `192.168.70.240`). Browse **Users / ServiceAccounts** OUs.
- **CLI (PowerShell + RSAT):**
  `Get-ADUser -Server 192.168.70.240 -Credential (Get-Credential nexus\nexusadmin) -Filter *`
- **CLI (ldapsearch, from any Linux node):**
  `ldapsearch -H ldaps://192.168.70.240 -D "nexusadmin@nexus.lab" -W -b "DC=nexus,DC=lab" "(objectClass=user)" sAMAccountName`

## §3 · SQL Server AG — OLTP (`OltpDb`, `nexus_demo`) — **SSMS**

- **Endpoint:** `192.168.70.16,1433` (FCI virtual) or `192.168.70.17,1433` (AG Listener).
- **Creds:** login `sa`, password `nexus/oltp/sqlserver/sa-password` (field `content`). (Or the
  operator login `nexus-cluster-admin` at `.../operator-password`.)
- **GUI (SSMS):** *Connect → Database Engine* → Server `192.168.70.16,1433` → Auth *SQL Server
  Authentication* → `sa` + password → **Options → Encryption: Mandatory, ✅ Trust server certificate**.
  Expand `Databases → OltpDb → Tables`, right-click a table → *Select Top 1000 Rows*.
- **CLI (sqlcmd):**
  `sqlcmd -S 192.168.70.16,1433 -U sa -P '<pw>' -N -C -Q "SELECT TOP 5 * FROM OltpDb.dbo.Customers"`
- **See more:** the DataFlow Studio pipeline walkthrough is
  [`dataflow-studio/docs/demos/watch-the-pipeline.md`](https://github.com/grezap/dataflow-studio/blob/main/docs/demos/watch-the-pipeline.md).

## §4 · Redis Cluster — **RedisInsight**

- **Endpoint:** any shard node, e.g. `192.168.70.81:6379` (TLS-only; 6 nodes `.81/.82/.83/.84/.87/.89`).
- **Creds:** mTLS client cert (issue from `pki_int/issue/…`); no password by default.
- **GUI (RedisInsight):** *Add Database → Host* `192.168.70.81` *Port* `6379` → enable **TLS**,
  import the client cert/key + CA (`~/.nexus/vault-ca-bundle.crt`) → connect. RedisInsight follows
  cluster redirections automatically.
- **CLI (from a redis node):**
  `redis-cli -c --tls --cert /etc/nexus-redis/tls/server.crt --key /etc/nexus-redis/tls/server.key --cacert /etc/nexus-redis/tls/ca.crt -h 192.168.10.81 -p 6379 CLUSTER INFO` then `SET k v` / `GET k`.

## §5 · MongoDB replica set (`nexus-rs`) — **NoSQLBooster / Compass**

- **Endpoint:** `192.168.70.71/72/73:27017`, replica set `nexus-rs` (mTLS + keyFile).
- **Creds:** user at `nexus/oltp/mongo/operator-password` (+ `smoke-user-password`).
- **GUI (NoSQLBooster):** *Connect → From URI*:
  `mongodb://192.168.70.71:27017,192.168.70.72:27017,192.168.70.73:27017/?replicaSet=nexus-rs&tls=true`
  → **TLS tab:** CA file + client cert (PEM) → auth *SCRAM* with the operator user → connect. Browse
  DBs/collections, right-click → *View documents*.
- **CLI (mongosh):**
  `mongosh --tls --tlsCAFile ca.pem --tlsCertificateKeyFile client.pem "mongodb://192.168.70.71/?replicaSet=nexus-rs" -u <user> -p <pw> --eval "rs.status().members.map(m=>m.name+':'+m.stateStr)"`

## §6 · MongoDB sharded cluster — **NoSQLBooster** (via `mongos`)

- **Endpoint:** the `mongos` routers, `192.168.70.58/59:27017` (RR-DNS `mongos.nexus.lab`).
- **Creds:** `nexus-sharded-admin`@`admin` (mongos forbids `local`); password in `nexus/oltp/mongo-sharded/…`.
- **GUI (NoSQLBooster):** connect to `mongodb://192.168.70.58:27017/?tls=true` (a router) with TLS +
  the sharded-admin creds → run `sh.status()` to see shards/chunks; browse the sharded collection.
- **CLI:** `mongosh --tls … "mongodb://192.168.70.58/admin" -u nexus-sharded-admin -p <pw> --eval "sh.status()"`

## §7 · Percona XtraDB Cluster (+ ProxySQL) — **DataGrip (MySQL)**

- **Endpoint:** VRRP VIP `192.168.70.50` — **ProxySQL** client port `:6033` (routes writes→writer,
  reads→readers); ProxySQL admin `:6032`. Direct PXC nodes `.51/.52/.53:3306`.
- **Creds:** app/root at `nexus/oltp/percona/{root,operator,cluster,monitor}-password`; ProxySQL admin
  at `.../proxysql-admin-password`.
- **GUI (DataGrip):** *New → Data Source → MySQL* → Host `192.168.70.50` Port `6033` → user + password
  → **SSH/SSL tab: Use SSL, CA `vault-ca-bundle.crt`** (or *"Use truststore… / require"*). Browse the
  schema; run `SELECT`.
- **CLI:** `mysql -h 192.168.70.50 -P 6033 -u <user> -p --ssl-ca=ca.pem -e "SHOW DATABASES; SELECT @@hostname"`
- **Cluster health (admin):** `mysql -h 192.168.70.50 -P 6032 -u admin -p -e "SELECT hostname,status FROM runtime_mysql_servers"`

## §8 · PostgreSQL Patroni HA — **DataGrip (PostgreSQL)**

- **Endpoint:** VRRP VIP `192.168.70.60:5432` (HAProxy → current Patroni leader).
- **Creds:** superuser `postgres` at `nexus/oltp/patroni/postgres-superuser-password`; operator at
  `.../operator-password`.
- **GUI (DataGrip):** *New → PostgreSQL* → Host `192.168.70.60` Port `5432` User `postgres` + password
  → **SSL: require, CA = vault-ca-bundle.crt**. Browse `public` schema; run `SELECT`.
- **CLI:** `psql "host=192.168.70.60 port=5432 dbname=postgres user=postgres sslmode=verify-ca sslrootcert=ca.pem" -c "SELECT inet_server_addr(), pg_is_in_recovery()"`
- **Who's leader:** Patroni REST — `curl -sk https://192.168.70.61:8008/cluster` (from a node).

## §9 · ClickHouse — **DataGrip (ClickHouse)** / `clickhouse-client`

- **Endpoint:** any data node `192.168.70.41`, secure native `:9440`, HTTPS `:8443` (RR-DNS
  `clickhouse.nexus.lab`, 6 data nodes `.41–.46`).
- **Creds:** `nexus/analytics/clickhouse/{admin,app,operator}-password`.
- **GUI (DataGrip):** *New → ClickHouse* → Host `192.168.70.41` Port `8443` → user + password →
  **SSL: enabled**, CA bundle. Query `system.clusters`, then your `analytics.*` tables.
- **CLI (from a node):** `clickhouse-client --host 192.168.10.41 --port 9440 --secure --user <u> --password <p> -q "SELECT * FROM system.clusters WHERE cluster='nexus_analytics'"`

## §10 · StarRocks (shared-nothing + shared-data) — **DataGrip (MySQL protocol)**

- **Endpoint:** FE MySQL port `192.168.70.31:9030` (RR-DNS `starrocks-fe.nexus.lab`; shared-data FE
  `.37:9030` / `starrocks-sd-fe.nexus.lab`); HTTP UI `:8030`.
- **Creds:** `root`/app at `nexus/analytics/starrocks/{root,app,operator}-password`.
- **GUI (DataGrip):** *New → MySQL* (StarRocks speaks the MySQL wire) → Host `192.168.70.31` Port `9030`
  → user `root` + password → connect. Run `SHOW FRONTENDS; SHOW BACKENDS; SHOW DATABASES;`.
- **CLI:** `mysql -h 192.168.70.31 -P 9030 -u root -p -e "SHOW BACKENDS\G"`
- **Web:** `http://192.168.70.31:8030` (FE dashboard).

## §11 · Kafka ecosystem — **Offset Explorer / Conduktor** (+ console tools)

- **Endpoints:** brokers `192.168.10.21/22/23:9092` (mTLS, backplane); Schema Registry
  `192.168.70.91/92:8081`; Kafka Connect `192.168.70.95/96:8083`; ksqlDB `:8088`; REST Proxy `.88:8082`.
- **Creds:** mTLS client cert from `pki_int/issue/kafka-broker` (CN `localhost` is allowed). Clients
  need **ACLs** (only broker principals are super-users) — grant with `kafka-acls.sh` from a broker.
- **GUI (Offset Explorer):** *Add Cluster* → Bootstrap `192.168.10.21:9092` → **Security: SSL**,
  import the client keystore (cert+key) + truststore (CA) → connect. Browse topics → *Data* tab.
- **CLI (from a broker node, `/tmp/client.properties` = SSL + PEM keystore):**
  `kafka-topics.sh --bootstrap-server 192.168.10.21:9092 --command-config /tmp/client.properties --list`
  · consume: `kafka-console-consumer.sh … --topic <t> --from-beginning`.
- **Schema Registry:** `curl -sk https://192.168.70.91:8081/subjects` (from a node).
- **Connect:** `curl -sk https://192.168.70.95:8083/connectors` → `/<name>/status`.
- **Full walkthrough:** [`dataflow-studio/docs/demos/watch-the-pipeline.md`](https://github.com/grezap/dataflow-studio/blob/main/docs/demos/watch-the-pipeline.md).

## §12 · Vitess (sharded MySQL) — **DataGrip (via `vtgate`)**

- **Endpoint:** `vtgate` MySQL listener `192.168.70.194:15306` (RR-DNS `vtgate.nexus.lab`, 2 gateways
  `.194/.195`); keyspace `commerce`, 2 shards.
- **Creds:** `nexus/vitess/{mysql-app,mysql-allprivs,mysql-root}-password`.
- **GUI (DataGrip):** *New → MySQL* → Host `192.168.70.194` Port `15306` → user + password → SSL. Query
  `SHOW DATABASES` (`commerce`), run a `SELECT` — vtgate fans out across shards transparently.
- **CLI:** `mysql -h 192.168.70.194 -P 15306 -u <user> -p -e "SHOW VITESS_SHARDS"` · admin:
  `vtctldclient --server 192.168.70.193:15999 GetTablets` (from the control node, with its certs).

## §13 · Citus (sharded PostgreSQL) — **DataGrip (via coordinator VIP)**

- **Endpoint:** coordinator VRRP VIP `192.168.70.211:5432` (`coord.citus.nexus.lab`); workers behind
  VIPs `.212/.213`.
- **Creds:** superuser/operator at `nexus/citus/{superuser,operator,citus-app}-password`.
- **GUI (DataGrip):** *New → PostgreSQL* → Host `192.168.70.211` Port `5432` → user + password → SSL.
  Query `SELECT * FROM pg_dist_node;` and `SELECT * FROM citus_shards;` to see the distribution.
- **CLI:** `psql "host=192.168.70.211 port=5432 user=postgres sslmode=verify-ca sslrootcert=ca.pem" -c "SELECT nodename,nodeport FROM pg_dist_node"`

## §14 · Swarm / Nomad / Consul / Portainer — **Web UIs**

- **Portainer:** `https://192.168.70.111:9443` (also `.112/.113`; RR-DNS `portainer.nexus.lab`).
  Login `admin` + password `nexus/portainer/admin-bcrypt` (the plaintext seed) → **Stacks / Services /
  Containers**.
- **Consul UI:** `https://192.168.70.111:8501/ui` → token = `nexus/swarm/consul-bootstrap-token`.
- **Nomad UI:** `https://192.168.70.111:4646/ui` → token = `nexus/swarm/nomad-bootstrap-token`.
- **CLI (from a manager):** `docker node ls` · `consul members` · `nomad server members`.

## §15 · Observability (Grafana LGTM) — **Grafana Web UI**

- **Endpoint:** `https://192.168.70.184:3000` (VRRP VIP, `grafana.nexus.lab`).
- **Creds:** `admin` + password `nexus/observability/grafana/…` (`-format=json` for the field).
- **GUI (browser):** log in → **Explore** → pick a datasource: **Prometheus** (metrics — `up`,
  `node_load1`), **Loki** (logs — `{fleet="nexusplatform"}`), **Tempo** (traces). Dashboards under
  **Dashboards**. This is how you watch the *whole fleet*.
- **CLI:** Prometheus `curl -sk https://192.168.70.180:9090/api/v1/query?query=up` (from a node).

## §16 · Lakehouse (MinIO · Spark · Iceberg) — **Web consoles**

- **MinIO console:** `https://192.168.70.140:9001` (S3 API `:9000`, RR-DNS `minio.nexus.lab`). Login =
  root creds at `nexus/lakehouse/minio/…`. Browse **buckets** (`warehouse`, `loki`, `tempo`, `harbor`,
  `starrocks`).
- **Spark master UI:** `http://192.168.70.147:8080` (the ZK-elected active master; workers + apps).
- **Iceberg/Nessie:** REST at `https://iceberg.nexus.lab:19120` (catalog `api/v1`); catalog DB is the
  PG pair behind VIP `192.168.70.151`.
- **CLI (MinIO, from a node):** `mc alias set lab https://192.168.10.140:9000 <ak> <sk> --api S3v4` then
  `mc ls lab/warehouse`.

## §17 · Registry (Harbor) — **Harbor Web UI**

- **Endpoint:** `https://192.168.70.115` (app nodes `.115/.116`, RR-DNS `registry.nexus.lab`, `:443`).
- **Creds:** `admin` + `nexus/registry/harbor-admin`; or **AD SSO** via Vault OIDC.
- **GUI (browser):** log in → **Projects** → a project → **Repositories** to see pushed images, Trivy
  scan results, and cosign signatures.
- **CLI:** `docker login registry.nexus.lab -u admin` then `docker pull registry.nexus.lab/<proj>/<img>:<tag>`
  (add the CA to Docker's trust, or `/etc/docker/certs.d/`). Metadata DB is the PG behind VIP `.119`.

---

## Appendix — tool ↔ tier quick map

| Tool | Tiers |
|---|---|
| **SSMS** | SQL Server AG (§3) |
| **DataGrip** | Percona (§7), Patroni PG (§8), ClickHouse (§9), StarRocks (§10), Vitess (§12), Citus (§13) |
| **NoSQLBooster / Compass** | MongoDB RS (§5), MongoDB sharded (§6) |
| **RedisInsight** | Redis Cluster (§4) |
| **Offset Explorer / Conduktor** | Kafka (§11) |
| **Web browser** | Vault (§1), Portainer/Consul/Nomad (§14), Grafana (§15), MinIO/Spark (§16), Harbor (§17) |
| **RSAT / ADUC** | Active Directory (§2) |
| **CLI (always available)** | every tier — `sqlcmd`, `redis-cli`, `mongosh`, `mysql`, `psql`, `clickhouse-client`, `kafka-*.sh`, `vtctldclient`, `mc`, `vault` |
