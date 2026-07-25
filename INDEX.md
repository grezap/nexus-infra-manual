# nexus-infra-manual — Guide Index (the roadmap)

The full set of by-hand build guides, in **build-dependency order**. Each guide is
session-sized and self-contained (with a stated prerequisites section). Work them
**top to bottom** — later guides assume the earlier tiers are alive.

**Status legend:** 📋 planned · 🚧 in progress · ✅ done

| # | Guide | Scope (what you build by hand) | Nodes | OS | Mirrors |
|:---:|---|---|:---:|---|---|
| 00 | ✅ [Lab host + base VM + OS install](./guides/00-lab-host-and-base-vm.md) | VMware host networks **VMnet10** (backplane, host-only) + **VMnet11** (service); create a VM in the VMware GUI (vCPU/RAM/disk, 2 NICs); manual **Debian 13 netinst** walkthrough; manual **Windows Server 2025** install (for AD + SQL); per-node baseline (hostname, dual-NIC static/DHCP, SSH, nftables, chrony) | — | — | `deb13`/`ws2025`/`vault` Packer |
| 01 | ✅ [Foundation · nexus-gateway](./guides/01-foundation-nexus-gateway.md) | Debian gateway: **dnsmasq** (DHCP reservations + DNS), **nftables** (NAT + filtering), **chrony** (NTP), **NFSv4** export, **iSCSI** target | 1 | deb13 | `nexus-infra-vmware/foundation` |
| 02 | ✅ [Foundation · AD DS forest](./guides/02-foundation-ad-ds-forest.md) | `dc-nexus` Install-ADDSForest by hand (DNS, OUs, password policy, KDS root key); `dc-nexus-2` replica promotion (0.M) | 2 | ws2025 | `foundation` (dc-*) |
| 03 | ✅ [Foundation · Vault HA](./guides/03-foundation-vault-ha.md) | `vault-1/2/3` 3-node **Raft** cluster (init, unseal, join, raft peers); `vault-transit` seal helper | 4 | deb13 | `security` (vault) |
| 04 | ✅ [Foundation · Vault PKI + auto-unseal + LDAP](./guides/04-foundation-vault-pki-ldap.md) | **transit auto-unseal**; **PKI hierarchy** (root + intermediate CA); **LDAPS auth** + **secrets/ldap** rotation; the per-tier scaffolding pattern (PKI roles, AppRoles, KV) done by hand | — | — | `security` |
| 05 | ✅ [Orchestration · Swarm + Nomad + Consul + Portainer](./guides/05-orchestration-swarm-nomad-consul-portainer.md) | Docker **Swarm** (3 mgr + 3 wkr), **Consul** (gossip+TLS+ACL), **Nomad** (mTLS+ACL+Vault), **Portainer** CE | 6 | deb13 | `nexus-infra-swarm-nomad` |
| 06 | ✅ [Kafka ecosystem](./guides/06-kafka-ecosystem.md) | 2 **KRaft** clusters (east+west) mTLS, Schema Registry HA, REST Proxy, Kafka Connect (Debezium), ksqlDB, **MirrorMaker 2** DR | 15 | deb13 | `nexus-infra-kafka` |
| 07 | ✅ [OLTP · Redis Cluster](./guides/07-oltp-redis-cluster.md) | 6-node Redis Cluster (3 shards × 2 replicas) over mTLS | 6 | deb13 | `nexus-infra-oltp` |
| 08 | ✅ [OLTP · MongoDB replica set](./guides/08-oltp-mongodb-replica-set.md) | 3-node RS, keyFile internal auth + mTLS x509 | 3 | deb13 | `nexus-infra-oltp` |
| 09 | ✅ [OLTP · Percona XtraDB Cluster](./guides/09-oltp-percona-xtradb-cluster.md) | 3 PXC (Galera) + 2 ProxySQL + **VRRP VIP** (keepalived) | 5 | deb13 | `nexus-infra-oltp` |
| 10 | ✅ [OLTP · Patroni PostgreSQL HA](./guides/10-oltp-patroni-postgresql-ha.md) | 3 Patroni + 3 etcd + 2 HAProxy + VRRP VIP | 8 | deb13 | `nexus-infra-oltp` |
| 11 | ✅ [OLTP · SQL Server FCI + Always-On AG](./guides/11-oltp-sqlserver-fci-ag.md) | WSFC + iSCSI shared storage FCI + AG listener | 4 | ws2025 | `nexus-infra-oltp` |
| 12 | ✅ [OLTP · MongoDB sharded](./guides/12-oltp-mongodb-sharded.md) | 3 config RS + 2 shard RS (×3) + 2 mongos; keyFile member auth + **wire mTLS x509** (0.N.1) | 11 | deb13 | `nexus-infra-oltp` |
| 13 | ✅ [Analytics · ClickHouse](./guides/13-analytics-clickhouse.md) | 3 shards × 2 replicas + 3-node ClickHouse Keeper | 9 | deb13 | `nexus-infra-analytics` |
| 14 | ✅ [Analytics · StarRocks (shared-nothing)](./guides/14-analytics-starrocks-shared-nothing.md) | 3 FE (BDB-JE quorum) + 3 BE (tablet sharding/replication) | 6 | deb13 | `nexus-infra-analytics` |
| 15 | ✅ [Analytics · StarRocks (shared-data)](./guides/15-analytics-starrocks-shared-data.md) | 3 FE + 2 CN on `run_mode=shared_data`, MinIO storage volume (needs Guide 16 first) | 5 | deb13 | `nexus-infra-analytics` |
| 16 | ✅ [Lakehouse · MinIO](./guides/16-lakehouse-minio.md) | 4-node distributed erasure-coded object store (S3 :9000, RR-DNS `minio.nexus.lab`, mTLS, dedicated data VMDK) — **unblocks Guide 15** | 4 | deb13 | `nexus-infra-lakehouse` |
| 17 | ✅ [Lakehouse · Iceberg / Nessie](./guides/17-lakehouse-iceberg-nessie.md) | 2× Nessie Iceberg REST catalog (RR-DNS) + dedicated PG 17 master-replica HA (streaming repl + keepalived VRRP VIP `.151`), warehouse on MinIO | 4 | deb13 | `nexus-infra-lakehouse` |
| 18 | ✅ [Lakehouse · Spark HA](./guides/18-lakehouse-spark-ha.md) | 2 Spark masters (ZooKeeper-elected HA) + 3-node ZK quorum + 3 workers; Iceberg/S3 client — proves the Spark→Nessie→MinIO write path | 8 | deb13 | `nexus-infra-lakehouse` |
| 19 | ✅ [Registry · Harbor HA](./guides/19-registry-harbor-ha.md) | 2 stateless Harbor app nodes (RR-DNS) + PG 17/Redis HA datastore (keepalived VRRP VIP `.119`), MinIO `s3://harbor` blobs, Trivy + cosign, Vault OIDC SSO | 4 | deb13 | `nexus-infra-registry` |
| 20 | ✅ [Observability · Grafana LGTM](./guides/20-observability-grafana-lgtm.md) | Prometheus HA + Alertmanager mesh + Loki + Tempo (MinIO) + Grafana HA over shared PG + OTel Collector + fleet Vector; 2 VRRP VIPs (`.184/.185`) | 14 | deb13 | `nexus-infra-observability` |
| 21 | ✅ [Sharding · Vitess (MySQL)](./guides/21-sharding-vitess-mysql.md) | 3 etcd topo + vtctld/VTOrc + 2 vtgate + 2 shards × 3 Percona 8.4 tablets; hash-vindex sharding, full mTLS, VTOrc auto-reparent, **engine-native `file` BackupStorage** (0.O.1) | 12 | deb13 | `nexus-infra-vitess` |
| 22 | ✅ [Sharding · Citus (PostgreSQL)](./guides/22-sharding-citus-postgresql.md) | 3 etcd DCS + coordinator Patroni pair + 2 worker Patroni pairs + 3 keepalived VIPs (VIP-follows-leader); 32-shard distributed table, full mTLS | 9 | deb13 | `nexus-infra-citus` |
| 23 | ✅ [Connect & Observe Cookbook](./guides/23-connect-and-observe-cookbook.md) | **Cross-tier reference** (not a build guide): how to connect to every tier and *see the data* — the right GUI tool per engine (SSMS · DataGrip · NoSQLBooster · RedisInsight · Offset Explorer · web consoles) **+** the CLI equivalent, with endpoints + Vault credential paths | — | — | all tiers |
| 24 | ✅ [Platform tools · Marquez (OpenLineage)](./guides/24-platform-tools-marquez-openlineage.md) | Marquez OpenLineage **data-lineage** backend: Docker-compose app (API + web + **nginx TLS** front door) + a dedicated **PostgreSQL 17 HA pair** + **VRRP VIP** (`marquez-db.nexus.lab .136`); the platform-tools tier (`09-platform`), Phase 0.Q.1 | 3 | deb13 | `nexus-infra-platform-tools` |

