# nexus-infra-manual — Conventions

The shared format + facts every guide inherits, so the structure never drifts and the
guides don't repeat the same lab basics 23 times. Read once.

---

## 1. Per-guide skeleton

Every `guides/NN-<slug>.md` has these sections in this order:

1. **Overview & purpose** — what this tier/cluster is, why it exists in the platform,
   and what it depends on.
2. **Component primer** — for *every* piece of software introduced (e.g. etcd, Patroni,
   Citus, keepalived): **what it is**, **why we use it here**, and **what it would
   otherwise be** (the alternative we rejected). This is the teaching half.
3. **Prerequisites** — exactly which machines must already exist + be alive (from earlier
   guides), and the **one command to verify each** before you start.
4. **Target topology** — the table of VMs for this guide: hostname · role · VMnet11 IP ·
   VMnet10 IP · MAC · vCPU/RAM/disk · ports. (Lifted from `vms.yaml`.)
5. **Step-by-step build** — numbered steps in the **step-block format** (§3 below).
6. **Validation** — a by-hand end-to-end smoke that proves the tier works, mirroring the
   automated `smoke-*.ps1` for that tier (same checks, run manually).
7. **Teardown / reset** — how to cleanly remove what this guide built.
8. **Troubleshooting** — the real gotchas (drawn from each repo's transient ledger):
   symptom → cause → fix.
9. **Production tuning** *(engine + OS guides only)* — the system variables a
   production operator sets that the **lab deliberately does not** (it runs lab-scale on
   2 GB VMs). This is an **additive reference layer**, explicitly *beyond the lab
   replica* — it never changes the verbatim lab configs in §5. Format in §6 below.
   Guides that introduce no tunable engine/OS (e.g. 01 gateway, 03 Vault HA) omit it.

---

## 2. Node repetition

**Every node is written out verbatim** — no "repeat steps X–Y" shorthand. A 9-node
cluster gets 9 full node sections. Steps that genuinely differ per node (initial-cluster
strings, Patroni `name`, tablet aliases, priorities) are spelled out per node, not
parameterised.

---

## 3. Step-block format

Each step is a numbered block. Commands are plain (no prompt prefixes) — the **WHERE**
line carries the machine + user, so you always know where your shell is:

> **Step N.M — <imperative title>**
> **WHERE:** `<hostname>` (`<VMnet11 IP>`), shell as `<user>` (e.g. `nexusadmin`, or
> `root` via `sudo -i`). For Windows nodes: "RDP to `<host>`, elevated PowerShell."
> **WHY:** what this step accomplishes and why it's necessary (1–3 sentences; this is
> where the teaching lives — explain the mechanism, not just the action).
> **WHAT:**
> ```bash
> <the exact command(s)>
> ```
> **EXPECTED:** the literal output / observable success signal (a log line, a status,
> a file appearing).
> **VERIFY:** a separate check command + its expected result (so you confirm the step
> took effect before moving on).

