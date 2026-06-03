# nexus-infra-manual — Guide Index (the roadmap)

The full set of by-hand build guides, in **build-dependency order**. Each guide is
session-sized and self-contained (with a stated prerequisites section). Work them
**top to bottom** — later guides assume the earlier tiers are alive.

**Status legend:** 📋 planned · 🚧 in progress · ✅ done

| # | Guide | Scope (what you build by hand) | Nodes | OS | Mirrors |
|:---:|---|---|:---:|---|---|
| 00 | ✅ [Lab host + base VM + OS install](./guides/00-lab-host-and-base-vm.md) | VMware host networks **VMnet10** (backplane, host-only) + **VMnet11** (service); create a VM in the VMware GUI (vCPU/RAM/disk, 2 NICs); manual **Debian 13 netinst** walkthrough; manual **Windows Server 2025** install (for AD + SQL); per-node baseline (hostname, dual-NIC static/DHCP, SSH, nftables, chrony) | — | — | `deb13`/`ws2025`/`vault` Packer |
| 01 | 📋 Foundation · nexus-gateway | Debian gateway: **dnsmasq** (DHCP reservations + DNS), **nftables** (NAT + filtering), **chrony** (NTP), **NFSv4** export, **iSCSI** target | 1 | deb13 | `nexus-infra-vmware/foundation` |
| 02 | 📋 Foundation · AD DS forest | `dc-nexus` Install-ADDSForest by hand (DNS, OUs, password policy, KDS root key); `dc-nexus-2` replica promotion (0.M) | 2 | ws2025 | `foundation` (dc-*) |
| 03 | 📋 Foundation · Vault HA | `vault-1/2/3` 3-node **Raft** cluster (init, unseal, join, raft peers); `vault-transit` seal helper | 4 | deb13 | `security` (vault) |
| 04 | 📋 Foundation · Vault PKI + auto-unseal + LDAP | **transit auto-unseal**; **PKI hierarchy** (root + intermediate CA); **LDAPS auth** + **secrets/ldap** rotation; the per-tier scaffolding pattern (PKI roles, AppRoles, KV) done by hand | — | — | `security` |
| 05 | 📋 Orchestration · Swarm + Nomad + Consul + Portainer | Docker **Swarm** (3 mgr + 3 wkr), **Consul** (gossip+TLS+ACL), **Nomad** (mTLS+ACL+Vault), **Portainer** CE | 6 | deb13 | `nexus-infra-swarm-nomad` |
| 06 | 📋 Kafka ecosystem | 2 **KRaft** clusters (east+west) mTLS, Schema Registry HA, REST Proxy, Kafka Connect (Debezium), ksqlDB, **MirrorMaker 2** DR | 15 | deb13 | `nexus-infra-kafka` |
| 07 | 📋 OLTP · Redis Cluster | 6-node Redis Cluster (3 shards × 2 replicas) over mTLS | 6 | deb13 | `nexus-infra-oltp` |
| 08 | 📋 OLTP · MongoDB replica set | 3-node RS, keyFile internal auth + mTLS x509 | 3 | deb13 | `nexus-infra-oltp` |
| 09 | 📋 OLTP · Percona XtraDB Cluster | 3 PXC (Galera) + 2 ProxySQL + **VRRP VIP** (keepalived) | 5 | deb13 | `nexus-infra-oltp` |
| 10 | 📋 OLTP · Patroni PostgreSQL HA | 3 Patroni + 3 etcd + 2 HAProxy + VRRP VIP | 8 | deb13 | `nexus-infra-oltp` |
| 11 | 📋 OLTP · SQL Server FCI + Always-On AG | WSFC + iSCSI shared storage FCI + AG listener | 4 | ws2025 | `nexus-infra-oltp` |
| 12 | 📋 OLTP · MongoDB sharded | 3 config RS + 2 shard RS (×3) + 2 mongos | 11 | deb13 | `nexus-infra-oltp` |
| 13 | 📋 Analytics · ClickHouse | 3 shards × 2 replicas + 3-node ClickHouse Keeper | 9 | deb13 | `nexus-infra-analytics` |
| 14 | 📋 Analytics · StarRocks (shared-nothing) | 3 FE (BDB-JE quorum) + 3 BE (tablet sharding/replication) | 6 | deb13 | `nexus-infra-analytics` |
| 15 | 📋 Analytics · StarRocks (shared-data) | 3 FE + 2 CN on `run_mode=shared_data`, MinIO storage volume | 5 | deb13 | `nexus-infra-analytics` |
| 16 | 📋 Lakehouse · MinIO | 4-node distributed erasure-coded object store | 4 | deb13 | `nexus-infra-lakehouse` |
| 17 | 📋 Lakehouse · Iceberg / Nessie | Nessie REST catalog + PG master-replica HA (VRRP VIP) | 4 | deb13 | `nexus-infra-lakehouse` |
| 18 | 📋 Lakehouse · Spark HA | 2 masters + ZooKeeper quorum + 3 workers | 8 | deb13 | `nexus-infra-lakehouse` |
| 19 | 📋 Registry · Harbor HA | 2 stateless app nodes (RR-DNS) + PG/Redis HA datastore, MinIO blobs, Vault OIDC SSO | 4 | deb13 | `nexus-infra-registry` |
| 20 | 📋 Observability · Grafana LGTM | Prometheus + Loki + Tempo + Grafana (+PG) + OTel collector, VRRP VIP | 14 | deb13 | `nexus-infra-observability` |
| 21 | 📋 Sharding · Vitess (MySQL) | 3 etcd + vtctld/VTOrc + 2 vtgate + 2 shards × 3 Percona tablets | 12 | deb13 | `nexus-infra-vitess` |
| 22 | 📋 Sharding · Citus (PostgreSQL) | 3 etcd DCS + coordinator Patroni pair + 2 worker Patroni pairs + 3 keepalived VIPs | 9 | deb13 | `nexus-infra-citus` |

**Total: 23 guides · ~140 VMs · the full infrastructure layer, by hand.**

## How we work

- **One guide per session.** Each session produces one complete, self-reviewed guide
  under `guides/NN-<slug>.md`, and flips its row above from 📋 → ✅.
- Every guide follows [`CONVENTIONS.md`](./CONVENTIONS.md) (the step-block format +
  global lab facts) so the format never drifts.
- Commands + configs are sourced **verbatim from the automated repos** (the rendered
  configs the overlays produce) so the manual path is a true 1:1 replay.
