# nexus-infra-manual

**By-hand, zero-automation build guides for the entire NexusPlatform infrastructure layer.**

This repo is the *manual twin* of the automated `grezap/nexus-infra-*` repos. Where
those use **Packer + Terraform + Ansible** to stand up the lab, this repo shows how to
build the **exact same lab — the same VMs, IPs, MACs, on-disk configs, and full
Vault-PKI mTLS — entirely by hand**, one command at a time.

It is three things at once:

1. **A runbook** — follow it top-to-bottom and you end up with a cluster byte-for-byte
   equivalent to what the automation produces.
2. **A teaching document** — every component is explained (what it is, why we use it,
   what it would otherwise be), and **every command** records **WHERE** it runs (which
   machine + user), **WHY** it's needed, the **EXPECTED** result, and how to **VERIFY** it.
3. **A production reference** — the engine + OS guides each close with a **§9 Production
   tuning** section: the kernel/OS and engine system variables a production deployment
   sets (the lab runs deliberately lab-scale on 2 GB VMs and omits them), in a
   `production value · lab value · why` format. It's an *additive* layer that never
   changes the verbatim §5 lab build — so a guide is a 1:1 replay **and** a
   production tuning reference at the same time.

No Terraform. No Ansible. No Packer. No `vmrun`/clone automation. You create the VMs in
the VMware GUI, install the OS from the ISO, and configure every service by hand.

## Start here

- **[`INDEX.md`](./INDEX.md)** — the full roadmap: 23 guides, build-dependency order,
  with status. Read this first.
- **[`CONVENTIONS.md`](./CONVENTIONS.md)** — the shared format every guide follows
  (the step-block shape, the global lab facts, the defaults) so you don't relearn it
  per guide.
- **[`guides/`](./guides/)** — the guides themselves, added one per session.

## Relationship to the automated repos

| This repo | Automates the same thing in |
|---|---|
| `guides/00`–`guides/04` (foundation) | `nexus-infra-vmware` (`foundation` + `security` envs, `deb13`/`vault`/`ws2025` Packer) |
| `guides/05` (orchestration) | `nexus-infra-swarm-nomad` |
| `guides/06` (kafka) | `nexus-infra-kafka` |
| `guides/07`–`guides/12` (oltp) | `nexus-infra-oltp` |
| `guides/13`–`guides/15` (analytics) | `nexus-infra-analytics` |
| `guides/16`–`guides/18` (lakehouse) | `nexus-infra-lakehouse` |
| `guides/19` (registry) | `nexus-infra-registry` |
| `guides/20` (observability) | `nexus-infra-observability` |
| `guides/21` (vitess) | `nexus-infra-vitess` |
| `guides/22` (citus) | `nexus-infra-citus` |

The canonical topology (VM specs, IPs, MACs) lives in
[`nexus-platform-plan/docs/infra/vms.yaml`](https://github.com/grezap/nexus-platform-plan/blob/main/docs/infra/vms.yaml);
these guides reproduce it exactly.

## Status

**✅ COMPLETE — all 23 guides authored (see `INDEX.md`).**
Foundation (00–04), orchestration (05), Kafka (06), the full **OLTP tier**
(07–12), the full **Analytics tier** (13–15: ClickHouse + StarRocks
shared-nothing + StarRocks shared-data), the full **Lakehouse tier** (16: MinIO;
17: Iceberg/Nessie + PG HA; 18: Spark HA + ZooKeeper), the **Registry** (19:
Harbor HA), the **Observability** tier (20: Grafana LGTM — 14 nodes + 2 VRRP
VIPs), and the two **Sharding** tiers (21: Vitess/MySQL; 22: Citus/PostgreSQL).
The entire NexusPlatform infrastructure layer — host networking through database
sharding — is now reproducible **entirely by hand**, one command at a time.

**Kept current (2026-07-10):** Guide 12 (sharded MongoDB) now documents the **0.N.1 wire
mTLS** layer (`requireTLS` + per-host `mongo-sharded-server` Vault-PKI leaves); Guide 21
(Vitess) documents the **0.O.1 engine-native `file` BackupStorage** (BackupShard /
RestoreFromBackup on NFS + xtrabackup). Guide 00 (OS layer) and **every engine guide
(06–22)** now carry a **§9 Production tuning** reference layer (see `CONVENTIONS.md` §6).
