# Guide 04 — Foundation · Vault PKI + LDAPS auth + `secrets/ldap` rotation

> **Mirrors:** the `security`-env overlays `role-overlay-vault-pki-*.tf`
> (mount → root → intermediate → roles → rotate → distribute), the LDAP set
> `role-overlay-vault-{ldap-policies,ldaps-cert,ldap-auth,ldap-secret-engine,ldap-rotate-role}.tf`,
> and the foundation-env AD-side `role-overlay-dc-vault-ad-bind*.tf`. Where the
> automated lab drives every step over SSH with base64 transit, this guide runs
> the `vault` CLI **on `vault-1`** and the AD work **on `dc-nexus` over RDP**.

---

## 1. Overview & purpose

Guide 03 left the Vault cluster running on **self-signed bootstrap TLS** and with
no directory integration. This guide closes both gaps and establishes the
**pattern every later tier reuses**:

1. **A PKI hierarchy in Vault** — an internal **root CA** (`pki/`) that signs one
   **intermediate CA** (`pki_int/`); all leaf certs issue from the intermediate.
   We then **re-issue `vault-1/2/3`'s own listener certs from PKI** (retiring the
   self-signed bootstrap certs) and **distribute the root CA** to the build host
   and every node — after which `VAULT_SKIP_VERIFY` goes away.
2. **LDAPS auth against the AD forest** — Vault's `auth/ldap` method, over
   **LDAPS** (TCP/636, with a PKI-issued cert installed on `dc-nexus`), maps AD
   groups to Vault policies so humans `vault login -method=ldap`.
3. **`secrets/ldap` rotation** — the unified AD secrets engine: Vault **owns and
   rotates** the passwords of designated AD service accounts (static roles);
   consumers read the current password via `vault read ldap/static-cred/<role>`.
4. **The per-tier scaffolding pattern** — the reusable recipe (a `pki_int` role +
   a scoped Vault **policy** + an **AppRole** + **KV** secrets) that Guides 05–22
   each invoke to give their tier mTLS certs + credentials. Part D documents it
   with a worked `foundation` example.

**Why all this:** every secured tier from here on needs (a) certs from a *shared*
CA so mTLS validates fleet-wide, (b) machine identity via AppRole, and (c)
directory-backed human login. This guide builds the machinery once.

**Dependency:**
- **Guide 03** — the 3-node Vault cluster is up, unsealed, with KV-v2 (`nexus/`),
  userpass, and AppRole enabled; you have the root token + the cluster init JSON.
- **Guide 02** — the `nexus.lab` AD forest is up on `dc-nexus` (`.240`), with
  `LDAPServerIntegrity=1` and the OUs (`ServiceAccounts`, `Groups`) created.
- **Guide 01** — gateway DNS forwards `nexus.lab` → `dc-nexus`.

> **Reading note — LDAPS, not plain LDAP.** This AD environment rejects **all**
> plain-LDAP (`:389`) simple binds from non-Windows clients regardless of
> `LDAPServerIntegrity` (tested 2/1/0 — all fail "Strong Auth Required"). So
> everything Vault↔AD here is **LDAPS (`:636`)**, which satisfies AD's integrity
> requirement structurally via TLS. That's why §5.B installs a PKI cert on the DC
> *before* wiring auth. (`LDAPServerIntegrity=1` from Guide 02 is still useful for
> other tooling, but LDAPS is what makes Vault work.)

---

## 2. Component primer

- **PKI: root CA vs. intermediate CA.** A **certificate authority** signs certs
  that assert "this hostname is who it says." Standard design keeps the **root**
  offline-ish: it signs the **intermediate** exactly once, then is dormant; all
  day-to-day **leaf** certs issue from the **intermediate**. *Why two:* if the
  intermediate key is ever compromised you revoke + reissue it without burning
  the root (and every cert under it). *Otherwise:* a single CA issuing leaves
  directly (one compromise = full PKI rebuild).
- **Leaf cert + SANs.** The per-server cert a TLS listener presents. Its
  **Subject Alternative Names** list every hostname/IP a client may use to reach
  it — all of which must be in the SAN or validation fails. *Why we enumerate
  both IPs + names:* the lab reaches services by FQDN, short name, and raw IP.
- **`pki_int` role.** A named template on the intermediate mount that constrains
  what a leaf may contain (allowed domains, IP-SANs allowed, key type, TTL).
  Issuing is then `vault write pki_int/issue/<role> common_name=… ip_sans=…`.
- **LDAPS.** LDAP wrapped in TLS on `:636`. *Why:* AD requires channel integrity
  for binds + *mandates* it for password writes (`unicodePwd`); LDAPS provides
  both. The DC serves it once a cert with the right chain sits in its
  `LocalMachine\My` store and NTDS restarts. *Otherwise:* plain LDAP (rejected
  here) or StartTLS (more moving parts).