**Total: 24 build guides (00–22 + 24) · 143 VMs · the full infrastructure + platform-tools layer, by hand — ✅ ALL COMPLETE — plus the cross-tier Connect & Observe Cookbook (Guide 23).**

> 🛠 **Production-tuning layer (§9).** Guide 00 (the OS layer) and **every engine guide
> (06–22)** each carry a **§9 Production tuning** section — the system variables a production operator
> sets that the deliberately lab-scale (2 GB) configs omit, in a `production value · lab
> value · why` format. It is an **additive reference layer** (see [`CONVENTIONS.md`](./CONVENTIONS.md)
> §6/§9) and never changes the verbatim §5 lab build, so each guide is both a 1:1 lab
> replay *and* a production reference. The OS layer lives once in **Guide 00 §9**; engine
> guides link back to it and add only their engine-specific knobs.

> ⚠️ **One out-of-numeric-order dependency:** build **Guide 16 (MinIO) BEFORE Guide 15
> (StarRocks shared-data)** — 15's `run_mode=shared_data` stores its data in a MinIO
> bucket, so MinIO must already be **alive**. The rows are numbered by tier, not by this
> single cross-tier edge. The full dependency graph — including which prerequisite tiers
> must be *running* (not just built) when you start a guide — is in
> [`OVERVIEW.md`](./OVERVIEW.md).

