# nexus-infra-manual — Build order & dependency graph

A single-page map of how the 23 by-hand guides ([`INDEX.md`](./INDEX.md)) fit together:
the **tier build order**, the **cross-guide dependencies**, and — critically — which
prerequisite tiers must be **alive (powered on)**, not merely *built*, when you start a
guide. Work the guides top-to-bottom (00 → 22); the only edge that runs against the numeric
order is **16 before 15** (called out below).

Fidelity note: this is a 1:1 by-hand replay of the automated `nexus-infra-*` repos. The
canonical topology (node names, IPs, MACs, counts) lives in
[`nexus-platform-plan/docs/infra/vms.yaml`](https://github.com/grezap/nexus-platform-plan/blob/main/docs/infra/vms.yaml)
(`metadata.vm_count = 143` — the 23 by-hand guides build 140 VMs; the +3 delta is the Marquez
platform-tools tier (Phase 0.Q.1, ADR-0043), deployed by automation and covered in Guide 23 rather than
a numbered by-hand build guide). If a guide and vms.yaml ever disagree, vms.yaml wins.

## Tier order (top = built first)

```
00  Lab host + base VM + OS install ............ prerequisite for EVERYTHING
     │
01  Foundation · nexus-gateway ................. DNS/DHCP/NTP + NFS + iSCSI for the whole lab
     │   (every later Linux VM pulls apt + resolves DNS through the gateway)
     ├── NFS export  ─────────────► 05 (Portainer data)
     └── iSCSI LUN   ─────────────► 11 (SQL FCI shared disk)
     │
02  Foundation · AD DS forest .................. nexus.lab domain + AD-integrated DNS
     └────────────────────────────► 11 (domain-join, GMSA), 04 (Vault LDAP), 19 (Vault OIDC↔AD)
     │
03  Foundation · Vault HA ...................... the secrets/trust root (3-node raft + transit)
     │
04  Foundation · Vault PKI + auto-unseal + LDAP  the PKI root every mTLS tier issues leaves from
     └────────────────────────────► ALL of 05–22 (each runs the §5.x "tier scaffolding" recipe)
     │
05  Orchestration · Swarm/Nomad/Consul/Portainer (needs 01 NFS for Portainer state)
06  Kafka ecosystem
07–12  OLTP (Redis · Mongo RS · Percona · Patroni · SQL FCI/AG · Mongo-sharded)
13–15  Analytics (ClickHouse · StarRocks shared-nothing · StarRocks shared-data)
16–18  Lakehouse (MinIO · Iceberg/Nessie · Spark)
19  Registry · Harbor HA
20  Observability · Grafana LGTM
21–22  Sharding (Vitess · Citus)
```

## Cross-guide dependencies (beyond "04 PKI + 01 gateway", which everything needs)

| Guide | Needs BUILT first | Needs ALIVE (running) when you start |
|---|---|---|
| 05 Swarm/Portainer | 01, 04 | 01 gateway (NFSv4 export mounted by Portainer) |
| 11 SQL FCI + AG | 01, 02, 04 | 02 AD DCs (domain-join, GMSA, KDS) · 01 gateway iSCSI LUN |
| **15 StarRocks shared-data** | **16 MinIO**, 04 | **16 MinIO** (its storage volume `s3://starrocks` must exist + be reachable) |
| 17 Iceberg/Nessie | 16, 04 | 16 MinIO (`s3://warehouse` bucket) |
| 18 Spark HA | 16, 17, 04 | 16 MinIO + 17 Iceberg REST (proves the Spark→Nessie→MinIO write path) |
| 19 Harbor HA | 16, 02→03→04 | 16 MinIO (`s3://harbor` blob backend) · Vault OIDC provider (AD-backed) |
| 20 Observability | 16, 04 | 16 MinIO (`s3://loki`, `s3://tempo`) |

**The 16-before-15 edge is the one place the numeric order lies.** Guide 15's INDEX row and its
§0 prerequisites both say "needs Guide 16 first"; build MinIO (16) before StarRocks shared-data (15).
MinIO (16) is a hub — five later guides (15, 17, 18, 19, 20) consume its buckets, so once you reach
the lakehouse tier, keep MinIO powered on for the rest of the analytics/lakehouse/registry/observability
bring-up.

## "Alive, not just built" — the rule

A guide's **§0 Prerequisites** always names which machines must be **running** to complete it. Building a
tier and then powering it off does not satisfy a later guide that reads from it at bring-up time (e.g. 15
runs `CREATE STORAGE VOLUME … s3://starrocks` against a live MinIO; 19 pushes an image into a live
`s3://harbor`). The [`feedback_minimal_running_vms`] discipline (run only the current guide's tier + its
live prerequisites; stop the rest) is compatible with this — just keep the *named live prerequisites* up.

## Foundation is always-on

Guides 00–04 (host networks, gateway, AD, Vault HA, PKI) are the **base fleet** — they must be alive for
every later guide (DNS/NTP/apt via the gateway; cert issuance + secret reads via Vault; domain services via
AD). In the running lab this is the 6-VM foundation base (gateway + 2 DCs' worth of AD + vault-1/2/3 +
vault-transit) that stays up between sessions.

## After the infrastructure — the application layer

The dependency graph above ends at a *running platform*. What runs on it is documented in the same
by-hand style, in each application repo rather than here (so the setup guide and the code it
describes version together):

- **dataflow-studio** — `docs/handbook.md` replays the whole CDC pipeline from zero, and
  `docs/demos/watch-the-pipeline.md` walks the data hop by hop across six faces. It consumes Guides
  **11** (SQL AG), **06** (Kafka), **14** (StarRocks) and **13** (ClickHouse) — all four must be
  *alive*, per the rule above, not merely built.
- **nexus-shared** — `docs/handbook.md` covers building, versioning and publishing the `Nexus.*`
  package family the application projects consume. Needs no lab VMs.

See [`INDEX.md`](./INDEX.md) for the full table.
