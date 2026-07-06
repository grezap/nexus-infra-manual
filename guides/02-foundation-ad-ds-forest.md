# Guide 02 — Foundation · AD DS forest (`dc-nexus` + `dc-nexus-2`)

> **Mirrors:** the `dc-nexus` role overlays in
> `nexus-infra-vmware/terraform/envs/foundation/role-overlay-dc-*.tf`
> (promotion, OUs, password policy, LDAP signing, reverse DNS, PDC time, KDS/GMSA)
> + `role-overlay-dc-nexus-2-promotion.tf` (the Phase 0.M replica). Where the
> automated lab drives every step over SSH with base64-encoded PowerShell, this
> guide runs the same cmdlets **by hand in an elevated PowerShell over RDP** on
> each DC — simpler and more reliable for interactive AD work, and the only way
> some steps (KDS root key) work at all on Server 2025.

---

## 1. Overview & purpose

This guide turns two baselined Windows Server 2025 VMs (built by Guide 00 §5.C)
into the lab's **Active Directory forest** — the identity backbone every Windows
node and every directory-aware service depends on.

- **`dc-nexus`** is promoted into the **forest root** domain controller for
  `nexus.lab` (NetBIOS `NEXUS`), running AD DS + AD-integrated DNS. On top of the
  bare promotion it gets the canonical OU layout, the domain password/lockout
  policy, a reverse-DNS zone, PDC-emulator time configuration, the relaxed LDAP
  signing setting Vault needs, and the KDS root key + GMSA scaffolding.
- **`dc-nexus-2`** is then domain-joined and promoted into a **replica** domain
  controller (Phase 0.M). Two DCs give the foundation **HA**: multi-master
  replication of the directory + a replicated `nexus.lab` DNS zone means a single
  DC failure does not take down authentication or name resolution.

**Why a real AD forest in a lab:** the relational-HA tier (SQL Server FCI +
Always-On AG, Guide 11) is genuinely Windows and *requires* a domain — WSFC,
Kerberos, and GMSA service identities all need AD. Vault's LDAP auth (Guide 04)
authenticates against it. The DCs also provide DNS for the `nexus.lab` zone that
the gateway forwards to.

**What this guide is *not*:** it does not issue certificates, wire Vault LDAP, or
create GMSA *consumers* — those are Guides 04 and 11. Here we stand up the forest
and the scaffolding (KDS root key, a sample GMSA) so those later guides just plug
in.

**Dependency:**
- **Guide 00** — two `ws2025-desktop` nodes baselined: `dc-nexus` and
  `dc-nexus-2` (RSAT/GPMC/DNS tools staged via Guide 00 §C.3.7).
- **Guide 01** — `nexus-gateway` up, so the VMs have DNS/NTP and so the gateway
  can forward `nexus.lab` queries to the DC (configured in §5.1).

---

## 2. Component primer

- **Active Directory Domain Services (AD DS).** Microsoft's LDAP+Kerberos
  directory: a replicated database of users, computers, groups, and policy,
  served by one or more **domain controllers**. *Why:* it's the identity and
  policy plane for the Windows half of the fleet, and the auth source for Vault
  LDAP. *Otherwise:* Samba AD (open-source, but WSFC/SQL want genuine Windows
  AD), or no domain at all (then SQL FCI/AG and GMSA are impossible).
- **Forest / domain / `Install-ADDSForest`.** A *forest* is the top-level AD
  trust + schema boundary; a *domain* (here `nexus.lab`) lives inside it.
  `Install-ADDSForest` creates a brand-new forest **and** promotes the local
  machine to its first DC in one shot. *Why a new forest:* the lab is
  self-contained. *Otherwise:* joining an existing corporate forest (not a
  standalone lab).
- **Domain controller (DC) + AD-integrated DNS.** A DC holds a replica of the
  directory and (with the `-InstallDns` flag) runs a DNS server whose zones are
  stored *in* AD and replicated automatically. *Why on the DC:* AD is
  fundamentally DNS-dependent (clients find DCs via `_ldap._tcp` SRV records);
  co-locating DNS is the standard pattern. *Otherwise:* external DNS (then you
  hand-maintain SRV records — fragile).
- **Replica DC / `Install-ADDSDomainController`.** Adds a *second* DC to an
  *existing* domain by replicating from a source DC. *Why:* HA — losing one DC
  doesn't lose auth/DNS. *Otherwise:* single DC (a SPOF for the entire Windows
  fleet).
- **Organizational Units (OUs).** Container objects for delegation + Group
  Policy scoping. We create `Servers`, `Workstations`, `ServiceAccounts`,
  `Groups`. *Why:* clean object placement + a target for future GPOs. **DCs stay
  in the built-in `CN=Domain Controllers`** — a Microsoft hard rule (moving them
  breaks replication/GPO scoping).
- **Default Domain Password & Lockout Policy.** The domain-wide rules for
  password length/complexity/age and account lockout. *Why one policy now:* a
  single cohort (`nexusadmin`); fine-grained PSOs come later when humans vs.
  service accounts diverge. *Otherwise:* PSOs (overkill today).