## Where the trail continues — the application layer

These 24 guides cover the **infrastructure + platform-tools** layer (host networking → PostgreSQL
sharding → the Marquez lineage backend). They stop where the platform is ready to *run something*. The
by-hand replay of what runs **on** it lives with each application project, in the same step-by-step
spirit:

| Project | By-hand replay | Watch the data move |
|---|---|---|
| **dataflow-studio** (Phase 1 — CDC pipeline) | [`docs/handbook.md`](https://github.com/grezap/dataflow-studio/blob/main/docs/handbook.md) — §0 prerequisites (exact creds + which tier per step) → migrate → CDC/Debezium → seed → ACLs → curate → StarRocks sink → ClickHouse telemetry → verify → tear down, with a transient ledger | [`watch-the-pipeline.md`](https://github.com/grezap/dataflow-studio/blob/main/docs/demos/watch-the-pipeline.md) — six faces, SSMS + Kafka console + DataGrip + `clickhouse-client` |
| **nexus-shared** (0.J — the `Nexus.*` NuGet family) | [`docs/handbook.md`](https://github.com/grezap/nexus-shared/blob/main/docs/handbook.md) — build → version → publish to GitHub Packages → consume downstream | — |
| **nexus-cli** | its own handbook + demo catalogue | — |

Guides that the application layer leans on most directly: **11** (SQL Server FCI+AG — the OLTP
source), **06** (Kafka — the transport), **13** (ClickHouse — telemetry, incl. §9.1 *ClickHouse as a
Kafka consumer*), **14** (StarRocks — the warehouse).

## How we work

- **One guide per session.** Each session produces one complete, self-reviewed guide
  under `guides/NN-<slug>.md`, and flips its row above from 📋 → ✅. **All 24 are now ✅.**
- Every guide follows [`CONVENTIONS.md`](./CONVENTIONS.md) (the step-block format +
  global lab facts) so the format never drifts.
- Commands + configs are sourced **verbatim from the automated repos** (the rendered
  configs the overlays produce) so the manual path is a true 1:1 replay. The one exception
  is each guide's **§9 Production tuning** layer, which is explicitly labelled *beyond the
  lab replica* and documents production values the lab does not use (see `CONVENTIONS.md` §6).
