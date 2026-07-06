# Guide 00 — Lab host + base VM + OS install

> **Mirrors:** `nexus-infra-vmware/packer/{deb13,ws2025-core,ws2025-desktop}` +
> `packer/_shared/ansible/roles/*` + `packer/_shared/powershell/scripts/*`.
> Where the automated lab uses **Packer** to bake a golden template and
> **Terraform** to clone + re-network it, this guide installs each OS **from the
> ISO by hand** and applies the same baseline **one command at a time**.

---

## 1. Overview & purpose

This is the **first** guide. It produces nothing you can log a cluster into — it
produces the two things every later guide assumes already exist:

1. **The host's two virtual networks.** Every NexusPlatform VM is dual-homed:
   one NIC on **VMnet11** (the management + application + egress plane,
   `192.168.70.0/24`) and one on **VMnet10** (the isolated cluster backplane,
   `192.168.10.0/24`). Those two VMware Workstation host-only networks have to
   be created and pinned to fixed host IPs before any VM can talk to anything.
2. **The repeatable "build a node" procedure.** There is no template and no
   cloning in the manual lab — so this guide is the **recipe** that Guides 01–22
   invoke by reference. It builds, end to end:
   - a **Debian 13** VM (the OS for ~120 of the ~140 fleet nodes), and
   - a **Windows Server 2025** VM (the OS for the AD domain controllers and the
     SQL Server FCI/AG nodes),

   each carried all the way from a blank VMware VM through the OS install to the
   **per-node baseline** (hostname, dual-NIC static networking, hardened SSH,
   `nftables`, time sync, node metrics) that makes a node a *NexusPlatform* node
   rather than a bare OS.

In the automated repo this work is split across `packer/deb13` (build on NAT →
golden image), `packer/ws2025-*` (same for Windows), the four shared Ansible
roles + five shared PowerShell scripts (the baseline), and Terraform's
`modules/vm` (clone + attach the real VMnet11/VMnet10 NICs + static IPs at
instantiation time). The manual path collapses that into one linear procedure:
**install over NAT so package fetches just work, apply the baseline, then
re-home the NICs to the dual-NIC VMnet11+VMnet10 static topology.**

**Dependency:** nothing. This guide is the root of the build-dependency tree —
it needs only the VMware Workstation host and the two install ISOs. Every other
guide depends, directly or transitively, on this one.

> **Reading note — the "golden install vs. per-node" model.** The automated lab
> builds *one* `deb13` image and clones it 120 times; the clone-time firstboot
> script assigns each clone its hostname/IPs by MAC. By hand there is no clone
> step — you repeat §5.B (Debian) or §5.C (Windows) for **each** node a later
> guide lists, substituting that node's hostname, IPs, and MACs from the guide's
> *Target topology* table. To keep the steps concrete, §5.B is written as a
> fully worked example that builds **`vault-1`** (the first real node, from
> Guide 03); every `<angle-bracket>` value is the per-node value you swap.

---

## 2. Component primer

Everything introduced in this guide — **what it is**, **why we use it here**,
and **what it would otherwise be** (the alternative we rejected).

### Virtualization + networking

