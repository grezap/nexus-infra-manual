# Guide 03 — Foundation · Vault HA (`vault-1/2/3` Raft + `vault-transit`)

> **Mirrors:** `nexus-infra-vmware/packer/vault/*` (the Vault node template +
> `vault_node` Ansible role + `vault.hcl.tpl` / `vault.service` /
> `vault-firstboot.sh`) and the `security` env overlays
> `role-overlay-vault-{transit-bringup,cluster-seal-config,cluster}.tf`. Where
> the automated lab Packer-bakes the binary then drives init/unseal/join from the
> build host over SSH, this guide installs Vault and runs every `vault operator`
> command **by hand on each node**.

---

## 1. Overview & purpose

This guide stands up the lab's **secrets backbone**: a 3-node HashiCorp Vault
cluster (`vault-1`, `vault-2`, `vault-3`) using **integrated Raft storage**, plus
a single-node **`vault-transit`** helper that holds the **auto-unseal key** for
the cluster.

Vault is where every later tier gets its identity and secrets: the PKI hierarchy
+ LDAP auth (Guide 04), and then the per-tier mTLS certs, AppRoles, and KV
credentials that Kafka, the OLTP/analytics/lakehouse engines, Vitess, Citus, and
the observability stack all consume. Without Vault, none of the secured tiers
can come up.

**The auto-unseal design (and why two pieces):**

- A freshly-started Vault is **sealed** — its storage is encrypted and it can't
  serve secrets until the master key is reconstructed. Classic **Shamir**
  unsealing means a human pastes 3-of-5 key shares on every boot. That doesn't
  scale to a fleet that cold-reboots.
- So `vault-1/2/3` are configured for **transit auto-unseal**: at boot they call
  `vault-transit` to decrypt their seal, and come up unsealed with no human in
  the loop.
- `vault-transit` itself is a tiny single-node Vault that **is** the unseal-key
  custodian — it can't auto-unseal itself, so it stays on **Shamir** (unsealed
  manually once per reboot, or by the recovery script). It runs only the
  **transit secrets engine** with one key, `nexus-cluster-unseal`.

So the build order is: **`vault-transit` first** (it must exist before the
cluster can reference it), then `vault-1` (init + auto-unseal), then `vault-2`
and `vault-3` (raft-join + auto-unseal).

**Dependency:**
- **Guide 00** — four `deb13` nodes baselined with dual-NIC static networking:
  `vault-1` (`.121`), `vault-2` (`.122`), `vault-3` (`.123`), `vault-transit`
  (`.124`); backplane IPs `192.168.10.121–.124`.
- **Guide 01** — `nexus-gateway` up (egress, so each node can download the Vault
  binary; DNS/NTP).
- Guide 02 (AD) is **not** required for the Raft cluster itself — it's needed
  only for Vault's LDAP auth, which is **Guide 04**. Build order still puts AD
  first.

> **Scope boundary:** this guide ends at a healthy 3-node cluster with KV-v2,
> userpass, AppRole, and a smoke secret — running on **self-signed bootstrap
> TLS**. The PKI hierarchy that re-issues every cert, plus LDAP auth, is
> **Guide 04**.

---

## 2. Component primer

- **HashiCorp Vault.** A secrets-management server: stores secrets encrypted,
  brokers dynamic credentials, issues certificates (PKI), and authenticates
  clients via pluggable methods. *Why:* one audited, policy-gated source of
  truth for every secret + identity in the platform. *Otherwise:* secrets
  scattered in config files / env vars (no rotation, no audit, no policy).
- **Integrated Raft storage.** Vault's built-in HA storage backend — the Raft
  consensus protocol replicates the encrypted store across the 3 nodes and
  elects a leader; followers forward writes to it. *Why:* no external storage
  dependency (unlike the old Consul backend) — the cluster is self-contained.
  *Otherwise:* Consul/etcd backend (an extra cluster to run), or a single node
  (no HA).
- **Seal / unseal + Shamir.** Vault's storage is sealed (encrypted) at rest;
  *unsealing* reconstructs the master key. **Shamir's Secret Sharing** splits
  that key into N shares needing M to reconstruct (here 5 shares, threshold 3).
  *Why M-of-N:* no single person holds the whole key. *Otherwise:* a single
  unseal key (one point of compromise).
- **Transit auto-unseal.** Instead of Shamir-by-hand at every boot, a node's
  seal is wrapped by a key held in *another* Vault's **transit** engine; at boot
  the node calls that Vault to unwrap it and auto-unseals. *Why:* a cold-rebootable
  fleet can't have a human pasting keys on every node. *Otherwise:* Shamir
  (manual), or a cloud KMS auto-unseal (no cloud in this lab — `vault-transit`
  is the on-prem stand-in).
- **Transit secrets engine.** Vault's "encryption as a service" — it holds keys
  and does encrypt/decrypt **without ever exposing the key**. Here it does
  exactly one job: wrap/unwrap the cluster's seal via key
  `nexus-cluster-unseal`. *Otherwise:* the cluster nodes would hold their own
  seal material (defeats the point).
- **Recovery keys.** In auto-unseal mode there are no Shamir *unseal* keys;
  instead `init` emits **recovery keys** (also 5/3) used only for break-glass
  operations (regenerate root token, etc.). Day-to-day unsealing is automatic.