- **`LDAPServerIntegrity` (signing).** A DC registry value: `2` = *require* LDAP
  signing, `1` = *negotiate*. Windows clients auto-negotiate sign-and-seal;
  non-Windows LDAP clients (Vault's go-ldap) doing a plain `:389` simple bind get
  rejected with "Strong Auth Required" at `2`. *Why lower to 1:* unblocks Vault
  LDAP (Guide 04) before LDAPS certs exist. *Otherwise:* keep `2` and require
  LDAPS everywhere from day one (deferred to the PKI guide).
- **PDC-emulator time (W32Time).** The forest-root PDC is the authoritative time
  source; every member inherits time from it, and the PDC syncs from external NTP
  peers. *Why:* Kerberos rejects tickets with >5 min clock skew, so domain time
  must be coherent. *Otherwise:* skew → auth failures.
- **KDS root key + GMSA.** The Key Distribution Service root key is a
  **forest-wide, one-time** seed that enables **Group Managed Service Accounts** —
  service identities whose passwords AD generates and rotates automatically (no
  human ever knows them). *Why scaffold now:* the SQL engine service account
  (Guide 11) is the first real GMSA consumer; seeding the KDS key here means it
  just plugs in. *Otherwise:* per-service static passwords (rotation burden,
  weaker). **Server-2025 caveat:** `Add-KdsRootKey` only works from an
  RDP/console session — it silently fails over SSH (see §8).

---

## 3. Prerequisites

| # | Requirement | One-command verify |
|---|---|---|
| 1 | `dc-nexus` baselined per Guide 00 §5.C (ws2025-desktop, RSAT/GPMC/DNS tools, at **`192.168.70.240`**) | Host: `Test-NetConnection 192.168.70.240 -Port 3389` → `True` |
| 2 | `dc-nexus-2` baselined per Guide 00 §5.C (at **`192.168.70.242`**) | Host: `Test-NetConnection 192.168.70.242 -Port 3389` → `True` |
| 3 | Guide 01 `nexus-gateway` up (DNS/NTP/egress) | Host: `Test-NetConnection 192.168.70.1 -Port 53` → `True` |
| 4 | You have an RDP client + the `nexusadmin` / Administrator build password (`NexusPackerBuild!1`) | RDP to `192.168.70.240` succeeds |
| 5 | A chosen **DSRM** (Directory Services Restore Mode) password — strong, ≥14 chars | (you pick it; stored securely; reused for both DCs) |

> **IP note — canon vs. reality.** `vms.yaml` lists `dc-nexus` canonically at
> `.10` and `dc-nexus-2` at `.11`, but the running lab uses **`.240`** and
> **`.242`** (a documented drift — `.11` is owned by `sql-fci-1`; see ADR-0039
> and the `vms.yaml` notes). **Every overlay in the automated lab hard-codes
> `.240`/`.242`,** and the reverse-DNS PTRs + DNS forwarding all key off those.
> So when you do Guide 00 §C.4.2 for these two nodes, assign `nic0`
> **`192.168.70.240`** (`dc-nexus`) and **`192.168.70.242`** (`dc-nexus-2`) — not
> `.10`/`.11`. This guide uses `.240`/`.242` throughout.

---

## 4. Target topology

| Field | `dc-nexus` (forest root) | `dc-nexus-2` (replica) |
|---|---|---|
| OS | ws2025-desktop | ws2025-desktop |
| vCPU / RAM / disk | 2 / 4 GB / 60 GB | 2 / 4 GB / 60 GB |
| VMnet11 IP (nic0) | `192.168.70.240/24` | `192.168.70.242/24` |
| VMnet10 IP (nic1) | `192.168.10.10/24` | `192.168.10.11/24` |
| VMware folder | `H:\VMS\NexusPlatform\01-foundation\dc-nexus\` | `…\dc-nexus-2\` |
| Role | AD DS forest root + DNS + PDC + KDS/GMSA | AD DS replica + DNS replica |

**Forest facts (canon):**

| Setting | Value |
|---|---|
| Domain / forest root | `nexus.lab` |
| NetBIOS name | `NEXUS` |
| Forest / domain functional level | Windows Server 2025 (Server 2016+ mode) |
| OUs | `Servers`, `Workstations`, `ServiceAccounts`, `Groups` (protected from accidental deletion) |
| Password policy | MinLength **14**, Complexity on, History 24, MaxAge 0 (never), MinAge 0 |
| Lockout | Threshold 5, Duration 15 min, Observation 15 min |
| `LDAPServerIntegrity` | **1** (negotiate — lab relaxation for Vault LDAP) |
| PDC time peers | `time.cloudflare.com`, `time.nist.gov`, `pool.ntp.org`, `time.windows.com` |
| Reverse DNS zone | `70.168.192.in-addr.arpa` (PTRs: `.240`→dc-nexus) |
| KDS root key | one, effective immediately (single-DC trick) |
| Sample GMSA | `gmsa-nexus-demo$` in `OU=ServiceAccounts`; consumers group `nexus-gmsa-consumers` in `OU=Groups` |

---

## 5. Step-by-step build

> **WHERE convention:** AD configuration is done **on the DC over RDP, in an
> elevated PowerShell** (the by-hand path — interactive AD cmdlets are far more
> reliable than the automated lab's SSH+base64 transit, and KDS *requires*
> console). Host steps are marked. Substitute your DSRM password for
> `<DSRM_PWD>`.

### 5.1 — Gateway forwards `nexus.lab` to the DC (one line on `nexus-gateway`)

> **Step 5.1.1 — Add a dnsmasq forward for the AD zone**
> **WHERE:** `nexus-gateway` (`192.168.70.1`), root shell (SSH from host).
> **WHY:** the gateway is the lab's resolver; AD owns `nexus.lab`. Forward
> `nexus.lab` queries to `dc-nexus` (`.240`) so any lab host resolving
> `_ldap._tcp.nexus.lab` / `dc-nexus.nexus.lab` reaches the DC. Without this, the
> replica's domain-join (and later Vault LDAP) can't find the domain.
> **WHAT:**
> ```bash
> echo 'server=/nexus.lab/192.168.70.240' | sudo tee /etc/dnsmasq.d/nexus-lab-forward.conf
> sudo dnsmasq --test && sudo systemctl reload dnsmasq
> ```
> **EXPECTED:** `dnsmasq: syntax check OK.`; reload succeeds.
> **VERIFY (after §5.2 promotes the DC):** from the host,
> `Resolve-DnsName -Server 192.168.70.1 dc-nexus.nexus.lab` → `192.168.70.240`.

> **Step 5.1.2 — Point both DCs' DNS at the forest root**
> **WHERE:** `dc-nexus` and `dc-nexus-2` consoles (RDP), elevated PowerShell.
> **WHY:** a DC must resolve `nexus.lab` SRV records to promote/join. `dc-nexus`
> points at **itself** (loopback) once it's a DNS server; `dc-nexus-2` points at
> `dc-nexus` (`.240`) for the join, then adds itself after it becomes a DNS
> server.
> **WHAT:**
> - On `dc-nexus` (after §5.2 promotion): `Set-DnsClientServerAddress -InterfaceAlias 'nic0' -ServerAddresses '127.0.0.1'`
> - On `dc-nexus-2` (before §5.7 join): `Set-DnsClientServerAddress -InterfaceAlias 'nic0' -ServerAddresses '192.168.70.240'`
> **EXPECTED:** DNS client settings updated.
> **VERIFY:** `Resolve-DnsName nexus.lab -Type SOA` succeeds on each (once the
> forest exists).

### 5.2 — Promote `dc-nexus` to the forest root

> **Step 5.2.1 — Reconcile the local Administrator password (promotion prereq)**
> **WHERE:** `dc-nexus` console (RDP as `nexusadmin`), elevated PowerShell.
> **WHY:** on forest creation, the local Administrator **becomes** the domain
> Administrator; `Install-ADDSForest`'s prereq check **refuses** if that account
> has a blank password. Set + enable it first. (In the automated lab the sysprep
> generalize can wipe this; by hand it's set from Guide 00 §C.2.3, but reconcile
> to be safe.)
> **WHAT:**
> ```powershell
> Set-LocalUser -Name 'Administrator' -Password (ConvertTo-SecureString 'NexusPackerBuild!1' -AsPlainText -Force) -PasswordNeverExpires $true
> Enable-LocalUser -Name 'Administrator'
> ```
> **EXPECTED:** no error.
> **VERIFY:** `(Get-LocalUser Administrator).Enabled` → `True`.

> **Step 5.2.2 — Install the AD DS role + create the forest**
> **WHERE:** `dc-nexus` console, elevated PowerShell.
> **WHY:** install the binaries, then `Install-ADDSForest` creates `nexus.lab`,
> promotes this box to its first DC, and stands up AD-integrated DNS. The NTDS
> database + SYSVOL go to the canonical paths. `-NoRebootOnCompletion` lets us run
> the post-promotion fixups (§5.2.3) **before** the reboot, in one sitting.
> **WHAT:**
> ```powershell
> Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
>
> Import-Module ADDSDeployment
> $dsrm = ConvertTo-SecureString '<DSRM_PWD>' -AsPlainText -Force
> Install-ADDSForest `
>   -DomainName 'nexus.lab' `
>   -DomainNetbiosName 'NEXUS' `
>   -SafeModeAdministratorPassword $dsrm `
>   -InstallDns `
>   -CreateDnsDelegation:$false `
>   -DatabasePath 'C:\Windows\NTDS' `
>   -LogPath 'C:\Windows\NTDS' `
>   -SysvolPath 'C:\Windows\SYSVOL' `
>   -Force `
>   -NoRebootOnCompletion
> ```
> **EXPECTED:** a long run (~3–5 min) ending with
> `Message: Operation completed successfully` and a warning that the machine will
> need to reboot (we control that next).
> **VERIFY:** `Get-WindowsFeature AD-Domain-Services` shows `Installed`; do **not**
> reboot yet.

> **Step 5.2.3 — Post-promotion remediation (run BEFORE the reboot)**
> **WHERE:** `dc-nexus` console, elevated PowerShell (same session).
> **WHY:** forest creation converts the local SAM into the AD database. The
> migrated `nexusadmin` (a) has its password wiped, (b) is **not** in Domain
> Admins, and (c) no longer matches sshd's `AllowUsers nexusadmin` (it's now
> `NEXUS\nexusadmin`). All three must be fixed in the **same automation block**
> or you lock yourself out (canonized in `feedback_addsforest_post_promotion.md`).
> **WHAT:**
> ```powershell
> Import-Module ActiveDirectory
> # (a)+(b) restore nexusadmin as a usable Domain Admin
> Set-ADAccountPassword -Identity nexusadmin -Reset -NewPassword (ConvertTo-SecureString 'NexusAdmin!1' -AsPlainText -Force)
> Enable-ADAccount -Identity nexusadmin
> Add-ADGroupMember -Identity 'Domain Admins' -Members nexusadmin
>
> # (c) sshd AllowUsers no longer matches the domain-qualified name -> drop it
> # (trust becomes pubkey + Administrators-group membership)
> $cfg = Get-Content 'C:\ProgramData\ssh\sshd_config'
> $cfg = $cfg -replace '^\s*AllowUsers.*$', '# AllowUsers (removed post-AD-promotion: trust = pubkey + Administrators group)'
> $cfg | Set-Content 'C:\ProgramData\ssh\sshd_config' -Encoding ascii
> Restart-Service sshd -Force
> ```
> **EXPECTED:** all commands succeed; `nexusadmin` is now a domain account in
> Domain Admins.
> **VERIFY:** `Get-ADGroupMember 'Domain Admins' | Select Name` lists
> `nexusadmin`.

> **Step 5.2.4 — Reboot and confirm the forest is live**
> **WHERE:** `dc-nexus` console, then host.
> **WHY:** the promotion needs a reboot to finish converting the box into a DC.
> **WHAT:** `Restart-Computer -Force` — wait ~3–5 min for AD DS to come up.
> Then point DNS at itself (§5.1.2) and verify.
> **EXPECTED:** the box boots as a domain controller.
> **VERIFY (on `dc-nexus` after reboot):**
> ```powershell
> Get-ADDomain  | Format-List Forest, DomainMode, NetBIOSName, DistinguishedName
> Get-ADForest  | Format-List Name, ForestMode, RootDomain, GlobalCatalogs
> ```
> shows `Forest: nexus.lab`, `NetBIOSName: NEXUS`. From the host:
> `Resolve-DnsName -Server 192.168.70.240 nexus.lab -Type SOA` returns the DC.

### 5.3 — OU layout

> **Step 5.3.1 — Create the four canonical OUs**
> **WHERE:** `dc-nexus` console, elevated PowerShell.
> **WHY:** clean homes for member servers, workstations, service accounts, and
> groups; each protected from accidental deletion. DCs deliberately stay in the
> built-in `CN=Domain Controllers` container (Microsoft hard rule).
> **WHAT:**
> ```powershell
> Import-Module ActiveDirectory
> $dnRoot = 'DC=nexus,DC=lab'
> foreach ($ou in 'Servers','Workstations','ServiceAccounts','Groups') {
>     $dn = "OU=$ou,$dnRoot"
>     $exists = $null
>     try { $exists = Get-ADOrganizationalUnit -Identity $dn -ErrorAction Stop } catch { $exists = $null }
>     if (-not $exists) {
>         New-ADOrganizationalUnit -Name $ou -Path $dnRoot -ProtectedFromAccidentalDeletion $true
>         Write-Host "created OU=$ou"
>     } else { Write-Host "OU=$ou already present" }
> }
> ```
> **EXPECTED:** four OUs created (or "already present" on re-run).
> **VERIFY:** `Get-ADOrganizationalUnit -Filter * | Select Name` lists the four.

### 5.4 — Domain password & lockout policy

> **Step 5.4.1 — Set the default domain password + lockout policy**
> **WHERE:** `dc-nexus` console, elevated PowerShell.
> **WHY:** one domain-wide policy for all accounts. Canon: MinLength **14**,
> complexity on, history 24, passwords never expire (modern NIST stance: rotate
> on compromise, not schedule), lockout after 5 bad attempts for 15 min.
> Lockout threshold of 5 (not 1–3) is deliberate so a stale-credential probe loop
> can't lock out `nexusadmin` and sever both SSH and RDP at once.
> **WHAT:**
> ```powershell
> Import-Module ActiveDirectory
> $span = New-TimeSpan -Minutes 15
> Set-ADDefaultDomainPasswordPolicy -Identity (Get-ADDomain).DistinguishedName `
>   -ComplexityEnabled $true `
>   -MinPasswordLength 14 `
>   -LockoutThreshold 5 `
>   -LockoutDuration $span `
>   -LockoutObservationWindow $span `
>   -MaxPasswordAge (New-TimeSpan -Days 0) `
>   -MinPasswordAge (New-TimeSpan -Days 0) `
>   -PasswordHistoryCount 24 `
>   -ReversibleEncryptionEnabled $false
> ```
> **EXPECTED:** no error.
> **VERIFY:**
> ```powershell
> Get-ADDefaultDomainPasswordPolicy | Format-List MinPasswordLength, ComplexityEnabled, LockoutThreshold, PasswordHistoryCount
> ```
> shows `MinPasswordLength : 14`, `LockoutThreshold : 5`.
>
> > **Note:** `NexusAdmin!1` (from §5.2.3) is 12 chars and satisfies the *prior*
> > 12-char default, but **not** the 14-char policy you just set. That's fine —
> > the policy applies to *future* password *changes*; the existing value isn't
> > retroactively rejected. Guide 04 rotates `nexusadmin` to a Vault-generated
> > 14+ char secret.

### 5.5 — Reverse DNS + PDC time

> **Step 5.5.1 — Create the reverse-DNS zone + PTR for the DC**
> **WHERE:** `dc-nexus` console, elevated PowerShell.
> **WHY:** PTR records make logs/Kerberos errors/sshd reverse-lookups read as
> hostnames instead of bare IPs. AD-integrated + secure-update so it replicates.
> **WHAT:**
> ```powershell
> Import-Module DnsServer
> if (-not (Get-DnsServerZone -Name '70.168.192.in-addr.arpa' -ErrorAction SilentlyContinue)) {
>     Add-DnsServerPrimaryZone -NetworkId '192.168.70.0/24' -ReplicationScope 'Domain' -DynamicUpdate 'Secure'
> }
> if (-not (Get-DnsServerResourceRecord -ZoneName '70.168.192.in-addr.arpa' -Name '240' -RRType Ptr -ErrorAction SilentlyContinue)) {
>     Add-DnsServerResourceRecordPtr -ZoneName '70.168.192.in-addr.arpa' -Name '240' -PtrDomainName 'dc-nexus.nexus.lab.'
> }
> ```
> **EXPECTED:** zone + PTR created.
> **VERIFY:** `Resolve-DnsName 192.168.70.240 -Type PTR` → `dc-nexus.nexus.lab`.
> (Add a `.242` PTR for `dc-nexus-2` after §5.8, same pattern.)

> **Step 5.5.2 — Configure the PDC as the authoritative time source**
> **WHERE:** `dc-nexus` console, elevated PowerShell.
> **WHY:** the forest-root PDC syncs from external NTP and every member inherits
> time from it. `0x8` = SpecialInterval (use the configured poll interval) — the
> Microsoft-recommended PDC flag.
> **WHAT:**
> ```powershell
> w32tm /config /manualpeerlist:"time.cloudflare.com,0x8 time.nist.gov,0x8 pool.ntp.org,0x8 time.windows.com,0x8" /syncfromflags:MANUAL /reliable:YES /update
> Restart-Service w32time
> Start-Sleep 3
> w32tm /resync /force
> ```
> **EXPECTED:** `The command completed successfully.`
> **VERIFY:** `w32tm /query /status | Select-String 'Source|Stratum'` shows an
> external `Source` and a low stratum.

### 5.6 — LDAP signing relaxation + KDS root key + GMSA scaffolding

> **Step 5.6.1 — Lower `LDAPServerIntegrity` to 1 (for Vault LDAP, Guide 04)**
> **WHERE:** `dc-nexus` console, elevated PowerShell.
> **WHY:** AD's default `LDAPServerIntegrity=2` rejects plain `:389` simple binds
> from non-Windows clients ("Strong Auth Required" / LDAP result 8). Vault's
> go-ldap bind account hits exactly this. Lowering to `1` (negotiate) unblocks
> Guide 04 until LDAPS certs land. Requires an NTDS restart (~5–30 s AD blip).
> **WHAT:**
> ```powershell
> $path = 'HKLM:\SYSTEM\CurrentControlSet\Services\NTDS\Parameters'
> Set-ItemProperty -Path $path -Name LDAPServerIntegrity -Value 1 -Type DWord
> Restart-Service NTDS -Force
> Start-Sleep 10
> ```
> **EXPECTED:** NTDS restarts and comes back.
> **VERIFY:** `(Get-ItemProperty $path -Name LDAPServerIntegrity).LDAPServerIntegrity`
> → `1`; `(Get-Service NTDS).Status` → `Running`.

> **Step 5.6.2 — Add the KDS root key (MUST be on the console/RDP)**
> **WHERE:** `dc-nexus` **console/RDP** (NOT over SSH), elevated PowerShell.
> **WHY:** the KDS root key is the forest-wide seed that enables GMSA. On a
> single-DC forest there's no replication-delay race, so we make it effective
> immediately via the `-10 hours` trick (production waits 10 h). **Critical:** on
> Server 2025, `Add-KdsRootKey` returns `ERROR_NOT_SUPPORTED` over SSH and even
> via `Invoke-Command` it returns a fake GUID without persisting — it **only**
> works from an interactive RDP/console session (canonized in
> `feedback_kds_rootkey_server2025_ssh.md`). This is *the* reason Guide 02's AD
> work is done over RDP, not SSH.
> **WHAT:**
> ```powershell
> Add-KdsRootKey -EffectiveTime ((Get-Date).AddHours(-10))
> ```
> **EXPECTED:** returns a GUID (the key id).
> **VERIFY:** `Get-KdsRootKey` returns **one** key (count ≥ 1). If it's empty,
> you ran it over SSH — re-run on the console.

> **Step 5.6.3 — Create the GMSA consumers group + sample GMSA**
> **WHERE:** `dc-nexus` console, elevated PowerShell.
> **WHY:** scaffolding so the first real GMSA consumer (the SQL engine account,
> Guide 11) just adds itself to `nexus-gmsa-consumers`. The sample
> `gmsa-nexus-demo$` proves the infrastructure end-to-end.
> **WHAT:**
> ```powershell
> Import-Module ActiveDirectory
> $dnRoot = 'DC=nexus,DC=lab'
>
> if (-not (Get-ADGroup -Filter "Name -eq 'nexus-gmsa-consumers'" -ErrorAction SilentlyContinue)) {
>     New-ADGroup -Name 'nexus-gmsa-consumers' -GroupScope Global -GroupCategory Security `
>       -Path "OU=Groups,$dnRoot" `
>       -Description 'GMSA consumers -- members can retrieve managed passwords for GMSAs that allow this group'
> }
>
> if (-not (Get-ADServiceAccount -Filter "Name -eq 'gmsa-nexus-demo'" -ErrorAction SilentlyContinue)) {
>     New-ADServiceAccount -Name 'gmsa-nexus-demo' `
>       -SamAccountName 'gmsa-nexus-demo$' `
>       -DNSHostName 'gmsa-nexus-demo.nexus.lab' `
>       -PrincipalsAllowedToRetrieveManagedPassword 'nexus-gmsa-consumers' `
>       -Path "OU=ServiceAccounts,$dnRoot" `
>       -Description 'Sample GMSA (scaffold; no real consumer yet)' `
>       -Enabled $true
> }
> ```
> **EXPECTED:** group + GMSA created.
> **VERIFY:** `Get-ADServiceAccount gmsa-nexus-demo` returns the object;
> `Get-ADGroup nexus-gmsa-consumers` returns the group. (With the KDS key present
> from §5.6.2, a domain-joined member that's in the consumers group can later
> `Test-ADServiceAccount gmsa-nexus-demo` → `True`.)

### 5.7 — Domain-join `dc-nexus-2`

> **Step 5.7.1 — Patch sshd + join `dc-nexus-2` to `nexus.lab`**
> **WHERE:** `dc-nexus-2` console (RDP as `nexusadmin`), elevated PowerShell.
> **WHY:** a box must be a domain member before it can become a replica DC.
> First patch sshd (post-join the user is `NEXUS\nexusadmin`, which won't match
> `AllowUsers nexusadmin` — same fix as §5.2.3), then `Add-Computer` with the
> domain admin credential and reboot. (Ensure DNS points at `.240` per §5.1.2.)
> **WHAT:**
> ```powershell
> # sshd AllowUsers fix (pre-reboot)
> $cfg = Get-Content 'C:\ProgramData\ssh\sshd_config'
> $cfg = $cfg -replace '^\s*AllowUsers.*$', '# AllowUsers (removed for AD-joined posture)'
> $cfg | Set-Content 'C:\ProgramData\ssh\sshd_config' -Encoding ascii
>
> $cred = New-Object System.Management.Automation.PSCredential(
>     'NEXUS\nexusadmin',
>     (ConvertTo-SecureString 'NexusAdmin!1' -AsPlainText -Force))
> Add-Computer -DomainName 'nexus.lab' -Credential $cred -Force -Restart
> ```
> **EXPECTED:** the box joins and reboots.
> **VERIFY (after reboot, RDP back in):**
> `(Get-WmiObject Win32_ComputerSystem).PartOfDomain` → `True`;
> `(Get-WmiObject Win32_ComputerSystem).Domain` → `nexus.lab`.

### 5.8 — Promote `dc-nexus-2` to a replica DC

> **Step 5.8.1 — Install AD DS + promote as a replica of `dc-nexus`**
> **WHERE:** `dc-nexus-2` console, elevated PowerShell.
> **WHY:** `Install-ADDSDomainController` adds this box as a second DC of
> `nexus.lab`, replicating from `dc-nexus.nexus.lab` into the
> `Default-First-Site-Name` site, with its own AD-integrated DNS replica. The
> `-ReplicationSourceDC` **must be a literal FQDN** — the automated lab's M1
> transient was a single-quoted PowerShell string that never interpolated, so the
> promotion silently no-op'd; by hand you type the literal, so just don't fat-
> finger it.
> **WHAT:**
> ```powershell
> Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
>
> Import-Module ADDSDeployment
> $dsrm = ConvertTo-SecureString '<DSRM_PWD>' -AsPlainText -Force
> $cred = New-Object System.Management.Automation.PSCredential(
>     'NEXUS\nexusadmin',
>     (ConvertTo-SecureString 'NexusAdmin!1' -AsPlainText -Force))
> Install-ADDSDomainController `
>   -DomainName 'nexus.lab' `
>   -Credential $cred `
>   -SafeModeAdministratorPassword $dsrm `
>   -InstallDns:$true `
>   -CreateDnsDelegation:$false `
>   -ReplicationSourceDC 'dc-nexus.nexus.lab' `
>   -SiteName 'Default-First-Site-Name' `
>   -DatabasePath 'C:\Windows\NTDS' `
>   -LogPath 'C:\Windows\NTDS' `
>   -SysvolPath 'C:\Windows\SYSVOL' `
>   -Force:$true `
>   -NoRebootOnCompletion:$false
> ```
> **EXPECTED:** a ~5–10 min run ending in success, then an **automatic reboot**.
> **VERIFY (after reboot):** on `dc-nexus-2`,
> `Get-ADDomain | Format-List Forest, DomainMode, NetBIOSName` → `nexus.lab`.

> **Step 5.8.1a — Pin `dc-nexus-2` to STATIC DNS (partner + self) — MANDATORY**
> **WHERE:** `dc-nexus-2` console, elevated PowerShell.
> **WHY:** a replica DC **must** use static DNS: the partner DC as **preferred**
> and itself (loopback) as **alternate**. `dc-nexus-2` boots with DHCP DNS (the
> gateway `.1`, which does **not** serve `nexus.lab`), so post-promotion it can't
> resolve the partner's `<dsa-guid>._msdcs.nexus.lab` CNAME → AD replication
> **fails with error 8524** ("DNS lookup failure") and the directory silently
> **diverges** whenever `dc-nexus-2` has been offline (it is normally OFF in the
> base-6 fleet). This is the canonical fix baked into the automated foundation
> terraform (`role-overlay-dc-nexus-2-promotion.tf` step 6.5, 2026-07-06). Order
> matters: **preferred = `.240` (partner), alternate = `127.0.0.1` (self)**.
> **WHAT:**
> ```powershell
> $a = (Get-NetIPAddress -IPAddress '192.168.70.242').InterfaceAlias
> Set-DnsClientServerAddress -InterfaceAlias $a -ServerAddresses '192.168.70.240','127.0.0.1'
> Clear-DnsClientCache
> ipconfig /registerdns
> repadmin /syncall /AdeP        # force a full replication sync both ways
> (Get-DnsClientServerAddress -InterfaceAlias $a -AddressFamily IPv4).ServerAddresses
> ```
> **EXPECTED:** DNS servers = `192.168.70.240, 127.0.0.1`; `repadmin /syncall`
> reports success, no 8524.
> **VERIFY:** `Get-DnsClientServerAddress` shows the static pair (not the gateway
> `.1`); `repadmin /showrepl` (next step) is clean.

> **Step 5.8.2 — Verify replication health (both directions)**
> **WHERE:** `dc-nexus` console, elevated PowerShell.
> **WHY:** prove the two DCs are replicating — the whole point of the HA pair.
> **WHAT:**
> ```powershell
> Get-ADDomainController -Identity dc-nexus-2 | Format-List HostName, IPv4Address, Site, IsGlobalCatalog
> Start-Sleep 60   # let the first replication cycle run
> repadmin /showrepl
> Get-ADReplicationPartnerMetadata -Target dc-nexus.nexus.lab -PartnerType Both |
>   Format-Table Partner, LastReplicationSuccess, LastReplicationResult
> ```
> **EXPECTED:** `dc-nexus-2` listed as a DC; `repadmin /showrepl` shows
> `LastReplicationResult: 0` (success) for the dc-nexus↔dc-nexus-2 links.
> **VERIFY:** `LastReplicationResult` is `0` both directions; no failures.

---

## 6. Validation — by-hand acceptance smoke

Run from the **host** (and one cross-DC object test). Both DCs powered on.

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | Forest root reachable | `Test-NetConnection 192.168.70.240 -Port 389` | `True` (LDAP) |
| 2 | Forest exists | `Resolve-DnsName -Server 192.168.70.240 nexus.lab -Type SOA` | returns the SOA |
| 3 | SRV records published | `Resolve-DnsName -Server 192.168.70.240 _ldap._tcp.nexus.lab -Type SRV` | lists `dc-nexus` (and `dc-nexus-2` after §5.8) |
| 4 | Gateway forwards the zone | `Resolve-DnsName -Server 192.168.70.1 dc-nexus.nexus.lab` | `192.168.70.240` |
| 5 | Reverse DNS works | `Resolve-DnsName -Server 192.168.70.240 192.168.70.240 -Type PTR` | `dc-nexus.nexus.lab` |
| 6 | OUs present | on DC: `Get-ADOrganizationalUnit -Filter * \| Measure` | `Count ≥ 4` |
| 7 | Password policy | on DC: `(Get-ADDefaultDomainPasswordPolicy).MinPasswordLength` | `14` |
| 8 | LDAP signing relaxed | on DC: `(Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Services\NTDS\Parameters' LDAPServerIntegrity).LDAPServerIntegrity` | `1` |
| 9 | PDC time source | on DC: `w32tm /query /status \| Select-String Source` | an external NTP source |
| 10 | KDS root key present | on DC: `Get-KdsRootKey \| Measure` | `Count ≥ 1` |
| 11 | Sample GMSA exists | on DC: `Get-ADServiceAccount gmsa-nexus-demo` | returns the object |
| 12 | Replica is a DC | on DC: `Get-ADDomainController -Filter * \| Select Name` | both `dc-nexus`, `dc-nexus-2` |
| 13 | Replication healthy | on DC: `repadmin /showrepl` | `LastReplicationResult: 0` both ways |
| 14 | Replica survives DC-1 loss | power off `dc-nexus`, then `Resolve-DnsName -Server 192.168.70.242 nexus.lab` + a domain logon | still resolves + authenticates (then power `dc-nexus` back on) |

**Checks 1–13 green ⇒ Guide 02 satisfied.** Check 14 is the HA proof — the whole
reason for the second DC.

---

## 7. Teardown / reset

**Demote a DC** (clean — removes it from the domain gracefully):

```powershell
# On the DC being removed (dc-nexus-2 first; never demote the last DC unless wiping the forest):
Uninstall-ADDSDomainController -DemoteOperationMasterRole -ForceRemoval `
  -LocalAdministratorPassword (ConvertTo-SecureString 'NexusPackerBuild!1' -AsPlainText -Force) -Force
```

**Wipe the whole forest** — delete both VMs (the directory lives on their disks):

```powershell
foreach ($vm in 'dc-nexus-2','dc-nexus') {
  $vmx = "H:\VMS\NexusPlatform\01-foundation\$vm\$vm.vmx"
  & 'C:\Program Files\VMware\VMware Workstation\vmrun.exe' stop $vmx soft 2>$null
  & 'C:\Program Files\VMware\VMware Workstation\vmrun.exe' deleteVM $vmx
  Remove-Item "H:\VMS\NexusPlatform\01-foundation\$vm" -Recurse -Force
}
# Also remove the gateway forward: on nexus-gateway -> sudo rm /etc/dnsmasq.d/nexus-lab-forward.conf; sudo systemctl reload dnsmasq
```

> If you destroy and rebuild the forest, **stale AD-related KV/credentials don't
> exist yet** (that's Guide 04), so no extra cleanup is needed here — but Guide
> 04+ rebuilds must re-seed Vault against the *new* forest.

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Install-ADDSForest` aborts: "local Administrator password is blank…" | The local Administrator (→ domain Administrator) has no password | Set + enable it first (§5.2.1). |
| After promotion, SSH to the DC is rejected ("not allowed because not listed in AllowUsers") | Post-promotion the user is `NEXUS\nexusadmin`, which doesn't match `AllowUsers nexusadmin` | Patch sshd_config to drop the `AllowUsers` line, restart sshd (§5.2.3) — do it in the **same block** as the promotion. See `feedback_addsforest_post_promotion.md`. |
| `nexusadmin` can't log in / isn't admin after promotion | Forest creation migrated it but wiped its password + left it out of Domain Admins | Reset password + `Add-ADGroupMember 'Domain Admins'` (§5.2.3). |
| `Get-KdsRootKey` shows **zero** keys even after `Add-KdsRootKey` "succeeded" | On Server 2025, `Add-KdsRootKey` doesn't persist over SSH / `Invoke-Command` (returns a fake GUID) | Run it from an **RDP/console** session (§5.6.2). See `feedback_kds_rootkey_server2025_ssh.md`. |
| Vault LDAP bind later fails "Strong Auth Required" (LDAP result 8) | AD `LDAPServerIntegrity=2` rejects plain-`:389` simple binds from Linux | Lower to `1` + restart NTDS (§5.6.1). See `feedback_ad_ldap_simple_bind_signing.md`. |
| Replica promotion "succeeds" but `dc-nexus-2` stays a Member Server | `-ReplicationSourceDC` got a non-resolvable value (the automated M1 bug: a literal `'dc-nexus.${domain}'`) | Pass a **literal FQDN** `'dc-nexus.nexus.lab'` (§5.8.1). |
| `Add-Computer` on `dc-nexus-2` fails to find the domain | `dc-nexus-2`'s DNS doesn't point at the DC, or the gateway forward is missing | Set DNS to `192.168.70.240` (§5.1.2) and add the gateway forward (§5.1.1). |
| `repadmin /showrepl` shows replication failures | First cycle hasn't run, or time skew between DCs | Wait ~1–5 min; confirm both DCs' clocks agree (PDC time, §5.5.2). |
| Replication **error 8524** ("DNS lookup failure") after `dc-nexus-2` has been offline; directory silently diverges | `dc-nexus-2` reverted to **DHCP DNS** (gateway `.1`, which doesn't serve `nexus.lab`) → can't resolve the partner's `_msdcs.nexus.lab` CNAME | Pin **static DNS** (preferred `192.168.70.240` / alternate `127.0.0.1`), `ipconfig /registerdns`, then `repadmin /syncall /AdeP` (§5.8.1a). This is the canonical 2026-07-06 fix. |
| Domain logons fail intermittently across the fleet | Clock skew > 5 min (Kerberos) | Ensure the PDC time config (§5.5.2) is healthy and members inherit it. |

---

### Cross-references

- **Network canon / DNS:** `nexus-platform-plan/docs/infra/network.md` (DNS section)
- **VM inventory:** `nexus-platform-plan/docs/infra/vms.yaml` (`foundation` cluster)
- **ADRs:** ADR-0039 (foundation HA — 2nd DC, Phase 0.M)
- **Automated equivalents:** `nexus-infra-vmware/terraform/envs/foundation/role-overlay-dc-*.tf` + `role-overlay-dc-nexus-2-promotion.tf`
- **Previous guide:** [`01-foundation-nexus-gateway.md`](./01-foundation-nexus-gateway.md)
- **Next guide:** Guide 03 — Foundation · Vault HA (3-node Raft cluster + transit). See [`INDEX.md`](../INDEX.md).
