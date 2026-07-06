# Guide 01 — Foundation · `nexus-gateway` (the lab edge router)

> **Mirrors:** `nexus-infra-vmware/packer/nexus-gateway/*` (Phase 0.B base
> router) + the gateway role overlays in
> `terraform/envs/foundation/role-overlay-gateway-{nfs-portainer,iscsi-sqlfci}.tf`
> (the NFSv4 + iSCSI add-ons from Phases 0.E.4a / 0.G.7). Where the automated lab
> Packer-builds the router on NAT then Terraform attaches its three real NICs +
> layers the storage exports, this guide installs it from the ISO and configures
> every service by hand.

---

## 1. Overview & purpose

`nexus-gateway` is **VM #0** — the first machine in the fleet, built before
anything else, because every other lab VM needs it for **internet egress, DNS,
DHCP, and time**. It is the lab's edge router.

It exists because of a hard VMware constraint: Workstation allows **exactly one
NAT network per host**, and that slot is already held by the pre-existing
VMnet8. So **VMnet11 cannot be a NAT network** — it is host-only at the VMware
layer, with no route off the workstation. `nexus-gateway` provides that route:
it has a **bridged** NIC onto the physical LAN (real internet) and **masquerades**
the VMnet11 subnet out through it. The same VM also serves the lab's DHCP (for
installs), DNS (authoritative for `*.nexus.local`, forwarding everything else),
and NTP.

Concretely, this guide stands up one Debian 13 VM with **three NICs** and these
services:

| Service | Package | Role |
|---|---|---|
| **NAT / firewall** | `nftables` | masquerade `192.168.70.0/24` → bridged NIC; default-deny inbound; isolate the backplane |
| **DHCP + DNS** | `dnsmasq` | DHCP `.200–.250` (installs only) + authoritative DNS for `nexus.local` + forwarder + round-robin service records |
| **NTP** | `chrony` | time **server** for the whole lab (clients sync to `192.168.70.1`) |
| **Metrics** | `prometheus-node-exporter` | host metrics on `:9100` for the future Prometheus tier |
| **Shared file store** | `nfs-kernel-server` | NFSv4-only export `/srv/nfs/portainer-data` for Portainer's `/data` (consumed by Guide 05's Swarm managers) |
| **Shared block store** | `tgt` (iSCSI target) | 60 GB sparse LUN `iqn.2026-05.local.nexus:sql-fci.lun1` for the SQL FCI shared disk (consumed by Guide 11) |

> **Why the gateway also hosts NFS + iSCSI** (rather than dedicated appliances):
> a lab consolidation — the gateway already plays infra-host (dnsmasq + nftables
> + chrony + node_exporter), so adding two storage exports avoids spinning up
> extra VMs/tiers. Production would split these onto dedicated NFS (NetApp /
> TrueNAS) and SAN appliances; the deviation is documented in ADR-0017 (NFS) and
> ADR-0026 (iSCSI). The two exports are **provisioned now** but their clients —
> the Swarm managers and the SQL FCI nodes — don't exist until Guides 05 and 11,
> so their firewall allow-lists are pre-seeded with those nodes' canonical IPs.

**Dependency:** **Guide 00 only** — the host's VMnet10 + VMnet11 must exist.
`nexus-gateway` is the *only* VM that does **not** install over NAT: it gets
internet immediately from its **bridged** NIC (DHCP from your home router), so
it is built with all three NICs attached from the start.

---

## 2. Component primer

(The OS-level components — Debian 13, `systemd-networkd`/`.link`, `nftables` as a
concept, `chrony`, `node_exporter`, OpenSSH, `unattended-upgrades` — are
explained in **Guide 00 §2**. This primer covers what's *new* in the router
role.)

- **NAT / masquerade (`nftables` `nat` table).** Network Address Translation
  rewrites the source address of packets leaving the lab so the physical LAN
  sees them as coming from the gateway, and reverses it on the way back.
  *Masquerade* is the dynamic form (uses whatever IP the bridged NIC currently
  has from home DHCP). *Why:* it's how the host-only VMnet11 gets to the
  internet without VMware NAT. *Otherwise:* VMware NAT (impossible — one slot,
  taken), or static `snat` (needs a fixed bridged IP we don't control on a home
  LAN).