- **VMware Workstation Pro (on Windows 11).** A type-2 hypervisor that runs the
  whole lab on one workstation. *Why:* it is what the lab host runs, it has a
  GUI for hand-building VMs (this guide's whole premise), and its host-only
  virtual networks give us isolated L2 segments without touching the physical
  LAN. *Otherwise:* Hyper-V (Windows-native, but its virtual-switch model and
  lack of a per-VM `.vmx` text file make the by-hand story messier), or a
  type-1 hypervisor like ESXi/Proxmox (needs dedicated tin — out of scope for a
  single workstation).
- **Host-only virtual network.** A VMware network with no bridge to the physical
  LAN and (here) no VMware NAT or DHCP — just an isolated subnet plus a host
  virtual adapter. *Why two of them:* separating **management/app** traffic
  (VMnet11) from **cluster replication/quorum** traffic (VMnet10) is how the
  whole fleet is designed — Raft peers, Galera SST, Patroni REST, Mongo
  replication, and Kafka controller quorum all bind to the backplane so they
  never contend with or leak onto the app plane. *Otherwise:* one flat network
  (simpler, but no traffic isolation and no way to firewall backplane ports off
  the app plane), or VMware NAT for VMnet11 (impossible — see the next item).
- **`nexus-gateway` as the lab edge router (forward reference: Guide 01).**
  VMware Workstation allows exactly **one NAT network per host**, and that slot
  is already held by the pre-existing VMnet8. So VMnet11 **cannot** be a NAT
  network. Instead, internet egress for the lab is provided by a dedicated
  Debian VM, `nexus-gateway`, sitting at `192.168.70.1` with a bridged NIC to
  the real LAN and `nftables` masquerade. It also serves DHCP (scope
  `.200–.250`, *for installs only*) and DNS for the lab. *Guide 00 builds VMs
  before that gateway exists*, which is exactly why §5 installs over the
  workstation's **NAT (VMnet8)** network for the OS install + baseline apt
  fetch, then switches to VMnet11/VMnet10 afterward.
- **Dual-NIC + static-everything policy.** Production VMs are **static on both
  NICs**. VMnet11 DHCP exists only for the install phase. *Why static:* a lab
  whose IPs are stable across reboots is reproducible and documentable;
  certificates carry IP SANs that must not move. *Otherwise:* DHCP reservations
  everywhere (the automated lab uses these as a convenience because Packer needs
  DHCP during build, but the canonical end state is static).

### Debian 13 (Trixie) baseline

- **Debian 13 "netinst" ISO.** A ~700 MB network-install image: it boots an
  installer that downloads the rest from a Debian mirror. *Why:* minimal
  footprint, current stable, and the installer's preseed/manual flow is
  well-understood. We pin **13.4.0** with a known SHA-256. *Otherwise:* Ubuntu
  Server (the repo keeps an `ubuntu24` template too, but Debian is the fleet
  default — leaner, no snap, predictable apt origins), or a full DVD ISO
  (needlessly large; we have internet during install).
- **`systemd-networkd` + `systemd .link` files.** `networkd` is systemd's
  network manager; a `.link` file renames a kernel interface
  (`ens160` → `nic0`) deterministically. *Why:* stable interface names across
  reboots so firewall rules and configs can reference `nic0`/`nic1` by name; on
  a dual-NIC box the rename **must** be keyed to the NIC's **MAC address**, not
  a glob like `en*`, or udev greedily renames the wrong NIC (a real failure we
  reproduce the fix for in §5.B). *Otherwise:* classic `ifupdown`
  (`/etc/network/interfaces` — fine, but networkd is the modern systemd-native
  path and matches the automated lab), or NetworkManager (heavier, desktop-
  oriented).
- **`nftables`.** The modern Linux in-kernel firewall (successor to iptables).
  *Why:* one ruleset file, atomic reloads, clean inet (v4+v6) tables; our
  baseline is default-deny-inbound, allowing only SSH (22) and node metrics
  (9100) from VMnet11. *Otherwise:* `ufw`/`firewalld` (wrappers that ultimately
  drive nftables anyway — we want the raw ruleset so it reads as code), or
  iptables-legacy (deprecated).
- **`chrony`.** An NTP client/server daemon. *Why:* a cluster with skewed clocks
  breaks Kerberos, TLS validity windows, Raft leases, and log correlation; every
  node is a chrony **client** of `nexus-gateway` (`192.168.70.1`) with a public
  pool fallback for the pre-gateway install phase. *Otherwise:* `systemd-timesyncd`
  (SNTP-only, no server mode, less control) or `ntpd` (heavier, legacy).
- **`prometheus-node-exporter`.** Exposes host metrics (CPU, memory, disk, net)
  as Prometheus text on `:9100`. *Why:* the Phase 0.I observability tier scrapes
  it; baking it into every node means a host is observable the moment it joins.
  *Otherwise:* collectd/telegraf (different ecosystem; node_exporter is the
  Prometheus-native standard).
- **OpenSSH server + key-only auth.** Remote administration. *Why:* during the
  build phase we keep password auth on (so we can log in before a key is
  placed), then harden to pubkey-only with a small set of session/auth limits.
  *Otherwise:* leaving passwords on (rejected — keys are the lab standard), or
  removing the password fallback entirely (rejected — the password fallback is
  *intentional* recovery insurance for console/SSH lockout).
- **`unattended-upgrades`.** Auto-applies **security** updates only, no auto-
  reboot. *Why:* keeps the long-lived lab patched without surprise restarts.
  *Otherwise:* manual patching (drifts), or full auto-upgrade incl. reboots
  (would knock nodes out from under a running cluster).

### Windows Server 2025 baseline

- **Windows Server 2025, Desktop Experience vs. Server Core.** Two SKUs from the
  same ISO. **Desktop Experience** (full GUI + RSAT/GPMC/DNS tools) is used for
  the **domain controllers** (`dc-nexus`, `dc-nexus-2`) and the SQL nodes that
  benefit from a console; **Server Core** (no GUI, smaller footprint) is the lean
  option. *Why two:* DC/admin roles want the management MMCs; lean roles
  shouldn't pay the GUI's RAM/disk cost. The SQL FCI/AG nodes in this lab use
  **Desktop Experience** (per `vms.yaml`). *Otherwise:* Windows client SKUs (not
  licensed/supported for AD DS or SQL FCI), or Linux (the relational-HA tier is
  deliberately Windows to exercise WSFC + Always-On).
- **`Autounattend.xml` → (here) interactive Setup.** The automated lab feeds
  Setup an unattended-answer file (disk layout, image pick, admin account, OOBE
  skips) and a `bootstrap-winrm.ps1` first-logon command so Packer can drive it
  over WinRM. **By hand we just answer those same screens in the installer GUI**
  and then RDP/console in — there is no WinRM and no sysprep/generalize, because
  there's no image to clone. §5.C maps each Autounattend setting to its manual
  click.
- **Windows OpenSSH Server (Feature-on-Demand).** WS2025 ships the OpenSSH
  Server payload in `install.wim`, so `Add-WindowsCapability` installs it fast
  and offline. *Why:* gives Windows nodes the **same** key-based SSH admin path
  as Linux nodes (consistent automation surface). *Otherwise:* WinRM-only
  (we use WinRM only transiently during the automated build; runtime is SSH),
  or the standalone Win32-OpenSSH GitHub build (only needed on client SKUs that
  lack the FOD payload).
- **Windows Firewall (default-deny inbound).** Same posture as nftables: block
  inbound by default, then allow SSH (22), `windows_exporter` (9182), RDP
  (3389), and ICMP echo — each **scoped to VMnet11** only. *Otherwise:* leaving
  the stock "allow from Any" rules (too open).
- **`windows_exporter`.** The Windows analog of node_exporter, on `:9182`.
  Same role, same scrape target model.
- **W32Time.** Windows' time service, pointed at `nexus-gateway` first with
  `time.windows.com` fallback — the chrony analog.
- **RSAT / GPMC / DNS+DHCP tools (Desktop Experience only).** The management
  MMC snap-ins the DC role needs. Installed as Windows Features. The actual AD
  forest promotion is **not** here — it's Guide 02; this guide only stages the
  tools so Guide 02 can run.

---

## 3. Prerequisites

This is the root guide, so the prerequisites are about the **host**, not other
VMs.

| # | Requirement | One-command verify (elevated PowerShell on the host) |
|---|---|---|
| 1 | VMware Workstation Pro 17.5+ installed | `& 'C:\Program Files\VMware\VMware Workstation\vmware.exe' -v` (or check **Help → About**) |
| 2 | The Virtual Network Editor is available | `Test-Path 'C:\Program Files (x86)\VMware\VMware Workstation\vmnetcfg.exe'` → `True` (path may be non-`(x86)` after an upgrade) |
| 3 | `vmrun` is available (used later to power VMs from the CLI) | `Test-Path 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'` → `True` |
| 4 | The Debian 13 netinst ISO is staged | `Test-Path 'H:\VMS\ISO\debian-13.4.0-amd64-netinst.iso'` → `True` |
| 5 | The Windows Server 2025 ISO is staged | `Test-Path 'H:\VMS\ISO\WindowsServer2025Evaluation.iso'` → `True` (or `WindowsServer2025.iso` for the MSDN/retail path) |
| 6 | The VM storage volume exists | `Test-Path 'H:\VMS\NexusPlatform'` → `True` |
| 7 | A NAT network exists for install-time internet | Virtual Network Editor shows **VMnet8** as **NAT** with DHCP on (the stock VMware default) |
| 8 | You have the owner SSH keypair | `Test-Path '~/.ssh/nexus_gateway_ed25519.pub'` → `True` (this guide installs its public half into `nexusadmin`'s `authorized_keys`) |

**Expected ISO checksums** (verify before first use — a corrupt ISO wastes an
entire install):

```powershell
# Debian 13.4.0 netinst — must equal 0b813535dd76f2ea96eff908c65e8521512c92a0631fd41c95756ffd7d4896dc
(Get-FileHash 'H:\VMS\ISO\debian-13.4.0-amd64-netinst.iso' -Algorithm SHA256).Hash

# WS2025 Evaluation — must equal 7b052573ba7894c9924e3e87ba732ccd354d18cb75a883efa9b900ea125bfd51
(Get-FileHash 'H:\VMS\ISO\WindowsServer2025Evaluation.iso' -Algorithm SHA256).Hash
```

> The `nexus_gateway_ed25519` public key shipped in the automated lab is:
> ```
> ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICOU5a2EcILbf/4050GGVRfw8FfdlyF59cHwGcnYUhT5 nexusadmin@nexus-gateway
> ```
> Use your own keypair's public half if you are rebuilding from scratch.

---

## 4. Target topology

Guide 00 doesn't build a fixed VM set — it builds the **host networks** plus the
**method**. Two reference tables follow: the networks (built once, in §5.A) and
the two base-VM hardware profiles (the shapes §5.B/§5.C produce).

### Host virtual networks (built once)

| VMnet | Mode | CIDR | DHCP | Host adapter IP | Role |
|---|---|---|---|---|---|
| **VMnet10** | Host-only (isolated) | `192.168.10.0/24` | **off** | `192.168.10.1/24` | Cluster backplane — replication, heartbeats, Raft/etcd peers, Galera SST, Patroni REST, Mongo replication, Kafka controller quorum |
| **VMnet11** | Host-only (routed via `nexus-gateway`) | `192.168.70.0/24` | **off** at the VMware layer (`nexus-gateway`'s dnsmasq serves `.200–.250` for installs only) | `192.168.70.254/24` | Mgmt, SSH/RDP, app traffic, internet egress (gateway `.1`) |
| VMnet8 | NAT (pre-existing) | (VMware default) | on | (VMware default) | **Install-time internet only.** Not part of the lab topology; used by §5 during OS install + baseline. |

Reserved on VMnet11: **`.1` = nexus-gateway**, **`.2`–`.9` = future edge
appliances**, **`.200`–`.250` = install DHCP pool**, **`.254` = host**.

### Base-VM hardware profiles

These match the automated templates' defaults. Per-node guides may shrink RAM
(the lab's standing rule is *smallest RAM that runs the workload* — e.g. Vault
runs at 2 GB) or grow disk.

| Profile | Used by | vCPU | RAM (build) | Disk | Firmware | NIC type | Disk controller |
|---|---|---:|---:|---:|---|---|---|
| **Debian 13** (`deb13`) | ~120 Linux nodes (vault, swarm, kafka, OLTP, analytics, lakehouse, registry, observability, vitess, citus) | 2 | **1024 MB** (installer needs ≥780 MB; runtime shrinks to 1–2 GB) | 10 GB growable | BIOS (default) | `vmxnet3` | default (LSI) |
| **WS2025 Desktop** (`ws2025-desktop`) | `dc-nexus`, `dc-nexus-2`, `sql-fci-1/2`, `sql-ag-rep-1/2` | 4 | 6144 MB | 80 GB | **EFI** | `e1000e` | **`lsisas1068`** (WinPE has no PVSCSI driver in-box) |
| WS2025 Core (`ws2025-core`) | lean Windows roles (none in the current fleet; profile kept for parity) | 4 | 4096 MB | 60 GB | EFI | `e1000e` | `lsisas1068` |

### Worked-example node (used throughout §5.B)

| Field | Value | Source |
|---|---|---|
| Hostname | `vault-1` | `vms.yaml` foundation tier |
| VMnet11 IP (nic0) | `192.168.70.121/24` | `vms.yaml` |
| VMnet10 IP (nic1) | `192.168.10.121/24` | `vms.yaml` |
| Primary MAC (nic0, VMnet11) | `00:50:56:3F:00:40` | `terraform/envs/security` MAC plan |
| Secondary MAC (nic1, VMnet10) | `00:50:56:3F:01:40` | `terraform/envs/security` MAC plan |
| VMware folder | `H:\VMS\NexusPlatform\01-foundation\vault-1\` | per-VM-folder convention |

> **MAC convention (whole fleet):** primary (VMnet11) =
> `00:50:56:3F:00:XX`; secondary (VMnet10) = `00:50:56:3F:01:XX` — same `XX`
> sixth byte, fifth byte flips `00`→`01`. The fourth byte is `3F` (inside
> VMware's `00`–`3F` user-managed OUI range). Each later guide's topology table
> states its nodes' exact `XX`.

---

## 5. Step-by-step build

Three parts: **A** creates the host networks (once). **B** builds a Debian 13
node end-to-end (worked as `vault-1`). **C** builds a Windows Server 2025 node
end-to-end (the DC/SQL profile).

> **WHERE convention:** the **WHERE** line names the machine + user. "Host" =
> the Windows 11 workstation (`10.0.70.101`). VM steps name the guest + the user
> (`nexusadmin`, or `root` via `sudo -i`). Windows-guest steps say "console/RDP,
> elevated PowerShell".

---

### Part A — Host virtual networks (one-time)

> **Step A.1 — Open the Virtual Network Editor as Administrator**
> **WHERE:** Host, elevated PowerShell.
> **WHY:** VMnet subnet/type/DHCP wiring must be done in the GUI —
> `vnetlib64.exe`'s `set addr` / `add dhcp` sub-commands silently no-op on
> Workstation Pro 17.5+. The editor needs admin to edit `netmap.conf`.
> **WHAT:**
> ```powershell
> Start-Process 'C:\Program Files (x86)\VMware\VMware Workstation\vmnetcfg.exe' -Verb RunAs
> ```
> **EXPECTED:** the **Virtual Network Editor** window opens, listing existing
> networks (typically VMnet0 bridged, VMnet1 host-only, VMnet8 NAT).
> **VERIFY:** the **Change Settings** button (lower right) is enabled — confirms
> the elevated handle.

> **Step A.2 — Create VMnet10 (isolated backplane)**
> **WHERE:** Host, Virtual Network Editor.
> **WHY:** the cluster backplane carries replication/quorum traffic, isolated
> from the app plane and from the internet. No DHCP — every backplane IP is
> assigned statically by the node baseline.
> **WHAT:** in the editor —
> 1. **Add Network…** → pick **VMnet10** → **OK**.
> 2. Select VMnet10, set **Type → Host-only**.
> 3. ✅ **Connect a host virtual adapter to this network** (the host sits at
>    `.1` for probes).
> 4. ❌ **Uncheck** *Use local DHCP service to distribute IP addresses to VMs*.
> 5. **Subnet IP** `192.168.10.0`, **Subnet mask** `255.255.255.0`.
> 6. **Apply**.
> **EXPECTED:** VMnet10 appears as *Host-only*, DHCP *disabled*, subnet
> `192.168.10.0`.
> **VERIFY:** after Apply, a *VMware Network Adapter VMnet10* appears in
> Windows' network adapters (Step A.4 confirms its IP).

> **Step A.3 — Create VMnet11 (routed service plane)**
> **WHERE:** Host, Virtual Network Editor.
> **WHY:** the management/app plane. It is **Host-only at the VMware layer**
> (it cannot be NAT — VMnet8 holds the only NAT slot); egress is provided later
> by the `nexus-gateway` VM at `.1`. DHCP is left **off** at the VMware layer —
> `nexus-gateway`'s dnsmasq serves the install-only `.200–.250` scope once it
> exists.
> **WHAT:** in the editor —
> 1. **Add Network…** → pick **VMnet11** → **OK**.
> 2. Select VMnet11, set **Type → Host-only** (**not** NAT).
> 3. ✅ **Connect a host virtual adapter to this network**.
> 4. ❌ **Uncheck** *Use local DHCP service…*.
> 5. **Subnet IP** `192.168.70.0`, **Subnet mask** `255.255.255.0`.
> 6. **Apply**, then **OK** to close the editor.
> **EXPECTED:** VMnet11 = *Host-only*, DHCP *disabled*, subnet `192.168.70.0`.
> **VERIFY:** *VMware Network Adapter VMnet11* now exists in Windows.

> **Step A.4 — Pin the host adapter IPs**
> **WHERE:** Host, elevated PowerShell.
> **WHY:** the host must sit at `192.168.10.1` (backplane probes) and
> `192.168.70.254` (service plane; `.1` is reserved for `nexus-gateway`).
> VMware sometimes leaves the new adapters on APIPA (`169.254.x.x`) until
> cycled, so we cycle then hard-set.
> **WHAT:**
> ```powershell
> Disable-NetAdapter -Name 'VMware Network Adapter VMnet10','VMware Network Adapter VMnet11' -Confirm:$false
> Start-Sleep 2
> Enable-NetAdapter  -Name 'VMware Network Adapter VMnet10','VMware Network Adapter VMnet11' -Confirm:$false
>
> # If either adapter is on 169.254.x.x after cycling, hard-set it:
> New-NetIPAddress -InterfaceAlias 'VMware Network Adapter VMnet10' -IPAddress 192.168.10.1   -PrefixLength 24 -ErrorAction SilentlyContinue
> New-NetIPAddress -InterfaceAlias 'VMware Network Adapter VMnet11' -IPAddress 192.168.70.254 -PrefixLength 24 -ErrorAction SilentlyContinue
> ```
> **EXPECTED:** no errors (or "already exists" if VMware's DHCP-less host
> adapter already took the configured `.1`/`.254`).
> **VERIFY:**
> ```powershell
> Get-NetIPAddress -InterfaceAlias 'VMware Network Adapter VMnet10','VMware Network Adapter VMnet11' |
>   Where-Object AddressFamily -eq IPv4 | Format-Table InterfaceAlias, IPAddress, PrefixLength
> ```
> shows `VMnet10 → 192.168.10.1/24` and `VMnet11 → 192.168.70.254/24`.

> **Step A.5 — Add the lab DNS resolver on the host's VMnet11 adapter (deferred)**
> **WHERE:** Host, elevated PowerShell.
> **WHY:** so the workstation can resolve `*.nexus.lab` / `*.nexus.local`
> service names host-wide once the gateway/DC exist. **This is a no-op until
> Guide 01 (`nexus-gateway`) is up** — record it here so the host-network setup
> is complete-by-reference.
> **WHAT (run after Guide 01):**
> ```powershell
> Set-DnsClientServerAddress -InterfaceAlias 'VMware Network Adapter VMnet11' `
>   -ServerAddresses '192.168.70.1','1.1.1.1'
> ```
> **EXPECTED:** the secondary DNS `192.168.70.1` is registered on VMnet11.
> **VERIFY:** after Guide 01, `Resolve-DnsName nexus-gateway.nexus.local` returns
> `192.168.70.1`.

---

### Part B — Build a Debian 13 node (worked example: `vault-1`)

Repeat this whole part once per Debian node a later guide lists, swapping the
`<angle-bracket>` values for that node's row in its topology table. The worked
values are `vault-1`'s (see §4).

#### B.1 — Create the VM in the VMware GUI

> **Step B.1.1 — New VM, custom, point at the Debian ISO**
> **WHERE:** Host, VMware Workstation GUI.
> **WHY:** a custom (advanced) VM lets us pick firmware, controllers, and NIC
> types deliberately rather than accept Workstation's auto-guess.
> **WHAT:**
> 1. **File → New Virtual Machine… → Custom (advanced) → Next**.
> 2. Hardware compatibility: **Workstation 17.x** (hardware version 20) → Next.
> 3. **Installer disc image file (iso)** → browse to
>    `H:\VMS\ISO\debian-13.4.0-amd64-netinst.iso` → Next. (VMware may say it
>    can't detect the OS — fine, we set it manually next.)
> 4. Guest OS: **Linux**, Version: **Debian 12.x 64-bit** (Workstation's catalog
>    lags; this is the compatible profile for Debian 13) → Next.
> 5. **Name** `<hostname>` (worked: `vault-1`); **Location**
>    `H:\VMS\NexusPlatform\<tier>\<hostname>\` (worked:
>    `H:\VMS\NexusPlatform\01-foundation\vault-1\`) → Next.
> **EXPECTED:** the per-VM folder is created and selected.
> **VERIFY:** `Test-Path 'H:\VMS\NexusPlatform\01-foundation\vault-1'` → `True`.

> **Step B.1.2 — CPU, RAM, disk, controller**
> **WHERE:** Host, VMware New-VM wizard (continued).
> **WHY:** match the `deb13` profile — 2 vCPU, **1024 MB** at install (the Debian
> installer drops to a degraded low-memory mode below ~780 MB, which disables
> fetching the preseed and slows the install), a single growable 10 GB disk.
> **WHAT:**
> 1. Processors: **1 processor × 2 cores** (= 2 vCPU) → Next.
> 2. Memory: **1024 MB** → Next.
> 3. Network type: **Use network address translation (NAT)** → Next. *(Install
>    over NAT — at Guide-00 time `nexus-gateway` doesn't exist yet, so VMnet11
>    has no DHCP/egress. We re-home to VMnet11+VMnet10 in B.4.)*
> 4. I/O controller types: **LSI Logic** (default) → Next.
> 5. Disk type: **SCSI** → Next. **Create a new virtual disk** → Next.
> 6. Maximum disk size **10 GB**; ✅ **Store virtual disk as a single file**
>    (growable single-file VMDK) → Next → Next → **Finish** (do **not** power on
>    yet).
> **EXPECTED:** the VM is created, powered off, with one NAT NIC.
> **VERIFY:** the VM's **Hardware** summary shows 2 processors, 1 GB RAM, a 10 GB
> SCSI disk, and one **Network Adapter (NAT)**.

> **Step B.1.3 — Set the NIC type to vmxnet3**
> **WHERE:** Host, VM Settings → Hardware → Network Adapter.
> **WHY:** the `deb13` profile uses the paravirtual `vmxnet3` adapter (better
> throughput than e1000). The single NIC here stays on **NAT** for the install.
> **WHAT:** **Edit virtual machine settings → Network Adapter → Advanced…** and
> confirm/select adapter type **VMXNET 3**. (If the GUI lacks the dropdown, edit
> `vault-1.vmx` and set `ethernet0.virtualDev = "vmxnet3"`.) → **OK**.
> **EXPECTED:** Network Adapter shows **NAT**, device **vmxnet3**.
> **VERIFY:** `Select-String 'ethernet0.virtualDev' 'H:\VMS\NexusPlatform\01-foundation\vault-1\vault-1.vmx'`
> → `ethernet0.virtualDev = "vmxnet3"`.

#### B.2 — Manual Debian 13 netinst walkthrough

Power the VM on (**▶ Power on this virtual machine**). At the GRUB boot menu
pick **Install** (text installer — lighter, identical outcome to Graphical
install). The screens below are the interactive equivalents of the automated
`preseed.cfg`; the **WHAT** lists exactly what to choose on each.

> **Step B.2.1 — Localisation + hostname/domain**
> **WHERE:** `vault-1` console (VMware VM window).
> **WHY:** matches the preseed locale/keymap and seeds an install-time hostname
> (the *real* hostname is set by the baseline in B.4, but the installer insists
> on one).
> **WHAT:** answer the screens —
> - Language **English** → Location **United States** → Locale **en_US.UTF-8**.
> - Keymap **American English**.
> - When networking auto-configures over NAT DHCP and asks **Hostname**: enter
>   `<hostname>` (worked: `vault-1`). **Domain name**: `nexus.local`.
> **EXPECTED:** the installer reports the network auto-configured (NAT DHCP gave
> it a `192.168.x.x` lease) and accepts the hostname/domain.
> **VERIFY:** the next screen (mirror/account) loads without a "network
> autoconfiguration failed" error — confirms NAT DHCP + internet reachability.

> **Step B.2.2 — Mirror**
> **WHERE:** `vault-1` console.
> **WHY:** the netinst pulls packages from a Debian mirror; we use the canonical
> `deb.debian.org`, no proxy (matches preseed).
> **WHAT:** Mirror country **United States** (or "enter manually" →
> `deb.debian.org`, directory `/debian`); **HTTP proxy** — leave blank.
> **EXPECTED:** the installer validates the mirror and downloads the release
> files.
> **VERIFY:** progress bar reaches "Loading additional components" without a
> mirror error.

> **Step B.2.3 — Accounts: root disabled, `nexusadmin` sudoer**
> **WHERE:** `vault-1` console.
> **WHY:** the lab disables direct root login and creates a single sudo-capable
> admin, `nexusadmin`, with the **build-time** password
> `nexus-packer-build-only` (this password is the intentional console/SSH
> recovery fallback — keep it).
> **WHAT:**
> - **Root password** — leave **both fields blank** and continue. (Blank root
>   password = root account locked, sudo-only — the preseed's
>   `passwd/root-login=false`.)
> - **Full name for the new user**: `NexusPlatform Admin`.
> - **Username**: `nexusadmin`.
> - **Password**: `nexus-packer-build-only` (twice). Accept the "weak password"
>   warning (**Yes**).
> **EXPECTED:** the installer accepts the locked root + `nexusadmin` account.
> **VERIFY:** later, post-boot, `sudo -n true` as `nexusadmin` succeeds (set up
> in B.3) and `getent passwd nexusadmin` shows the account.

> **Step B.2.4 — Clock + time zone**
> **WHERE:** `vault-1` console.
> **WHAT:** Time zone **UTC** (if asked to pick a US zone, choose any then
> correct to UTC in B.3 — the baseline pins UTC). Hardware clock set to **UTC**.
> **EXPECTED:** clock configured.
> **VERIFY:** post-boot `timedatectl` shows `Time zone: UTC`.

> **Step B.2.5 — Partition the whole disk (single ext4 root, NO swap partition — growable root)**
> **WHERE:** `vault-1` console.
> **WHY:** the automated preseed uses the `atomic` recipe with
> `partman-basicfilesystems/no_swap = true` → the disk becomes **one ext4 root
> partition and no swap partition**. This is deliberate. A swap partition placed
> *after* root (partman's default) makes root **not** the last partition, so
> `growpart /dev/sda 1` cannot extend it after a disk grow — `nexus scale-up
> --disk` would grow the vmdk but leave the guest root FS stranded behind the
> swap. With root as the single/last partition it grows cleanly (`growpart` +
> `resize2fs`). Swap parity is restored by a **2 GB `/swapfile`** (Step B.2.5a),
> matching the preseed `late_command`.
> **WHAT:**
> - Partitioning method: **Manual**.
> - Select the single **SCSI3 (0,0,0) (sda)** disk → **Yes** to create a new
>   empty partition table on it.
> - Select the **FREE SPACE** → **Create a new partition** → accept the **max**
>   size (the whole disk) → **Primary** → **Beginning** → set **Mount point: /**,
>   **Use as: Ext4**, **Bootable flag: on** → **Done setting up the partition**.
>   **Do NOT create a swap partition.**
> - **Finish partitioning and write changes to disk** → **Yes** to write. If the
>   installer warns *"You have not selected any partitions for use as swap
>   space"*, answer **No** (continue with no swap partition — the /swapfile in
>   B.2.5a replaces it).
> **EXPECTED:** the installer formats a single ext4 root on `sda1` and begins the
> base-system install.
> **VERIFY:** post-boot `lsblk` shows `sda` with ONE root ext4 partition (`sda1`)
> and no swap partition; `swapon --show` is empty (until B.2.5a).

> **Step B.2.5a — Create the 2 GB /swapfile (swap parity, keeps root growable)**
> **WHERE:** `vault-1` shell — either the installer's *"Execute a shell"* (the
> preseed does this in `late_command`) or the first post-boot login.
> **WHY:** restores 2 GB of swap without a partition, so root stays the single
> growable partition.
> **WHAT:**
> ```
> sudo dd if=/dev/zero of=/swapfile bs=1M count=2048 status=none
> sudo chmod 0600 /swapfile
> sudo mkswap /swapfile
> echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
> sudo swapon /swapfile
> ```
> **EXPECTED:** a 2 GB swapfile is active and persisted in `/etc/fstab`.
> **VERIFY:** `swapon --show` lists `/swapfile` (2G); `free -h` shows ~2.0Gi swap.

> **Step B.2.6 — Software selection: standard + SSH server only**
> **WHERE:** `vault-1` console.
> **WHY:** a minimal base — **no desktop**, no extra tasks; the baseline installs
> exactly the packages we need in B.3. Matches the preseed
> `tasksel: standard, ssh-server`.
> **WHAT:**
> - "Participate in the package usage survey?" → **No**.
> - **Software selection** (space-bar toggles): select **SSH server** and
>   **standard system utilities** ONLY. **Deselect** "Debian desktop
>   environment" and any GNOME/KDE entries. Continue.
> **EXPECTED:** apt installs the base + openssh-server + standard utils (a few
> minutes).
> **VERIFY:** post-boot `systemctl is-enabled ssh` → `enabled`.

> **Step B.2.7 — GRUB to /dev/sda, finish, first boot**
> **WHERE:** `vault-1` console.
> **WHY:** install the bootloader to the disk MBR so the VM boots without the
> ISO.
> **WHAT:**
> - "Install the GRUB boot loader to your primary drive?" → **Yes**.
> - Device: **/dev/sda**.
> - At "Installation complete" the installer reboots. The ISO auto-ejects; if
>   the VM re-enters the installer, power off and detach the ISO in VM Settings →
>   CD/DVD (uncheck *Connect at power on*), then power on.
> **EXPECTED:** the VM boots to a `vault-1 login:` prompt.
> **VERIFY:** log in as `nexusadmin` / `nexus-packer-build-only` at the console —
> you reach a shell.

#### B.3 — Apply the per-node baseline

These steps reproduce, by hand, the four shared Ansible roles + the Debian tail
(`nexus_identity`, `nexus_network`, `nexus_firewall`, `nexus_observability`,
`debian_base`). Run them as **root** over the console (or SSH once B.3.1 is done).

> **Step B.3.1 — Confirm passwordless sudo for `nexusadmin`**
> **WHERE:** `vault-1`, console as `nexusadmin`.
> **WHY:** the rest of the baseline runs via `sudo`; the automated path drops a
> sudoers file at install time. Reproduce it (idempotent).
> **WHAT:**
> ```bash
> echo 'nexusadmin ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/nexusadmin
> sudo chmod 0440 /etc/sudoers.d/nexusadmin
> sudo -i   # become root for the remaining baseline steps
> ```
> **EXPECTED:** `tee` echoes the line; you land in a root shell (`#` prompt).
> **VERIFY:** `sudo -n true && echo OK` prints `OK` (no password prompt).

> **Step B.3.2 — Install the baseline packages**
> **WHERE:** `vault-1`, root shell.
> **WHY:** the full Debian baseline package set (firewall, time, metrics,
> security-updates tooling, diagnostics). Mirrors `debian_base`'s apt task +
> the preseed `pkgsel/include`.
> **WHAT:**
> ```bash
> apt-get update -qq
> apt-get install -y \
>   openssh-server sudo curl ca-certificates gnupg \
>   nftables chrony prometheus-node-exporter \
>   unattended-upgrades apt-listchanges tcpdump mtr-tiny \
>   python3 python3-apt
> ```
> **EXPECTED:** apt installs/confirms all packages present.
> **VERIFY:**
> ```bash
> dpkg -l nftables chrony prometheus-node-exporter unattended-upgrades | grep '^ii' | wc -l
> ```
> → `4`.

> **Step B.3.3 — Deploy the owner SSH public key (nexus_identity)**
> **WHERE:** `vault-1`, root shell.
> **WHY:** key-based login for `nexusadmin`. The public key is the lab's
> `nexus_gateway_ed25519.pub`.
> **WHAT:**
> ```bash
> install -d -m 0700 -o nexusadmin -g nexusadmin /home/nexusadmin/.ssh
> cat > /home/nexusadmin/.ssh/authorized_keys <<'EOF'
> ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICOU5a2EcILbf/4050GGVRfw8FfdlyF59cHwGcnYUhT5 nexusadmin@nexus-gateway
> EOF
> chown nexusadmin:nexusadmin /home/nexusadmin/.ssh/authorized_keys
> chmod 0600 /home/nexusadmin/.ssh/authorized_keys
> ```
> **EXPECTED:** the file is written, owned by `nexusadmin`, mode `600`.
> **VERIFY:** `stat -c '%U %a' /home/nexusadmin/.ssh/authorized_keys` →
> `nexusadmin 600`.

> **Step B.3.4 — Harden sshd + first-boot host-key regeneration (nexus_identity)**
> **WHERE:** `vault-1`, root shell.
> **WHY:** the hardening drop-in sets sane server minimums (no root login, short
> grace, keepalives). The host-key drop-in matters for the *automated* clone
> model (Packer wipes host keys so each clone is unique); by hand your host keys
> already exist, but we install the same drop-in so the node is byte-identical
> to an automated one and survives a future key wipe. Password auth stays **on**
> for now — it is the intentional recovery fallback.
> **WHAT:**
> ```bash
> # sshd hardening drop-in
> cat > /etc/ssh/sshd_config.d/10-nexus-hardening.conf <<'EOF'
> PermitRootLogin no
> X11Forwarding no
> AllowAgentForwarding no
> MaxAuthTries 3
> LoginGraceTime 20
> ClientAliveInterval 60
> ClientAliveCountMax 3
> EOF
>
> # ssh.service drop-in: regenerate host keys before sshd -t, in order
> install -d -m 0755 /etc/systemd/system/ssh.service.d
> cat > /etc/systemd/system/ssh.service.d/10-regenerate-host-keys.conf <<'EOF'
> [Service]
> # Clear inherited ExecStartPre list, then re-declare in order:
> ExecStartPre=
> ExecStartPre=-/usr/bin/ssh-keygen -A
> ExecStartPre=/usr/sbin/sshd -t
> EOF
>
> systemctl daemon-reload
> systemctl restart ssh
> ```
> **EXPECTED:** `ssh` restarts cleanly.
> **VERIFY:** `sshd -t && echo SSHD_OK` → `SSHD_OK`; `systemctl is-active ssh`
> → `active`. From the host: `ssh -i ~/.ssh/nexus_gateway_ed25519 nexusadmin@<install-DHCP-IP>`
> logs in **without** a password prompt.

> **Step B.3.5 — chrony client config (nexus_network)**
> **WHERE:** `vault-1`, root shell.
> **WHY:** point time sync at `nexus-gateway` (`192.168.70.1`) with a public
> pool fallback for the pre-gateway phase. This node serves NTP to no one.
> **WHAT:**
> ```bash
> cat > /etc/chrony/chrony.conf <<'EOF'
> # deb13 — chrony client config
> # NexusPlatform time source: nexus-gateway (192.168.70.1) on VMnet11.
> # Public pool retained as fallback for cold-boot before the gateway is up.
>
> server 192.168.70.1 iburst prefer
> pool   2.debian.pool.ntp.org iburst
>
> driftfile /var/lib/chrony/chrony.drift
> ntsdumpdir /var/lib/chrony
>
> # Step the clock if >1s off; only during the first 3 updates
> makestep 1.0 3
>
> # Sync RTC
> rtcsync
>
> # This box is a CLIENT only — do not serve NTP to anyone
> deny all
>
> logdir /var/log/chrony
> EOF
> systemctl enable --now chrony
> systemctl restart chrony
> ```
> **EXPECTED:** chrony starts; over NAT it syncs from the public pool now, and
> will prefer `192.168.70.1` once the node is on VMnet11.
> **VERIFY:** `chronyc tracking | grep 'Leap status'` → `Normal` (give it
> ~30 s); `chronyc sources` lists `2.debian.pool.ntp.org`.

> **Step B.3.6 — nftables baseline ruleset (nexus_firewall)**
> **WHERE:** `vault-1`, root shell.
> **WHY:** default-deny inbound, allowing only SSH + node_exporter **from
> VMnet11** on `nic0`. (Role-specific guides add their service ports on top — on
> VMnet10 `nic1` for backplane ports.)
> **WHAT:**
> ```bash
> cat > /etc/nftables.conf <<'EOF'
> #!/usr/sbin/nft -f
> #
> # deb13 — nftables baseline ruleset
> # deny-all-inbound, allow management (SSH) + observability (node_exporter)
> # from the lab network (VMnet11, 192.168.70.0/24). Outbound unrestricted.
>
> flush ruleset
>
> table inet filter {
>     chain input {
>         type filter hook input priority 0; policy drop;
>
>         iif "lo" accept
>         ct state { established, related } accept
>         ct state invalid drop
>
>         # ICMP — let everything ping (debug-friendly in a lab)
>         ip protocol icmp   accept
>         ip6 nexthdr icmpv6 accept
>
>         # Management from the lab network only
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 22   accept comment "SSH from VMnet11"
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 9100 accept comment "node_exporter from VMnet11"
>
>         counter drop
>     }
>
>     chain forward {
>         type filter hook forward priority 0; policy drop;
>     }
>
>     chain output {
>         type filter hook output priority 0; policy accept;
>     }
> }
> EOF
> chmod 0755 /etc/nftables.conf
> nft -c -f /etc/nftables.conf   # syntax-check before enabling
> systemctl enable --now nftables
> systemctl restart nftables
> ```
> **EXPECTED:** `nft -c` prints nothing (valid); nftables enables + starts.
> **VERIFY:** `nft list chain inet filter input | grep 'dport 22'` shows the SSH
> accept rule; `systemctl is-active nftables` → `active`.
>
> > ⚠️ The baseline references `nic0` — which does **not exist yet** (the
> > interface is still `ens160` or similar over NAT). The rules are *installed*
> > now but only *match* after B.4 renames the primary NIC to `nic0`. SSH stays
> > reachable in the meantime because the `established,related` + the NAT path
> > already cover your current session; do B.4 before relying on the firewall.

> **Step B.3.7 — Enable node_exporter (nexus_observability)**
> **WHERE:** `vault-1`, root shell.
> **WHY:** expose host metrics on `:9100` for the future Prometheus tier.
> **WHAT:**
> ```bash
> systemctl enable --now prometheus-node-exporter
> ```
> **EXPECTED:** the service is active.
> **VERIFY:** `curl -s localhost:9100/metrics | head -1` returns a
> `# HELP ...` Prometheus metric line.

> **Step B.3.8 — Unattended security upgrades + MOTD (debian_base tail)**
> **WHERE:** `vault-1`, root shell.
> **WHY:** auto-apply Debian **security** updates (no auto-reboot), and stamp the
> base-image MOTD. Mirrors the `debian_base` role's tail.
> **WHAT:**
> ```bash
> cat > /etc/apt/apt.conf.d/50nexus-unattended <<'EOF'
> Unattended-Upgrade::Origins-Pattern {
>     "origin=Debian,codename=${distro_codename},label=Debian-Security";
> };
> Unattended-Upgrade::Automatic-Reboot "false";
> APT::Periodic::Update-Package-Lists "1";
> APT::Periodic::Unattended-Upgrade "1";
> EOF
>
> cat > /etc/motd <<'EOF'
> ╔═══════════════════════════════════════════════════════════════╗
> ║  deb13 — NexusPlatform generic Debian 13 base                 ║
> ║  Gateway: 192.168.70.1 (DNS/NTP/egress)                       ║
> ║  Services: nftables · chrony-client · node_exporter           ║
> ║  Built BY HAND per nexus-infra-manual/guides/00               ║
> ╚═══════════════════════════════════════════════════════════════╝
> EOF
> ```
> **EXPECTED:** both files written.
> **VERIFY:** `unattended-upgrade --dry-run -d 2>&1 | grep -i 'allowed origins'`
> lists the `Debian-Security` origin.

#### B.4 — Re-home to the dual-NIC VMnet11 + VMnet10 static topology

This is the manual equivalent of Terraform `modules/vm` attaching the real NICs
+ the clone-time firstboot assigning the static IPs/hostname. Two halves:
reconfigure the VM's virtual NICs (host, VM powered off), then configure the
guest's networking (boot, root shell).

> **Step B.4.1 — Replace the NAT NIC with two static-MAC NICs**
> **WHERE:** Host, VM Settings (power the VM **off** first: `sudo poweroff`).
> **WHY:** the node's final topology is dual-NIC: `nic0` on **VMnet11**, `nic1`
> on **VMnet10**, each with a **fixed MAC** from the fleet plan so the guest can
> identify them deterministically and so `nexus-gateway`'s DHCP reservation (and
> later guides' configs) line up.
> **WHAT:** in **Edit virtual machine settings → Hardware**:
> 1. Select the existing **Network Adapter** → set it to **Custom: VMnet11**.
> 2. **Add… → Network Adapter** → set it to **Custom: VMnet10**.
> 3. **OK**, then edit `<hostname>.vmx` to pin the MACs + adapter type
>    (VMware regenerates MACs on network change; pin them explicitly). For
>    `vault-1`:
> ```ini
> ethernet0.present              = "TRUE"
> ethernet0.connectionType       = "custom"
> ethernet0.vnet                 = "VMnet11"
> ethernet0.virtualDev           = "vmxnet3"
> ethernet0.addressType          = "static"
> ethernet0.address              = "00:50:56:3F:00:40"
>
> ethernet1.present              = "TRUE"
> ethernet1.connectionType       = "custom"
> ethernet1.vnet                 = "VMnet10"
> ethernet1.virtualDev           = "vmxnet3"
> ethernet1.addressType          = "static"
> ethernet1.address              = "00:50:56:3F:01:40"
> ```
> **EXPECTED:** VM Settings shows two adapters — one **VMnet11**, one
> **VMnet10**.
> **VERIFY:**
> `Select-String 'ethernet[01].address ' 'H:\VMS\NexusPlatform\01-foundation\vault-1\vault-1.vmx'`
> shows the two pinned MACs `...:00:40` and `...:01:40`.

> **Step B.4.2 — MAC-keyed `.link` rename + static `.network` for both NICs**
> **WHERE:** `vault-1`, root shell (power on; log in at console — over VMnet11
> it has no IP yet until this step runs).
> **WHY:** the automated `deb13` baseline ships a single
> `10-nic0.link` with `OriginalName=en*`, which on a **dual-NIC** box greedily
> matches **both** interfaces and renames the wrong one — a real failure
> (diagnosed 2026-05-01 during Vault bring-up). The fix is to key **each**
> `.link` to its NIC's **MAC**: primary MAC → `nic0` (VMnet11, DHCP→soon-static),
> secondary MAC → `nic1` (VMnet10, static). We assign `nic0` its static VMnet11
> IP here too (the lab is static-everywhere).
> **WHAT:** (substitute the node's MACs + IPs)
> ```bash
> PRIMARY_MAC=00:50:56:3F:00:40     # nic0  — VMnet11
> SECONDARY_MAC=00:50:56:3F:01:40   # nic1  — VMnet10
> VMNET11_IP=192.168.70.121
> VMNET10_IP=192.168.10.121
>
> # nic0 — MAC-matched rename (replaces the greedy en* baseline)
> cat > /etc/systemd/network/10-nic0.link <<EOF
> [Match]
> MACAddress=$PRIMARY_MAC
>
> [Link]
> Name=nic0
> EOF
>
> # nic1 — MAC-matched rename
> cat > /etc/systemd/network/20-nic1.link <<EOF
> [Match]
> MACAddress=$SECONDARY_MAC
>
> [Link]
> Name=nic1
> EOF
>
> # nic0 — static VMnet11 (mgmt/app/egress) with gateway + DNS
> cat > /etc/systemd/network/20-nic0.network <<EOF
> [Match]
> Name=nic0
>
> [Network]
> Address=$VMNET11_IP/24
> Gateway=192.168.70.1
> DNS=192.168.70.1
> IPv6AcceptRA=no
> EOF
>
> # nic1 — static VMnet10 (backplane); NO gateway (isolated), no DNS
> cat > /etc/systemd/network/20-nic1.network <<EOF
> [Match]
> Name=nic1
>
> [Network]
> Address=$VMNET10_IP/24
> LinkLocalAddressing=no
> DHCP=no
> IPv6AcceptRA=no
> EOF
>
> systemctl enable systemd-networkd
> udevadm control --reload
> reboot
> ```
> **EXPECTED:** after reboot the two interfaces are named `nic0`/`nic1` with the
> static IPs.
> **VERIFY (post-reboot, console):**
> ```bash
> ip -br addr show
> ```
> shows `nic0 ... 192.168.70.121/24` and `nic1 ... 192.168.10.121/24`. From the
> host: `ssh -i ~/.ssh/nexus_gateway_ed25519 nexusadmin@192.168.70.121` logs in.
>
> > **Why MAC-keyed and not `en*`:** with `OriginalName=en*`, on every boot udev
> > processes `10-nic0.link` first (lexical order), matches `en*` against **both**
> > NICs, renames the first to `nic0`, then fails to rename the second to `nic0`
> > (name clash) — leaving it at its kernel name, so `nic1` never appears and the
> > backplane has no IP. Quorum-dependent services (Vault Raft on `:8201`,
> > etcd, Patroni) then break on the next restart. MAC-keying makes each `.link`
> > match exactly one NIC. See `feedback_systemd_link_precedence_multi_nic.md`.

> **Step B.4.3 — Set the canonical hostname + `/etc/hosts` self-entry**
> **WHERE:** `vault-1`, root shell.
> **WHY:** set the real hostname (the installer's was a placeholder) and add the
> `127.0.1.1` self-resolution line — without it, **every** `sudo` prints
> `unable to resolve host <hostname>: Temporary failure in name resolution` to
> stderr, polluting all logs (canonized in
> `feedback_smoke_gate_probe_robustness.md`).
> **WHAT:** (substitute the hostname)
> ```bash
> HOSTNAME=vault-1
> hostnamectl set-hostname "$HOSTNAME"
> sed -i '/^127\.0\.1\.1\s/d' /etc/hosts
> echo "127.0.1.1 $HOSTNAME.nexus.lab $HOSTNAME" >> /etc/hosts
> ```
> **EXPECTED:** hostname set; `/etc/hosts` has the self-entry.
> **VERIFY:** `hostnamectl --static` → `vault-1`; `sudo -n true` prints **no**
> "unable to resolve host" warning.

> **Step B.4.4 — Add the VMnet10 backplane firewall rule (per dual-NIC node)**
> **WHERE:** `vault-1`, root shell.
> **WHY:** the baseline ruleset only opens VMnet11. Every dual-NIC node must also
> accept its cluster's backplane ports on `nic1` from `192.168.10.0/24` — the
> *role* guides add the specific dport rules, but the **principle** (and the
> generic "allow backplane subnet on nic1") belongs to the node baseline. The
> rule below is the generic backplane allow; later guides extend it with their
> exact ports (e.g. Vault `8201`).
> **WHAT:** insert into `/etc/nftables.conf` `chain input`, just after the
> VMnet11 lines, then reload:
> ```bash
> # (edit /etc/nftables.conf — add inside chain input, before `counter drop`)
> #   iifname "nic1" ip saddr 192.168.10.0/24 accept comment "backplane (VMnet10)"
> nft -c -f /etc/nftables.conf && systemctl reload nftables
> ```
> **EXPECTED:** `nft -c` validates; reload succeeds.
> **VERIFY:** `nft list chain inet filter input | grep nic1` shows the backplane
> accept rule.
>
> > For a node whose backplane ports are known at build time, prefer the
> > tightened per-port form the role guide gives (e.g.
> > `iifname "nic1" ip saddr 192.168.10.0/24 tcp dport 8201 accept`). The
> > broad-subnet form above is the safe baseline before the role is layered on.

`vault-1` is now a fully baselined Debian node. **Power it off**
(`sudo poweroff`) until Guide 03 needs it (minimal-running-VMs rule). To build
the next Debian node, repeat §5.B with that node's hostname/IPs/MACs.

---

### Part C — Build a Windows Server 2025 node (DC / SQL profile)

Repeat per Windows node (the DCs in Guide 02; the SQL FCI/AG nodes in Guide 11),
swapping the node's hostname/IPs/MACs. The example uses the
**Desktop Experience** profile (4 vCPU / 6 GB / 80 GB) since every Windows node
in the current fleet uses it. The actual **role** work (forest promotion, SQL
install, WSFC) belongs to the later guides — Guide 00 stops at a fully baselined,
domain-join-ready Windows node with the admin tooling staged.

#### C.1 — Create the VM in the VMware GUI

> **Step C.1.1 — New VM, custom, EFI firmware, WS2025 ISO**
> **WHERE:** Host, VMware Workstation GUI.
> **WHY:** WS2025 Setup refuses legacy BIOS — the VM must be **EFI**. Disk and
> NIC controller choices below avoid the in-box driver gaps in WinPE.
> **WHAT:**
> 1. **File → New Virtual Machine… → Custom (advanced) → Next**; hardware
>    compatibility **Workstation 17.x** → Next.
> 2. **Installer disc image file (iso)** → `H:\VMS\ISO\WindowsServer2025Evaluation.iso`
>    → Next. (VMware may not auto-detect — fine.)
> 3. Guest OS **Microsoft Windows**, Version **Windows Server 2022** (catalog
>    lags; compatible with 2025) → Next.
> 4. **Name** `<hostname>` (e.g. `dc-nexus`); **Location**
>    `H:\VMS\NexusPlatform\01-foundation\dc-nexus\` (SQL nodes:
>    `...\02-sqlserver\<hostname>\`) → Next.
> 5. **Firmware type: UEFI** (do **not** tick Secure Boot — Workstation can't
>    expose a real vTPM headless; Server SKUs install fine without it) → Next.
> **EXPECTED:** the per-VM folder is created.
> **VERIFY:** `Test-Path 'H:\VMS\NexusPlatform\01-foundation\dc-nexus'` → `True`.

> **Step C.1.2 — CPU, RAM, disk (lsisas1068 controller), finish**
> **WHERE:** Host, New-VM wizard (continued).
> **WHY:** Desktop Experience needs the bigger profile; the disk controller must
> be **LSI Logic SAS** (`lsisas1068`) because WS2025's WinPE has **no PVSCSI
> driver in-box** — a paravirtual SCSI disk would be invisible to Setup.
> **WHAT:**
> 1. Processors **2 × 2 cores** (= 4 vCPU) → Next.
> 2. Memory **6144 MB** → Next.
> 3. Network type **NAT** (install-time internet; re-homed in C.4) → Next.
> 4. I/O controller types **LSI Logic SAS** → Next.
> 5. Disk type **SCSI** → Create a new virtual disk → **80 GB**, ✅ single file
>    → Finish (**don't** power on).
> **EXPECTED:** VM created, powered off.
> **VERIFY:** Hardware summary: 4 processors, 6 GB RAM, 80 GB SCSI disk on an
> **LSI Logic SAS** controller, one NAT NIC.

> **Step C.1.3 — Set the NIC to e1000e**
> **WHERE:** Host, VM Settings → Network Adapter → Advanced (or `.vmx`).
> **WHY:** the `ws2025` profile uses **e1000e** (Windows Setup has the Intel
> driver in-box; vmxnet3 needs VMware Tools, which isn't installed yet).
> **WHAT:** set adapter device to **E1000e** (or `.vmx`:
> `ethernet0.virtualDev = "e1000e"`). Keep connection **NAT**.
> **EXPECTED:** Network Adapter = NAT, device e1000e.
> **VERIFY:** `Select-String 'ethernet0.virtualDev' '...\dc-nexus\dc-nexus.vmx'`
> → `e1000e`.

#### C.2 — Manual Windows Server 2025 install walkthrough

Power on. WS2025 boots into Setup. The screens below are the interactive
equivalents of the automated `Autounattend.xml`; **WHAT** lists the choice per
screen, and the **note** maps it to the answer-file setting.

> **Step C.2.1 — Language, edition, EULA**
> **WHERE:** `dc-nexus` console.
> **WHY:** pick the **Desktop Experience** edition (the answer file's
> `image_name` = "Windows Server 2025 Standard Evaluation (Desktop Experience)").
> **WHAT:**
> - Language **English (United States)**, time/format **en-US**, keyboard
>   **US** → Next → **Install now**.
> - Edition: **Windows Server 2025 Standard Evaluation (Desktop Experience)** →
>   Next. *(For a Core node, pick the non-"Desktop Experience" edition.)*
> - Accept the license terms → Next.
> **EXPECTED:** Setup advances to the install-type screen.
> **VERIFY:** the chosen edition shows "(Desktop Experience)".

> **Step C.2.2 — Custom install, partition the disk (UEFI/GPT)**
> **WHERE:** `dc-nexus` console.
> **WHY:** matches the answer file's GPT layout — EFI System Partition + MSR +
> OS partition. Letting Setup auto-create them on the empty 80 GB disk yields
> exactly this.
> **WHAT:**
> - Install type: **Custom: Install Microsoft Server Operating System only
>   (advanced)**.
> - Select the single **Drive 0 Unallocated Space** (80 GB) → **New** → **Apply**
>   → **OK** (Setup creates the ESP, MSR, and primary partitions automatically) →
>   select the large **Primary** partition → **Next**.
> **EXPECTED:** Setup copies + expands the image and reboots one or more times.
> **VERIFY:** the VM reaches the OOBE "Customize settings" screen
> (Administrator password).

> **Step C.2.3 — OOBE: set the Administrator password**
> **WHERE:** `dc-nexus` console.
> **WHY:** the answer file sets the local Administrator (and creates `nexusadmin`
> with the same password) to the build-time value. By hand we set Administrator
> now and create `nexusadmin` in the baseline (C.3.1).
> **WHAT:** at "Customize settings", set the **Administrator** password to
> `NexusPackerBuild!1` (twice) → **Finish**. Log in as **Administrator** with
> that password.
> **EXPECTED:** you reach the Windows Server desktop (Server Manager auto-opens).
> **VERIFY:** `whoami` in an elevated PowerShell → `<hostname>\administrator`.

> **Step C.2.4 — Install VMware Tools**
> **WHERE:** `dc-nexus` console, then VMware menu.
> **WHY:** Tools provides the paravirtual drivers, time sync, and graceful
> shutdown the lab relies on. (The automated path attaches the Tools ISO; by hand
> use the menu.)
> **WHAT:** VMware menu **VM → Install VMware Tools…**; in the guest, open the
> mounted **DVD Drive (VMware Tools)** → run **setup64.exe** → **Typical** →
> finish → **reboot** when prompted.
> **EXPECTED:** Tools installs; after reboot the VMware toolbox is present.
> **VERIFY (elevated PowerShell):**
> `Get-Service VMTools | Select-Object Status` → `Running`.

#### C.3 — Apply the per-node baseline

Reproduces the five shared PowerShell baseline scripts
(`01-nexus-identity` … `05-windows-baseline`) by hand. Run all of these in an
**elevated PowerShell** as Administrator on the node.

> **Step C.3.1 — Create `nexusadmin` + install OpenSSH Server + deploy key (01-nexus-identity)**
> **WHERE:** `dc-nexus`, elevated PowerShell.
> **WHY:** give the node the same key-based SSH admin path as the Linux fleet.
> WS2025 ships the OpenSSH Server payload in `install.wim`, so the FOD installs
> offline. For **admin** users, Windows OpenSSH reads
> `C:\ProgramData\ssh\administrators_authorized_keys` (not `~\.ssh`), so we write
> there.
> **WHAT:**
> ```powershell
> $ErrorActionPreference = 'Stop'
> $pw = ConvertTo-SecureString 'NexusPackerBuild!1' -AsPlainText -Force
>
> # 1. Local admin user nexusadmin
> if (-not (Get-LocalUser -Name nexusadmin -ErrorAction SilentlyContinue)) {
>     New-LocalUser -Name nexusadmin -Password $pw -FullName 'NexusPlatform Admin' `
>         -Description 'NexusPlatform build + runtime admin' -PasswordNeverExpires
> }
> Add-LocalGroupMember -Group Administrators -Member nexusadmin -ErrorAction SilentlyContinue
>
> # 2. OpenSSH Server via Feature-on-Demand (offline from install.wim)
> if (-not (Get-Service sshd -ErrorAction SilentlyContinue)) {
>     Add-WindowsCapability -Online -Name 'OpenSSH.Server~~~~0.0.1.0' | Out-Null
> }
>
> # 3. Owner public key -> administrators_authorized_keys (admin-group path)
> $pubkey = 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICOU5a2EcILbf/4050GGVRfw8FfdlyF59cHwGcnYUhT5 nexusadmin@nexus-gateway'
> New-Item -Path 'C:\ProgramData\ssh' -ItemType Directory -Force | Out-Null
> $adminAK = 'C:\ProgramData\ssh\administrators_authorized_keys'
> Set-Content -Path $adminAK -Value "$pubkey`n" -NoNewline -Encoding ascii
> icacls $adminAK /inheritance:r /grant 'Administrators:F' 'SYSTEM:F' | Out-Null
> ```
> **EXPECTED:** the `sshd` service is registered; the key file exists with locked
> ACL.
> **VERIFY:** `Get-Service sshd` exists; `(Get-Content $adminAK)` shows the key;
> `icacls $adminAK` lists only `Administrators` + `SYSTEM`.

> **Step C.3.2 — Harden sshd_config + start the service (01-nexus-identity cont.)**
> **WHERE:** `dc-nexus`, elevated PowerShell.
> **WHY:** key-only auth, no root, single allowed user. Windows OpenSSH has no
> `sshd_config.d` drop-in dir, so we write the whole `sshd_config`.
> **WHAT:**
> ```powershell
> $config = @'
> # sshd_config -- NexusPlatform Windows baseline
> Port 22
> AddressFamily inet
> ListenAddress 0.0.0.0
>
> PubkeyAuthentication yes
> PasswordAuthentication no
> PermitRootLogin no
> PermitEmptyPasswords no
>
> AuthorizedKeysFile C:/ProgramData/ssh/administrators_authorized_keys
>
> ClientAliveInterval 300
> ClientAliveCountMax 2
> MaxAuthTries 3
> MaxSessions 10
> LoginGraceTime 30
>
> AllowUsers nexusadmin
>
> Subsystem sftp sftp-server.exe
> '@
> Set-Content -Path 'C:\ProgramData\ssh\sshd_config' -Value $config -Encoding ascii
>
> Set-Service -Name sshd -StartupType Automatic
> Restart-Service -Name sshd -Force
> Set-Service -Name ssh-agent -StartupType Automatic
> Start-Service -Name ssh-agent
> ```
> **EXPECTED:** `sshd` restarts and listens on 22.
> **VERIFY:** `Get-Service sshd | Select Status` → `Running`;
> `Test-NetConnection localhost -Port 22` → `TcpTestSucceeded : True`.

> **Step C.3.3 — Rename the NIC + point W32Time at the gateway (02-nexus-network)**
> **WHERE:** `dc-nexus`, elevated PowerShell.
> **WHY:** label the single install-time NIC `nic0` (consistent metric labels);
> configure Windows time (`W32Time`) to prefer `nexus-gateway` with a public
> fallback (the chrony analog). DNS is pointed at the gateway so it's right the
> moment the NIC moves to VMnet11.
> **WHAT:**
> ```powershell
> # Rename the single active Ethernet adapter -> nic0
> $nic = Get-NetAdapter -Physical | Where-Object Status -eq 'Up' | Select-Object -First 1
> if ($nic -and $nic.Name -ne 'nic0') { Rename-NetAdapter -Name $nic.Name -NewName 'nic0' }
>
> # W32Time: gateway primary, time.windows.com fallback
> Set-Service -Name w32time -StartupType Automatic
> Start-Service -Name w32time -ErrorAction SilentlyContinue
> w32tm /config /manualpeerlist:"192.168.70.1,0x8 time.windows.com,0x9" `
>     /syncfromflags:manual /reliable:no /update | Out-Null
>
> # DNS -> gateway (right after the NIC moves to VMnet11)
> Set-DnsClientServerAddress -InterfaceAlias 'nic0' -ServerAddresses '192.168.70.1'
> ```
> **EXPECTED:** the adapter is named `nic0`; W32Time reconfigures.
> **VERIFY:** `Get-NetAdapter -Name nic0` exists;
> `w32tm /query /peers` lists `192.168.70.1`.

> **Step C.3.4 — Default-deny firewall + VMnet11 allowlist (03-nexus-firewall)**
> **WHERE:** `dc-nexus`, elevated PowerShell.
> **WHY:** same posture as nftables — block inbound by default, allow only SSH,
> `windows_exporter`, RDP, and ICMP echo **from VMnet11**.
> **WHAT:**
> ```powershell
> $vmnet11 = '192.168.70.0/24'
>
> Set-NetFirewallProfile -Profile Domain,Public,Private `
>     -DefaultInboundAction Block -DefaultOutboundAction Allow -Enabled True `
>     -LogBlocked True -LogFileName '%SystemRoot%\System32\LogFiles\Firewall\pfirewall.log'
>
> $rules = @(
>     @{ Name='Nexus-SSH-In';             Display='Nexus SSH (22) from VMnet11';               Port=22;   Proto='TCP' },
>     @{ Name='Nexus-WindowsExporter-In'; Display='Nexus windows_exporter (9182) from VMnet11'; Port=9182; Proto='TCP' },
>     @{ Name='Nexus-RDP-In';             Display='Nexus RDP (3389) from VMnet11';             Port=3389; Proto='TCP' }
> )
> foreach ($r in $rules) {
>     Remove-NetFirewallRule -Name $r.Name -ErrorAction SilentlyContinue
>     New-NetFirewallRule -Name $r.Name -DisplayName $r.Display -Direction Inbound `
>         -Action Allow -Protocol $r.Proto -LocalPort $r.Port -RemoteAddress $vmnet11 `
>         -Profile Domain,Public,Private -Enabled True | Out-Null
> }
>
> Remove-NetFirewallRule -Name 'Nexus-ICMPv4-In' -ErrorAction SilentlyContinue
> New-NetFirewallRule -Name 'Nexus-ICMPv4-In' -DisplayName 'Nexus ICMPv4 Echo from VMnet11' `
>     -Direction Inbound -Action Allow -Protocol ICMPv4 -IcmpType 8 `
>     -RemoteAddress $vmnet11 -Profile Domain,Public,Private -Enabled True | Out-Null
> ```
> **EXPECTED:** four allow rules created; default inbound = Block.
> **VERIFY:** `(Get-NetFirewallProfile -Profile Domain).DefaultInboundAction`
> → `Block`; `Get-NetFirewallRule -Name 'Nexus-*' | Measure-Object` → `Count 4`.
>
> > **Don't open WinRM/RDP to Any.** RDP is allowed only from VMnet11 above; it
> > is your console fallback if SSH breaks. Keep it scoped.

> **Step C.3.5 — Install `windows_exporter` (04-nexus-observability)**
> **WHERE:** `dc-nexus`, elevated PowerShell (needs install-time internet — NAT).
> **WHY:** Windows host metrics on `:9182` for the Prometheus tier.
> **WHAT:**
> ```powershell
> $version = '0.30.4'
> $msi = "C:\Windows\Temp\windows_exporter-$version.msi"
> [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12 -bor [Net.SecurityProtocolType]::Tls13
> Invoke-WebRequest -UseBasicParsing -OutFile $msi `
>   -Uri "https://github.com/prometheus-community/windows_exporter/releases/download/v$version/windows_exporter-$version-amd64.msi"
>
> $args = @('/i', $msi, '/qn', '/norestart',
>           'LISTEN_PORT=9182',
>           'ENABLED_COLLECTORS=cpu,cs,logical_disk,memory,net,os,service,system,tcp,textfile')
> $p = Start-Process msiexec.exe -ArgumentList $args -Wait -PassThru
> if ($p.ExitCode -notin 0,3010) { throw "windows_exporter MSI failed ($($p.ExitCode))" }
> Set-Service windows_exporter -StartupType Automatic
> Start-Service windows_exporter
> Remove-Item $msi -Force
> ```
> **EXPECTED:** the `windows_exporter` service installs + starts.
> **VERIFY:** `(Invoke-WebRequest http://127.0.0.1:9182/metrics -UseBasicParsing).StatusCode`
> → `200`.

> **Step C.3.6 — Windows baseline hardening (05-windows-baseline)**
> **WHERE:** `dc-nexus`, elevated PowerShell.
> **WHY:** the Windows OS-tail — security-only auto-updates, minimal telemetry,
> `RemoteSigned` execution policy, login banner, disable Server Manager
> auto-launch + CEIP, disable deprecated TLS, fixed pagefile.
> **WHAT:**
> ```powershell
> # 1. Windows Update: security auto, no reboot with users
> $wu = 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU'
> New-Item $wu -Force | Out-Null
> Set-ItemProperty $wu NoAutoUpdate 0 -Type DWord
> Set-ItemProperty $wu AUOptions 4 -Type DWord
> Set-ItemProperty $wu ScheduledInstallDay 0 -Type DWord
> Set-ItemProperty $wu ScheduledInstallTime 3 -Type DWord
> Set-ItemProperty $wu NoAutoRebootWithLoggedOnUsers 1 -Type DWord
>
> # 2. Telemetry -> Security
> $dc = 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection'
> New-Item $dc -Force | Out-Null
> Set-ItemProperty $dc AllowTelemetry 0 -Type DWord
>
> # 3. PowerShell execution policy (machine scope) -> RemoteSigned
> $ps = 'HKLM:\SOFTWARE\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell'
> New-Item $ps -Force | Out-Null
> Set-ItemProperty $ps ExecutionPolicy 'RemoteSigned' -Type String
>
> # 4. Login banner
> $sys = 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System'
> Set-ItemProperty $sys LegalNoticeCaption 'NexusPlatform'
> Set-ItemProperty $sys LegalNoticeText 'Authorized access only. All activity is logged.'
>
> # 5. Disable Server Manager auto-launch + CEIP
> $sm = Get-ScheduledTask -TaskPath '\Microsoft\Windows\Server Manager\' -TaskName 'ServerManager' -ErrorAction SilentlyContinue
> if ($sm) { Disable-ScheduledTask -InputObject $sm | Out-Null }
> Get-ScheduledTask -TaskPath '\Microsoft\Windows\Customer Experience Improvement Program\' -ErrorAction SilentlyContinue | Disable-ScheduledTask | Out-Null
>
> # 6. Disable SSL3/TLS1.0/1.1 at SCHANNEL
> $root = 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols'
> foreach ($proto in 'SSL 2.0','SSL 3.0','TLS 1.0','TLS 1.1') {
>   foreach ($side in 'Client','Server') {
>     $k = "$root\$proto\$side"; New-Item $k -Force | Out-Null
>     Set-ItemProperty $k Enabled 0 -Type DWord
>     Set-ItemProperty $k DisabledByDefault 1 -Type DWord
>   }
> }
>
> # 7. Fixed 4 GB pagefile
> $cs = Get-CimInstance Win32_ComputerSystem
> if ($cs.AutomaticManagedPagefile) { $cs | Set-CimInstance -Property @{ AutomaticManagedPagefile = $false } }
> $pf = Get-CimInstance Win32_PageFileSetting
> if ($pf) { $pf | Set-CimInstance -Property @{ InitialSize = 4096; MaximumSize = 4096 } }
> else { New-CimInstance -ClassName Win32_PageFileSetting -Property @{ Name='C:\pagefile.sys'; InitialSize=4096; MaximumSize=4096 } | Out-Null }
> ```
> **EXPECTED:** all registry writes + scheduled-task disables succeed.
> **VERIFY:** `(Get-ItemProperty $ps).ExecutionPolicy` → `RemoteSigned`;
> `(Get-ItemProperty $wu).AUOptions` → `4`.

> **Step C.3.7 — (DC nodes only) install RSAT / GPMC / DNS+DHCP tools (10-desktop-admin-tools)**
> **WHERE:** `dc-nexus`, elevated PowerShell (skip for SQL nodes).
> **WHY:** stage the management MMC tooling the **domain-controller** role
> (Guide 02) needs. This installs the *tools*, **not** the AD role — promotion is
> Guide 02.
> **WHAT:**
> ```powershell
> foreach ($feat in 'RSAT-AD-Tools','RSAT-DNS-Server','RSAT-DHCP','GPMC') {
>     if ((Get-WindowsFeature -Name $feat).InstallState -ne 'Installed') {
>         $r = Install-WindowsFeature -Name $feat -IncludeManagementTools
>         if (-not $r.Success) { throw "Install-WindowsFeature $feat failed" }
>     }
> }
> ```
> **EXPECTED:** features install (a restart may be requested — take it in C.4).
> **VERIFY:** `Get-WindowsFeature RSAT-AD-Tools,GPMC | Select Name,InstallState`
> → both `Installed`.

#### C.4 — Re-home to the dual-NIC VMnet11 + VMnet10 static topology

> **Step C.4.1 — Replace the NAT NIC with two static-MAC NICs**
> **WHERE:** Host, VM Settings (shut the guest down first:
> `Stop-Computer -Force` in the guest, or VMware **Shut Down Guest**).
> **WHY:** same dual-NIC end state as Linux — `nic0` on VMnet11, `nic1` on
> VMnet10, fixed MACs from the node's row in its guide's topology table. (Example
> MACs below are illustrative; use the node's real ones.)
> **WHAT:** in **VM Settings → Hardware**: set the existing adapter to **Custom:
> VMnet11**; **Add → Network Adapter → Custom: VMnet10**. Then pin MACs +
> `e1000e` in `<hostname>.vmx` (e.g. for `dc-nexus`, primary `00:50:56:3F:00:01`,
> secondary `00:50:56:3F:01:01` — substitute the guide's values):
> ```ini
> ethernet0.present        = "TRUE"
> ethernet0.connectionType = "custom"
> ethernet0.vnet           = "VMnet11"
> ethernet0.virtualDev     = "e1000e"
> ethernet0.addressType    = "static"
> ethernet0.address        = "00:50:56:3F:00:01"
>
> ethernet1.present        = "TRUE"
> ethernet1.connectionType = "custom"
> ethernet1.vnet           = "VMnet10"
> ethernet1.virtualDev     = "e1000e"
> ethernet1.addressType    = "static"
> ethernet1.address        = "00:50:56:3F:01:01"
> ```
> **EXPECTED:** two adapters — VMnet11 + VMnet10.
> **VERIFY:** `Select-String 'ethernet[01].vnet' '...\dc-nexus\dc-nexus.vmx'`
> shows `VMnet11` and `VMnet10`.

> **Step C.4.2 — Assign static IPs to both NICs + set hostname**
> **WHERE:** `dc-nexus`, elevated PowerShell (power on; log in at console).
> **WHY:** static everywhere. Identify each NIC by its MAC (the
> `00:..:00:..`=VMnet11 / `00:..:01:..`=VMnet10 convention), set the VMnet11 IP
> with gateway+DNS, the VMnet10 IP with no gateway, then set the real computer
> name.
> **WHAT:** (substitute the node's IPs/MACs/hostname)
> ```powershell
> $primaryMac   = '00-50-56-3F-00-01'   # VMnet11 (note Windows uses dashes)
> $secondaryMac = '00-50-56-3F-01-01'   # VMnet10
> $vmnet11Ip = '192.168.70.10';  $vmnet10Ip = '192.168.10.10'
>
> $nic0 = Get-NetAdapter | Where-Object { $_.MacAddress -eq $primaryMac }
> $nic1 = Get-NetAdapter | Where-Object { $_.MacAddress -eq $secondaryMac }
> Rename-NetAdapter -Name $nic0.Name -NewName 'nic0'
> Rename-NetAdapter -Name $nic1.Name -NewName 'nic1'
>
> # nic0 — VMnet11 static + gateway + DNS (gateway/DC resolver)
> New-NetIPAddress -InterfaceAlias 'nic0' -IPAddress $vmnet11Ip -PrefixLength 24 -DefaultGateway 192.168.70.1
> Set-DnsClientServerAddress -InterfaceAlias 'nic0' -ServerAddresses '192.168.70.1'
>
> # nic1 — VMnet10 backplane static, NO gateway
> New-NetIPAddress -InterfaceAlias 'nic1' -IPAddress $vmnet10Ip -PrefixLength 24
>
> # Computer name (reboot applies it)
> Rename-Computer -NewName 'dc-nexus' -Force
> Restart-Computer -Force
> ```
> **EXPECTED:** after reboot, `nic0`=`.70.10`, `nic1`=`.10.10`, computer name set.
> **VERIFY (post-reboot):** `Get-NetIPAddress -AddressFamily IPv4 |
> Where-Object InterfaceAlias -in 'nic0','nic1' | Format-Table InterfaceAlias,IPAddress`
> shows both; `hostname` → `dc-nexus`. From the host:
> `ssh -i ~/.ssh/nexus_gateway_ed25519 nexusadmin@192.168.70.10` logs in.

This Windows node is now baselined and **domain-join-ready** (DCs are *promoted*
in Guide 02; SQL nodes are *domain-joined* then built in Guide 11). **Shut it
down** until its role guide needs it.

---

## 6. Validation — by-hand acceptance smoke

Run these from the **host** (elevated PowerShell) after building at least one
Debian node (`vault-1`) and one Windows node (`dc-nexus`), with both powered on.
This mirrors the spirit of the automated `*-smoke.ps1` gates: prove the host
networks, both NICs, SSH, firewall, time, and metrics on each node.

| # | Check | Command (host) | Pass criteria |
|---|---|---|---|
| 1 | Host VMnet adapters correct | `Get-NetIPAddress -InterfaceAlias 'VMware Network Adapter VMnet10','VMware Network Adapter VMnet11' \| ? AddressFamily -eq IPv4 \| ft InterfaceAlias,IPAddress` | VMnet10 `192.168.10.1`, VMnet11 `192.168.70.254` |
| 2 | Debian node reachable on VMnet11 | `Test-NetConnection 192.168.70.121 -Port 22` | `TcpTestSucceeded : True` |
| 3 | Debian node reachable on VMnet10 backplane | `Test-Connection 192.168.10.121 -Count 2 -Quiet` | `True` |
| 4 | Debian key-only SSH works | `ssh -i ~/.ssh/nexus_gateway_ed25519 -o BatchMode=yes nexusadmin@192.168.70.121 'hostname'` | prints `vault-1` (no password prompt) |
| 5 | Debian dual-NIC + services | `ssh ... nexusadmin@192.168.70.121 'ip -br addr show; systemctl is-active nftables chrony prometheus-node-exporter ssh'` | `nic0 .70.121`, `nic1 .10.121`; four `active` |
| 6 | Debian node metrics | `ssh ... nexusadmin@192.168.70.121 'curl -s localhost:9100/metrics \| head -1'` | a `# HELP` line |
| 7 | Debian firewall denies non-VMnet11 | from a VMnet10-only context, `Test-NetConnection 192.168.10.121 -Port 22` | `TcpTestSucceeded : False` (22 only open on `nic0`/VMnet11) |
| 8 | Windows node reachable on VMnet11 | `Test-NetConnection 192.168.70.10 -Port 22` | `TcpTestSucceeded : True` |
| 9 | Windows key-only SSH works | `ssh -i ~/.ssh/nexus_gateway_ed25519 nexusadmin@192.168.70.10 'hostname'` | prints `dc-nexus` |
| 10 | Windows metrics endpoint | `ssh ... nexusadmin@192.168.70.10 'powershell -c "(iwr http://127.0.0.1:9182/metrics -UseBasicParsing).StatusCode"'` | `200` |
| 11 | Windows firewall default-deny | `ssh ... nexusadmin@192.168.70.10 'powershell -c "(Get-NetFirewallProfile -Profile Domain).DefaultInboundAction"'` | `Block` |
| 12 | Windows DC tooling staged (DC nodes) | `ssh ... nexusadmin@192.168.70.10 'powershell -c "(Get-WindowsFeature GPMC).InstallState"'` | `Installed` |

**All 12 green ⇒ Guide 00 is satisfied** for those two nodes, and the host is
ready for Guide 01 (`nexus-gateway`). Re-run checks 2–7 for every additional
Debian node and 8–12 for every additional Windows node you build.

> **Note on check 7 / time:** until Guide 01 brings up `nexus-gateway` at
> `192.168.70.1`, VMnet11 has **no egress and no gateway-served DNS/NTP**. A
> baselined node will still pass 1–12, but `chrony`/`W32Time` sync from the
> *public* fallback only and `ping 192.168.70.1` fails. That is expected at the
> Guide-00 boundary and clears the moment Guide 01 is done.

---

## 7. Teardown / reset

**Remove one node** (does not touch the host networks):

```powershell
# Graceful guest shutdown, then delete the VM + its folder.
& 'C:\Program Files\VMware\VMware Workstation\vmrun.exe' stop 'H:\VMS\NexusPlatform\01-foundation\vault-1\vault-1.vmx' soft
& 'C:\Program Files\VMware\VMware Workstation\vmrun.exe' deleteVM 'H:\VMS\NexusPlatform\01-foundation\vault-1\vault-1.vmx'
Remove-Item 'H:\VMS\NexusPlatform\01-foundation\vault-1' -Recurse -Force
```

**Remove the host networks** (only if rebuilding the whole lab — this breaks
*every* VM):

```powershell
$vl = 'C:\Program Files (x86)\VMware\VMware Workstation\vnetlib64.exe'
& $vl -- remove adapter vmnet10
& $vl -- remove adapter vmnet11
# Then re-add + reconfigure via §5.A if you intend to rebuild.
```

> The full host-network panic-button rebuild (after a VMware upgrade or an
> accidental "Restore Defaults") is the procedure in
> `nexus-platform-plan/docs/infra/network.md` → *"Panic button — rebuild both
> VMnets"*. Re-running §5.A by hand is equivalent.

---

## 8. Troubleshooting

Real gotchas from building this lab (drawn from the repo transient ledgers + the
operator memory). Symptom → cause → fix.

| Symptom | Cause | Fix |
|---|---|---|
| Debian installer drops to a **low-memory** mode and won't fetch the preseed/components | VM has < ~780 MB RAM at install | Give the VM **1024 MB** for the install (§B.1.2). Shrink RAM afterward in VM Settings if the role wants less. |
| After §B.4.2 reboot, `nic1` is **missing** and the VMnet10 IP never comes up | The stock `10-nic0.link` used `OriginalName=en*`, which greedily matches **both** NICs; udev renames one to `nic0` and can't rename the second (name clash) | Use **MAC-keyed** `.link` files (one per NIC) exactly as §B.4.2 — never `OriginalName=en*` on a dual-NIC box. See `feedback_systemd_link_precedence_multi_nic.md`. |
| Every `sudo` on a Linux node prints `unable to resolve host <name>: Temporary failure in name resolution` | No `127.0.1.1` self-entry in `/etc/hosts` after the hostname change | Add the self-entry (§B.4.3): `echo "127.0.1.1 <host>.nexus.lab <host>" >> /etc/hosts`. |
| Linux node has no internet during the baseline apt installs | You moved it to VMnet11 *before* `nexus-gateway` exists (no egress yet) | Keep the install + baseline on **NAT** (§B.1.2); only re-home to VMnet11/VMnet10 in §B.4, after the apt steps. |
| `nft -f` reload appears to succeed but a needed port is still blocked | A runtime `nft add rule` lands **after** the chain's `counter drop` (unreachable), or the rule references `nic0` before §B.4 renamed the NIC | Edit `/etc/nftables.conf` and `nft -f` the whole file (atomic replace); ensure `nic0`/`nic1` exist first. See `feedback_nftables_runtime_add_after_drop.md`. |
| WS2025 Setup shows **no disks** to install to | WinPE lacks the PVSCSI driver; the disk is on a paravirtual controller | Create the VM with the **LSI Logic SAS (`lsisas1068`)** controller (§C.1.2). |
| WS2025 install refuses to proceed / wrong firmware | VM is BIOS, or Secure Boot/TPM checks fail | Create the VM as **UEFI** without Secure Boot (§C.1.1); Server SKUs install without a TPM. |
| `Add-WindowsCapability OpenSSH.Server` hangs or "Access is denied" | (On client SKUs) FOD pulls over Windows Update + token filtering. On WS2025 the payload is in-box and this is fast | On WS2025 it should be offline + quick; if it stalls, confirm install-time **NAT internet** is up and you're in an **elevated** session. |
| Windows SSH key login still prompts for a password | Key written to `~\.ssh\authorized_keys` instead of the admin file; Windows OpenSSH reads `C:\ProgramData\ssh\administrators_authorized_keys` for **admin** users | Write the key to `administrators_authorized_keys` with the locked ACL (§C.3.1). |
| Can't reach a node on VMnet11 from the host though the VM is running | Host VMnet adapter on APIPA (`169.254.x.x`) after a reboot/VMware upgrade | Re-pin host IPs (§A.4); the segment goes dark if VMnet11's host adapter loses `.254`. See `feedback_vmnet_host_adapter_ip_reset.md`. |
| `vmrun` / VM Settings can't find `vmrun.exe` at the `(x86)` path | A VMware upgrade relocated it to `C:\Program Files\VMware\...` | Use the non-`(x86)` path (this guide's commands already do). See `feedback_vmrun_path_moved_nonx86.md`. |
| PowerShell baseline line like `"https://$ip:8200"` renders wrong / a `$Host` assignment silently fails | `$ip:port` parses as a scope-qualified variable; `$Host` etc. are read-only automatic variables | Use `${ip}:8200`; never assign `$Host`/`$Error`/`$args`. See `feedback_powershell_url_scope_qualifier.md` + `feedback_powershell_automatic_variables.md`. |

---

### Cross-references

- **Network canon:** `nexus-platform-plan/docs/infra/network.md`
- **VM inventory (hostnames/IPs/MACs/specs):** `nexus-platform-plan/docs/infra/vms.yaml`
- **Automated equivalents:** `nexus-infra-vmware/packer/{deb13,ws2025-core,ws2025-desktop}` + `packer/_shared/{ansible/roles,powershell/scripts}`
- **Next guide:** Guide 01 — Foundation · `nexus-gateway` (builds the lab edge router that lights up VMnet11 egress/DHCP/DNS). See [`INDEX.md`](../INDEX.md).
