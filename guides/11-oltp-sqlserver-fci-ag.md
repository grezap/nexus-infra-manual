# Guide 11 — OLTP · SQL Server FCI + Always-On AG (WSFC + iSCSI)

> **Mirrors:** `nexus-infra-oltp` — the `oltp-sqlserver-node` Packer template
> (`10-sql-install.ps1`, `11-cluster-features.ps1`) + the `oltp-sqlserver` env
> overlays (`sqlserver-domain-join`, `sqlserver-tls`, `iscsi-attach`,
> `wsfc-bootstrap`, `fci-install`, `ag-bootstrap`, `ag-listener`). The automated
> lab drives every step over SSH as **local** `nexusadmin`, so it wraps each
> domain-privileged command in a **scheduled-task running as `NEXUS\nexusadmin`**.
> **By hand we RDP in as `NEXUS\nexusadmin` directly** — a native domain Kerberos
> context — so the scheduled-task dance disappears and we run the commands inline.

---

## 1. Overview & purpose

The most intricate OLTP tier: a **hybrid SQL Server 2025 FCI + Always-On AG** on
**4 Windows Server 2025** nodes. It demonstrates the full enterprise SQL HA
pattern — shared-storage clustering **and** replica-based availability — in one
deployment.

- **FCI (`sql-fci-1` + `sql-fci-2`, `.11/.12`)** — a 2-node **Failover Cluster
  Instance**: both nodes share one **iSCSI LUN** (from `nexus-gateway`, Guide 01
  §5.9) as a clustered disk (`S:\`); only one node runs the SQL instance at a
  time, and it fails over to the other on node loss. The FCI's virtual identity
  is **`sqlfci` at `.16`**.
- **AG replicas (`sql-ag-rep-1` + `sql-ag-rep-2`, `.13/.14`)** — two standalone
  SQL instances holding **async** Always-On copies of the FCI's databases.
- **WSFC** — the Windows Failover Cluster (`sql-fci-cluster`, mgmt IP **`.15`**)
  underpins both the FCI and the AG; all 4 nodes are members (NodeMajority
  quorum).
- **AG Listener (`sql-ag-listener`, `.17`)** — the single client endpoint;
  WSFC migrates the IP with the AG primary on failover (the LB-tier HA primitive,
  ADR-0025 — no external LB needed).

**Why this hybrid** (vs. AG-only): to demonstrate **both** SQL HA mechanisms —
FCI's shared-storage failover and AG's replica failover — on the canonical
enterprise topology. SQL service identity on the FCI is a **GMSA**
(`gmsa-sql-engine$`); AG endpoints authenticate by **certificate** (ADR-0027).

**Dependency:**
- **Guide 00** — 4 `ws2025-desktop` nodes baselined (`.11–.14`).
- **Guide 02** — AD forest + the **KDS root key** (for GMSA) + GMSA tooling.
- **Guide 01** — the **iSCSI target** + LUN on `nexus-gateway` (§5.9).
- **Guide 04** — Vault PKI (cert issuance for the unified per-node TLS cert).

> **By-hand divergence (the big one):** the automated lab runs every WSFC/FCI/AG
> command through a **scheduled task as `NEXUS\nexusadmin`** because its SSH
> session is *local* `nexusadmin` with no Kerberos TGT. By hand you **RDP in as
> `NEXUS\nexusadmin`** — you already have the domain context, so run the commands
> directly in an elevated PowerShell. Also: `ws2025` has no `openssl`, so the cert
> is issued via the Vault **HTTP API** + assembled into a **PFX** with .NET, then
> `Import-PfxCertificate`. SQL 2025 dropped bundled `sqlcmd` — install **ODBC
> Driver 18** + **MSSQL CmdLineUtils** manually, and use `sqlcmd -C` (trust the
> chain via the installed CA).

---

## 2. Component primer

- **WSFC (Windows Server Failover Clustering).** The Windows clustering substrate:
  a set of nodes sharing a quorum + a heartbeat fabric (NetFT), able to host
  *cluster roles* that fail over between nodes. Both the FCI and the AG are WSFC
  roles. *Quorum:* NodeMajority across the 4 nodes (tolerates 1 loss).
- **FCI (Failover Cluster Instance).** A single SQL instance whose storage + IP +
  network name are WSFC resources; it runs on one node at a time and **fails over
  with its shared disk** to another. *Why shared storage:* both nodes see the same
  data — no replication, instant failover of the *same* database files.
  *Otherwise:* AG-only (no shared storage; each node has its own copy).
- **iSCSI shared LUN.** The block device both FCI nodes mount. `nexus-gateway`
  exports it (`tgt`, Guide 01 §5.9); both FCI nodes are initiators (CHAP +
  IP-ACL); **only one formats it** (`S:\`, NTFS 64K, GPT), the other just
  attaches. It becomes a clustered **Physical Disk** (a single-instance FCI takes
  a dedicated disk, *not* a CSV).
- **Always-On Availability Group (AG).** Replica-based HA: a *primary* ships
  changes to *secondary* replicas. Here the FCI is the primary; the two standalone
  replicas are **async** secondaries. **Endpoint auth is certificate-based**
  (ADR-0027) — each participant creates a `Hadr_endpoint` cert, imports the other
  two's public certs, and maps a login. *Why both FCI + AG:* FCI protects the
  instance (shared storage); AG protects the *data* across independent copies.
- **Manual seeding.** AG can auto-seed a secondary, but **not here** — the FCI's
  DBs live on the shared `S:\` while replicas use local `C:\`, which defeats
  automatic seeding. So we **backup → `RESTORE WITH MOVE … NORECOVERY` → join**.
  And a brand-new DB needs a **full backup first** (else `ADD DATABASE` fails Msg
  1475).
- **AG Listener.** A virtual network name + IP (`.17`) that WSFC migrates with the
  AG primary; clients connect to it and always reach the current primary. It needs
  **Kerberos SPNs** (`MSSQLSvc/sql-ag-listener.nexus.lab[:1433]`) on the GMSA, or
  remote `-E` (integrated auth) falls back to NTLM → `ANONYMOUS LOGON`.
- **GMSA (`gmsa-sql-engine$`).** The FCI's SQL service identity — a Group Managed
  Service Account whose password AD rotates automatically (no human knows it).
  Needs the **KDS root key** from Guide 02. The replicas run as `NETWORK SERVICE`
  (AG uses cert auth, so a GMSA buys them nothing).
- **The unified per-node TLS cert.** One cert per instance carrying **all** the
  SANs that instance serves — node name + the FCI virtual name `sqlfci` + the
  listener name. SQL has a single `SuperSocketNetLib\Certificate` slot, so the one
  cert covers node, FCI, and listener identities for `Encrypt=True;
  TrustServerCertificate=False`.

---

## 3. Prerequisites

| # | Requirement | One-command verify |
|---|---|---|
| 1 | 4 `ws2025-desktop` nodes baselined (Guide 00 §5.C), `.11–.14` | each answers `:3389` (RDP) |
| 2 | AD forest + **KDS root key** present (Guide 02 §5.6.2) | on a DC: `Get-KdsRootKey` returns ≥1 key |
| 3 | iSCSI target + LUN live on `nexus-gateway` (Guide 01 §5.9) | `ssh …@1 'sudo tgtadm --mode target --op show \| grep sql-fci-shared.img'` |
| 4 | Vault PKI usable; `sqlserver-server` PKI role can include `sqlfci`/listener SANs | created in §5.2 |
| 5 | The SQL Server 2025 Developer ISO on the build host | `Test-Path H:\VMS\ISO\<sql2025>.iso` |
| 6 | iSCSI CHAP secret known (Guide 01 §5.9 generated it) | in Vault KV `nexus/oltp/sqlserver/iscsi-chap-secret` |

> SQL edition: **Developer** (free, full Enterprise feature set incl. AG sync
> replicas). SQL 2025 = registry instance `MSSQL17.MSSQLSERVER`.

---

## 4. Target topology

| Node | Role | VMnet11 | VMnet10 | SQL service identity | RAM |
|---|---|---|---|---|---|
| `sql-fci-1` | FCI node 1 (iSCSI initiator) | `.11` | `.10.11` | `nexus.lab\gmsa-sql-engine$` | 16 GB |
| `sql-fci-2` | FCI node 2 (iSCSI initiator) | `.12` | `.10.12` | `nexus.lab\gmsa-sql-engine$` | 16 GB |
| `sql-ag-rep-1` | AG async replica | `.13` | `.10.13` | `NT AUTHORITY\NETWORK SERVICE` | 12 GB |
| `sql-ag-rep-2` | AG async replica | `.14` | `.10.14` | `NT AUTHORITY\NETWORK SERVICE` | 12 GB |

**WSFC-owned virtual IPs (no keepalived — WSFC migrates these):**

| VIP | Name | Purpose |
|---|---|---|
| `.15` | `sql-fci-cluster` (CNO) | WSFC cluster management IP |
| `.16` | `sqlfci` (FCI virtual server) | FCI client endpoint |
| `.17` | `sql-ag-listener` | AG Listener — the client front door |

> Client: `sqlcmd -S sql-ag-listener.nexus.lab -E -N` (Encrypt + strict cert
> validate). PKI role **`sqlserver-server`** (SANs include node + `sqlfci` +
> `sql-ag-listener`). AG `nexus-ag`, demo DB `nexus_demo`.

---

## 5. Step-by-step build

> **WHERE:** **RDP into the named node as `NEXUS\nexusadmin`**, elevated
> PowerShell — you have the domain Kerberos context natively (no scheduled-task
> wrapper). `vault` runs on **`vault-1`**. SQL T-SQL via `sqlcmd -C` (trust the
> installed CA chain).

### 5.1 — Domain-join all 4 + GMSA + cluster group

> **Step 5.1.1 — Domain-join the 4 nodes to `nexus.lab`**
> **WHERE:** each node (RDP as local Administrator first), elevated PowerShell.
> **WHY:** WSFC + the GMSA + Kerberos all require domain membership.
> **WHAT (on each, then reboot):**
> ```powershell
> $cred = New-Object System.Management.Automation.PSCredential('NEXUS\nexusadmin',
>   (ConvertTo-SecureString '<nexusadmin AD password>' -AsPlainText -Force))
> Add-Computer -DomainName nexus.lab -Credential $cred -Force -Restart
> ```
> **EXPECTED:** each joins + reboots.
> **VERIFY:** after reboot, RDP as `NEXUS\nexusadmin`; `(Get-WmiObject Win32_ComputerSystem).Domain` → `nexus.lab`.

> **Step 5.1.2 — Create the SQL GMSA + the cluster-members group (on a DC)**
> **WHERE:** `dc-nexus` (RDP), elevated PowerShell.
> **WHY:** `gmsa-sql-engine$` is the FCI's service identity; the two FCI computer
> accounts must be allowed to retrieve its managed password. Needs the KDS root
> key (Guide 02).
> **WHAT:**
> ```powershell
> Import-Module ActiveDirectory
> New-ADGroup -Name nexus-sql-cluster-members -GroupScope Global -GroupCategory Security -Path 'OU=Groups,DC=nexus,DC=lab'
> Add-ADGroupMember nexus-sql-cluster-members -Members 'sql-fci-1$','sql-fci-2$'
> New-ADServiceAccount -Name gmsa-sql-engine -DNSHostName gmsa-sql-engine.nexus.lab `
>   -SamAccountName 'gmsa-sql-engine$' `
>   -PrincipalsAllowedToRetrieveManagedPassword nexus-sql-cluster-members `
>   -Path 'OU=ServiceAccounts,DC=nexus,DC=lab' -Enabled $true
> ```
> **EXPECTED:** group + GMSA created.
> **VERIFY:** `Get-ADServiceAccount gmsa-sql-engine` returns the object.

> **Step 5.1.3 — Install the GMSA on both FCI nodes**
> **WHERE:** `sql-fci-1` + `sql-fci-2` (RDP), elevated PowerShell.
> **WHY:** caches the GMSA so the SQL service can run as it. (May need a reboot
> for the group membership to take effect in the node's Kerberos token first.)
> **WHAT:**
> ```powershell
> Install-WindowsFeature RSAT-AD-PowerShell
> Install-ADServiceAccount gmsa-sql-engine
> Test-ADServiceAccount gmsa-sql-engine     # -> True
> ```
> **EXPECTED:** `Test-ADServiceAccount` → `True` on both FCI nodes.
> **VERIFY:** `True` on each.

### 5.2 — SQL Server 2025 install + tooling + TLS cert

> **Step 5.2.1 — Install SQL Server 2025 Developer (standalone on the 2 replicas; skip on FCI nodes)**
> **WHERE:** `sql-ag-rep-1` + `sql-ag-rep-2` (RDP), elevated PowerShell.
> **WHY:** the replicas run standalone instances. The **FCI nodes get SQL via the
> FCI installer in §5.5** (not a standalone install), so do **not** install
> standalone there. Mount the ISO, run setup unattended.
> **WHAT (on each replica — mount the ISO at `D:` first):**
> ```powershell
> D:\setup.exe /Q /ACTION=Install /FEATURES=SQLEngine,FullText `
>   /INSTANCENAME=MSSQLSERVER /SECURITYMODE=SQL `
>   /SQLSVCACCOUNT="NT AUTHORITY\NETWORK SERVICE" `
>   /SQLSYSADMINACCOUNTS="NEXUS\Domain Admins" `
>   /AGTSVCACCOUNT="NT AUTHORITY\NETWORK SERVICE" `
>   /IACCEPTSQLSERVERLICENSETERMS /SUPPRESSPRIVACYSTATEMENTNOTICE
> ```
> **EXPECTED:** standalone SQL installs (Developer edition).
> **VERIFY:** `Get-Service MSSQLSERVER` → `Running`.

> **Step 5.2.2 — Install ODBC Driver 18 + `sqlcmd` (SQL 2025 dropped the bundle)**
> **WHERE:** all 4 nodes (RDP), elevated PowerShell.
> **WHY:** SQL 2025 no longer ships `sqlcmd`; install ODBC Driver 18 + the MSSQL
> command-line utilities, and use `sqlcmd -C` (trust the chain) thereafter.
> **WHAT:**
> ```powershell
> $msi = "$env:TEMP\msodbcsql18.msi"
> Invoke-WebRequest 'https://go.microsoft.com/fwlink/?linkid=2358430' -OutFile $msi -UseBasicParsing
> Start-Process msiexec -ArgumentList "/i `"$msi`" /qn IACCEPTMSODBCSQLLICENSETERMS=YES" -Wait
> $msi2 = "$env:TEMP\MsSqlCmdLnUtils.msi"
> Invoke-WebRequest 'https://go.microsoft.com/fwlink/?linkid=2230791' -OutFile $msi2 -UseBasicParsing
> Start-Process msiexec -ArgumentList "/i `"$msi2`" /qn IACCEPTMSSQLCMDLNUTILSLICENSETERMS=YES" -Wait
> ```
> **EXPECTED:** both MSIs install.
> **VERIFY:** `sqlcmd -?` runs (a new shell, so `sqlcmd` is on PATH).

> **Step 5.2.3 — Issue + install the unified per-node TLS cert (Vault HTTP API → PFX)**
> **WHERE:** issue on `vault-1`; install on each node (RDP).
> **WHY:** `ws2025` has no `openssl`, so issue the cert via Vault's HTTP API,
> assemble a PFX, and `Import-PfxCertificate`. The cert's SANs include the **node
> name + `sqlfci` + `sql-ag-listener`** so one cert validates for every identity
> the instance serves. Install the **Intermediate → `CA`** + **Root → `Root`** so
> strict TLS chains.
> **WHAT (on vault-1 — create the role + issue):**
> ```bash
> vault write pki_int/roles/sqlserver-server \
>   allowed_domains='nexus.lab,sql-fci-1,sql-fci-2,sql-ag-rep-1,sql-ag-rep-2,sqlfci,sql-ag-listener,localhost' \
>   allow_subdomains=true allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
> # per node — issue (CN=<host>.nexus.lab, SANs add sqlfci + listener + the node IP + .16/.17):
> vault write -format=json pki_int/issue/sqlserver-server \
>   common_name="<host>.nexus.lab" \
>   alt_names="<host>,sqlfci,sqlfci.nexus.lab,sql-ag-listener,sql-ag-listener.nexus.lab" \
>   ip_sans="<node-ip>,192.168.70.16,192.168.70.17" ttl=2160h
> ```
> **WHAT (on each node — assemble PFX from the PEM via .NET, import):**
> ```powershell
> # paste the leaf + key PEM into $leafPem / $keyPem, the intermediate + root into files
> $cert = [System.Security.Cryptography.X509Certificates.X509Certificate2]::CreateFromPem($leafPem, $keyPem)
> $pfx  = $cert.Export([System.Security.Cryptography.X509Certificates.X509ContentType]::Pfx, 'transient-pw')
> [IO.File]::WriteAllBytes("$env:TEMP\sql.pfx", $pfx)
> Import-PfxCertificate -FilePath "$env:TEMP\sql.pfx" -CertStoreLocation Cert:\LocalMachine\My -Password (ConvertTo-SecureString 'transient-pw' -AsPlainText -Force) -Exportable
> Import-Certificate -FilePath "$env:TEMP\intermediate.pem" -CertStoreLocation Cert:\LocalMachine\CA
> Import-Certificate -FilePath "$env:TEMP\root.pem" -CertStoreLocation Cert:\LocalMachine\Root
> ```
> **EXPECTED:** the leaf is in `LocalMachine\My` with its private key; chain builds.
> **VERIFY:** `Get-ChildItem Cert:\LocalMachine\My | ? Subject -match '<host>'` shows
> `HasPrivateKey: True`. (The cert is bound to the SQL wire in §5.7.)

### 5.3 — Attach the iSCSI shared LUN (FCI nodes)

> **Step 5.3.1 — Connect both FCI nodes to the LUN; format `S:` on `sql-fci-1` only**
> **WHERE:** `sql-fci-1` + `sql-fci-2` (RDP), elevated PowerShell.
> **WHY:** both FCI nodes mount the same LUN from `nexus-gateway`. **Only
> `sql-fci-1` formats it** (NTFS 64K, GPT → `S:`); `sql-fci-2` just attaches (a
> format there would destroy node-1's data).
> **WHAT (on BOTH FCI nodes — CHAP secret from Vault KV):**
> ```powershell
> Set-Service msiscsi -StartupType Automatic ; Start-Service msiscsi
> $chap = '<iSCSI CHAP secret from Vault KV nexus/oltp/sqlserver/iscsi-chap-secret>'
> New-IscsiTargetPortal -TargetPortalAddress 192.168.70.1 -InitiatorPortalAddress <this node's VMnet11 IP>
> Connect-IscsiTarget -NodeAddress 'iqn.2026-05.local.nexus:sql-fci.lun1' `
>   -AuthenticationType ONEWAYCHAP -ChapUsername sql-fci-initiator -ChapSecret $chap -IsPersistent $true
> ```
> **WHAT (on `sql-fci-1` ONLY — format the new disk):**
> ```powershell
> $disk = Get-Disk | Where-Object { $_.BusType -eq 'iSCSI' -and $_.PartitionStyle -eq 'RAW' } | Select -First 1
> Initialize-Disk -Number $disk.Number -PartitionStyle GPT
> New-Partition -DiskNumber $disk.Number -UseMaximumSize -DriveLetter S
> Format-Volume -DriveLetter S -FileSystem NTFS -AllocationUnitSize 65536 -NewFileSystemLabel 'SQLDATA' -Confirm:$false
> ```
> **EXPECTED:** both nodes see the LUN; `sql-fci-1` has `S:`.
> **VERIFY:** `Get-Disk | ? BusType -eq iSCSI` on both; `Get-Volume -DriveLetter S` on node-1.

### 5.4 — Create the WSFC cluster

> **Step 5.4.1 — `New-Cluster sql-fci-cluster` (4 nodes, IP `.15`) + add the disk**
> **WHERE:** `sql-fci-1` (RDP as `NEXUS\nexusadmin`), elevated PowerShell.
> **WHY:** the failover-clustering substrate. All 4 nodes; static cluster IP `.15`;
> the iSCSI LUN becomes a clustered Physical Disk. (Skip `Test-Cluster`'s storage
> validation — the iSCSI single-LUN lab trips its checks; the cluster forms fine.)
> **WHAT:**
> ```powershell
> Install-WindowsFeature Failover-Clustering -IncludeManagementTools   # on all 4 nodes first
> Import-Module FailoverClusters
> New-Cluster -Name sql-fci-cluster -Node sql-fci-1,sql-fci-2,sql-ag-rep-1,sql-ag-rep-2 `
>   -StaticAddress 192.168.70.15 -NoStorage
> # add the iSCSI LUN as a clustered disk, then make sure it's S: on the owner:
> Get-ClusterAvailableDisk | Add-ClusterDisk
> ```
> **EXPECTED:** the 4-node cluster forms; the disk is added to Available Storage.
> **VERIFY:** `Get-Cluster` → `sql-fci-cluster`; `Get-ClusterNode` → 4 `Up`;
> `Get-ClusterResource | ? ResourceType -eq 'Physical Disk'` shows the LUN. If the
> drive letter got stripped to `E:`, move the storage group to `sql-fci-1` and
> `Set-Partition … -NewDriveLetter S` (transient #32).

### 5.5 — Install the Failover Cluster Instance

> **Step 5.5.1 — `InstallFailoverCluster` on `sql-fci-1`**
> **WHERE:** `sql-fci-1` (RDP), elevated PowerShell (SQL ISO mounted at `D:`).
> **WHY:** creates the FCI resource group in WSFC — virtual name **`sqlfci`**, IP
> **`.16`**, data on the clustered `S:`. Service account = the **GMSA**.
> **Pre-create `S:\SQLData`** (setup needs the `/INSTALLSQLDATADIR` base to exist —
> the fresh LUN format leaves `S:` empty; transient #31).
> **WHAT:**
> ```powershell
> New-Item -ItemType Directory -Force S:\SQLData | Out-Null
> D:\setup.exe /Q /ACTION=InstallFailoverCluster /FEATURES=SQLEngine,FullText `
>   /INSTANCENAME=MSSQLSERVER `
>   /FAILOVERCLUSTERGROUP="SQL Server (MSSQLSERVER)" `
>   /FAILOVERCLUSTERNETWORKNAME=sqlfci `
>   /FAILOVERCLUSTERIPADDRESSES="IPv4;192.168.70.16;Cluster Network 1;255.255.255.0" `
>   /INSTALLSQLDATADIR=S:\SQLData `
>   /SQLSVCACCOUNT="NEXUS\gmsa-sql-engine$" /SQLSVCPASSWORD="" `
>   /SQLSYSADMINACCOUNTS="NEXUS\Domain Admins" `
>   /SKIPRULES=Cluster_VerifyForErrors `
>   /IACCEPTSQLSERVERLICENSETERMS /SUPPRESSPRIVACYSTATEMENTNOTICE
> ```
> **EXPECTED:** setup exits `0` (or `3010` reboot-pending). The FCI group appears
> in WSFC.
> **VERIFY:** `Get-ClusterGroup "SQL Server (MSSQLSERVER)"` is `Online`;
> `sqlcmd -S sqlfci -C -Q "SELECT @@SERVERNAME"` → `SQLFCI`.

> **Step 5.5.2 — `AddNode` on `sql-fci-2`**
> **WHERE:** `sql-fci-2` (RDP), elevated PowerShell (ISO at `D:`).
> **WHY:** make node-2 a possible owner of the FCI.
> **WHAT:**
> ```powershell
> D:\setup.exe /Q /ACTION=AddNode /INSTANCENAME=MSSQLSERVER `
>   /SQLSVCACCOUNT="NEXUS\gmsa-sql-engine$" /SQLSVCPASSWORD="" `
>   /SKIPRULES=Cluster_VerifyForErrors `
>   /IACCEPTSQLSERVERLICENSETERMS /SUPPRESSPRIVACYSTATEMENTNOTICE
> ```
> **EXPECTED:** node-2 joins the FCI as a possible owner.
> **VERIFY:** `Get-ClusterOwnerNode "SQL Server (MSSQLSERVER)"` lists both
> `sql-fci-1` + `sql-fci-2`; `Move-ClusterGroup "SQL Server (MSSQLSERVER)" -Node sql-fci-2`
> then re-query `@@SERVERNAME` → still `SQLFCI` (failover works). Move it back.

### 5.6 — Create the Always-On Availability Group

> **Step 5.6.1 — Enable AG (Hadr) + fix `@@SERVERNAME` on the replicas**
> **WHERE:** each replica (RDP), elevated PowerShell.
> **WHY:** Always-On must be enabled per instance + a restart. The Packer bake's
> `@@SERVERNAME` is stuck at the bake hostname on the replicas — fix it
> (`sp_dropserver`/`sp_addserver`) or AG joins fail (transient #28n).
> **WHAT (on `sql-ag-rep-1` + `sql-ag-rep-2`):**
> ```powershell
> Enable-SqlAlwaysOn -ServerInstance . -Force    # enables Hadr + restarts SQL
> sqlcmd -S . -C -Q "IF @@SERVERNAME <> CONVERT(sysname, SERVERPROPERTY('MachineName')) BEGIN EXEC sp_dropserver @@SERVERNAME; EXEC sp_addserver @@SERVERNAME=N'$(hostname)', @local='local'; END"
> # then restart SQL once so the rename takes effect
> Restart-Service MSSQLSERVER -Force
> ```
> Enable Hadr on the FCI too (`Enable-SqlAlwaysOn -ServerInstance sqlfci -Force` —
> on the active FCI owner; it restarts the clustered instance via the cluster).
> **EXPECTED:** Hadr enabled on all 3 AG participants; replica `@@SERVERNAME`
> matches the machine name.
> **VERIFY:** `sqlcmd -S . -C -Q "SELECT SERVERPROPERTY('IsHadrEnabled')"` → `1`.

> **Step 5.6.2 — Certificate endpoints + create the AG with manual seeding**
> **WHERE:** the FCI (`sqlcmd -S sqlfci`) + each replica, RDP PowerShell.
> **WHY:** AG endpoint auth is **certificate-based** (ADR-0027): each participant
> makes a `Hadr_endpoint` cert, exchanges public `.cer`s, maps logins, grants
> CONNECT. Then create `nexus-ag` with the FCI primary + the 2 async replicas, and
> **manually seed** `nexus_demo` (full backup → `RESTORE … WITH MOVE, NORECOVERY`
> → join).
> **WHAT (outline — run the T-SQL on each participant via `sqlcmd -S <inst> -C`):**
> ```sql
> -- on EACH of the 3 participants: master key + endpoint cert + Hadr_endpoint
> CREATE MASTER KEY ENCRYPTION BY PASSWORD = '<mk-pw>';
> CREATE CERTIFICATE Hadr_endpoint_cert WITH SUBJECT = 'AG endpoint <host>';
> BACKUP CERTIFICATE Hadr_endpoint_cert TO FILE = 'C:\Windows\Temp\<host>_endpoint.cer';
> CREATE ENDPOINT Hadr_endpoint STATE = STARTED
>   AS TCP (LISTENER_PORT = 5022)
>   FOR DATABASE_MIRRORING (ROLE = ALL, AUTHENTICATION = CERTIFICATE Hadr_endpoint_cert, ENCRYPTION = REQUIRED ALGORITHM AES);
> -- exchange .cer files (scp via the build host), then on each node import the
> -- OTHER two participants' certs + a login mapped to each + GRANT CONNECT:
> CREATE LOGIN <peer>_login FROM CERTIFICATE <peer>_cert ... ; GRANT CONNECT ON ENDPOINT::Hadr_endpoint TO <peer>_login;
> ```
> ```sql
> -- on the FCI primary (sqlfci): create the demo DB + a FULL backup FIRST (Msg 1475)
> CREATE DATABASE nexus_demo;
> BACKUP DATABASE nexus_demo TO DISK = 'S:\SQLData\nexus_demo.bak' WITH INIT;
> BACKUP LOG nexus_demo TO DISK = 'S:\SQLData\nexus_demo.trn' WITH INIT;
> CREATE AVAILABILITY GROUP nexus_ag
>   FOR DATABASE nexus_demo
>   REPLICA ON 'SQLFCI'        WITH (ENDPOINT_URL='TCP://sqlfci.nexus.lab:5022',        AVAILABILITY_MODE=SYNCHRONOUS_COMMIT, FAILOVER_MODE=MANUAL),
>           'SQL-AG-REP-1' WITH (ENDPOINT_URL='TCP://sql-ag-rep-1.nexus.lab:5022', AVAILABILITY_MODE=ASYNCHRONOUS_COMMIT, FAILOVER_MODE=MANUAL),
>           'SQL-AG-REP-2' WITH (ENDPOINT_URL='TCP://sql-ag-rep-2.nexus.lab:5022', AVAILABILITY_MODE=ASYNCHRONOUS_COMMIT, FAILOVER_MODE=MANUAL);
> ```
> ```sql
> -- on EACH replica: join the AG + manually seed (FCI DB on S:\, replica on C:\)
> ALTER AVAILABILITY GROUP nexus_ag JOIN;
> -- copy the .bak/.trn from the FCI, then:
> RESTORE DATABASE nexus_demo FROM DISK='C:\Temp\nexus_demo.bak' WITH MOVE 'nexus_demo' TO 'C:\Data\nexus_demo.mdf', MOVE 'nexus_demo_log' TO 'C:\Data\nexus_demo_log.ldf', NORECOVERY;
> RESTORE LOG nexus_demo FROM DISK='C:\Temp\nexus_demo.trn' WITH NORECOVERY;
> ALTER DATABASE nexus_demo SET HADR AVAILABILITY GROUP = nexus_ag;
> ```
> **EXPECTED:** the AG forms; both replicas `SYNCHRONIZING`/`HEALTHY`.
> **VERIFY:** on the primary,
> `sqlcmd -S sqlfci -C -Q "SELECT ar.replica_server_name, drs.synchronization_state_desc FROM sys.dm_hadr_database_replica_states drs JOIN sys.availability_replicas ar ON drs.replica_id=ar.replica_id"`
> shows all 3 participants synchronizing.

### 5.7 — AG Listener + SPNs + cert binding

> **Step 5.7.1 — Add the Listener (`.17`), register SPNs, bind the TLS cert**
> **WHERE:** the FCI primary (`sqlcmd -S sqlfci`) + a DC (SPNs) + each node (cert).
> **WHY:** the Listener is the client front door (WSFC migrates `.17` with the AG
> primary). It needs **SPNs** on the GMSA (or remote `-E` → NTLM → ANONYMOUS
> LOGON), and each node binds its unified cert to the SQL wire so the listener
> name validates under strict TLS.
> **WHAT:**
> ```sql
> -- on the primary:
> ALTER AVAILABILITY GROUP nexus_ag
>   ADD LISTENER 'sql-ag-listener' (WITH IP (('192.168.70.17','255.255.255.0')), PORT = 1433);
> ```
> ```powershell
> # on a DC: register the virtual-name SPNs on the GMSA (SQL can't auto-register these)
> setspn -S MSSQLSvc/sqlfci.nexus.lab:1433 nexus\gmsa-sql-engine$
> setspn -S MSSQLSvc/sqlfci.nexus.lab nexus\gmsa-sql-engine$
> setspn -S MSSQLSvc/sql-ag-listener.nexus.lab:1433 nexus\gmsa-sql-engine$
> setspn -S MSSQLSvc/sql-ag-listener.nexus.lab nexus\gmsa-sql-engine$
> ```
> ```powershell
> # on EACH node: bind the per-node cert to the SQL wire (SQL 2025 = MSSQL17)
> $thumb = (Get-ChildItem Cert:\LocalMachine\My | ? Subject -match $env:COMPUTERNAME | Select -First 1).Thumbprint.ToLower()
> $key = 'HKLM:\SOFTWARE\Microsoft\Microsoft SQL Server\MSSQL17.MSSQLSERVER\MSSQLServer\SuperSocketNetLib'
> Set-ItemProperty $key -Name Certificate -Value $thumb
> # FCI nodes: cycle the SQL CLUSTER GROUP (a direct Restart-Service races the cluster):
> #   Stop-ClusterGroup "SQL Server (MSSQLSERVER)"; Start-ClusterGroup "SQL Server (MSSQLSERVER)"
> # replicas: Restart-Service MSSQLSERVER -Force
> ```
> **EXPECTED:** the Listener resource is Online; SPNs registered; certs bound.
> **VERIFY (from a domain client / the build host with DNS to the DC):**
> ```
> sqlcmd -S sql-ag-listener.nexus.lab -E -N -Q "SELECT @@SERVERNAME"
> ```
> → `SQLFCI` (Encrypt + strict cert validate succeeds, integrated auth via
> Kerberos resolves to the primary).

---

## 6. Validation — by-hand acceptance smoke

Mirrors the 56-check `smoke-0.G.7.ps1`, condensed. From a domain client / RDP.

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | WSFC 4 nodes up | `Get-ClusterNode` | 4 `Up`, NodeMajority quorum |
| 2 | FCI online | `Get-ClusterGroup "SQL Server (MSSQLSERVER)"` | `Online` |
| 3 | FCI virtual server | `sqlcmd -S sqlfci -C -Q "SELECT @@SERVERNAME"` | `SQLFCI` |
| 4 | FCI failover | `Move-ClusterGroup "SQL Server (MSSQLSERVER)" -Node sql-fci-2`; re-query | still `SQLFCI` (then move back) |
| 5 | iSCSI shared disk | `Get-ClusterResource \| ? ResourceType -eq 'Physical Disk'` | Online on the FCI owner |
| 6 | GMSA identity | `Get-CimInstance Win32_Service -Filter "Name='MSSQLSERVER'"` on FCI | `StartName = NEXUS\gmsa-sql-engine$` |
| 7 | AG healthy | `sys.dm_hadr_database_replica_states` query (§5.6.2) | 3 participants synchronizing/healthy |
| 8 | Listener answers | `sqlcmd -S sql-ag-listener.nexus.lab -E -N -Q "SELECT @@SERVERNAME"` | `SQLFCI` (Kerberos + strict TLS) |
| 9 | Replicated DB | write to `nexus_demo` on primary; read on a (recovered) replica | value present |
| 10 | **AG failover** | `ALTER AVAILABILITY GROUP nexus_ag FAILOVER` to a sync replica; Listener re-query | listener follows the new primary |

**1–9 green ⇒ Guide 11 satisfied** (= the spirit of the 56/56 cold-rebuild gate).
10 is the AG failover proof.

---

## 7. Teardown / reset

```powershell
# On sql-fci-1: remove the AG + the cluster (destroys cluster roles).
Invoke-Sqlcmd -ServerInstance sqlfci -Query "DROP AVAILABILITY GROUP nexus_ag" -ErrorAction SilentlyContinue
Get-Cluster | Remove-Cluster -Force -CleanupAD
# then vmrun stop + deleteVM each of the 4 (Guide 00 §7).
```

> **Cold-rebuild prerequisite (mirrors `scripts/cold-rebuild-prereqs.ps1`):** a
> destroy leaves a footprint **outside** the VMs that breaks a from-zero re-apply.
> Before rebuilding, on `dc-nexus` remove the stale **CNO `sql-fci-cluster`** +
> the FCI/Listener **VCOs** (`sqlfci`, `sql-ag-listener`) + the 4 node computer
> accounts + their DNS A records; and on `nexus-gateway` **wipe the iSCSI LUN
> backing file** `/srv/iscsi/sql-fci-shared.img` (else the fresh FCI install
> collides with the old NTFS + DBs — the attach step only inits a *RAW* disk).

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| WSFC/FCI/AG cmdlet: "Access is denied" | running as **local** `nexusadmin` (no domain TGT) — the automated lab's SSH problem | **RDP in as `NEXUS\nexusadmin`** (domain context) — the whole reason this guide uses RDP, not SSH. |
| `InstallFailoverCluster` fails -2067660798 "Root of path S:\SQLData… does not exist" | the fresh LUN format leaves `S:` empty; setup needs the `/INSTALLSQLDATADIR` base to pre-exist | `New-Item -ItemType Directory S:\SQLData` before setup (§5.5.1, transient #31). |
| iSCSI disk comes back as `E:` / storage group on the wrong node | `Add-ClusterDisk` strips the drive letter | move the storage group to `sql-fci-1` + `Set-Partition … -NewDriveLetter S` (§5.4.1, transient #32). |
| `ALTER AVAILABILITY GROUP ADD DATABASE` fails Msg 1475 | a brand-new DB has no full-backup LSN baseline | take a **full backup first**, then ADD DATABASE, then a log backup (§5.6.2, transient #33). |
| Automatic AG seeding never completes | FCI DBs on `S:\` vs replicas on `C:\` defeats auto-seed | use **manual seeding** (backup → `RESTORE WITH MOVE, NORECOVERY` → join) (§5.6.2). |
| AG join fails / `@@SERVERNAME` wrong on a replica | the Packer bake hostname is stuck | `sp_dropserver`/`sp_addserver` + restart (§5.6.1, transient #28n). |
| Remote `sqlcmd -E` to the listener → `ANONYMOUS LOGON` / login failed | missing SPNs → Kerberos falls to NTLM | register `MSSQLSvc/{sqlfci,sql-ag-listener}.nexus.lab[:1433]` on the GMSA (§5.7.1). |
| `sqlcmd` not found / ODBC cert errors | SQL 2025 dropped bundled `sqlcmd`; ODBC18 validates the chain | install ODBC18 + CmdLineUtils; use `sqlcmd -C` (§5.2.2). |
| Strict TLS (`-N`) fails to the listener | cert not bound, or bound under the wrong instance key | bind the per-node cert under **`MSSQL17.MSSQLSERVER`** (SQL 2025, not MSSQL16); cycle the SQL **cluster group** on the FCI (not `Restart-Service`) (§5.7.1, #29p/#29q). |
| `Test-ADServiceAccount gmsa-sql-engine` → False | node not yet in `nexus-sql-cluster-members`'s effective token | reboot the node so the group membership lands in its Kerberos token, then `Install-ADServiceAccount` (§5.1.3). |

---

## 9. Production tuning — SQL Server (Windows)

> **Everything in this section is *beyond the lab replica*.** The §5 build installs SQL
> Server 2025 with **stock defaults** — no `max server memory` cap (the instance grabs all
> RAM), MAXDOP `0`, single-file tempdb, no **Lock Pages in Memory** / **Instant File
> Initialization** grants, Windows on the *Balanced* power plan — because the lab nodes are
> modest (FCI `16 GB`, replicas `12 GB`) and a 1:1 replay must not diverge from what the
> automated `nexus-infra-oltp` overlays render. This section is what you would set on a
> **production** SQL Server FCI + AG deployment and *why*. **Do not apply these to the lab
> VMs blindly.** Because this is a **Windows** tier, the OS layer below is the *Windows*
> equivalent of Guide 00 §9's Linux tuning — **do not link there for the OS knobs**; the
> mechanisms (power plan, page file, User Rights Assignment) are Windows-native.

> **The FCI vs. AG-replica split matters throughout.** The FCI is **one clustered instance
> with a single set of system databases on the shared `S:`** — so its `sp_configure`
> settings (memory, MAXDOP, cost threshold, ad-hoc, backup compression) live in the shared
> `master` and are **automatically identical** on whichever FCI node owns it. The two AG
> replicas are **independent standalone instances** — every `sp_configure` value must be set
> on **each** of them separately to match the FCI. And the **OS-layer** grants (LPIM, IFI,
> power plan, page file, node-local tempdb paths) are **per-Windows-node** and are **not**
> shared by the cluster — they must be applied on **all four** nodes. §9.6 is the checklist.

### 9.1 Windows OS layer (per node — all four)

These are host-level and do **not** travel with the clustered instance; apply on each of
`sql-fci-1`, `sql-fci-2`, `sql-ag-rep-1`, `sql-ag-rep-2`.

```powershell
# PRODUCTION — not applied in the lab. RDP as NEXUS\nexusadmin, elevated PowerShell.

# --- High Performance power plan (SCHEME_MIN) ---
powercfg /setactive SCHEME_MIN
powercfg /getactivescheme          # VERIFY -> "High performance"

# --- Fixed page file on a non-data volume (disable automatic management) ---
$cs = Get-CimInstance Win32_ComputerSystem
if ($cs.AutomaticManagedPagefile) {
    $cs | Set-CimInstance -Property @{ AutomaticManagedPagefile = $false }
}
# set an explicit, fixed page file (Initial = Maximum avoids runtime resizing)
Set-CimInstance -Query "SELECT * FROM Win32_PageFileSetting WHERE Name='C:\\pagefile.sys'" `
    -Property @{ InitialSize = 16384; MaximumSize = 16384 }   # 16 GB; reboot to apply
```

Lock Pages in Memory (**SeLockMemoryPrivilege**) and Instant File Initialization
(**SeManageVolumePrivilege**) are **User Rights Assignments** granted to the *SQL service
account* — which is **`NEXUS\gmsa-sql-engine$`** on the FCI nodes but **`NT AUTHORITY\NETWORK
SERVICE`** on the replicas. Grant via `secpol.msc` (**Local Policies → User Rights
Assignment**) or script it with `secedit` (native; no Resource-Kit `ntrights` needed):

```powershell
# PRODUCTION — grant LPIM + IFI to THIS node's SQL service account. Run per node,
# substituting the correct account (GMSA on FCI nodes, NETWORK SERVICE on replicas).
$acct = 'NEXUS\gmsa-sql-engine$'    # replicas: 'NT AUTHORITY\NETWORK SERVICE'
$sid  = (New-Object System.Security.Principal.NTAccount($acct)
        ).Translate([System.Security.Principal.SecurityIdentifier]).Value

$inf = "$env:TEMP\lpim-ifi.inf"; $db = "$env:TEMP\lpim-ifi.sdb"
secedit /export /cfg "$env:TEMP\cur.cfg" /areas USER_RIGHTS | Out-Null
# merge *$sid onto the two privilege lines (create them if absent), then re-apply:
@"
[Unicode]
Unicode=yes
[Privilege Rights]
SeLockMemoryPrivilege = *$sid
SeManageVolumePrivilege = *$sid
[Version]
signature="`$CHICAGO`$"
Revision=1
"@ | Set-Content $inf -Encoding Unicode
secedit /configure /db $db /cfg $inf /areas USER_RIGHTS
# NOTE: /configure MERGES rights additively; verify existing holders aren't dropped.
```

> IFI can alternatively be granted **at install time** with `/SQLSVCINSTANTFILEINIT="True"`
> on the `setup.exe` line (§5.2.1 / §5.5.1) — the by-hand equivalent of the grant above.
> Both LPIM and IFI take effect on the **next SQL service (re)start**; on the FCI, cycle the
> **cluster group** (`Stop-ClusterGroup`/`Start-ClusterGroup "SQL Server (MSSQLSERVER)"`), not
> `Restart-Service` (§5.7.1).

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| **Lock Pages in Memory** (`SeLockMemoryPrivilege`, per node) | **⚠️ granted** to the SQL service account | not granted | Stops Windows trimming SQL's working set: under memory pressure the OS can page the entire buffer pool to disk, causing sudden latency cliffs and `A significant part of sql server process memory has been paged out` alerts. Pair with a **capped** `max server memory` (§9.2) so LPIM can't starve the OS. |
| **Instant File Initialization** (`SeManageVolumePrivilege`, per node) | **⚠️ granted** to the SQL service account | not granted | Skips zero-filling **data** files on create/grow/restore — a 100 GB restore or autogrow goes from minutes of zeroing to near-instant. (Does not apply to log files, which are always zeroed.) |
| Power plan | **High Performance** (`powercfg /setactive SCHEME_MIN`) | unset (*Balanced*) | *Balanced* parks cores and scales CPU frequency down; SQL latency-sensitive workloads see jitter and lower throughput. High Performance pins max frequency. |
| Page file | **fixed** Initial = Maximum, on a non-data volume | unset (system-managed) | A resizing/fragmented page file adds stalls; with LPIM the buffer pool is never paged, so a modest fixed file (for crash dumps + OS) is enough — but leave one, don't disable it. |
| NTFS allocation unit (data/log volumes) | **64 KB** | **PRESENT** — `S:` formatted `-AllocationUnitSize 65536` (§5.3.1) | 64 KB matches SQL's 64 KB extent (8×8 KB pages) → fewer I/O ops per extent. Already correct in the lab for the shared LUN; apply the same to any local data/log volume on the replicas. |

### 9.2 Server memory — `max server memory` + `min server memory`

⚠️ **This is the single most important FCI setting.** SQL's default `max server memory` is
effectively unlimited, so the instance consumes all RAM and leaves nothing for Windows, the
cluster service, or filesystem cache — and on **failover the instance must fit the passive
node too**. Leave the OS **~10–20 %, or ≥ 4 GB**, whichever is larger. `min server memory`
stops Windows reclaiming SQL's pool back down after a spike; with **LPIM** set (§9.1) it also
defines the floor the pinned pages defend.

```sql
-- PRODUCTION — not applied in the lab. Run once on the FCI (shared master → both nodes);
-- run again, independently, on EACH replica with that node's own sizing.
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;

-- FCI nodes are 16 GB → cap SQL at ~12 GB, leaving ~4 GB for Windows + cluster:
EXEC sp_configure 'max server memory (MB)', 12288; RECONFIGURE;
EXEC sp_configure 'min server memory (MB)',  4096; RECONFIGURE;
-- On the 12 GB AG replicas use e.g. max 9216 / min 3072 instead.
```

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `max server memory (MB)` | RAM − max(≈15 %, 4 GB) — FCI 16 GB → **`12288`**; replica 12 GB → **`9216`** | unset (`2147483647` → takes all RAM) | **⚠️** Uncapped, SQL starves Windows/WSFC → paging, cluster-heartbeat timeouts, hard failovers. **With FCI the cap must also fit the passive node**; with **LPIM** an uncapped instance can lock so much RAM the OS can't function. |
| `min server memory (MB)` | ≈ 25 % of the max (FCI **`4096`**) | unset (`0`) | Prevents Windows clawing the buffer pool below a useful floor after a memory-pressure spike, avoiding a cold cache and a latency spike while SQL re-reads from disk. |

### 9.3 Parallelism — MAXDOP + cost threshold

```sql
-- PRODUCTION — FCI: set once (shared master). Replicas: set on each to MATCH.
EXEC sp_configure 'max degree of parallelism', 4;  RECONFIGURE;   -- see sizing below
EXEC sp_configure 'cost threshold for parallelism', 50; RECONFIGURE;
```

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `max degree of parallelism` | **= logical cores per NUMA node, capped at 8** — these nodes are 4 vCPU / 1 NUMA node → **`4`** | unset (`0` = use all cores) | `0` lets one query fan out across *every* core, starving concurrent OLTP requests and inflating `CXPACKET`/`CXCONSUMER` waits. Cap it to the per-NUMA core count (≤ 8) so parallel plans stay within one memory node. |
| `cost threshold for parallelism` | **`50`** | unset (`5`) | The default `5` is a 1990s value — trivial queries go parallel, paying thread-coordination overhead that makes them *slower* on an OLTP box. Raising to `50` keeps small queries serial and reserves parallelism for genuinely expensive plans. |

### 9.4 tempdb — multiple equal files on fast storage

tempdb allocation contention (PFS/GAM/SGAM latch waits) is a classic OLTP bottleneck; the
fix is **multiple equal-sized data files** so allocations round-robin across them.

```sql
-- PRODUCTION — one data file per logical core, up to 8. These nodes = 4 vCPU → 4 files,
-- all EQUAL size with EQUAL autogrow (unequal sizes defeat proportional-fill round-robin).
ALTER DATABASE tempdb MODIFY FILE (NAME = tempdev, SIZE = 1024MB, FILEGROWTH = 256MB);
ALTER DATABASE tempdb ADD FILE (NAME = temp2, FILENAME = 'T:\tempdb\tempdb2.ndf', SIZE = 1024MB, FILEGROWTH = 256MB);
ALTER DATABASE tempdb ADD FILE (NAME = temp3, FILENAME = 'T:\tempdb\tempdb3.ndf', SIZE = 1024MB, FILEGROWTH = 256MB);
ALTER DATABASE tempdb ADD FILE (NAME = temp4, FILENAME = 'T:\tempdb\tempdb4.ndf', SIZE = 1024MB, FILEGROWTH = 256MB);
-- restart the instance for the file layout to take effect (FCI: cycle the cluster group).
```

> The by-hand equivalent at install is the `setup.exe` tempdb switches:
> `/SQLTEMPDBFILECOUNT=4 /SQLTEMPDBFILESIZE=1024 /SQLTEMPDBFILEGROWTH=256`
> (and `/SQLTEMPDBDIR=T:\tempdb`). On the **FCI**, put tempdb on **node-local fast disk**
> (SQL 2016+ supports FCI tempdb on local storage — it need not sit on the shared `S:`), but
> the chosen path (`T:\tempdb`) **must exist on both FCI nodes** or the instance won't start
> after a failover.

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| tempdb data-file count | **`min(vCPU, 8)`** = **`4`** here, all equal-sized | unset (setup default layout) | One tempdb file serialises allocation-page latches under concurrent temp-table/sort/spill load → `PAGELATCH_UP` contention. Multiple equal files spread it. Equal *size + growth* is required for proportional fill to round-robin evenly. |
| tempdb autogrow | fixed **`256MB`** (not the `%` default), pre-sized large | unset (small file, `%` growth) | Percentage growth grows ever-larger chunks and, without IFI, stalls the instance while zeroing; a fixed MB growth is predictable. Pre-size so growth is rare. |
| tempdb placement | **local fast NVMe/SSD** per node | shared `S:` (default) | tempdb is write-heavy and disposable; keeping it off the replicated/shared data path removes it from the FCI storage bottleneck. |

### 9.5 Instance options — ad-hoc plans, backup compression, backup-log noise

```sql
-- PRODUCTION — FCI: set once (shared master). Replicas: set on each to MATCH.
EXEC sp_configure 'optimize for ad hoc workloads', 1; RECONFIGURE;
EXEC sp_configure 'backup compression default',    1; RECONFIGURE;
```

Trace flag **3226** is a **startup** parameter, not `sp_configure`. Set it via **SQL Server
Configuration Manager → SQL Server (MSSQLSERVER) → Properties → Startup Parameters → add
`-T3226`** (persists across FCI failover because it is the clustered instance's own
configuration), then restart the instance (FCI: cycle the cluster group).

| Setting | Production value | Lab value (§5) | Why it matters |
|---|---|---|---|
| `optimize for ad hoc workloads` | **`1`** | unset (`0`) | Stores only a small *plan stub* on a query's first execution instead of a full plan; stops single-use ad-hoc queries bloating the plan cache and evicting reusable plans. Safe, universally recommended. |
| `backup compression default` | **`1`** | unset (`0`) | Compresses backups by default → smaller files, **faster** backup/restore (less I/O), and faster AG manual-seed transfers (§5.6.2). Developer edition includes it. |
| Trace flag **`3226`** (startup `-T3226`) | **enabled** | not set | Suppresses the *successful backup* entry the engine writes to the errorlog on **every** backup; with frequent log backups this spam drowns the errorlog and hides real errors. |

### 9.6 Consistency across all nodes — the production failover trap

A mismatched failover **target** is a real production incident: the workload lands on a node
whose tuning differs, and behaviour changes silently. Enforce this matrix before go-live.

| Setting | Scope | FCI (`sql-fci-1/2`) | AG replicas (`sql-ag-rep-1/2`) |
|---|---|---|---|
| `max server memory` / `min server memory` | **`sp_configure`** | shared `master` → auto-identical across both FCI nodes; still verify each node's **RAM is equal** so the cap fits after failover | ⚠️ **set on each replica independently** — and size to that replica's RAM (12 GB, not 16) |
| MAXDOP + cost threshold | **`sp_configure`** | auto-identical (shared `master`) | ⚠️ **set on each replica to match the FCI** |
| `optimize for ad hoc` / `backup compression` / TF `3226` | `sp_configure` + startup param | auto-identical / `-T3226` on the clustered instance | ⚠️ **set on each replica to match** |
| tempdb file count + sizes + **path** | per-node (files are node-local) | ⚠️ tempdb path (e.g. `T:\tempdb`) **must exist on both** FCI nodes or startup fails post-failover | ⚠️ configure the same file count/sizing on each replica |
| **LPIM** + **IFI** (User Rights Assignment) | per-Windows-node | ⚠️ **grant on both** FCI nodes to `gmsa-sql-engine$` — the grant does **not** replicate with the cluster | ⚠️ **grant on both** replicas to `NETWORK SERVICE` |
| Power plan / page file | per-Windows-node | ⚠️ set on **both** | ⚠️ set on **both** |

> **Rule of thumb:** anything in `sp_configure` is *shared* on the FCI (one instance, one
> `master`) but *not* on the AG replicas (independent instances); anything at the **Windows
> or file-system layer** (LPIM, IFI, power plan, page file, tempdb paths) is **per-node
> everywhere** and is the most common thing forgotten on the *passive* FCI node until a
> failover exposes it. Verify with the §6 smoke plus a `sp_configure` diff between the FCI
> and each replica.

---

### Cross-references

- **0.G.7 architecture + transients:** memory `project_nexus_infra_oltp_0g7_phase`; ADR-0026 (iSCSI), ADR-0027 (AG cert endpoints), ADR-0025 (Listener as LB-tier HA)
- **Network/VIP canon:** `nexus-platform-plan/docs/infra/network.md` (SQL `.11`–`.17`)
- **Automated equivalents:** `nexus-infra-oltp/packer/oltp-sqlserver-node/` + `terraform/envs/oltp-sqlserver/role-overlay-*.tf`
- **iSCSI target consumed:** [`01-foundation-nexus-gateway.md`](./01-foundation-nexus-gateway.md) §5.9 · **GMSA/KDS:** [`02-foundation-ad-ds-forest.md`](./02-foundation-ad-ds-forest.md) §5.6
- **Previous guide:** [`10-oltp-patroni-postgresql-ha.md`](./10-oltp-patroni-postgresql-ha.md)
- **Next guide:** Guide 12 — OLTP · MongoDB sharded (3 config RS + 2 shard RS×3 + 2 mongos). See [`INDEX.md`](../INDEX.md).