- **IP forwarding (`net.ipv4.ip_forward=1`).** The kernel switch that lets a
  host route packets between interfaces. Off by default (a host isn't a router);
  we turn it on so VMnet11 traffic can cross to the bridged NIC. *Otherwise:*
  nothing forwards and masquerade never sees the packets.
- **`dnsmasq`.** A lightweight combined **DHCP server + DNS forwarder/resolver**.
  *Why here:* one small daemon gives the lab (a) DHCP leases in the install-only
  `.200–.250` range, (b) authoritative answers for `*.nexus.local`, (c) caching
  forwarding to public resolvers for everything else, and (d) round-robin
  multi-A "service" records (e.g. `portainer.nexus.lab` → all 3 Swarm managers)
  that give cluster front doors a single stable name. *Otherwise:* ISC
  `dhcpd` + `bind9` as two separate daemons (more moving parts than a lab edge
  needs), or `systemd-networkd`'s built-in DHCP server (no DNS, no records).
- **DHCP reservation (`dhcp-host`) vs. static IP.** The lab's production VMs are
  **static** — but Packer-built clones need DHCP during their build, so the
  automated lab pins each clone's MAC to its canonical IP with a `dhcp-host`
  reservation. By hand we assign statics directly (Guide 00 §B.4), so the DHCP
  pool is genuinely only used by the rare from-ISO install. The `dhcp-host`
  syntax is shown anyway (§5.4) because it's part of "DHCP reservations + DNS."
- **NFSv4 + `fsid=0` pseudo-root.** NFS shares a filesystem over the network.
  NFSv4 collapses the v3 multi-port mess (portmapper + mountd + lockd) into a
  single TCP/2049 listener — firewall-friendly. Setting `fsid=0` on an export
  makes it the NFSv4 **pseudo-root**, so clients mount via `server:/` (not
  `server:/srv/nfs/portainer-data`). *Why:* Portainer CE has no native HA — one
  Server replica runs at a time, and Swarm reschedules it across managers; the
  shared NFS `/data` means a reschedule doesn't lose its BoltDB state.
  *Otherwise:* a Docker volume plugin (more complex), or accepting state loss on
  reschedule (rejected).
- **iSCSI target (`tgt`) + CHAP.** iSCSI presents a **block device** (a raw
  disk) over TCP/3260; the gateway is the *target* (server), the SQL FCI nodes
  are *initiators* (clients). The backing store is a 60 GB sparse file. CHAP is
  the iSCSI password handshake; the target also restricts by initiator IP.
  *Why:* a Windows Failover Cluster Instance needs **shared block storage** both
  nodes can mount — iSCSI gives that without a physical SAN. *Otherwise:* a real
  SAN/FC array (no hardware), or a clustered file share (FCI requires block, not
  file).

---

## 3. Prerequisites

| # | Requirement | One-command verify (host, elevated PowerShell) |
|---|---|---|
| 1 | **Guide 00 complete** — VMnet10 + VMnet11 exist with the host at `.1`/`.254` | `Get-NetIPAddress -InterfaceAlias 'VMware Network Adapter VMnet10','VMware Network Adapter VMnet11' \| ? AddressFamily -eq IPv4 \| ft InterfaceAlias,IPAddress` → `10.1` + `70.254` |
| 2 | A **bridged**-capable physical NIC with internet (home router DHCP) | `Test-NetConnection 1.1.1.1 -Port 443` on the host → `True` |
| 3 | Debian 13 netinst ISO staged + checksum-verified | see Guide 00 §3 (`debian-13.4.0-amd64-netinst.iso`) |
| 4 | VM storage volume exists | `Test-Path 'H:\VMS\NexusPlatform'` → `True` |
| 5 | Owner SSH keypair available | `Test-Path '~/.ssh/nexus_gateway_ed25519.pub'` → `True` |

> No other VM is required — this **is** the first VM. The NFS/iSCSI clients it
> serves (Swarm managers `.111–.113`, SQL FCI `sql-fci-1/2` at `.11`/`.12`) are
> built later; their allow-lists are pre-seeded here.

---

## 4. Target topology

| Field | Value |
|---|---|
| Hostname | `nexus-gateway` (`nexus-gateway.nexus.local`) |
| OS | Debian 13 (Trixie), minimal |
| vCPU / RAM / disk | **1 / 512 MB / 4 GB** (an edge router is light; bump RAM to 1024 MB for the install per Guide 00, then shrink) |
| VMware folder | `H:\VMS\NexusPlatform\00-edge\nexus-gateway\` |
| **nic0** (bridged) | physical LAN, **DHCP from home router** — internet uplink | MAC `00:50:56:3F:00:10` |
| **nic1** (VMnet11) | `192.168.70.1/24`, **IPForward=yes** — lab gateway | MAC `00:50:56:3F:00:11` |
| **nic2** (VMnet10) | `192.168.10.1/24`, **IPForward=no** — backplane visibility only | MAC `00:50:56:3F:00:12` |

> **MAC note:** the gateway's three NICs all use the `…:00:` (primary) byte —
> `:10` bridged, `:11` VMnet11, `:12` VMnet10 — because all three are
> "front-side" interfaces on a single router (this is the one VM that breaks the
> `…:01:` = backplane convention, since its backplane NIC is just for visibility,
> not a clustered service). These are the canonical values from
> `terraform/gateway/example.tfvars`.

**Storage exports provisioned here (clients arrive in later guides):**

| Export | Path / IQN | Clients (allow-list) | Consumed by |
|---|---|---|---|
| NFSv4 | `/srv/nfs/portainer-data` (`fsid=0`) | `192.168.70.111`, `.112`, `.113` (Swarm managers) | Guide 05 (Portainer) |
| iSCSI | `iqn.2026-05.local.nexus:sql-fci.lun1` → `/srv/iscsi/sql-fci-shared.img` (60 GB sparse) | `192.168.70.11`, `.12` (sql-fci-1/2) | Guide 11 (SQL FCI) |

---

## 5. Step-by-step build

> **WHERE convention:** "Host" = the Windows 11 workstation. VM steps name the
> guest + user (`nexusadmin`, or `root` via `sudo -i`).

### 5.1 — Create the VM in the VMware GUI (three NICs)

> **Step 5.1.1 — New custom VM, point at the Debian ISO**
> **WHERE:** Host, VMware Workstation GUI.
> **WHY:** a custom VM lets us add the three NICs and set the adapter type
> deliberately. The gateway is built with all three NICs from the start (unlike
> leaf nodes) because its internet comes from the bridged NIC immediately.
> **WHAT:**
> 1. **File → New Virtual Machine… → Custom (advanced) → Next**; hardware
>    compatibility **Workstation 17.x** → Next.
> 2. **Installer disc image file (iso)** → `H:\VMS\ISO\debian-13.4.0-amd64-netinst.iso`
>    → Next.
> 3. Guest OS **Linux**, Version **Debian 12.x 64-bit** (catalog lags Debian 13)
>    → Next.
> 4. **Name** `nexus-gateway`; **Location**
>    `H:\VMS\NexusPlatform\00-edge\nexus-gateway\` → Next.
> **EXPECTED:** the per-VM folder is created.
> **VERIFY:** `Test-Path 'H:\VMS\NexusPlatform\00-edge\nexus-gateway'` → `True`.

> **Step 5.1.2 — CPU, RAM, disk; first NIC = bridged**
> **WHERE:** Host, New-VM wizard (continued).
> **WHY:** the router is light (1 vCPU, 512 MB at runtime), but give it **1024 MB
> for the install** (Debian's installer degrades below ~780 MB) and shrink later.
> The first NIC is **bridged** so the install reaches the internet directly.
> **WHAT:**
> 1. Processors **1 × 1 core** (= 1 vCPU) → Next.
> 2. Memory **1024 MB** (shrink to 512 MB after §5.9) → Next.
> 3. Network type → **Use bridged networking** → Next.
> 4. I/O controller **LSI Logic** → Next; Disk type **SCSI** → Next;
>    **Create a new virtual disk** → **4 GB**, ✅ **single file** → Next →
>    Finish (**don't** power on).
> **EXPECTED:** VM created, powered off, one bridged NIC.
> **VERIFY:** Hardware summary: 1 processor, 1 GB RAM, 4 GB disk, one
> **Network Adapter (Bridged)**.

> **Step 5.1.3 — Add the two host-only NICs + pin all three MACs**
> **WHERE:** Host, VM Settings → Hardware, then edit `nexus-gateway.vmx`.
> **WHY:** nic1 on VMnet11 (the lab gateway address) and nic2 on VMnet10
> (backplane visibility). Pinning MACs makes the in-guest `.link` rename
> deterministic (§5.3).
> **WHAT:**
> 1. **Edit virtual machine settings → Add… → Network Adapter → Custom: VMnet11**
>    → OK.
> 2. **Add… → Network Adapter → Custom: VMnet10** → OK.
> 3. Power off if needed, then edit `H:\VMS\NexusPlatform\00-edge\nexus-gateway\nexus-gateway.vmx`:
> ```ini
> ethernet0.present        = "TRUE"
> ethernet0.connectionType = "bridged"
> ethernet0.virtualDev     = "vmxnet3"
> ethernet0.addressType    = "static"
> ethernet0.address        = "00:50:56:3F:00:10"
>
> ethernet1.present        = "TRUE"
> ethernet1.connectionType = "custom"
> ethernet1.vnet           = "VMnet11"
> ethernet1.virtualDev     = "vmxnet3"
> ethernet1.addressType    = "static"
> ethernet1.address        = "00:50:56:3F:00:11"
>
> ethernet2.present        = "TRUE"
> ethernet2.connectionType = "custom"
> ethernet2.vnet           = "VMnet10"
> ethernet2.virtualDev     = "vmxnet3"
> ethernet2.addressType    = "static"
> ethernet2.address        = "00:50:56:3F:00:12"
> ```
> **EXPECTED:** three adapters — Bridged, VMnet11, VMnet10.
> **VERIFY:**
> `Select-String 'ethernet[012].vnet|ethernet0.connectionType' 'H:\VMS\NexusPlatform\00-edge\nexus-gateway\nexus-gateway.vmx'`
> shows `bridged`, `VMnet11`, `VMnet10`.

### 5.2 — Debian 13 netinst walkthrough

Power on; at GRUB pick **Install**. This is the same flow as Guide 00 §5.B.2 —
follow that for screen-by-screen detail. The gateway-specific answers:

> **Step 5.2.1 — Install Debian with the gateway hostname**
> **WHERE:** `nexus-gateway` console.
> **WHY:** identical localisation/account/partition flow to Guide 00, with this
> VM's hostname.
> **WHAT (deltas from Guide 00 §5.B.2):**
> - Networking auto-configures over the **bridged** NIC (home-router DHCP).
>   **Hostname** `nexus-gateway`; **Domain** `nexus.local`.
> - Mirror `deb.debian.org` `/debian`, no proxy.
> - Root password **blank** (locked); user **NexusPlatform Admin** /
>   `nexusadmin` / `nexus-packer-build-only`.
> - Time zone **UTC**.
> - Partitioning **Guided - use entire disk** → **All files in one partition**
>   → write.
> - Software selection: **SSH server** + **standard system utilities** only
>   (no desktop).
> - GRUB to **/dev/sda**.
> **EXPECTED:** install completes and reboots to a `nexus-gateway login:` prompt.
> **VERIFY:** log in as `nexusadmin` at the console; `hostname` → `nexus-gateway`.
>
> > If the VM re-enters the installer on reboot, power off and uncheck
> > *Connect at power on* for the CD/DVD in VM Settings, then power on.

### 5.3 — Identity baseline + three-NIC networking

> **Step 5.3.1 — Apply the shared identity baseline (key, sudo, sshd hardening)**
> **WHERE:** `nexus-gateway`, console as `nexusadmin` → `sudo -i`.
> **WHY:** the gateway is still a NexusPlatform node — it gets the same identity
> baseline as every Debian node: passwordless sudo, the owner SSH key, sshd
> hardening, and the host-key-regen drop-in.
> **WHAT:** run **Guide 00 steps B.3.1, B.3.3, and B.3.4 verbatim** on this VM
> (passwordless sudo for `nexusadmin`; deploy `nexus_gateway_ed25519.pub` to
> `authorized_keys`; sshd hardening + host-key drop-in).
> **EXPECTED:** key login works; sshd hardened.
> **VERIFY:** from the host,
> `ssh -i ~/.ssh/nexus_gateway_ed25519 nexusadmin@<bridged-DHCP-IP> 'sshd -t && echo OK'`
> → `OK`. (Find the bridged IP at the console with `ip -4 addr show`.)

> **Step 5.3.2 — Install the gateway package set**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** the router needs `dnsmasq` (DHCP+DNS), `systemd-resolved` (replaced as
> the resolver but the package is pulled for libs), metrics, security-updates
> tooling, and diagnostics. (`nftables`/`chrony` came in from the install.)
> **WHAT:**
> ```bash
> apt-get update -qq
> apt-get install -y \
>   nftables dnsmasq chrony systemd-resolved \
>   prometheus-node-exporter unattended-upgrades apt-listchanges \
>   curl ca-certificates gnupg tcpdump iptraf-ng mtr-tiny \
>   python3 python3-apt
> ```
> **EXPECTED:** all packages install.
> **VERIFY:** `dpkg -s dnsmasq nftables chrony prometheus-node-exporter | grep -c '^Status: install ok'`
> → `4`.

> **Step 5.3.3 — MAC-keyed `.link` renames for all three NICs**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** deterministically name the three NICs `nic0`/`nic1`/`nic2` by MAC, so
> the firewall/dnsmasq/networkd configs that reference them by name are stable
> across reboots (the same MAC-keyed pattern Guide 00 §B.4.2 uses, extended to
> three interfaces).
> **WHAT:**
> ```bash
> cat > /etc/systemd/network/10-nic0.link <<'EOF'
> [Match]
> MACAddress=00:50:56:3f:00:10
>
> [Link]
> Name=nic0
> EOF
> cat > /etc/systemd/network/11-nic1.link <<'EOF'
> [Match]
> MACAddress=00:50:56:3f:00:11
>
> [Link]
> Name=nic1
> EOF
> cat > /etc/systemd/network/12-nic2.link <<'EOF'
> [Match]
> MACAddress=00:50:56:3f:00:12
>
> [Link]
> Name=nic2
> EOF
> ```
> **EXPECTED:** three `.link` files written.
> **VERIFY:** `ls /etc/systemd/network/1?-nic?.link` lists all three.

> **Step 5.3.4 — Static `.network` files (nic0 DHCP, nic1/nic2 static + forwarding)**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** nic0 (bridged) takes DHCP from the home router (internet uplink);
> nic1 is the VMnet11 gateway address `192.168.70.1` with **IP forwarding on**;
> nic2 is the VMnet10 backplane address `192.168.10.1` with forwarding **off**
> (the backplane never routes through this host).
> **WHAT:**
> ```bash
> cat > /etc/systemd/network/20-nic0.network <<'EOF'
> [Match]
> Name=nic0
>
> [Network]
> DHCP=yes
> EOF
>
> cat > /etc/systemd/network/21-nic1.network <<'EOF'
> [Match]
> Name=nic1
>
> [Network]
> Address=192.168.70.1/24
> IPForward=yes
> EOF
>
> cat > /etc/systemd/network/22-nic2.network <<'EOF'
> [Match]
> Name=nic2
>
> [Network]
> Address=192.168.10.1/24
> IPForward=no
> EOF
>
> systemctl enable systemd-networkd
> ```
> **EXPECTED:** three `.network` files written; networkd enabled.
> **VERIFY (after the reboot in §5.3.6):** `ip -br addr show` shows
> `nic1 … 192.168.70.1/24` and `nic2 … 192.168.10.1/24`; `nic0` has a
> home-LAN IP.

> **Step 5.3.5 — Enable IP forwarding + small-NAT-box sysctls**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** `ip_forward=1` makes the box route between NICs (required for NAT).
> `nf_conntrack` must be loaded before its `_max` tunable can be set; the
> backlog/conntrack bumps suit a small router under lab churn.
> **WHAT:**
> ```bash
> # Load conntrack at boot + now (so the _max sysctl below has its knob)
> echo nf_conntrack > /etc/modules-load.d/nexus-gateway.conf
> modprobe nf_conntrack
>
> cat > /etc/sysctl.d/10-nexus-gateway.conf <<'EOF'
> net.ipv4.ip_forward = 1
> net.core.somaxconn = 4096
> net.ipv4.tcp_max_syn_backlog = 4096
> net.netfilter.nf_conntrack_max = 131072
> EOF
> sysctl --system
> ```
> **EXPECTED:** `sysctl --system` echoes the applied keys.
> **VERIFY:** `sysctl net.ipv4.ip_forward` → `net.ipv4.ip_forward = 1`.

> **Step 5.3.6 — Reboot to bring up the renamed NICs**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** apply the `.link` renames + static `.network` config cleanly.
> **WHAT:** `reboot`
> **EXPECTED:** the VM comes back with `nic0`/`nic1`/`nic2`.
> **VERIFY:** from the host,
> `ssh -i ~/.ssh/nexus_gateway_ed25519 nexusadmin@192.168.70.1 'ip -br addr show'`
> shows the three NICs with their IPs. **From now on, SSH to the gateway at its
> stable VMnet11 IP `192.168.70.1`.**

### 5.4 — `nftables` (NAT + filtering)

> **Step 5.4.1 — Install the gateway nftables ruleset**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** this ruleset (a) default-denies inbound but allows SSH/DNS/DHCP/NTP/
> node_exporter from VMnet11, (b) **masquerades** VMnet11 out the bridged nic0,
> and (c) **drops** any VMnet10 egress (the backplane is isolated). It is the
> heart of the router.
> **WHAT:**
> ```bash
> cat > /etc/nftables.conf <<'EOF'
> #!/usr/sbin/nft -f
> #
> # nexus-gateway — nftables ruleset
> # Masquerade VMnet11 (192.168.70.0/24) through nic0 (bridged to physical LAN).
> # Drop VMnet10 (192.168.10.0/24) egress — backplane is isolated.
> #   nic0 — bridged (internet)   nic1 — vmnet11 (.70.1)   nic2 — vmnet10 (.10.1)
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
>         ip protocol icmp  accept
>         ip6 nexthdr icmpv6 accept
>
>         # Management + lab services from VMnet11 only
>         iifname "nic1" tcp dport 22  accept comment "SSH from VMnet11"
>         iifname "nic1" udp dport 53  accept comment "DNS from lab"
>         iifname "nic1" tcp dport 53  accept
>         iifname "nic1" udp dport 67  accept comment "DHCP from lab"
>         iifname "nic1" udp dport 123 accept comment "NTP from lab"
>         iifname "nic1" tcp dport 9100 accept comment "node_exporter"
>
>         # Backplane visibility: ICMP only
>         iifname "nic2" ip protocol icmp accept
>
>         counter drop
>     }
>
>     chain forward {
>         type filter hook forward priority 0; policy drop;
>
>         ct state { established, related } accept
>
>         # Lab (VMnet11) → internet (bridged nic0)
>         iifname "nic1" oifname "nic0" accept
>
>         # Backplane never egresses
>         iifname "nic2" drop
>     }
>
>     chain output {
>         type filter hook output priority 0; policy accept;
>     }
> }
>
> table ip nat {
>     chain prerouting {
>         type nat hook prerouting priority -100;
>     }
>     chain postrouting {
>         type nat hook postrouting priority 100; policy accept;
>
>         oifname "nic0" ip saddr 192.168.70.0/24 masquerade
>     }
> }
> EOF
> chmod 0755 /etc/nftables.conf
> nft -c -f /etc/nftables.conf       # syntax-check
> systemctl enable --now nftables
> systemctl restart nftables
> ```
> **EXPECTED:** `nft -c` is silent (valid); nftables starts.
> **VERIFY:** `nft list table ip nat | grep masquerade` shows the masquerade
> rule; `nft list chain inet filter input | grep 'dport 53'` shows the DNS rule.

### 5.5 — `dnsmasq` (DHCP + DNS)

> **Step 5.5.1 — Free port 53 from systemd-resolved**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** Debian's `systemd-resolved` binds a stub listener on `:53`, which
> would collide with dnsmasq. Stop+disable it and point the host's own resolver
> at dnsmasq (loopback).
> **WHAT:**
> ```bash
> systemctl disable --now systemd-resolved
> rm -f /etc/resolv.conf
> cat > /etc/resolv.conf <<'EOF'
> nameserver 127.0.0.1
> search nexus.local
> EOF
> ```
> **EXPECTED:** resolved stopped; `/etc/resolv.conf` points at `127.0.0.1`.
> **VERIFY:** `ss -tulnp | grep ':53 '` shows **no** systemd-resolved listener
> (dnsmasq will claim it next).

> **Step 5.5.2 — Install the dnsmasq config + seed the static-hosts file**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** the canonical dnsmasq config — bind DHCP/DNS to nic1 (VMnet11) only,
> never the bridged NIC; serve the install-only DHCP scope; be authoritative for
> `nexus.local`; forward + DNSSEC-validate everything else; cache aggressively.
> **WHAT:**
> ```bash
> cat > /etc/dnsmasq.conf <<'EOF'
> # nexus-gateway — dnsmasq
> # DHCP on VMnet11 (.200-.250 — installs only; production VMs are static).
> # Authoritative DNS for *.nexus.local; forwards everything else.
>
> # ── Networking ──
> interface=nic1
> bind-interfaces
> except-interface=nic0
> no-dhcp-interface=nic0
> no-dhcp-interface=nic2
> interface=nic2
>
> # ── DHCP ──
> dhcp-range=192.168.70.200,192.168.70.250,255.255.255.0,30m
> dhcp-option=option:router,192.168.70.1
> dhcp-option=option:dns-server,192.168.70.1
> dhcp-option=option:ntp-server,192.168.70.1
> dhcp-option=option:domain-name,nexus.local
> dhcp-authoritative
> dhcp-leasefile=/var/lib/misc/dnsmasq.leases
>
> # ── DNS ──
> domain=nexus.local
> local=/nexus.local/
> expand-hosts
> domain-needed
> bogus-priv
>
> no-resolv
> server=1.1.1.1
> server=1.0.0.1
> server=9.9.9.9
>
> dnssec
> trust-anchor=.,20326,8,2,E06D44B80B8F1D39A95C0B0D7C65D08458E880409BBC683457104237C7F8EC8D
> dnssec-check-unsigned
>
> cache-size=2048
> min-cache-ttl=60
>
> log-queries=extra
> log-dhcp
>
> # ── Static hosts (superset; populated as VMs come online) ──
> addn-hosts=/etc/dnsmasq.d/hosts.nexus
>
> # ── Security ──
> stop-dns-rebind
> rebind-localhost-ok
> EOF
>
> install -d -m 0755 /etc/dnsmasq.d
> cat > /etc/dnsmasq.d/hosts.nexus <<'EOF'
> # Static host entries — add lab VMs here as they come online.
> # 192.168.70.10  dc-nexus.nexus.local   dc-nexus
> EOF
> ```
> **EXPECTED:** config + addn-hosts written.
> **VERIFY:** `dnsmasq --test` → `dnsmasq: syntax check OK.`

> **Step 5.5.3 — Start dnsmasq**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** with nic1 now up (§5.3.6) and `:53` freed, dnsmasq can bind.
> **WHAT:**
> ```bash
> systemctl enable --now dnsmasq
> systemctl restart dnsmasq
> ```
> **EXPECTED:** dnsmasq active.
> **VERIFY:**
> ```bash
> systemctl is-active dnsmasq          # active
> dig @127.0.0.1 debian.org +short     # resolves (forwarding works)
> ```

> **Step 5.5.4 — (Pattern) add a DHCP reservation + a round-robin service record**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** "DHCP reservations + DNS" — show the two mechanisms later guides use.
> A `dhcp-host` pins a MAC to an IP (used by the automated lab during clone
> builds); a multi-A `host-record` gives a cluster front door one stable name
> resolving round-robin to all members (e.g. `portainer.nexus.lab` → the 3 Swarm
> managers, added in Guide 05). These are **examples** — only add the real
> entries when their guide builds those nodes.
> **WHAT (illustrative — do not add until the nodes exist):**
> ```bash
> # Example DHCP reservation (pins a clone's MAC to its canonical IP):
> #   echo 'dhcp-host=00:50:56:3F:00:40,192.168.70.121,vault-1' >> /etc/dnsmasq.d/reservations.conf
> #
> # Example round-robin service record (Guide 05 — Portainer front door):
> #   echo 'host-record=portainer.nexus.lab,192.168.70.111,192.168.70.112,192.168.70.113' >> /etc/dnsmasq.d/records.conf
> #
> # After editing any /etc/dnsmasq.d/*.conf:
> #   dnsmasq --test && systemctl reload dnsmasq
> echo 'dnsmasq reservation/record pattern documented — no live entries added in Guide 01'
> ```
> **EXPECTED:** the echo prints (nothing live changes).
> **VERIFY:** n/a — this is the documented pattern; later guides add the real
> entries and verify with `dig @192.168.70.1 <name> +short`.

### 5.6 — `chrony` (NTP server for the lab)

> **Step 5.6.1 — Install the chrony server config**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** unlike leaf nodes (chrony *clients*), the gateway is the lab's NTP
> **server** — it sources time from public pools and **serves** it to VMnet11 +
> VMnet10, even when itself unsynchronised (cold-boot tolerance via
> `local stratum 10`), but never to the internet.
> **WHAT:**
> ```bash
> cat > /etc/chrony/chrony.conf <<'EOF'
> # nexus-gateway — chrony
> # Source time from public pools; serve the lab on VMnet11/VMnet10 only.
>
> pool 2.debian.pool.ntp.org iburst
> pool time.cloudflare.com   iburst
>
> driftfile /var/lib/chrony/chrony.drift
> ntsdumpdir /var/lib/chrony
>
> # Allow lab VMs to query time
> allow 192.168.70.0/24
> allow 192.168.10.0/24
>
> # Never serve the internet
> deny 0.0.0.0/0
>
> # Serve time even when unsynchronised (lab cold-boot tolerance)
> local stratum 10
>
> makestep 1.0 3
> rtcsync
>
> logdir /var/log/chrony
> log tracking measurements statistics
> hwtimestamp *
> EOF
> systemctl enable --now chrony
> systemctl restart chrony
> ```
> **EXPECTED:** chrony starts and begins syncing from the public pools.
> **VERIFY:** `chronyc tracking | grep 'Leap status'` → `Normal` (after ~30 s);
> `chronyc sources` lists the two pools.

### 5.7 — `node_exporter` + unattended upgrades + MOTD

> **Step 5.7.1 — Enable node_exporter; configure unattended security upgrades; MOTD**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** host metrics for the Prometheus tier; auto security patching (no
> reboot); the gateway banner. Same `unattended-upgrades` config as every Debian
> node (Guide 00 §B.3.8).
> **WHAT:**
> ```bash
> systemctl enable --now prometheus-node-exporter
>
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
> ║  nexus-gateway — NexusPlatform lab edge router (VM #0)        ║
> ║  VMnet11 gateway: 192.168.70.1   VMnet10: 192.168.10.1        ║
> ║  Services: nftables · dnsmasq · chrony · node_exporter        ║
> ║  Built BY HAND per nexus-infra-manual/guides/01               ║
> ╚═══════════════════════════════════════════════════════════════╝
> EOF
> ```
> **EXPECTED:** node_exporter active; files written.
> **VERIFY:** `curl -s localhost:9100/metrics | head -1` → a `# HELP` line.

### 5.8 — NFSv4 export (Portainer `/data`, for Guide 05)

> **Step 5.8.1 — Install NFS server, NFSv4-only, export `/srv/nfs/portainer-data`**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** Guide 05's Portainer needs shared `/data` so its single Server replica
> survives a Swarm reschedule across managers. NFSv4-only (`-N 2 -N 3`) keeps it
> to one TCP/2049 listener; `fsid=0` makes the export the pseudo-root so clients
> mount via `192.168.70.1:/`.
> **WHAT:**
> ```bash
> apt-get install -y nfs-kernel-server
>
> # Force NFSv4-only (disable v2/v3)
> grep -q 'RPCNFSDOPTS=.*-N 2 -N 3' /etc/default/nfs-kernel-server 2>/dev/null || \
>   echo 'RPCNFSDOPTS="-N 2 -N 3"' >> /etc/default/nfs-kernel-server
>
> install -d -m 0755 -o root -g root /srv/nfs/portainer-data
> install -d -m 0755 /etc/exports.d   # not created by the package on Debian 13
>
> cat > /etc/exports.d/portainer.exports <<'EOF'
> # NFSv4-only export of /srv/nfs/portainer-data for the swarm managers (Portainer CE /data).
> # fsid=0 makes this path the NFSv4 pseudo-root.
> /srv/nfs/portainer-data  192.168.70.111(rw,sync,no_root_squash,no_subtree_check,fsid=0)
> /srv/nfs/portainer-data  192.168.70.112(rw,sync,no_root_squash,no_subtree_check,fsid=0)
> /srv/nfs/portainer-data  192.168.70.113(rw,sync,no_root_squash,no_subtree_check,fsid=0)
> EOF
> chmod 0644 /etc/exports.d/portainer.exports
>
> systemctl enable --now nfs-kernel-server
> exportfs -ra
> ```
> **EXPECTED:** NFS server active; export listed.
> **VERIFY:** `exportfs -v | grep portainer-data` shows the three manager
> clients; `ss -tlnp | grep ':2049 '` shows a listener.

> **Step 5.8.2 — Open TCP/2049 from the Swarm managers (nftables patch)**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** the baseline ruleset doesn't open 2049. Add it from the manager IPs on
> nic1 — and **patch the file + `nft -f`**, not a runtime `nft add rule` (which
> lands *after* `counter drop`, unreachable — see
> `feedback_nftables_runtime_add_after_drop.md`).
> **WHAT:** edit `/etc/nftables.conf`, inserting **before** the `counter drop`
> line in `chain input`:
> ```
>         # portainer NFSv4 access
>         iifname "nic1" ip saddr 192.168.70.111 tcp dport 2049 accept comment "NFSv4 from manager-1"
>         iifname "nic1" ip saddr 192.168.70.112 tcp dport 2049 accept comment "NFSv4 from manager-2"
>         iifname "nic1" ip saddr 192.168.70.113 tcp dport 2049 accept comment "NFSv4 from manager-3"
> ```
> then reload atomically:
> ```bash
> nft -c -f /etc/nftables.conf && nft -f /etc/nftables.conf
> ```
> **EXPECTED:** ruleset reloads with no error.
> **VERIFY:** `nft list chain inet filter input | grep 2049` shows the three
> rules, **above** the `counter drop`.

### 5.9 — iSCSI target (SQL FCI shared LUN, for Guide 11)

> **Step 5.9.1 — Install `tgt`, create the 60 GB sparse LUN, write the target**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** Guide 11's SQL Server FCI needs shared block storage both cluster
> nodes can mount. `tgt` exports a sparse 60 GB file as iSCSI LUN
> `iqn.2026-05.local.nexus:sql-fci.lun1`, restricted by CHAP **and** initiator
> IP (`sql-fci-1`/`sql-fci-2` at `.11`/`.12`). The data-segment tunables match
> the Windows initiator defaults for SQL workloads.
> **WHAT:** (pick a CHAP secret; store it where Guide 11 will read it)
> ```bash
> apt-get install -y tgt
> systemctl enable --now tgt.service
>
> # 60 GB sparse backing file (thin until SQL writes)
> install -d -m 0750 -o root -g root /srv/iscsi
> [ -f /srv/iscsi/sql-fci-shared.img ] || { truncate -s 60G /srv/iscsi/sql-fci-shared.img; chmod 0640 /srv/iscsi/sql-fci-shared.img; }
>
> CHAP_USER=sql-fci-initiator
> # IDEMPOTENT: reuse the existing CHAP secret if the target is already
> # configured. Re-running this step must NOT rotate the secret — Guide 11's
> # SQL FCI initiator stores it and would fail CHAP auth after a silent
> # rotation. Only generate a fresh secret on the FIRST run. (The automated lab
> # sources this from Vault KV `nexus/oltp/sqlserver/iscsi-chap-secret` via a
> # sticky sidecar; the by-hand equivalent is reuse-from-existing-config.)
> if [ -f /etc/tgt/conf.d/sql-fci.conf ]; then
>   CHAP_SECRET=$(awk '/incominguser/ {print $3}' /etc/tgt/conf.d/sql-fci.conf)
> fi
> CHAP_SECRET=${CHAP_SECRET:-$(openssl rand -hex 16)}   # record on FIRST run — Guide 11 needs it (≥12 chars, RFC 3720)
> echo "iSCSI CHAP: user=$CHAP_USER secret=$CHAP_SECRET"
>
> cat > /etc/tgt/conf.d/sql-fci.conf <<EOF
> default-driver iscsi
>
> <target iqn.2026-05.local.nexus:sql-fci.lun1>
>     backing-store /srv/iscsi/sql-fci-shared.img
>     incominguser $CHAP_USER $CHAP_SECRET
>     initiator-address 192.168.70.11
>     initiator-address 192.168.70.12
>     MaxRecvDataSegmentLength 262144
>     FirstBurstLength 262144
>     MaxBurstLength 1048576
> </target>
> EOF
> chmod 0640 /etc/tgt/conf.d/sql-fci.conf
>
> # A freshly-written target only attaches its backing-store on a service
> # RESTART (tgt-admin --update does NOT attach it). Restart is ~1s.
> systemctl restart tgt.service
> sleep 2
> ```
> **EXPECTED:** tgt restarts and loads the target.
> **VERIFY:**
> `tgtadm --mode target --op show | grep -q '/srv/iscsi/sql-fci-shared.img' && echo OK`
> → `OK`.

> **Step 5.9.2 — Open TCP/3260 from the FCI initiators (nftables patch)**
> **WHERE:** `nexus-gateway`, root shell.
> **WHY:** allow iSCSI only from `sql-fci-1`/`-2`. Same file-patch + `nft -f`
> discipline as §5.8.2.
> **WHAT:** edit `/etc/nftables.conf`, inserting **before** `counter drop` in
> `chain input`:
> ```
>         # === iSCSI for SQL FCI ===
>         iifname "nic1" ip saddr 192.168.70.11 tcp dport 3260 accept comment "iSCSI from sql-fci-1"
>         iifname "nic1" ip saddr 192.168.70.12 tcp dport 3260 accept comment "iSCSI from sql-fci-2"
>         # === end iSCSI ===
> ```
> then:
> ```bash
> nft -c -f /etc/nftables.conf && nft -f /etc/nftables.conf
> ```
> **EXPECTED:** reload succeeds.
> **VERIFY:** `nft list chain inet filter input | grep 3260` shows both rules.

> **Step 5.9.3 — (Optional) shrink RAM to the runtime profile**
> **WHERE:** Host, VM Settings (after a clean `sudo poweroff`).
> **WHY:** the install needed 1024 MB; the running router is comfortable at
> **512 MB** (the canonical spec). Power off, set Memory to 512 MB, power on.
> **WHAT:** VM Settings → Memory → **512 MB** → OK → power on.
> **EXPECTED:** the gateway boots and all services come up at 512 MB.
> **VERIFY:** re-run §6 checks after the resize — all green.

---

## 6. Validation — by-hand acceptance smoke

Run from the **host** (elevated PowerShell), gateway powered on. Mirrors the
spirit of the automated gateway smoke.

| # | Check | Command (host) | Pass criteria |
|---|---|---|---|
| 1 | Gateway reachable on VMnet11 | `Test-NetConnection 192.168.70.1 -Port 22` | `TcpTestSucceeded : True` |
| 2 | Key-only SSH works | `ssh -i ~/.ssh/nexus_gateway_ed25519 nexusadmin@192.168.70.1 'hostname'` | `nexus-gateway` |
| 3 | Three NICs up with correct IPs | `ssh … 'ip -br addr show \| grep -E "nic[012]"'` | `nic0` home-LAN IP, `nic1 192.168.70.1`, `nic2 192.168.10.1` |
| 4 | IP forwarding on | `ssh … 'sysctl -n net.ipv4.ip_forward'` | `1` |
| 5 | Core services active | `ssh … 'systemctl is-active nftables dnsmasq chrony prometheus-node-exporter nfs-kernel-server tgt'` | six `active` |
| 6 | DNS forwarding works | `ssh … 'dig @127.0.0.1 debian.org +short'` | an A record |
| 7 | DNS authoritative for nexus.local | `ssh … 'dig @127.0.0.1 nexus-gateway.nexus.local +short'` | (resolves once the host is in `hosts.nexus`; empty is OK at this stage) |
| 8 | NAT masquerade present | `ssh … 'sudo nft list table ip nat \| grep masquerade'` | the `oifname "nic0" … masquerade` rule |
| 9 | Backplane isolated (no egress) | `ssh … 'sudo nft list chain inet filter forward \| grep "nic2"'` | `iifname "nic2" drop` |
| 10 | NTP serving the lab | `ssh … 'chronyc tracking \| grep "Leap status"'` | `Normal` |
| 11 | node_exporter responding | `ssh … 'curl -s localhost:9100/metrics \| head -1'` | a `# HELP` line |
| 12 | NFSv4 export live | `ssh … 'sudo exportfs -v \| grep portainer-data'` | the three manager clients |
| 13 | iSCSI LUN attached | `ssh … 'sudo tgtadm --mode target --op show \| grep sql-fci-shared.img'` | the backing-store path |
| 14 | **End-to-end egress** — a future VMnet11 host can reach the internet *through* the gateway | (after Guide 03+ a leaf node: `ssh leaf 'curl -sI https://debian.org \| head -1'`) | `HTTP/2 200` (proves masquerade) |

**Checks 1–13 green ⇒ Guide 01 is satisfied.** Check 14 is verified the moment
the first leaf node (Guide 03) is on VMnet11 — it's the real proof that NAT
egress works.

> **Host-side DNS:** now that the gateway is up, finish **Guide 00 §A.5** —
> `Set-DnsClientServerAddress -InterfaceAlias 'VMware Network Adapter VMnet11' -ServerAddresses '192.168.70.1','1.1.1.1'` — so the workstation resolves
> `*.nexus.local` names.

---

## 7. Teardown / reset

`nexus-gateway` is the **root** of the running lab — tearing it down takes the
whole lab offline (no DNS/DHCP/egress). Only do this to rebuild it.

```powershell
# Graceful shutdown, then delete the VM + folder.
& 'C:\Program Files\VMware\VMware Workstation\vmrun.exe' stop 'H:\VMS\NexusPlatform\00-edge\nexus-gateway\nexus-gateway.vmx' soft
& 'C:\Program Files\VMware\VMware Workstation\vmrun.exe' deleteVM 'H:\VMS\NexusPlatform\00-edge\nexus-gateway\nexus-gateway.vmx'
Remove-Item 'H:\VMS\NexusPlatform\00-edge\nexus-gateway' -Recurse -Force
```

To remove **only** an add-on export without rebuilding the VM:

```bash
# NFS:  sudo systemctl disable --now nfs-kernel-server; sudo rm /etc/exports.d/portainer.exports
# iSCSI: sudo rm /etc/tgt/conf.d/sql-fci.conf; sudo systemctl restart tgt.service
#   (the LUN backing file /srv/iscsi/sql-fci-shared.img holds data — rm it manually only if you mean to)
# then remove the matching nftables accept rules from /etc/nftables.conf and: sudo nft -f /etc/nftables.conf
```

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Lab VMs can't reach the internet though the gateway is up | `ip_forward` off, or the masquerade rule missing/below a drop | `sysctl net.ipv4.ip_forward` must be `1` (§5.3.5); confirm the `ip nat postrouting … masquerade` rule (§5.4.1). |
| `dnsmasq` won't start — "address already in use" on `:53` | `systemd-resolved` still owns the port | Stop+disable it and point `/etc/resolv.conf` at `127.0.0.1` (§5.5.1). |
| After reboot, `nic1`/`nic2` are missing or swapped | A `.link` matched the wrong NIC (e.g. an `OriginalName=en*` glob on a multi-NIC box) | Use **MAC-keyed** `.link` files, one per NIC (§5.3.3) — never `en*` globs. See `feedback_systemd_link_precedence_multi_nic.md`. |
| Added an nftables allow rule at runtime but the port is still blocked | `nft add rule` appends **after** the chain's `counter drop` (unreachable) | Edit `/etc/nftables.conf` and `nft -f` the whole file (atomic). §5.8.2 / §5.9.2. See `feedback_nftables_runtime_add_after_drop.md`. |
| `mount.nfs4: No such file or directory` on a client (Guide 05) | The server's `fsid=0` makes the export the NFSv4 pseudo-root | Client mounts via `192.168.70.1:/` (not `:/srv/nfs/portainer-data`). |
| iSCSI initiator (Guide 11) sees the IQN but no disk | `tgt-admin --update` doesn't attach a freshly-written backing-store | `systemctl restart tgt.service` after writing `/etc/tgt/conf.d/*.conf` (§5.9.1). |
| Gateway segment goes dark after a host reboot / VMware upgrade | The host's VMnet11 adapter dropped its static `.254` (APIPA) | Re-pin host IPs — Guide 00 §A.4. See `feedback_vmnet_host_adapter_ip_reset.md`. |
| The Debian installer drops to low-memory mode | < ~780 MB RAM at install | Install at **1024 MB** (§5.1.2), shrink to 512 MB afterward (§5.9.3). |
| Every `sudo` warns "unable to resolve host nexus-gateway" | No `127.0.1.1` self-entry | Add it: `echo "127.0.1.1 nexus-gateway.nexus.local nexus-gateway" >> /etc/hosts` (same pattern as Guide 00 §B.4.3). |

---

### Cross-references

- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (`nexus-gateway` section)
- **VM inventory:** `nexus-platform-plan/docs/infra/vms.yaml` (`edge` cluster)
- **ADRs:** ADR-0017 (Portainer NFS via gateway), ADR-0026 (SQL FCI iSCSI shared storage)
- **Automated equivalents:** `nexus-infra-vmware/packer/nexus-gateway/` + `terraform/envs/foundation/role-overlay-gateway-{nfs-portainer,iscsi-sqlfci}.tf`
- **Previous guide:** [`00-lab-host-and-base-vm.md`](./00-lab-host-and-base-vm.md) — host networks + the base-VM/identity baseline this guide builds on.
- **Next guide:** Guide 02 — Foundation · AD DS forest (`dc-nexus`). See [`INDEX.md`](../INDEX.md).