- **KV-v2 / userpass / AppRole.** The first things mounted on the live cluster:
  **KV-v2** (versioned key/value secret store at `nexus/`), **userpass** (a
  human operator login), and **AppRole** (machine login — a RoleID + SecretID
  pair, the method every later tier's nodes use to authenticate). *Why now:* so
  the cluster is immediately usable + has a smoke target. The per-tier AppRoles
  themselves are added by later guides.

---

## 3. Prerequisites

| # | Requirement | One-command verify (host) |
|---|---|---|
| 1 | Four `deb13` nodes baselined (Guide 00 §5.B), dual-NIC static | `1..3 + 124 \| % { Test-NetConnection 192.168.70.$_ -Port 22 }` style — all `True` for `.121`–`.124` |
| 2 | Guide 01 `nexus-gateway` up (egress for the binary download) | `ssh nexusadmin@192.168.70.121 'curl -sI https://releases.hashicorp.com \| head -1'` → `HTTP/2 200` |
| 3 | A secure place on the build host to store init JSON (root token + keys) | `Test-Path ~/.nexus/secrets` (create it; lock the ACL) |

> The four nodes are ordinary Debian baseline nodes from Guide 00 — Guide 00
> already set their hostnames, dual-NIC static IPs, `/etc/hosts`, `nftables`,
> chrony, and SSH. This guide layers Vault on top. (The automated lab uses a
> `vault-firstboot.sh` to derive the per-clone hostname/IP/TLS at boot; by hand
> you already set those in Guide 00, so we write the literal values directly.)

---

## 4. Target topology

| Node | Role | VMnet11 (nic0) | VMnet10 (nic1) | Primary MAC | vCPU / RAM / disk |
|---|---|---|---|---|---|
| `vault-transit` | transit auto-unseal custodian (Shamir; single node) | `192.168.70.124/24` | `192.168.10.124/24` | `00:50:56:3F:00:43` | 2 / 2 GB / 40 GB |
| `vault-1` | Raft node 1 (init leader) | `192.168.70.121/24` | `192.168.10.121/24` | `00:50:56:3F:00:40` | 2 / 2 GB / 40 GB |
| `vault-2` | Raft node 2 | `192.168.70.122/24` | `192.168.10.122/24` | `00:50:56:3F:00:41` | 2 / 2 GB / 40 GB |
| `vault-3` | Raft node 3 | `192.168.70.123/24` | `192.168.10.123/24` | `00:50:56:3F:00:42` | 2 / 2 GB / 40 GB |

Secondary (VMnet10) MACs flip the fifth byte to `01` (`…:01:40`–`…:01:43`).
`vault-transit`'s `:43` is the next contiguous MAC after `vault-1/2/3`'s
`:40`–`:42` (confirm against your Guide 00 assignment).

**Vault facts (canon):**

| Setting | Value |
|---|---|
| Vault version | `1.18.4` (`linux_amd64`) |
| Binary | `/usr/local/bin/vault` (`setcap cap_ipc_lock=+ep`) |
| User / group | `vault` / `vault` (system, `nologin`) |
| Data dir | `/opt/vault/data` (`0700 vault:vault`) — Raft store |
| Config dir | `/etc/vault.d` (`0750 root:vault`) — `vault.hcl` [+ `seal-transit.hcl`] |
| TLS dir | `/etc/vault.d/tls` (`0750 vault:vault`) — per-node self-signed cert |
| Cluster name | `nexus-vault` |
| API listener | `0.0.0.0:8200` (clients via VMnet11) |
| Raft RPC | `:8201` advertised on the **VMnet10** backplane |
| Seal mode | `vault-transit` = Shamir 5/3; `vault-1/2/3` = transit auto-unseal (recovery 5/3) |
| Transit key | `nexus-cluster-unseal` at `mount_path=transit/` on `vault-transit` |
| Post-init mounts | KV-v2 at `nexus/`, `userpass`, `approle`, smoke secret `nexus/smoke/canary` |

> **Bootstrap TLS caveat:** every node starts with a per-node **self-signed**
> cert (4096-bit RSA, 10-yr). All `vault` CLI calls in this guide therefore use
> `VAULT_SKIP_VERIFY=true`. Guide 04's PKI re-issues these from a shared CA;
> after that, skip-verify goes away.

---

## 5. Step-by-step build

> **WHERE convention:** unless noted, run on the named node via
> `ssh -i ~/.ssh/nexus_gateway_ed25519 nexusadmin@<ip>`, then `sudo -i`. Vault
> CLI is run **locally on each node** against `https://127.0.0.1:8200` with
> `VAULT_SKIP_VERIFY=true`. "Host" steps run on the Windows workstation.

### 5.1 — Install the Vault binary + node scaffolding (on all four nodes)

This sub-section is **identical on all four nodes** (`vault-transit`, `vault-1`,
`vault-2`, `vault-3`) — run it on each. Only the config (§5.2/5.4/5.5/5.6)
differs per node.

> **Step 5.1.1 — Create the `vault` user, group, and directories**
> **WHERE:** the node, root shell.
> **WHY:** Vault runs as an unprivileged system user with tightly-permissioned
> directories — data `0700`, config `0750` (group-readable by `vault`), TLS dir
> `0750`.
> **WHAT:**
> ```bash
> groupadd --system vault 2>/dev/null || true
> id vault >/dev/null 2>&1 || useradd --system -g vault -s /usr/sbin/nologin -d /opt/vault -M vault
> install -d -m 0750 -o vault -g vault /opt/vault
> install -d -m 0700 -o vault -g vault /opt/vault/data
> install -d -m 0750 -o root  -g vault /etc/vault.d
> install -d -m 0750 -o vault -g vault /etc/vault.d/tls
> ```
> **EXPECTED:** no errors.
> **VERIFY:** `stat -c '%U:%G %a' /opt/vault/data` → `vault:vault 700`.

> **Step 5.1.2 — Download, checksum-verify, and install Vault 1.18.4**
> **WHERE:** the node, root shell.
> **WHY:** install the pinned version from HashiCorp releases, verifying the
> SHA256SUMS before trusting the binary. `setcap cap_ipc_lock` lets Vault
> `mlock` its memory (prevent secrets paging to disk) without running as root.
> **WHAT:**
> ```bash
> cd /tmp
> ver=1.18.4 ; arch=linux_amd64
> curl -fsSL "https://releases.hashicorp.com/vault/${ver}/vault_${ver}_${arch}.zip"  -o "vault_${ver}_${arch}.zip"
> curl -fsSL "https://releases.hashicorp.com/vault/${ver}/vault_${ver}_SHA256SUMS"   -o "vault_${ver}_SHA256SUMS"
> grep "vault_${ver}_${arch}.zip" "vault_${ver}_SHA256SUMS" | sha256sum -c -
> unzip -o "vault_${ver}_${arch}.zip"
> install -m 755 -o root -g root vault /usr/local/bin/vault
> setcap cap_ipc_lock=+ep /usr/local/bin/vault
> rm -f "vault_${ver}_${arch}.zip" "vault_${ver}_SHA256SUMS" vault
> ```
> **EXPECTED:** `vault_1.18.4_linux_amd64.zip: OK` from `sha256sum -c`.
> **VERIFY:** `vault version` → `Vault v1.18.4`.

> **Step 5.1.3 — Install the `vault.service` systemd unit**
> **WHERE:** the node, root shell.
> **WHY:** runs Vault as the `vault` user with the IPC_LOCK capability, and —
> critically — points `-config` at the **directory** `/etc/vault.d/` so Vault
> merges `vault.hcl` **plus** any `seal-transit.hcl` drop-in (that's how
> auto-unseal is switched on for `vault-1/2/3` without editing `vault.hcl`).
> **WHAT:**
> ```bash
> cat > /etc/systemd/system/vault.service <<'EOF'
> [Unit]
> Description=HashiCorp Vault
> Documentation=https://www.vaultproject.io/docs
> Requires=network-online.target
> After=network-online.target
> ConditionFileNotEmpty=/etc/vault.d/vault.hcl
>
> [Service]
> User=vault
> Group=vault
> ProtectSystem=full
> ProtectHome=read-only
> PrivateTmp=yes
> PrivateDevices=yes
> SecureBits=keep-caps
> AmbientCapabilities=CAP_IPC_LOCK
> CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
> NoNewPrivileges=yes
> ExecStart=/usr/local/bin/vault server -config=/etc/vault.d/
> ExecReload=/bin/kill --signal HUP $MAINPID
> KillMode=process
> KillSignal=SIGINT
> Restart=on-failure
> RestartSec=5
> TimeoutStopSec=30
> StartLimitInterval=60
> StartLimitBurst=3
> LimitNOFILE=65536
> LimitMEMLOCK=infinity
>
> [Install]
> WantedBy=multi-user.target
> EOF
> systemctl daemon-reload
> ```
> **EXPECTED:** no error. (Do **not** start yet — there's no `vault.hcl`.)
> **VERIFY:** `systemctl cat vault.service | grep ExecStart` shows
> `-config=/etc/vault.d/` (directory, trailing slash).

### 5.2 — Bring up `vault-transit` (the auto-unseal custodian)

> **Step 5.2.1 — Write `vault-transit`'s TLS cert + `vault.hcl`, then start**
> **WHERE:** `vault-transit` (`192.168.70.124`), root shell.
> **WHY:** the transit node is a standalone Vault on Raft storage. Its config has
> **no** seal stanza, so it uses Shamir. Generate its self-signed cert (SANs
> cover both IPs + hostname) and write `vault.hcl` with its literal addresses.
> **WHAT:**
> ```bash
> # Per-node self-signed TLS (SANs: FQDN, short name, both IPs, loopback)
> openssl req -new -newkey rsa:4096 -days 3650 -nodes -x509 \
>   -subj "/CN=vault-transit.nexus.lab" \
>   -addext "subjectAltName=DNS:vault-transit.nexus.lab,DNS:vault-transit,IP:192.168.70.124,IP:192.168.10.124,IP:127.0.0.1" \
>   -keyout /etc/vault.d/tls/vault.key -out /etc/vault.d/tls/vault.crt
> chown vault:vault /etc/vault.d/tls/vault.key /etc/vault.d/tls/vault.crt
> chmod 600 /etc/vault.d/tls/vault.key ; chmod 644 /etc/vault.d/tls/vault.crt
>
> cat > /etc/vault.d/vault.hcl <<'EOF'
> ui            = true
> disable_mlock = false
> cluster_name  = "nexus-vault"
>
> storage "raft" {
>   path    = "/opt/vault/data"
>   node_id = "vault-transit"
> }
>
> listener "tcp" {
>   address       = "0.0.0.0:8200"
>   tls_cert_file = "/etc/vault.d/tls/vault.crt"
>   tls_key_file  = "/etc/vault.d/tls/vault.key"
> }
>
> cluster_addr = "https://192.168.10.124:8201"
> api_addr     = "https://192.168.70.124:8200"
>
> telemetry {
>   prometheus_retention_time = "24h"
>   disable_hostname          = true
> }
>
> log_level  = "info"
> log_format = "json"
> EOF
> chown root:vault /etc/vault.d/vault.hcl ; chmod 640 /etc/vault.d/vault.hcl
>
> systemctl enable --now vault.service
> ```
> **EXPECTED:** `vault.service` active.
> **VERIFY:** `VAULT_SKIP_VERIFY=true vault status -address=https://127.0.0.1:8200`
> → `Initialized: false`, `Sealed: true`.

> **Step 5.2.2 — Init + unseal `vault-transit` (Shamir 5/3)**
> **WHERE:** `vault-transit`, root shell. **Save the output on the build host.**
> **WHY:** the transit node is the unseal-key custodian, so it can't auto-unseal
> itself — it's Shamir, 5 keys threshold 3. The init JSON (root token + 5 unseal
> keys) is the only copy — persist it to the build host immediately.
> **WHAT:**
> ```bash
> VAULT_SKIP_VERIFY=true vault operator init -format=json \
>   -key-shares=5 -key-threshold=3 -address=https://127.0.0.1:8200 \
>   > /root/vault-transit-init.json
> cat /root/vault-transit-init.json   # copy this to the build host, then shred locally
>
> # Unseal with the first 3 keys
> for i in 0 1 2; do
>   key=$(jq -r ".unseal_keys_b64[$i]" /root/vault-transit-init.json)
>   VAULT_SKIP_VERIFY=true vault operator unseal -address=https://127.0.0.1:8200 "$key"
> done
> ```
> **EXPECTED:** after the 3rd key, `Sealed  false`.
> **VERIFY:** `VAULT_SKIP_VERIFY=true vault status -address=https://127.0.0.1:8200`
> → `Sealed: false`, `Initialized: true`.
> **Host:** copy `vault-transit-init.json` to `~/.nexus/secrets/` (locked ACL),
> then `shred -u /root/vault-transit-init.json` on the node (keep only the host copy).

> **Step 5.2.3 — Enable transit, create the seal key, policy, and cluster token**
> **WHERE:** `vault-transit`, root shell (export the root token from the init JSON).
> **WHY:** stand up the one job this node does — a transit key
> `nexus-cluster-unseal`, a least-privilege policy (encrypt/decrypt on *that key
> only*), and a long-lived token bound to it. That token is what `vault-1/2/3`
> use to auto-unseal; if a cluster node is compromised, the token grants nothing
> but seal wrap/unwrap.
> **WHAT:**
> ```bash
> export VAULT_SKIP_VERIFY=true
> export VAULT_ADDR=https://127.0.0.1:8200
> export VAULT_TOKEN=$(jq -r .root_token ~/vault-transit-init.json 2>/dev/null || echo PASTE_ROOT_TOKEN)
>
> vault secrets enable transit
> vault write -f transit/keys/nexus-cluster-unseal
>
> vault policy write nexus-cluster-unseal - <<'EOF'
> path "transit/encrypt/nexus-cluster-unseal" { capabilities = ["update"] }
> path "transit/decrypt/nexus-cluster-unseal" { capabilities = ["update"] }
> EOF
>
> # Long-lived (non-expiring, 720h period) token for the cluster's seal config
> vault token create -policy=nexus-cluster-unseal -ttl=0 -period=720h \
>   -display-name=cluster-unseal -format=json | tee /root/cluster-unseal-token.json
> ```
> **EXPECTED:** the token JSON prints; note `.auth.client_token`.
> **VERIFY:** `vault read transit/keys/nexus-cluster-unseal` returns the key;
> `vault token lookup $(jq -r .auth.client_token /root/cluster-unseal-token.json) | grep policies`
> shows `nexus-cluster-unseal`. **Save the client_token** — §5.3 needs it. Copy
> `cluster-unseal-token.json` to the build host and shred the node copy.

### 5.3 — The `seal-transit.hcl` drop-in (used by `vault-1/2/3`)

> **Step 5.3.1 — Compose the seal stanza (you'll drop it on each cluster node)**
> **WHERE:** reference — the exact file you write in §5.4.2 / §5.5.1 / §5.6.1.
> **WHY:** presence of `/etc/vault.d/seal-transit.hcl` (merged by the
> directory-mode `-config`) switches a node into **transit-seal** mode — it
> auto-unseals by calling `vault-transit`. The token is the
> `nexus-cluster-unseal` client token from §5.2.3.
> **WHAT (the file content; substitute your token):**
> ```hcl
> seal "transit" {
>   address         = "https://192.168.70.124:8200"
>   token           = "<CLUSTER_UNSEAL_TOKEN>"
>   disable_renewal = "false"
>   key_name        = "nexus-cluster-unseal"
>   mount_path      = "transit/"
>   tls_skip_verify = "true"
> }
> ```
> **EXPECTED / VERIFY:** n/a here — it's installed + verified per node below.
> `tls_skip_verify=true` is acceptable on the lab's trusted backplane during
> bootstrap (vault-transit still has its self-signed cert); Guide 04 issues its
> cert from PKI.

### 5.4 — `vault-1` (init leader, transit auto-unseal)

> **Step 5.4.1 — TLS cert + `vault.hcl` for `vault-1`**
> **WHERE:** `vault-1` (`192.168.70.121`), root shell.
> **WHY:** identical shape to `vault-transit`'s config but with `vault-1`'s
> `node_id` and addresses (Raft RPC on the `.10.121` backplane; API on
> `.70.121`).
> **WHAT:**
> ```bash
> openssl req -new -newkey rsa:4096 -days 3650 -nodes -x509 \
>   -subj "/CN=vault-1.nexus.lab" \
>   -addext "subjectAltName=DNS:vault-1.nexus.lab,DNS:vault-1,IP:192.168.70.121,IP:192.168.10.121,IP:127.0.0.1" \
>   -keyout /etc/vault.d/tls/vault.key -out /etc/vault.d/tls/vault.crt
> chown vault:vault /etc/vault.d/tls/vault.* ; chmod 600 /etc/vault.d/tls/vault.key ; chmod 644 /etc/vault.d/tls/vault.crt
>
> cat > /etc/vault.d/vault.hcl <<'EOF'
> ui            = true
> disable_mlock = false
> cluster_name  = "nexus-vault"
> storage "raft" {
>   path    = "/opt/vault/data"
>   node_id = "vault-1"
> }
> listener "tcp" {
>   address       = "0.0.0.0:8200"
>   tls_cert_file = "/etc/vault.d/tls/vault.crt"
>   tls_key_file  = "/etc/vault.d/tls/vault.key"
> }
> cluster_addr = "https://192.168.10.121:8201"
> api_addr     = "https://192.168.70.121:8200"
> telemetry { prometheus_retention_time = "24h"  disable_hostname = true }
> log_level  = "info"
> log_format = "json"
> EOF
> chown root:vault /etc/vault.d/vault.hcl ; chmod 640 /etc/vault.d/vault.hcl
> ```
> **EXPECTED:** files written.
> **VERIFY:** `grep node_id /etc/vault.d/vault.hcl` → `vault-1`.

> **Step 5.4.2 — Drop `seal-transit.hcl` and start `vault-1`**
> **WHERE:** `vault-1`, root shell.
> **WHY:** with the seal drop-in present, `vault-1` will auto-unseal via
> `vault-transit` the moment it's initialized.
> **WHAT:** (paste the `nexus-cluster-unseal` token from §5.2.3)
> ```bash
> cat > /etc/vault.d/seal-transit.hcl <<'EOF'
> seal "transit" {
>   address         = "https://192.168.70.124:8200"
>   token           = "<CLUSTER_UNSEAL_TOKEN>"
>   disable_renewal = "false"
>   key_name        = "nexus-cluster-unseal"
>   mount_path      = "transit/"
>   tls_skip_verify = "true"
> }
> EOF
> chown root:vault /etc/vault.d/seal-transit.hcl ; chmod 640 /etc/vault.d/seal-transit.hcl
> systemctl enable --now vault.service
> ```
> **EXPECTED:** `vault.service` active.
> **VERIFY:** `VAULT_SKIP_VERIFY=true vault status -address=https://127.0.0.1:8200`
> → `Initialized: false`, `Sealed: true`, **`Seal Type: transit`**.

> **Step 5.4.3 — Initialize `vault-1` (recovery keys; transit auto-unseals)**
> **WHERE:** `vault-1`, root shell. **Save the output on the build host.**
> **WHY:** in transit-seal mode, `init` uses `-recovery-shares/-recovery-threshold`
> (not `-key-shares`); it emits **recovery** keys (break-glass only) + the root
> token, then `vault-1` auto-unseals via transit — no manual unseal.
> **WHAT:**
> ```bash
> VAULT_SKIP_VERIFY=true vault operator init -format=json \
>   -recovery-shares=5 -recovery-threshold=3 -address=https://127.0.0.1:8200 \
>   > /root/vault-cluster-init.json
> cat /root/vault-cluster-init.json    # copy to build host; contains root_token + recovery_keys_b64
> sleep 5
> VAULT_SKIP_VERIFY=true vault status -address=https://127.0.0.1:8200
> ```
> **EXPECTED:** init succeeds; after ~5 s `Sealed  false` (auto-unsealed via
> transit) and `HA Mode  active`.
> **VERIFY:** `vault status` shows `Sealed: false`, `Initialized: true`,
> `Seal Type: transit`. **Host:** copy `vault-cluster-init.json` to
> `~/.nexus/secrets/`, shred the node copy.
>
> > If `vault-1` stays **sealed** here, `vault-transit` is unreachable or the
> > token in `seal-transit.hcl` is wrong — there are no Shamir keys to fall back
> > on in transit mode. Check `vault-transit` is unsealed (§5.2.2) and the token.

### 5.5 — `vault-2` (raft join)

> **Step 5.5.1 — TLS + `vault.hcl` + `seal-transit.hcl`, then start**
> **WHERE:** `vault-2` (`192.168.70.122`), root shell.
> **WHY:** same config as `vault-1` with `vault-2`'s identity; the same seal
> drop-in so it auto-unseals after joining.
> **WHAT:**
> ```bash
> openssl req -new -newkey rsa:4096 -days 3650 -nodes -x509 \
>   -subj "/CN=vault-2.nexus.lab" \
>   -addext "subjectAltName=DNS:vault-2.nexus.lab,DNS:vault-2,IP:192.168.70.122,IP:192.168.10.122,IP:127.0.0.1" \
>   -keyout /etc/vault.d/tls/vault.key -out /etc/vault.d/tls/vault.crt
> chown vault:vault /etc/vault.d/tls/vault.* ; chmod 600 /etc/vault.d/tls/vault.key ; chmod 644 /etc/vault.d/tls/vault.crt
>
> cat > /etc/vault.d/vault.hcl <<'EOF'
> ui = true
> disable_mlock = false
> cluster_name  = "nexus-vault"
> storage "raft" { path = "/opt/vault/data"  node_id = "vault-2" }
> listener "tcp" {
>   address       = "0.0.0.0:8200"
>   tls_cert_file = "/etc/vault.d/tls/vault.crt"
>   tls_key_file  = "/etc/vault.d/tls/vault.key"
> }
> cluster_addr = "https://192.168.10.122:8201"
> api_addr     = "https://192.168.70.122:8200"
> telemetry { prometheus_retention_time = "24h"  disable_hostname = true }
> log_level = "info"
> log_format = "json"
> EOF
> chown root:vault /etc/vault.d/vault.hcl ; chmod 640 /etc/vault.d/vault.hcl
>
> # Same seal drop-in as vault-1 (paste the cluster-unseal token)
> cat > /etc/vault.d/seal-transit.hcl <<'EOF'
> seal "transit" {
>   address         = "https://192.168.70.124:8200"
>   token           = "<CLUSTER_UNSEAL_TOKEN>"
>   disable_renewal = "false"
>   key_name        = "nexus-cluster-unseal"
>   mount_path      = "transit/"
>   tls_skip_verify = "true"
> }
> EOF
> chown root:vault /etc/vault.d/seal-transit.hcl ; chmod 640 /etc/vault.d/seal-transit.hcl
> systemctl enable --now vault.service
> ```
> **EXPECTED:** `vault.service` active; `vault status` → `Initialized: false`,
> `Sealed: true` (not yet joined).
> **VERIFY:** `grep node_id /etc/vault.d/vault.hcl` → `vault-2`;
> `grep 192.168.70.122 /etc/vault.d/vault.hcl` matches `api_addr`.

> **Step 5.5.2 — Trust `vault-1`'s cert, then raft-join**
> **WHERE:** `vault-2`, root shell.
> **WHY:** `vault-2` must verify `vault-1`'s self-signed API cert to join, so
> install it into the system trust store first. After joining, transit
> auto-unseals `vault-2` — no manual unseal. (Guide 04's shared PKI retires this
> per-node cert shuffle.)
> **WHAT:** (copy `vault-1`'s cert over first — from the build host:
> `scp nexusadmin@192.168.70.121:... ` or `ssh ... 'sudo cat ...'`)
> ```bash
> # On vault-2: fetch vault-1's cert (run from the build host, or paste it in)
> ssh -i ~/.ssh/nexus_gateway_ed25519 nexusadmin@192.168.70.121 'sudo cat /etc/vault.d/tls/vault.crt' \
>   | sudo tee /usr/local/share/ca-certificates/vault-leader.crt >/dev/null
> sudo update-ca-certificates
>
> VAULT_SKIP_VERIFY=true vault operator raft join -address=https://127.0.0.1:8200 https://192.168.70.121:8200
> sleep 15
> VAULT_SKIP_VERIFY=true vault status -address=https://127.0.0.1:8200
> ```
> **EXPECTED:** `raft join` reports `Joined: true`; after ~15 s `Sealed  false`
> (auto-unsealed).
> **VERIFY:** `vault status` → `Sealed: false`, `HA Mode: standby`.

### 5.6 — `vault-3` (raft join)

> **Step 5.6.1 — TLS + `vault.hcl` + `seal-transit.hcl`, then start**
> **WHERE:** `vault-3` (`192.168.70.123`), root shell.
> **WHY:** identical to `vault-2` with `vault-3`'s identity.
> **WHAT:**
> ```bash
> openssl req -new -newkey rsa:4096 -days 3650 -nodes -x509 \
>   -subj "/CN=vault-3.nexus.lab" \
>   -addext "subjectAltName=DNS:vault-3.nexus.lab,DNS:vault-3,IP:192.168.70.123,IP:192.168.10.123,IP:127.0.0.1" \
>   -keyout /etc/vault.d/tls/vault.key -out /etc/vault.d/tls/vault.crt
> chown vault:vault /etc/vault.d/tls/vault.* ; chmod 600 /etc/vault.d/tls/vault.key ; chmod 644 /etc/vault.d/tls/vault.crt
>
> cat > /etc/vault.d/vault.hcl <<'EOF'
> ui = true
> disable_mlock = false
> cluster_name  = "nexus-vault"
> storage "raft" { path = "/opt/vault/data"  node_id = "vault-3" }
> listener "tcp" {
>   address       = "0.0.0.0:8200"
>   tls_cert_file = "/etc/vault.d/tls/vault.crt"
>   tls_key_file  = "/etc/vault.d/tls/vault.key"
> }
> cluster_addr = "https://192.168.10.123:8201"
> api_addr     = "https://192.168.70.123:8200"
> telemetry { prometheus_retention_time = "24h"  disable_hostname = true }
> log_level = "info"
> log_format = "json"
> EOF
> chown root:vault /etc/vault.d/vault.hcl ; chmod 640 /etc/vault.d/vault.hcl
>
> cat > /etc/vault.d/seal-transit.hcl <<'EOF'
> seal "transit" {
>   address         = "https://192.168.70.124:8200"
>   token           = "<CLUSTER_UNSEAL_TOKEN>"
>   disable_renewal = "false"
>   key_name        = "nexus-cluster-unseal"
>   mount_path      = "transit/"
>   tls_skip_verify = "true"
> }
> EOF
> chown root:vault /etc/vault.d/seal-transit.hcl ; chmod 640 /etc/vault.d/seal-transit.hcl
> systemctl enable --now vault.service
> ```
> **EXPECTED:** `vault.service` active; `vault status` → `Sealed: true`.
> **VERIFY:** `grep node_id /etc/vault.d/vault.hcl` → `vault-3`.

> **Step 5.6.2 — Trust `vault-1`'s cert, then raft-join**
> **WHERE:** `vault-3`, root shell.
> **WHY / WHAT / EXPECTED:** identical to §5.5.2 but on `vault-3`:
> ```bash
> ssh -i ~/.ssh/nexus_gateway_ed25519 nexusadmin@192.168.70.121 'sudo cat /etc/vault.d/tls/vault.crt' \
>   | sudo tee /usr/local/share/ca-certificates/vault-leader.crt >/dev/null
> sudo update-ca-certificates
> VAULT_SKIP_VERIFY=true vault operator raft join -address=https://127.0.0.1:8200 https://192.168.70.121:8200
> sleep 15
> VAULT_SKIP_VERIFY=true vault status -address=https://127.0.0.1:8200
> ```
> **VERIFY:** `vault status` → `Sealed: false`, `HA Mode: standby`.

### 5.7 — Confirm the cluster + post-init mounts

> **Step 5.7.1 — Verify the 3-peer Raft cluster**
> **WHERE:** `vault-1`, root shell (root token from §5.4.3).
> **WHY:** prove all three nodes are raft peers with one leader.
> **WHAT:**
> ```bash
> export VAULT_SKIP_VERIFY=true VAULT_ADDR=https://127.0.0.1:8200
> export VAULT_TOKEN=$(jq -r .root_token /root/vault-cluster-init.json 2>/dev/null || echo PASTE_ROOT_TOKEN)
> vault operator raft list-peers
> ```
> **EXPECTED:** **3** servers — `vault-1` `leader`, `vault-2`/`vault-3`
> `follower`, all `voter`.
> **VERIFY:** `vault operator raft list-peers -format=json | jq '.data.config.servers | length'`
> → `3`.

> **Step 5.7.2 — Mount KV-v2, enable userpass + AppRole, write the smoke secret**
> **WHERE:** `vault-1`, root shell (root token exported).
> **WHY:** make the cluster usable: a versioned secret store at `nexus/`, a human
> operator login (userpass), the machine-login method every later tier uses
> (AppRole), and a canary secret the smoke gate reads.
> **WHAT:**
> ```bash
> export VAULT_SKIP_VERIFY=true VAULT_ADDR=https://127.0.0.1:8200   # VAULT_TOKEN already set
>
> vault secrets enable -path=nexus -version=2 kv
> vault auth enable userpass
> vault write auth/userpass/users/operator password='<PICK_A_STRONG_PASSWORD>' policies=default
> vault auth enable approle
> vault write auth/approle/role/nexus-bootstrap \
>   token_policies=default token_ttl=1h token_max_ttl=4h secret_id_ttl=24h
> vault kv put nexus/smoke/canary value=ok timestamp="$(date -Iseconds)" phase=0.D.1
> vault kv get nexus/smoke/canary
> ```
> **EXPECTED:** each command succeeds; the final `kv get` prints
> `value  ok`.
> **VERIFY:** `vault secrets list | grep '^nexus/'` and
> `vault auth list | grep -E 'userpass|approle'` both present.

---

## 6. Validation — by-hand acceptance smoke

From the **host** (and a couple of on-node checks). All four nodes powered on.

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | All four API endpoints reachable | `121,122,123,124 \| % { Test-NetConnection 192.168.70.$_ -Port 8200 }` | all `True` |
| 2 | `vault-transit` unsealed | `ssh …@124 'VAULT_SKIP_VERIFY=true vault status -address=https://127.0.0.1:8200 \| grep Sealed'` | `Sealed false` |
| 3 | Transit key present | `ssh …@124 'VAULT_TOKEN=… VAULT_SKIP_VERIFY=true vault read transit/keys/nexus-cluster-unseal'` | returns the key |
| 4 | `vault-1` unsealed, transit seal | `ssh …@121 '… vault status'` | `Sealed false`, `Seal Type transit`, `HA Mode active` |
| 5 | `vault-2` unsealed, standby | `ssh …@122 '… vault status'` | `Sealed false`, `HA Mode standby` |
| 6 | `vault-3` unsealed, standby | `ssh …@123 '… vault status'` | `Sealed false`, `HA Mode standby` |
| 7 | 3 raft peers | `ssh …@121 'VAULT_TOKEN=… … vault operator raft list-peers'` | 3 servers, 1 leader, all voter |
| 8 | KV-v2 + auth methods | `ssh …@121 '… vault secrets list; vault auth list'` | `nexus/`, `userpass/`, `approle/` |
| 9 | Smoke secret readable | `ssh …@121 '… vault kv get nexus/smoke/canary'` | `value  ok` |
| 10 | **Auto-unseal survives reboot** | reboot `vault-2`; after boot: `ssh …@122 '… vault status \| grep Sealed'` | `Sealed false` **without** any manual unseal (transit did it) |
| 11 | **HA leader failover** | `ssh …@121 'sudo systemctl stop vault'`; then `ssh …@122 '… vault status \| grep "HA Mode"'` | one of `vault-2`/`vault-3` becomes `active` (then restart vault-1) |

**Checks 1–9 green ⇒ cluster healthy.** 10 proves auto-unseal; 11 proves Raft
HA — the two reasons this is a 3-node cluster.

> **The boot-race caveat (memory `feedback_vault_transit_boot_race_recovery`):**
> if the **whole host** reboots, `vault-transit` comes up **sealed** (it's
> Shamir), so `vault-1/2/3` can't auto-unseal until you unseal `vault-transit`
> first. Recovery order: unseal `vault-transit` (its 3 keys), then restart
> `vault.service` on the cluster nodes. See §8.

---

## 7. Teardown / reset

```powershell
# Stop + delete all four VMs (Raft data + transit keys live on their disks).
foreach ($n in 'vault-3','vault-2','vault-1','vault-transit') {
  $vmx = "H:\VMS\NexusPlatform\01-foundation\$n\$n.vmx"
  & 'C:\Program Files\VMware\VMware Workstation\vmrun.exe' stop $vmx soft 2>$null
  & 'C:\Program Files\VMware\VMware Workstation\vmrun.exe' deleteVM $vmx
  Remove-Item "H:\VMS\NexusPlatform\01-foundation\$n" -Recurse -Force
}
```

To reset just the cluster (keep `vault-transit`): on each of `vault-1/2/3`,
`systemctl stop vault`, `rm -rf /opt/vault/data/*`, delete
`/etc/vault.d/vault.hcl` state, and re-run §5.4–5.7.

> **Cold-rebuild note (memory `feedback_cold_rebuild_stale_kv_tokens`):** later
> tiers store bootstrap tokens in Vault KV. After a destroy+rebuild of this
> cluster, those old tokens are gone with it — re-seed them when rebuilding the
> dependent tiers; don't expect a stale token to work against a fresh cluster.

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `vault-1` stays **sealed** after init (transit mode) | `vault-transit` unreachable or the token in `seal-transit.hcl` is wrong/expired | Confirm `vault-transit` is unsealed (§5.2.2) + `8200` reachable from the cluster node; re-check the `nexus-cluster-unseal` token (§5.2.3). No Shamir fallback exists in transit mode. |
| Whole host rebooted → cluster won't come up | `vault-transit` boots **sealed** (it's Shamir), so the cluster can't auto-unseal | Unseal `vault-transit` first (its 3 keys), then `systemctl restart vault` on `vault-1/2/3`. Recover order: **network → vault-transit → cluster**. See `feedback_vault_transit_boot_race_recovery`. |
| `raft join` fails `x509: certificate signed by unknown authority` | Follower doesn't trust `vault-1`'s self-signed cert | Install `vault-1`'s `vault.crt` into the follower's system trust (`/usr/local/share/ca-certificates/` + `update-ca-certificates`) **before** joining (§5.5.2). |
| `vault.service` won't start, `ConditionFileNotEmpty` not met | `/etc/vault.d/vault.hcl` missing or empty | Write `vault.hcl` (§5.2.1 / §5.4.1) before `systemctl start`. |
| Vault errors on `mlock` / won't lock memory | `setcap` not applied to the binary | `setcap cap_ipc_lock=+ep /usr/local/bin/vault` (§5.1.2); confirm `LimitMEMLOCK=infinity` in the unit. |
| Raft peers stuck at 1, followers can't reach leader | Backplane (VMnet10) port `8201` blocked or `cluster_addr` wrong | Confirm each `vault.hcl` `cluster_addr` is the node's **`.10.x:8201`**, and `nftables` allows `8201` from `192.168.10.0/24` on `nic1` (Guide 00 §B.4.4 backplane rule + the cluster's port). |
| `vault status` over SSH errors on TLS | self-signed bootstrap cert | Use `VAULT_SKIP_VERIFY=true` (every command in this guide does) until Guide 04's PKI lands. |

---

### Cross-references

- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (Vault `.121`–`.124`)
- **VM inventory:** `nexus-platform-plan/docs/infra/vms.yaml` (`foundation` cluster)
- **Automated equivalents:** `nexus-infra-vmware/packer/vault/` + `terraform/envs/security/role-overlay-vault-{transit-bringup,cluster-seal-config,cluster}.tf`
- **Previous guide:** [`02-foundation-ad-ds-forest.md`](./02-foundation-ad-ds-forest.md)
- **Next guide:** Guide 04 — Foundation · Vault PKI + auto-unseal + LDAP (PKI root/intermediate, LDAPS auth, secrets/ldap rotation). See [`INDEX.md`](../INDEX.md).