- **Vault `auth/ldap` (search-then-bind).** Vault binds as a **service account**
  (`svc-vault-ldap`), searches for the login user, then **rebinds as that user's
  DN** with the supplied password to authenticate; group memberships then map to
  policies. *Why a bind account:* Vault needs to find users before it knows their
  DN. **Critical:** leave `upndomain=""` — setting it rewrites `{{.Username}}` in
  the user filter to `user@domain`, which AD's `sAMAccountName` never matches
  (Vault #27276).
- **Vault `secrets/ldap` + static roles.** The engine that *manages* AD account
  passwords: a **static role** binds an AD account to Vault, which rotates its
  password on a schedule and serves the current value via
  `ldap/static-cred/<role>`. *Why:* no human ever knows or hard-codes a service
  password again. **Fresh-onboarding gotcha:** Vault's default is to "import" the
  account by binding *as it* with its *current* password — which we don't know —
  so we set `skip_static_role_import_rotation=true` and trigger the first
  rotation explicitly with `vault write -force ldap/rotate-role/<role>` (the bind
  account's *Reset Password* right does the write).
- **AppRole + policy + KV (the scaffolding pattern).** **AppRole** is the machine
  login method: a node presents a `RoleID` + `SecretID` and gets a token carrying
  a **policy** that scopes exactly which paths it may touch (its KV prefix, its
  `pki_int/issue/<role>`). **KV-v2** holds the tier's seeded secrets. *Why:* every
  tier authenticates the same way, with least privilege. Part D is the recipe.
- **The CA bundle + retiring skip-verify.** Once the root CA is written to the
  build host (`VAULT_CACERT`) and into each node's system trust store, clients
  validate Vault's now-PKI-issued listener certs properly and `VAULT_SKIP_VERIFY`
  is no longer needed.

---

## 3. Prerequisites

| # | Requirement | One-command verify |
|---|---|---|
| 1 | Guide 03 cluster healthy (3 peers, unsealed) | `ssh …@121 'VAULT_SKIP_VERIFY=true VAULT_TOKEN=… vault operator raft list-peers'` → 3 servers |
| 2 | Root token available (cluster init JSON) | `Test-Path ~/.nexus/secrets/vault-cluster-init.json` (host) → `True` |
| 3 | AD forest up; `ServiceAccounts` + `Groups` OUs exist (Guide 02) | `ssh …@240 'powershell -c "Get-ADOrganizationalUnit -Filter * | Select Name"'` lists them |
| 4 | Gateway forwards `nexus.lab` | host: `Resolve-DnsName -Server 192.168.70.1 dc-nexus.nexus.lab` → `192.168.70.240` |
| 5 | `jq`, `openssl` on the Vault nodes (Debian base has them) | `ssh …@121 'which jq openssl'` |

> Throughout Part A, run `vault` on **`vault-1`** with `VAULT_SKIP_VERIFY=true`
> and the root token exported:
> ```bash
> export VAULT_ADDR=https://127.0.0.1:8200 VAULT_SKIP_VERIFY=true
> export VAULT_TOKEN=$(jq -r .root_token /root/vault-cluster-init.json 2>/dev/null || echo PASTE_ROOT_TOKEN)
> ```
> After §5.A.6 distributes the CA, you can swap `VAULT_SKIP_VERIFY=true` for
> `VAULT_CACERT=/usr/local/share/ca-certificates/nexus-vault-pki-root.crt`.

---

## 4. Target topology

No new VMs — this configures the existing `vault-1/2/3` + `dc-nexus`.

**Vault objects created:**

| Path | What |
|---|---|
| `pki/` | Root CA mount (max-lease 10y / `87600h`); CN `NexusPlatform Root CA` |
| `pki_int/` | Intermediate CA mount (max-lease 5y / `43800h`); CN `NexusPlatform Intermediate CA` |
| `pki_int/roles/vault-server` | leaf role: allowed `nexus.lab`, `vault-1/2/3`(+FQDN), `dc-nexus`(+FQDN), `localhost`; IP-SANs allowed; RSA-4096; TTL **90d** (`2160h`) |
| `auth/ldap/` | LDAPS to `dc-nexus.nexus.lab:636`, search-then-bind, group→policy maps |
| `ldap/` (secrets) | AD engine, `schema=ad`, `password_policy=nexus-ad-rotated`, static role `svc-demo-rotated` |
| policies | `nexus-admin`, `nexus-operator`, `nexus-reader` |
| build host | root CA bundle at `~/.nexus/vault-ca-bundle.crt` |

**AD objects created (on `dc-nexus`):**

| Object | Where | Purpose |
|---|---|---|
| `svc-vault-ldap` | `OU=ServiceAccounts` | Vault's bind account (4 delegated ACEs) |
| `svc-demo-rotated` | `OU=ServiceAccounts` | demo account whose password Vault rotates |
| `nexus-vault-admins` / `-operators` / `-readers` | `OU=Groups` | mapped to `nexus-admin`/`-operator`/`-reader` |
| LDAPS cert | `dc-nexus` `LocalMachine\{Root,CA,My}` | serves LDAPS/636 with a PKI-issued cert |

**Canon:** PKI leaf TTL **90 days**; AD MinPwLen 14 (Guide 02); LDAPS only.

---

## 5. Step-by-step build

> **WHERE:** Part A + C run on **`vault-1`** (root shell, root token exported).
> Part B runs on **`dc-nexus`** over **RDP** (elevated PowerShell) plus two
> issue-cert steps on `vault-1`. Part D is the reusable pattern.

### Part A — PKI hierarchy + retire self-signed TLS

> **Step 5.A.1 — Mount `pki/` (root) and `pki_int/` (intermediate)**
> **WHERE:** `vault-1`, root shell (token exported).
> **WHY:** two separate PKI engines — root with a 10-year ceiling, intermediate
> with 5.
> **WHAT:**
> ```bash
> vault secrets enable -path=pki     -max-lease-ttl=87600h pki
> vault secrets enable -path=pki_int -max-lease-ttl=43800h pki
> vault secrets tune -max-lease-ttl=87600h pki
> vault secrets tune -max-lease-ttl=43800h pki_int
> ```
> **EXPECTED:** both mounts enabled.
> **VERIFY:** `vault secrets list | grep -E '^pki(_int)?/'` shows both.

> **Step 5.A.2 — Generate the root CA**
> **WHERE:** `vault-1`, root shell.
> **WHY:** the internal root; its private key never leaves Vault. It signs the
> intermediate (next step) and is otherwise dormant. The URLs config gives
> chain-walking tools an HTTP fetch path on the leader.
> **WHAT:**
> ```bash
> vault write -format=json pki/root/generate/internal \
>   common_name='NexusPlatform Root CA' \
>   issuer_name='nexus-platform-root-ca' \
>   ttl=87600h key_bits=4096 >/dev/null
> vault write pki/config/urls \
>   issuing_certificates='https://192.168.70.121:8200/v1/pki/ca' \
>   crl_distribution_points='https://192.168.70.121:8200/v1/pki/crl'
> ```
> **EXPECTED:** no error.
> **VERIFY:** `vault read pki/cert/ca | openssl x509 -noout -subject` →
> `CN = NexusPlatform Root CA`.

> **Step 5.A.3 — Generate + sign the intermediate CA**
> **WHERE:** `vault-1`, root shell.
> **WHY:** `pki_int/` produces a CSR; the root signs it; `pki_int/` stores the
> signed cert. The intermediate is what issues all leaf certs.
> **WHAT:**
> ```bash
> CSR=$(vault write -format=json pki_int/intermediate/generate/internal \
>   common_name='NexusPlatform Intermediate CA' \
>   issuer_name='nexus-platform-intermediate-ca' key_bits=4096 | jq -r '.data.csr')
> SIGNED=$(vault write -format=json pki/root/sign-intermediate \
>   csr="$CSR" format=pem_bundle ttl=43800h | jq -r '.data.certificate')
> echo "$SIGNED" | vault write pki_int/intermediate/set-signed certificate=-
> vault write pki_int/config/urls \
>   issuing_certificates='https://192.168.70.121:8200/v1/pki_int/ca' \
>   crl_distribution_points='https://192.168.70.121:8200/v1/pki_int/crl'
> ```
> **EXPECTED:** `set-signed` returns the imported issuer.
> **VERIFY:** `vault read pki_int/cert/ca | openssl x509 -noout -subject -issuer`
> → subject `Intermediate CA`, issuer `Root CA`.

> **Step 5.A.4 — Define the `vault-server` leaf role**
> **WHERE:** `vault-1`, root shell.
> **WHY:** the role that issues the cluster's listener certs (and `dc-nexus`'s
> LDAPS cert in Part B). It allows the bare lab names + IP-SANs, RSA-4096, 90-day
> leaves.
> **WHAT:**
> ```bash
> vault write pki_int/roles/vault-server \
>   allowed_domains='nexus.lab,vault-1,vault-2,vault-3,vault-1.nexus.lab,vault-2.nexus.lab,vault-3.nexus.lab,dc-nexus,dc-nexus.nexus.lab,DC-NEXUS,DC-NEXUS.nexus.lab,localhost' \
>   allow_subdomains=false allow_bare_domains=true allow_glob_domains=false \
>   allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=false \
>   key_type=rsa key_bits=4096 ttl=2160h max_ttl=2160h
> ```
> **EXPECTED:** role written.
> **VERIFY:** `vault read pki_int/roles/vault-server | grep allowed_domains` lists
> the names.

> **Step 5.A.5 — Re-issue each node's listener cert from PKI (retire self-signed)**
> **WHERE:** each of `vault-1/2/3`, root shell.
> **WHY:** swap the Guide-03 self-signed listener cert for a PKI-issued one (leaf
> + intermediate chain), then SIGHUP Vault to serve it. Atomic swap so a failure
> leaves the old cert in place. Do `vault-1` last is fine; order doesn't matter
> since each issues against the leader. **Values shown for `vault-1`; repeat with
> each node's IPs.**
> **WHAT (on `vault-1`; for `vault-2` use `.122`/`.10.122`, `vault-3` `.123`/`.10.123`):**
> ```bash
> H=vault-1 ; V11=192.168.70.121 ; V10=192.168.10.121
> ISSUED=$(vault write -format=json pki_int/issue/vault-server \
>   common_name="$H.nexus.lab" alt_names="$H,localhost" \
>   ip_sans="$V11,$V10,127.0.0.1" ttl=2160h)
> echo "$ISSUED" | jq -r '.data.certificate, .data.issuing_ca' > /tmp/vault.crt
> echo "$ISSUED" | jq -r '.data.private_key' > /tmp/vault.key
> install -o vault -g vault -m 0644 /tmp/vault.crt /etc/vault.d/tls/vault.crt.new
> install -o vault -g vault -m 0600 /tmp/vault.key /etc/vault.d/tls/vault.key.new
> mv /etc/vault.d/tls/vault.crt.new /etc/vault.d/tls/vault.crt
> mv /etc/vault.d/tls/vault.key.new /etc/vault.d/tls/vault.key
> rm -f /tmp/vault.crt /tmp/vault.key
> systemctl reload vault.service ; sleep 3
> ```
> **EXPECTED:** Vault reloads; listener now serves the PKI cert + chain.
> **VERIFY:**
> ```bash
> echo Q | openssl s_client -connect 127.0.0.1:8200 -servername vault-1.nexus.lab 2>/dev/null \
>   | openssl x509 -noout -issuer
> ```
> → issuer `CN = NexusPlatform Intermediate CA`. Repeat the whole block on
> `vault-2` and `vault-3`.

> **Step 5.A.6 — Distribute the root CA (build host + every node) and drop skip-verify**
> **WHERE:** `vault-1` (export the CA), then host + each node.
> **WHY:** so clients validate the new PKI certs without `VAULT_SKIP_VERIFY`.
> Writes the root to the build host (for `VAULT_CACERT`) and to each node's system
> trust store.
> **WHAT:**
> ```bash
> # On vault-1: print the root CA (copy it to the build host + nodes)
> vault read -format=json pki/cert/ca | jq -r '.data.certificate'
>
> # On EACH node (vault-1/2/3): install into the system trust store
> #   paste the root CA into /usr/local/share/ca-certificates/nexus-vault-pki-root.crt
> sudo update-ca-certificates
> ```
> On the **host**, save the same PEM to `~/.nexus/vault-ca-bundle.crt`.
> **EXPECTED:** `update-ca-certificates` reports `1 added`.
> **VERIFY:** on a node, drop skip-verify:
> `VAULT_CACERT=/usr/local/share/ca-certificates/nexus-vault-pki-root.crt vault status -address=https://vault-1.nexus.lab:8200`
> succeeds **without** `VAULT_SKIP_VERIFY` and **without** a TLS error.

### Part B — AD prep on `dc-nexus` (bind account, ACL, LDAPS cert)

> **Step 5.B.1 — Create the bind account, demo account, and the three groups**
> **WHERE:** `dc-nexus` (RDP), elevated PowerShell.
> **WHY:** `svc-vault-ldap` is Vault's bind identity; `svc-demo-rotated` is the
> account Vault will own + rotate; the three groups map to Vault policies. Add
> `nexusadmin` to the admins group so your LDAP login gets `nexus-admin`.
> **WHAT:** (pick a strong initial password for the service accounts ≥14 chars)
> ```powershell
> Import-Module ActiveDirectory
> $dn = 'OU=ServiceAccounts,DC=nexus,DC=lab'
> $pw = ConvertTo-SecureString '<STRONG_INITIAL_PW>' -AsPlainText -Force
>
> New-ADUser -Name svc-vault-ldap  -SamAccountName svc-vault-ldap  -Path $dn `
>   -AccountPassword $pw -Enabled $true -PasswordNeverExpires $true `
>   -Description 'Vault LDAP bind account'
> New-ADUser -Name svc-demo-rotated -SamAccountName svc-demo-rotated -Path $dn `
>   -AccountPassword $pw -Enabled $true -PasswordNeverExpires $true `
>   -Description 'Demo account whose password Vault rotates'
>
> foreach ($g in 'nexus-vault-admins','nexus-vault-operators','nexus-vault-readers') {
>   New-ADGroup -Name $g -GroupScope Global -GroupCategory Security -Path 'OU=Groups,DC=nexus,DC=lab'
> }
> Add-ADGroupMember -Identity nexus-vault-admins -Members nexusadmin
> ```
> **EXPECTED:** two users + three groups created.
> **VERIFY:** `Get-ADUser svc-vault-ldap` and
> `Get-ADGroupMember nexus-vault-admins | Select Name` → lists `nexusadmin`.

> **Step 5.B.2 — Delegate password-reset rights to `svc-vault-ldap`**
> **WHERE:** `dc-nexus` (RDP), elevated **cmd** (dsacls) or PowerShell.
> **WHY:** the `secrets/ldap` engine rotates `svc-demo-rotated`'s password by
> writing `unicodePwd` over LDAPS as `svc-vault-ldap`. That bind account needs
> four ACEs on the OU — *Reset Password* + *Change Password* extended rights, and
> read/write on `userAccountControl` — or rotation fails with LDAP code 50
> (insufficient access).
> **WHAT:**
> ```cmd
> dsacls "OU=ServiceAccounts,DC=nexus,DC=lab" /I:S /G "NEXUS\svc-vault-ldap:CA;Reset Password;user"
> dsacls "OU=ServiceAccounts,DC=nexus,DC=lab" /I:S /G "NEXUS\svc-vault-ldap:CA;Change Password;user"
> dsacls "OU=ServiceAccounts,DC=nexus,DC=lab" /I:S /G "NEXUS\svc-vault-ldap:RPWP;userAccountControl;user"
> ```
> **EXPECTED:** each prints `The command completed successfully`.
> **VERIFY:** `dsacls "OU=ServiceAccounts,DC=nexus,DC=lab" | findstr /i "svc-vault-ldap"`
> shows the granted rights.

> **Step 5.B.3 — Issue the LDAPS cert + install the full chain on `dc-nexus`**
> **WHERE:** issue on `vault-1` (root shell); install on `dc-nexus` (RDP).
> **WHY:** the DC serves LDAPS only once a cert whose chain builds **locally to a
> trusted root** sits in `LocalMachine\My`. **All three stores are load-bearing:**
> root→`Root`, intermediate→`CA`, leaf+key→`My`. Miss the root and Schannel logs
> Event 36886 and resets every LDAPS handshake. Build the PFX with **openssl on
> `vault-1`** (the .NET PFX path yields ephemeral keys Schannel can't use).
> **WHAT (on `vault-1`):**
> ```bash
> P=$(openssl rand -hex 16)   # transit-only PFX password
> ISSUED=$(vault write -format=json pki_int/issue/vault-server \
>   common_name=dc-nexus.nexus.lab alt_names='dc-nexus,DC-NEXUS,DC-NEXUS.nexus.lab' \
>   ip_sans=192.168.70.240 ttl=2160h)
> echo "$ISSUED" | jq -r '.data.certificate'  > /tmp/dc.crt
> echo "$ISSUED" | jq -r '.data.private_key'  > /tmp/dc.key
> echo "$ISSUED" | jq -r '.data.issuing_ca'   > /tmp/dc-int.pem
> openssl pkcs12 -export -inkey /tmp/dc.key -in /tmp/dc.crt -name dc-nexus-ldaps \
>   -passout "pass:$P" -out /tmp/dc.pfx
> echo "PFX password: $P"   # note it; you'll paste it on the DC
> # copy /tmp/dc.pfx, /tmp/dc-int.pem, and the root CA (~/.nexus/vault-ca-bundle.crt) to dc-nexus
> ```
> **WHAT (on `dc-nexus`, RDP, elevated PowerShell — files staged in `C:\Windows\Temp`):**
> ```powershell
> # Order matters: root + intermediate FIRST, then the leaf, so the chain builds.
> Import-Certificate -FilePath C:\Windows\Temp\dc-root.pem -CertStoreLocation Cert:\LocalMachine\Root
> Import-Certificate -FilePath C:\Windows\Temp\dc-int.pem  -CertStoreLocation Cert:\LocalMachine\CA
> $pw = ConvertTo-SecureString '<PFX_PASSWORD>' -AsPlainText -Force
> $leaf = Import-PfxCertificate -FilePath C:\Windows\Temp\dc.pfx `
>   -CertStoreLocation Cert:\LocalMachine\My -Password $pw -Exportable
> # Prove the chain builds locally BEFORE restarting NTDS
> $chain = New-Object System.Security.Cryptography.X509Certificates.X509Chain
> $chain.ChainPolicy.RevocationMode = 'NoCheck'
> if (-not $chain.Build($leaf)) { throw "chain build failed: $($chain.ChainStatus.Status)" }
> Restart-Service NTDS -Force ; Start-Sleep 20
> ```
> **EXPECTED:** chain build returns `True`; NTDS restarts (~10–30 s AD blip).
> **VERIFY (from the host):**
> ```powershell
> $t=[Net.Sockets.TcpClient]::new('192.168.70.240',636)
> $s=[Net.Security.SslStream]::new($t.GetStream(),$false,{$true})
> $s.AuthenticateAsClient('dc-nexus.nexus.lab'); $s.RemoteCertificate.Issuer; $s.Dispose(); $t.Close()
> ```
> → issuer `CN=NexusPlatform Intermediate CA` (LDAPS handshake completes).

### Part C — Vault LDAP auth + `secrets/ldap` rotation

> **Step 5.C.1 — Write the three Vault policies**
> **WHERE:** `vault-1`, root shell.
> **WHY:** `nexus-admin` = full sudo; `nexus-operator` = read/write `nexus/*` +
> issue certs (no sudo); `nexus-reader` = read-only. AD groups map to these.
> **WHAT:**
> ```bash
> vault policy write nexus-admin - <<'EOF'
> path "*" { capabilities = ["create","read","update","delete","list","sudo"] }
> EOF
>
> vault policy write nexus-operator - <<'EOF'
> path "nexus/data/*"     { capabilities = ["create","read","update","delete","list"] }
> path "nexus/metadata/*" { capabilities = ["read","list","delete"] }
> path "nexus/delete/*"   { capabilities = ["update"] }
> path "nexus/undelete/*" { capabilities = ["update"] }
> path "nexus/destroy/*"  { capabilities = ["update"] }
> path "pki_int/issue/*"  { capabilities = ["create","update"] }
> path "pki_int/roles"    { capabilities = ["read","list"] }
> path "pki_int/roles/*"  { capabilities = ["read"] }
> path "pki/cert/ca"      { capabilities = ["read"] }
> path "pki_int/cert/ca"  { capabilities = ["read"] }
> path "auth/token/lookup-self" { capabilities = ["read"] }
> path "auth/token/renew-self"  { capabilities = ["update"] }
> EOF
>
> vault policy write nexus-reader - <<'EOF'
> path "nexus/data/*"     { capabilities = ["read"] }
> path "nexus/metadata/*" { capabilities = ["read","list"] }
> path "pki/cert/ca"      { capabilities = ["read"] }
> path "pki_int/cert/ca"  { capabilities = ["read"] }
> path "auth/token/lookup-self" { capabilities = ["read"] }
> path "auth/token/renew-self"  { capabilities = ["update"] }
> EOF
> ```
> **EXPECTED:** three policies written.
> **VERIFY:** `vault policy list | grep -E 'nexus-(admin|operator|reader)'` → all three.

> **Step 5.C.2 — Configure `auth/ldap` (LDAPS, search-then-bind) + group maps**
> **WHERE:** `vault-1`, root shell.
> **WHY:** point Vault at the DC over LDAPS, bind as `svc-vault-ldap`, search by
> `sAMAccountName`, map groups. **`upndomain` stays empty** (setting it breaks the
> user filter — see primer); the `certificate=` is the root CA so Vault trusts the
> DC's LDAPS cert.
> **WHAT:** (the CA bundle staged at `/tmp/nexus-root-ca.pem` on `vault-1`)
> ```bash
> vault auth enable ldap 2>/dev/null || true
> vault write auth/ldap/config \
>   url='ldaps://dc-nexus.nexus.lab:636' \
>   binddn='CN=svc-vault-ldap,OU=ServiceAccounts,DC=nexus,DC=lab' \
>   bindpass='<svc-vault-ldap PW>' \
>   userdn='DC=nexus,DC=lab' \
>   userattr='sAMAccountName' \
>   upndomain='' \
>   userfilter='(&(objectClass=user)(sAMAccountName={{.Username}}))' \
>   groupdn='DC=nexus,DC=lab' \
>   groupattr='cn' \
>   groupfilter='(&(objectClass=group)(member:1.2.840.113556.1.4.1941:={{.UserDN}}))' \
>   certificate=@/tmp/nexus-root-ca.pem \
>   insecure_tls=false starttls=false request_timeout=30 username_as_alias=true
>
> vault write auth/ldap/groups/nexus-vault-admins    policies=nexus-admin
> vault write auth/ldap/groups/nexus-vault-operators policies=nexus-operator
> vault write auth/ldap/groups/nexus-vault-readers   policies=nexus-reader
> ```
> **EXPECTED:** config + 3 group maps written.
> **VERIFY:** `vault login -method=ldap username=nexusadmin` (enter the AD
> password) issues a token; `vault token lookup | grep policies` shows
> `nexus-admin`.

> **Step 5.C.3 — Enable `secrets/ldap` + the rotation password policy**
> **WHERE:** `vault-1`, root shell.
> **WHY:** the AD secrets engine. The `nexus-ad-rotated` password policy makes
> 24-char passwords using all four character classes (always satisfies AD
> complexity). `skip_static_role_import_rotation=true` stops Vault trying to bind
> *as* a target account with a password we never knew.
> **WHAT:**
> ```bash
> vault write sys/policies/password/nexus-ad-rotated policy=- <<'EOF'
> length = 24
> rule "charset" { charset = "abcdefghijklmnopqrstuvwxyz" min-chars = 2 }
> rule "charset" { charset = "ABCDEFGHIJKLMNOPQRSTUVWXYZ" min-chars = 2 }
> rule "charset" { charset = "0123456789"                 min-chars = 2 }
> rule "charset" { charset = "!#$%&*+-.=?@_"               min-chars = 2 }
> EOF
>
> vault secrets enable ldap 2>/dev/null || true
> vault write ldap/config \
>   binddn='CN=svc-vault-ldap,OU=ServiceAccounts,DC=nexus,DC=lab' \
>   bindpass='<svc-vault-ldap PW>' \
>   url='ldaps://dc-nexus.nexus.lab:636' \
>   schema=ad password_policy=nexus-ad-rotated \
>   skip_static_role_import_rotation=true \
>   certificate=@/tmp/nexus-root-ca.pem \
>   request_timeout=30 insecure_tls=false starttls=false
> ```
> **EXPECTED:** policy + engine config written.
> **VERIFY:** `vault read ldap/config | grep -E 'schema|password_policy'` →
> `ad` + `nexus-ad-rotated`.

> **Step 5.C.4 — Create the static rotate-role + take ownership**
> **WHERE:** `vault-1`, root shell.
> **WHY:** bind `svc-demo-rotated` to Vault. `skip_import_rotation=true` is
> **create-only** (Vault rejects it on updates). The explicit
> `vault write -force ldap/rotate-role/...` is what actually seizes the password
> (binding as `svc-vault-ldap`, writing `unicodePwd` over LDAPS — needs the §5.B.2
> ACL).
> **WHAT:**
> ```bash
> vault write ldap/static-role/svc-demo-rotated \
>   username='svc-demo-rotated' \
>   dn='CN=svc-demo-rotated,OU=ServiceAccounts,DC=nexus,DC=lab' \
>   rotation_period=24h skip_import_rotation=true
> vault write -force ldap/rotate-role/svc-demo-rotated
> vault read ldap/static-cred/svc-demo-rotated
> ```
> **EXPECTED:** the role is created; `static-cred` returns `username`, a
> `password`, and `last_vault_rotation`.
> **VERIFY:** `vault read -format=json ldap/static-cred/svc-demo-rotated | jq -r '.data.last_vault_rotation'`
> → a timestamp (Vault now owns the password).

### Part D — The per-tier scaffolding pattern (worked: `foundation`)

Every later guide gives its tier mTLS certs + credentials with the **same four
moves**. This is the recipe to invoke — shown with a small `foundation` example.

> **Step 5.D.1 — (1) a `pki_int` role · (2) a scoped policy · (3) an AppRole · (4) KV seed**
> **WHERE:** `vault-1`, root shell.
> **WHY:** the repeatable shape — a tier gets a cert-issuing role limited to its
> hostnames, a policy limited to its KV prefix + that issue path, an AppRole bound
> to the policy (the node's machine login), and its seeded secrets in KV.
> **WHAT (template — substitute `<tier>`, hostnames, KV prefix):**
> ```bash
> # (1) PKI role — only this tier's identities may be issued
> vault write pki_int/roles/<tier>-server \
>   allowed_domains='<tier>-1.nexus.lab,<tier>-2.nexus.lab' \
>   allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h
>
> # (2) policy — least privilege: this tier's KV + its issue path
> vault policy write <tier> - <<EOF
> path "nexus/data/<tier>/*"     { capabilities = ["read"] }
> path "pki_int/issue/<tier>-server" { capabilities = ["create","update"] }
> path "pki/cert/ca"             { capabilities = ["read"] }
> EOF
>
> # (3) AppRole — the node's machine login, bound to the policy
> vault write auth/approle/role/<tier> \
>   token_policies=<tier> token_ttl=1h token_max_ttl=4h secret_id_ttl=720h
> vault read  auth/approle/role/<tier>/role-id            # -> RoleID (bake into the node)
> vault write -f auth/approle/role/<tier>/secret-id       # -> SecretID (deliver to the node)
>
> # (4) KV seed — the tier's secrets
> vault kv put nexus/<tier>/bootstrap some_key=some_value
> ```
> **EXPECTED:** role + policy + AppRole + KV path created; RoleID/SecretID printed.
> **VERIFY:** a node can `vault write auth/approle/login role_id=… secret_id=…`
> and the resulting token can `vault kv get nexus/<tier>/bootstrap` but **not**
> read another tier's path.
>
> > Each Guide 05–22 runs exactly this, with its real hostnames/KV prefix. The
> > node then uses its AppRole token to `vault write pki_int/issue/<tier>-server …`
> > for its mTLS leaf — the same issue call §5.A.5 used for Vault's own certs.

---

## 6. Validation — by-hand acceptance smoke

From the **host**/`vault-1`. Cluster + DC up.

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | Root + intermediate CAs exist | `vault read pki_int/cert/ca \| openssl x509 -noout -issuer` | issuer = `NexusPlatform Root CA` |
| 2 | Vault listeners serve PKI certs | `echo Q \| openssl s_client -connect vault-1.nexus.lab:8200 2>/dev/null \| openssl x509 -noout -issuer` | `Intermediate CA` |
| 3 | skip-verify retired | `VAULT_CACERT=…/nexus-vault-pki-root.crt vault status -address=https://vault-1.nexus.lab:8200` | succeeds, no TLS error |
| 4 | LDAPS handshake on the DC | (host SslStream to `192.168.70.240:636`, §5.B.3 verify) | issuer = `Intermediate CA` |
| 5 | Human LDAP login works | `vault login -method=ldap username=nexusadmin` | token issued |
| 6 | LDAP login carries the right policy | `vault token lookup \| grep policies` (after #5) | `nexus-admin` |
| 7 | `secrets/ldap` engine configured | `vault read ldap/config \| grep schema` | `ad` |
| 8 | Static role rotates | `vault read ldap/static-cred/svc-demo-rotated \| grep last_vault_rotation` | a timestamp |
| 9 | Rotated password actually works in AD | `vault read -field=password ldap/static-cred/svc-demo-rotated` then test an LDAPS bind as `svc-demo-rotated` | bind succeeds |
| 10 | Scaffolding AppRole least-privilege | log in with a tier AppRole, read its KV (ok) + another tier's KV (denied) | own ok, other `permission denied` |

**1–8 green ⇒ Guide 04 satisfied.** 9 proves Vault truly owns the AD password;
10 proves the per-tier isolation later guides rely on.

---

## 7. Teardown / reset

```bash
# On vault-1 (root token). Disabling a mount destroys its data.
vault secrets disable ldap
vault auth disable ldap
vault policy delete nexus-admin ; vault policy delete nexus-operator ; vault policy delete nexus-reader
vault secrets disable pki_int   # destroys the intermediate (and every leaf under it)
vault secrets disable pki       # destroys the root
```
On `dc-nexus` (RDP): remove the LDAPS cert from `LocalMachine\My` (+ the
intermediate/root you added) and restart NTDS; delete the AD users/groups if
wiping. After disabling `pki_int`, **every tier's certs are invalid** — only do
this on a full rebuild.

> Re-issuing the **root** is destructive to the entire chain. To rotate the
> intermediate only, sign a new one from the existing root (§5.A.3) and re-run the
> leaf rotations (§5.A.5) — you don't burn the root.

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `vault login -method=ldap` fails "Strong Auth Required" / result 8 | Vault is talking plain LDAP, or the DC has no LDAPS cert | Use `ldaps://…:636` + install the DC cert (§5.B.3). Plain LDAP is rejected here regardless of `LDAPServerIntegrity`. See `feedback_ad_ldaps_password_writes`. |
| LDAP search returns 0 users though the bind works | `upndomain` is set, rewriting `{{.Username}}` to `user@domain` (no `sAMAccountName` match) | Set `upndomain=''` (§5.C.2). Vault #27276 / `feedback_vault_ldap_ad_upn_bind`. |
| LDAPS handshake on `:636` resets immediately; Schannel Event 36886 | Root CA missing from `dc-nexus` `LocalMachine\Root` → chain builds `PartialChain` | Install **all three**: root→`Root`, intermediate→`CA`, leaf→`My`; prove `X509Chain.Build` before restarting NTDS (§5.B.3). |
| LDAPS cert imports but key is "not accessible" / handshake still resets | PFX built with .NET (ephemeral key) | Build the PFX with **openssl on `vault-1`** (§5.B.3), not .NET. |
| `static-role` create fails "failed to bind with current password" (data 52e) | Vault tried to import-rotate by binding as the target | Set `skip_static_role_import_rotation=true` on the engine + `skip_import_rotation=true` on the role (create-only), then `vault write -force ldap/rotate-role/<r>`. See `feedback_vault_ldap_static_role_skip_import`. |
| `rotate-role` fails LDAP code 50 (insufficient access) | Bind account lacks the password-reset ACEs | Delegate the 4 ACEs via `dsacls` (§5.B.2). See `feedback_vault_ldap_ad_acl_delegation`. |
| `skip_import_rotation has no effect on updates` (400) on re-apply | The flag is **create-only**; you sent it on an existing role | Don't re-write the role; just `vault write -force ldap/rotate-role/<r>`. |
| TLS errors after the cert swap | clients still on self-signed trust | Distribute the root CA (§5.A.6) + use `VAULT_CACERT`; confirm the leaf served is intermediate-issued. |

---

### Cross-references

- **Network/DNS canon:** `nexus-platform-plan/docs/infra/network.md`
- **Automated equivalents:** `nexus-infra-vmware/terraform/envs/security/role-overlay-vault-{pki-*,ldap-*,ldaps-cert}.tf` + foundation-env `role-overlay-dc-vault-ad-bind*.tf`
- **Previous guide:** [`03-foundation-vault-ha.md`](./03-foundation-vault-ha.md)
- **Next guide:** Guide 05 — Orchestration · Swarm + Nomad + Consul + Portainer. See [`INDEX.md`](../INDEX.md). It is the first guide to **invoke Part D's scaffolding pattern** for a real tier.