Conventions inside blocks:
- Show **real values**, not placeholders, wherever the lab has a fixed value (IPs,
  hostnames, paths). Use `<angle-brackets>` only for genuine per-reader secrets.
  *(Exception: the §9 Production-tuning layer shows recommended production values the
  lab does **not** use — always paired with the lab's actual value + a "why". That layer
  is the only place a guide states a value that isn't the verbatim lab render.)*
- Multi-line files: show the **entire file content** in a `cat > /path <<'EOF'` block,
  then a VERIFY that re-reads it.
- Secrets (passwords, keys): generate them in-step (`openssl rand -hex 16`, `vault …`)
  and show exactly where they're stored + with what permissions — never hand-wave.

---

## 4. Global lab facts (inherited by all guides — not repeated per guide)

- **Hypervisor:** VMware Workstation Pro on a Windows 11 host. VMs created by hand in the
  GUI (Guide 00).
- **Networks:** **VMnet11** = service/management network `192.168.70.0/24` (gateway
  `.1`, host `.254`); **VMnet10** = cluster backplane `192.168.10.0/24` (host-only, no
  DHCP). Most cluster nodes are **dual-NIC**: `nic0` = VMnet11, `nic1` = VMnet10.
- **Domain / DNS:** AD domain `nexus.lab`; the gateway's dnsmasq serves DHCP + DNS on
  VMnet11 and forwards `.nexus.lab`/AD to the DCs.
- **Naming:** hostnames match `vms.yaml` (e.g. `citus-coord-1`); VMnet10 IP = same last
  octet as VMnet11 (`.70.205` ↔ `.10.205`).
- **MAC convention:** `00:50:56:3F:00:XX` primary (VMnet11), `00:50:56:3F:01:XX`
  secondary (VMnet10); the `XX` block per tier is in `vms.yaml` + `network.md`.
- **Default admin user:** `nexusadmin` (sudoer on Linux; SSH key
  `~/.ssh/nexus_gateway_ed25519`, password fallback). Windows: domain admin.
- **Always-on base:** the 6-VM foundation (`nexus-gateway` + `dc-nexus` + `vault-1/2/3`
  + `vault-transit`) underpins every later guide — Guides 01–04 build it; Guides 05+
  assume it alive.
- **Security posture:** full **Vault-PKI mTLS** is reproduced **by hand** — manual
  `vault write pki_int/issue/<role>` (or `openssl` CSR → Vault sign), manual placement of
  `server-cert.pem` / `server-key.pem` / `ca.pem`, manual AppRole/policy/KV setup. No
  shortcuts; the manual lab has the same security as the automated one.
- **Source of truth:** the canonical topology is
  [`nexus-platform-plan/docs/infra/vms.yaml`](https://github.com/grezap/nexus-platform-plan/blob/main/docs/infra/vms.yaml);
  the §5 build configs are the verbatim equivalents of what the automated repos' overlays
  render. **The §9 Production-tuning layer is the sole, clearly-labelled exception** — it
  documents production system variables the lab omits (see §6), so a guide can serve both
  as a 1:1 lab replay *and* as a production reference without the two ever being confused.

---

## 5. What "by hand" excludes

No Packer (you install the OS from the ISO), no Terraform (you create + configure VMs
yourself), no Ansible (you run every package install + write every config file by hand),
no `vmrun` scripting. Where the automated path uses a Vault Agent to *render* a cert or
secret, the manual path issues + places it with the `vault` CLI directly.

---

## 6. Production-tuning section format (guide §9)

The lab runs deliberately lab-scale on 2 GB VMs, so its verbatim configs (§5) omit the
system variables a production deployment must set. Section §9 adds them back as an
**explicitly-labelled reference layer that does not alter §5**. Rules:

- **Open with the disclaimer:** "Everything below is *beyond the lab replica* — the lab
  ships the §5 values; this section is what you would change for a production-scale
  deployment and why. Do not apply these to the 2 GB lab VMs blindly."
- **One table per subsystem** (OS-layer first, then engine-layer), columns:

  | Setting | Production value | Lab value (§5) | Why it matters |
  |---|---|---|---|

  - **Setting** — the exact knob (`vm.max_map_count`, `max server memory (MB)`,
    `shared_buffers`, `maxmemory-policy`), with the file/scope it lives in.
  - **Production value** — a concrete recommended value *or* a sizing formula
    (`≈ 25% of RAM`, `50% of (RAM − 1 GB)`), not "tune as needed".
  - **Lab value (§5)** — what this guide's verbatim config actually sets (or "unset").
  - **Why it matters** — the mechanism + the failure mode if left at default (1 sentence).
- **Show the how**, not just the what: for anything that needs a command/file to apply
  (sysctl drop-in, `limits.conf`, THP unit, `ALTER SYSTEM`, `sp_configure`), give the
  exact snippet in a fenced block — same rigour as a §5 step, but flagged "production,
  not applied in the lab".
- **Flag hard requirements vs optimisations.** A few knobs are engine *requirements*, not
  tuning (e.g. `vm.max_map_count=262144` for StarRocks, THP-disabled for MongoDB/Redis) —
  mark these **⚠️ required** so they're never mistaken for optional polish.
- **OS-layer knobs live once, in Guide 00 §9** (swappiness, overcommit, THP, file-max,
  somaxconn, `limits.conf`, I/O scheduler). Per-engine guides link back to it and only
  restate the engine-specific overrides (e.g. Redis needs `overcommit_memory=1`).
