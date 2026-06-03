# Guide 05 — Orchestration · Docker Swarm + Consul + Nomad + Portainer

> **Mirrors:** `nexus-infra-swarm-nomad` — the `swarm-node` Packer template
> (`consul-{server,client}.hcl`, `nomad-{server,client}.hcl`, `nftables.conf`,
> `swarm-node-firstboot.sh`) + the `swarm-nomad` env overlays
> (`swarm-init`, `consul-{gossip,tls,acl}`, `nomad-{tls,acl,consul-rewire,vault}`,
> `portainer-{nfs,tls,dns,admin,stack}`). Where the automated lab drives every
> step from the build host over SSH and renders secrets via per-node Vault
> Agents, this guide installs each binary by hand and **issues certs/tokens
> directly with the `vault` CLI** (per CONVENTIONS §5).

---

## 1. Overview & purpose

This is the lab's **orchestration tier** — the first non-foundation tier, and the
first to invoke **Guide 04 Part D's scaffolding pattern** for real. Six Debian
nodes form a clustered container platform:

- **3 managers** (`swarm-manager-1/2/3`, `.111–.113`) + **3 workers**
  (`swarm-worker-1/2/3`, `.131–.133`).
- **Docker Swarm** — the container orchestrator (3-manager Raft quorum + 3
  workers).
- **Consul** — service discovery + health + KV + service mesh, hardened with
  **gossip encryption → mTLS → ACL deny-mode**.
- **Nomad** — a workload scheduler co-resident with Swarm, hardened with
  **mTLS → ACL → Consul-over-HTTPS → Vault integration**.
- **Portainer CE** — the Swarm management UI, deployed *as a Swarm service*, with
  its state on an **NFSv4 share from `nexus-gateway`** so it survives rescheduling.

**Why all four:** Swarm runs the containers; Consul gives them discovery + a mesh;
Nomad adds batch/system scheduling alongside Swarm on the same Docker engine;
Portainer is the operator's pane of glass. Every control plane is secured with
Vault-PKI mTLS and ACLs — this tier is where the foundation's PKI + AppRole
machinery first earns its keep.

**Dependency:**
- **Guides 00–04 complete** — the 6-VM foundation (gateway, AD, Vault HA, PKI,
  LDAP) is alive. Specifically this guide needs: Guide 01's **NFSv4 export**
  (`/srv/nfs/portainer-data`, clients `.111–.113`), Guide 04's **PKI**
  (`pki_int/issue/...`) and **AppRole/KV scaffolding pattern** (Part D), and the
  gateway's **dnsmasq** for the round-robin `portainer.nexus.lab` record.
- Six `deb13` nodes baselined per Guide 00 §5.B with dual-NIC static IPs.

> **By-hand divergence from the automated lab:** the automated path runs a
> per-node **Vault Agent** that AppRole-authenticates and *renders* certs/tokens
> from templates. By hand we skip the agent and **issue each cert/token directly
> with the `vault` CLI on `vault-1`, then scp it to the node** — the same idiom
> Guides 03–04 used for the raft-join and LDAPS certs. The AppRole + KV scaffolding
> from Guide 04 Part D is still created (it's how a node *would* self-renew), but
> cert placement here is manual.

---

## 2. Component primer

- **Docker Swarm.** Docker's built-in orchestrator: **managers** hold a Raft
  quorum of cluster state and schedule **services**; **workers** run tasks.
  *Why:* simplest multi-node Docker orchestration, native to the engine.
  *Otherwise:* Kubernetes (heavier; this lab keeps K8s out of the
  foundation/orchestration tier deliberately), or single-host Docker (no HA).
- **Consul.** A service-discovery + health-check + KV + service-mesh agent. A
  **server** quorum (the 3 managers) holds state; **client** agents (the 3
  workers) forward to them. Hardened in three moves: **gossip encryption** (a
  shared symmetric key over the Serf gossip LAN), **TLS** (mTLS for internal RPC
  + Raft; server-TLS for the HTTPS API), and **ACLs** (default-deny; every agent
  + operator carries a token). *Why:* services find each other by name + health,
  not hardcoded IPs. *Otherwise:* DNS-only (no health/mesh) or etcd (no service
  catalog).
- **Nomad.** A workload scheduler that runs alongside Swarm on the same Docker
  daemon — servers (managers) schedule, clients (workers) execute. Hardened with
  **mTLS**, **ACLs**, **Consul-over-HTTPS**, and **Vault integration** (jobs get
  Vault-issued secrets). *Why:* batch/system jobs + a second scheduler exercise
  that don't fit the Swarm service model. *Otherwise:* Swarm-only.
- **Gossip vs. RPC vs. Raft (Consul/Nomad).** *Gossip* (Serf, UDP+TCP 8301 /
  4648) is the membership/failure-detection layer — encrypted with a shared key.
  *RPC* (8300 / 4647) is agent↔server calls — mTLS. *Raft* is the consensus log
  among servers — also over the RPC TLS. All bind to the **VMnet10 backplane**.
- **Vault Agent / direct issue.** The automated lab's per-node **Vault Agent**
  logs in via AppRole and renders certs + tokens from templates, auto-renewing.
  By hand we issue the same artifacts with the `vault` CLI and place them — see
  the divergence note in §1.
- **Portainer CE + NFSv4 shared state.** Portainer is the Swarm UI. CE has **no
  native HA** — one Server replica runs at a time, but Swarm reschedules it
  across managers; its `/data` (BoltDB) lives on the **`fsid=0` NFSv4 export**
  from the gateway so a reschedule keeps the state. The **agent** runs *global*
  (one per node) for full-cluster visibility. *Otherwise:* losing Portainer state
  on every manager failover.

---

## 3. Prerequisites

| # | Requirement | One-command verify (host) |
|---|---|---|
| 1 | Foundation alive (Guides 00–04): gateway, Vault HA + PKI + LDAP | `vault status` via `VAULT_CACERT` succeeds; `Resolve-DnsName -Server 192.168.70.1 vault-1.nexus.lab` resolves |
| 2 | Six `deb13` nodes baselined (Guide 00 §5.B), dual-NIC static `.111–.113` + `.131–.133` | `111,112,113,131,132,133 \| % { Test-NetConnection 192.168.70.$_ -Port 22 }` → all `True` |
| 3 | Guide 01 NFSv4 export live, allow-list = managers | `ssh …@1 'sudo exportfs -v \| grep portainer-data'` → `.111/.112/.113` |
| 4 | Vault root token on the build host | `Test-Path ~/.nexus/secrets/vault-cluster-init.json` |
| 5 | Internet egress on the nodes (binary downloads) | `ssh …@111 'curl -sI https://releases.hashicorp.com \| head -1'` → `200` |

> Vault CLI on `vault-1` uses the root token (Part-D scaffolding) — same exports
> as Guide 04. After Guide 04, prefer `VAULT_CACERT` over `VAULT_SKIP_VERIFY`.

---

## 4. Target topology

| Node | Role | VMnet11 (nic0) | VMnet10 (nic1) | Primary MAC | vCPU / RAM / disk |
|---|---|---|---|---|---|
| `swarm-manager-1` | Swarm mgr · Consul server · Nomad server | `.70.111` | `.10.111` | `…:00:6F` | 2 / 4 GB / 40 GB |
| `swarm-manager-2` | (same) | `.70.112` | `.10.112` | `…:00:70` | 2 / 4 GB / 40 GB |
| `swarm-manager-3` | (same) | `.70.113` | `.10.113` | `…:00:71` | 2 / 4 GB / 40 GB |
| `swarm-worker-1` | Swarm wkr · Consul client · Nomad client | `.70.131` | `.10.131` | `…:00:83` | 2 / 4 GB / 40 GB |
| `swarm-worker-2` | (same) | `.70.132` | `.10.132` | `…:00:84` | 2 / 4 GB / 40 GB |
| `swarm-worker-3` | (same) | `.70.133` | `.10.133` | `…:00:85` | 2 / 4 GB / 40 GB |

> MACs follow the fleet convention (`…:00:XX` primary, `…:01:XX` secondary);
> confirm each node's exact `XX` against its Guide 00 assignment / the gateway
> reservations. Versions: **Docker** stable, **Consul 1.20.1**, **Nomad 1.9.3**.
> Datacenter `nexus-lab`. All cluster traffic on the **VMnet10 backplane**.

**Ports (all on VMnet10 except operator UIs on VMnet11):** Swarm `2377` (mgmt),
`7946` (gossip), `4789` (overlay VXLAN); Consul `8300` (RPC), `8301` (serf LAN),
`8501` (HTTPS API, VMnet11), `8600` (DNS); Nomad `4646` (HTTPS API, VMnet11),
`4647` (RPC), `4648` (serf); Portainer `9443` (HTTPS UI), `8000`, `9001` (agent).

---

## 5. Step-by-step build

> **WHERE:** node steps run as `nexusadmin`→`sudo -i`. `vault` commands run on
> **`vault-1`** (root token exported). "mgr-1" = `swarm-manager-1` (`.111`), the
> Swarm/Consul/Nomad bootstrap node.

### 5.1 — Per-node base: install Docker + Consul + Nomad, write configs (all 6)

Run §5.1.1–5.1.5 on **every** node. The only per-node differences are
hostname/IPs and **server (managers) vs. client (workers)** configs — both
spelled out below.

> **Step 5.1.1 — Install Docker Engine**
> **WHERE:** each node, root shell.
> **WHY:** the container runtime both Swarm and Nomad drive.
> **WHAT:**
> ```bash
> curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
> echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian trixie stable" \
>   > /etc/apt/sources.list.d/docker.list
> apt-get update -qq
> apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
> systemctl enable --now docker
> ```
> **EXPECTED:** Docker installs + starts.
> **VERIFY:** `docker version --format '{{.Server.Version}}'` prints a version.

> **Step 5.1.2 — Install Consul 1.20.1 + Nomad 1.9.3 binaries**
> **WHERE:** each node, root shell.
> **WHY:** the discovery + scheduling agents. Create their system users + dirs.
> **WHAT:**
> ```bash
> cd /tmp
> for prod in consul:1.20.1 nomad:1.9.3; do
>   name=${prod%%:*}; ver=${prod##*:}
>   curl -fsSL "https://releases.hashicorp.com/$name/$ver/${name}_${ver}_linux_amd64.zip" -o "$name.zip"
>   curl -fsSL "https://releases.hashicorp.com/$name/$ver/${name}_${ver}_SHA256SUMS" -o "$name.sums"
>   grep "${name}_${ver}_linux_amd64.zip" "$name.sums" | sha256sum -c -
>   unzip -o "$name.zip" ; install -m755 "$name" /usr/local/bin/ ; rm -f "$name" "$name.zip" "$name.sums"
> done
> for svc in consul nomad; do
>   groupadd --system $svc 2>/dev/null || true
>   id $svc >/dev/null 2>&1 || useradd --system -g $svc -s /usr/sbin/nologin -d /opt/$svc -M $svc
>   install -d -o $svc -g $svc -m0750 /opt/$svc /opt/$svc/data /etc/$svc.d /etc/$svc.d/tls
> done
> ```
> **EXPECTED:** both checksums `OK`; users + dirs created.
> **VERIFY:** `consul version` → `1.20.1`; `nomad version` → `1.9.3`.

> **Step 5.1.3 — Write the Consul config (server on managers, client on workers)**
> **WHERE:** each node, root shell.
> **WHY:** managers form the server quorum (`bootstrap_expect=3`); workers are
> clients. Gossip + Raft bind to the **VMnet10** backplane; the API binds broadly
> (nftables gates it). `retry_join` auto-joins via the managers' backplane IPs.
> **WHAT (on a MANAGER — substitute `@HOSTNAME@`/`@VMNET10_IP@`/`@VMNET11_IP@`):**
> ```bash
> cat > /etc/consul.d/consul.hcl <<'EOF'
> datacenter = "nexus-lab"
> data_dir   = "/opt/consul/data"
> node_name  = "@HOSTNAME@"
> log_level  = "INFO"
> server           = true
> bootstrap_expect = 3
> ui_config { enabled = true }
> bind_addr      = "@VMNET10_IP@"
> advertise_addr = "@VMNET10_IP@"
> client_addr    = "0.0.0.0"
> ports { http = 8500  dns = 8600  grpc = 8502 }
> retry_join = ["192.168.10.111","192.168.10.112","192.168.10.113"]
> connect { enabled = true }
> performance { raft_multiplier = 1 }
> EOF
> chown consul:consul /etc/consul.d/consul.hcl ; chmod 640 /etc/consul.d/consul.hcl
> ```
> **WHAT (on a WORKER — `server=false`, `client_addr=127.0.0.1`, no `bootstrap_expect`):**
> ```bash
> cat > /etc/consul.d/consul.hcl <<'EOF'
> datacenter = "nexus-lab"
> data_dir   = "/opt/consul/data"
> node_name  = "@HOSTNAME@"
> log_level  = "INFO"
> server = false
> bind_addr      = "@VMNET10_IP@"
> advertise_addr = "@VMNET10_IP@"
> client_addr    = "127.0.0.1"
> ports { http = 8500  dns = 8600  grpc = 8502 }
> retry_join = ["192.168.10.111","192.168.10.112","192.168.10.113"]
> connect { enabled = true }
> EOF
> chown consul:consul /etc/consul.d/consul.hcl ; chmod 640 /etc/consul.d/consul.hcl
> ```
> **EXPECTED:** config written with this node's real hostname/IPs.
> **VERIFY:** `consul validate /etc/consul.d/` → `Configuration is valid!`

> **Step 5.1.4 — Write the Nomad config + a systemd unit using directory `-config`**
> **WHERE:** each node, root shell.
> **WHY:** managers are Nomad servers (`bootstrap_expect=3`), workers are clients
> with the Docker driver. **Critical:** the stock Nomad unit uses
> `-config=/etc/nomad.d/nomad.hcl` (single file) — which would ignore the
> `tls.hcl`/`acl.hcl`/etc. drop-ins we add later. The unit must point at the
> **directory** `/etc/nomad.d/` (per `feedback_nomad_systemd_unit_single_file_config`).
> **WHAT (server config on a MANAGER):**
> ```bash
> cat > /etc/nomad.d/nomad.hcl <<'EOF'
> datacenter = "nexus-lab"
> data_dir   = "/opt/nomad/data"
> name       = "@HOSTNAME@"
> log_level  = "INFO"
> bind_addr  = "@VMNET10_IP@"
> addresses  { http = "0.0.0.0" }
> advertise  { http = "@VMNET10_IP@:4646"  rpc = "@VMNET10_IP@:4647"  serf = "@VMNET10_IP@:4648" }
> server     { enabled = true  bootstrap_expect = 3 }
> consul     { address = "127.0.0.1:8500" }
> ports      { http = 4646  rpc = 4647  serf = 4648 }
> EOF
> ```
> **WHAT (client config on a WORKER):**
> ```bash
> cat > /etc/nomad.d/nomad.hcl <<'EOF'
> datacenter = "nexus-lab"
> data_dir   = "/opt/nomad/data"
> name       = "@HOSTNAME@"
> log_level  = "INFO"
> bind_addr  = "@VMNET10_IP@"
> addresses  { http = "0.0.0.0" }
> advertise  { http = "@VMNET10_IP@:4646"  rpc = "@VMNET10_IP@:4647"  serf = "@VMNET10_IP@:4648" }
> client     { enabled = true  reserved { cpu = 200  memory = 512 } }
> consul     { address = "127.0.0.1:8500" }
> plugin "docker" { config { allow_privileged = false  volumes { enabled = true } } }
> EOF
> ```
> **WHAT (the systemd units — both node types):**
> ```bash
> cat > /etc/systemd/system/consul.service <<'EOF'
> [Unit]
> Description=Consul
> After=network-online.target
> Wants=network-online.target
> [Service]
> User=consul
> Group=consul
> ExecStart=/usr/local/bin/consul agent -config-dir=/etc/consul.d/
> ExecReload=/bin/kill --signal HUP $MAINPID
> Restart=on-failure
> LimitNOFILE=65536
> [Install]
> WantedBy=multi-user.target
> EOF
>
> cat > /etc/systemd/system/nomad.service <<'EOF'
> [Unit]
> Description=Nomad
> After=network-online.target
> Wants=network-online.target
> [Service]
> User=root
> Group=root
> ExecStart=/usr/local/bin/nomad agent -config=/etc/nomad.d/
> ExecReload=/bin/kill --signal HUP $MAINPID
> KillMode=process
> Restart=on-failure
> LimitNOFILE=65536
> [Install]
> WantedBy=multi-user.target
> EOF
> chmod 640 /etc/nomad.d/nomad.hcl
> systemctl daemon-reload
> ```
> **EXPECTED:** configs + units written. (Nomad runs as `root` — it needs the
> Docker socket + to mount task volumes.)
> **VERIFY:** `systemctl cat nomad.service | grep ExecStart` → `-config=/etc/nomad.d/`
> (directory, not the single `.hcl`).

> **Step 5.1.5 — Install the swarm-node nftables ruleset, then start the agents**
> **WHERE:** each node, root shell.
> **WHY:** the baseline only opens SSH + node_exporter. The swarm-node ruleset
> **trusts the whole VMnet10 backplane on `nic1`** (too many cluster ports to
> enumerate), opens the operator UIs (`8500/8501/4646`) from VMnet11, and — for
> Swarm's ingress mesh — adds **`forward`-chain accepts** for `docker_gwbridge`/
> `docker0` (without these, `host:9443 → DNAT → container` is dropped; per
> `feedback_nftables_flush_ruleset_wipes_docker` also **restart Docker after any
> `nft -f`**).
> **WHAT:**
> ```bash
> cat > /etc/nftables.conf <<'EOF'
> #!/usr/sbin/nft -f
> flush ruleset
> table inet filter {
>     chain input {
>         type filter hook input priority 0; policy drop;
>         iif "lo" accept
>         ct state { established, related } accept
>         ct state invalid drop
>         ip protocol icmp accept
>         ip6 nexthdr icmpv6 accept
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 22   accept comment "SSH from VMnet11"
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport 9100 accept comment "node_exporter"
>         iifname "nic1" ip saddr 192.168.10.0/24 accept comment "trusted cluster backplane (VMnet10)"
>         iifname "nic0" ip saddr 192.168.70.0/24 tcp dport { 8500, 8501, 4646, 9443 } accept comment "Consul + Nomad + Portainer UIs from VMnet11"
>         counter drop
>     }
>     chain forward {
>         type filter hook forward priority 0; policy drop;
>         ct state { established, related } accept
>         iifname "docker_gwbridge" accept
>         oifname "docker_gwbridge" accept
>         iifname "docker0" accept
>         oifname "docker0" accept
>     }
>     chain output { type filter hook output priority 0; policy accept; }
> }
> EOF
> chmod 755 /etc/nftables.conf
> nft -c -f /etc/nftables.conf && systemctl enable --now nftables && nft -f /etc/nftables.conf
> systemctl restart docker      # re-add Docker's iptables-nft rules wiped by `nft -f`
> systemctl enable --now consul nomad
> ```
> **EXPECTED:** nftables loads; docker/consul/nomad start.
> **VERIFY:** `systemctl is-active docker consul nomad` → three `active`.

### 5.2 — Initialize the Swarm + join all nodes

> **Step 5.2.1 — `docker swarm init` on `swarm-manager-1`**
> **WHERE:** mgr-1 (`.111`), root shell.
> **WHY:** bootstrap the Swarm Raft on the backplane IP, then capture the join
> tokens. Advertising on `.10.111` keeps cluster traffic on VMnet10.
> **WHAT:**
> ```bash
> docker swarm init --advertise-addr 192.168.10.111 --listen-addr 192.168.10.111:2377
> docker swarm join-token -q manager   # -> MANAGER_TOKEN (note it)
> docker swarm join-token -q worker    # -> WORKER_TOKEN  (note it)
> ```
> **EXPECTED:** "Swarm initialized"; two tokens printed.
> **VERIFY:** `docker node ls` → mgr-1 `Leader`.

> **Step 5.2.2 — Join `swarm-manager-2/3` as managers**
> **WHERE:** mgr-2 (`.112`) and mgr-3 (`.113`), root shell.
> **WHY:** complete the 3-manager Raft quorum (tolerates 1 manager loss).
> **WHAT (on each, with its own advertise IP):**
> ```bash
> # mgr-2:
> docker swarm join --token <MANAGER_TOKEN> --advertise-addr 192.168.10.112 192.168.10.111:2377
> # mgr-3:
> docker swarm join --token <MANAGER_TOKEN> --advertise-addr 192.168.10.113 192.168.10.111:2377
> ```
> **EXPECTED:** "This node joined a swarm as a manager."
> **VERIFY (on mgr-1):** `docker node ls` → 3 managers, one `Leader` two `Reachable`.

> **Step 5.2.3 — Join `swarm-worker-1/2/3` as workers**
> **WHERE:** wrk-1/2/3 (`.131/.132/.133`), root shell.
> **WHY:** add the task-running capacity.
> **WHAT (on each):**
> ```bash
> # wrk-1: --advertise-addr 192.168.10.131  | wrk-2: .132 | wrk-3: .133
> docker swarm join --token <WORKER_TOKEN> --advertise-addr 192.168.10.13X 192.168.10.111:2377
> ```
> **EXPECTED:** "This node joined a swarm as a worker."
> **VERIFY (on mgr-1):** `docker node ls` → **6 nodes**, all `Ready`/`Active`.

> **Step 5.2.4 — Confirm Consul + Nomad clustered**
> **WHERE:** mgr-1, root shell.
> **WHY:** the agents started in §5.1.5 should have auto-joined via `retry_join` /
> Consul discovery.
> **WHAT:** `consul members` and `nomad server members`
> **EXPECTED:** Consul lists **6** members (3 server + 3 client); Nomad lists **3**
> servers.
> **VERIFY:** `consul members | grep -c alive` → `6`;
> `nomad server members | grep -c alive` → `3`.

### 5.3 — Tier scaffolding in Vault (Guide 04 Part D, for this tier)

> **Step 5.3.1 — Create PKI roles, AppRoles, and the gossip key in Vault**
> **WHERE:** `vault-1`, root shell (root token exported).
> **WHY:** invoke Guide 04 Part D for the orchestration tier — three cert-issuing
> roles (Consul/Nomad/Portainer servers), a per-node AppRole + scoped policy
> (machine identity), and the Consul **gossip key** in KV. The gossip key is the
> one shared secret; the certs are issued per-node in the harden steps.
> **WHAT:**
> ```bash
> # (1) PKI roles — server identities. Consul needs server.nexus-lab.consul in
> #     allowed_domains for verify_server_hostname; Nomad needs server.global.nomad.
> vault write pki_int/roles/consul-server \
>   allowed_domains='consul.nexus.lab,nexus.lab,server.nexus-lab.consul,localhost' \
>   allow_subdomains=true allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
> vault write pki_int/roles/nomad-server \
>   allowed_domains='global.nomad,nexus.lab,server.global.nomad,client.global.nomad,localhost' \
>   allow_subdomains=true allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
> vault write pki_int/roles/portainer-server \
>   allowed_domains='portainer.nexus.lab,nexus.lab,localhost' \
>   allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=false key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
>
> # (2) gossip key in KV (one shared symmetric key for the whole cluster)
> vault kv put nexus/swarm/consul-gossip-key gossip_key="$(consul keygen)"
>
> # (3) a tier policy + AppRole (machine identity; see Guide 04 §5.D.1)
> vault policy write swarm - <<'EOF'
> path "pki_int/issue/consul-server"    { capabilities = ["create","update"] }
> path "pki_int/issue/nomad-server"     { capabilities = ["create","update"] }
> path "pki_int/issue/portainer-server" { capabilities = ["create","update"] }
> path "nexus/data/swarm/*"             { capabilities = ["read"] }
> path "pki/cert/ca"                     { capabilities = ["read"] }
> EOF
> vault write auth/approle/role/swarm token_policies=swarm token_ttl=1h token_max_ttl=4h secret_id_ttl=720h
> ```
> **EXPECTED:** roles + policy + AppRole + gossip key created.
> **VERIFY:** `vault read pki_int/roles/consul-server | grep allowed_domains`;
> `vault kv get -field=gossip_key nexus/swarm/consul-gossip-key` prints a base64 key.

### 5.4 — Harden Consul (gossip → mTLS → ACL)

> **Step 5.4.1 — Enable gossip encryption (rolling, sequential)**
> **WHERE:** each of the 6 nodes, root shell; orchestrate from the build host.
> **WHY:** encrypt the Serf gossip with the shared key from §5.3.1. Roll it one
> node at a time so the cluster never loses quorum.
> **WHAT (on each node, gossip key from `vault kv get`):**
> ```bash
> printf 'encrypt = "%s"\n' '<GOSSIP_KEY>' > /etc/consul.d/10-encrypt.hcl
> chown consul:consul /etc/consul.d/10-encrypt.hcl ; chmod 640 /etc/consul.d/10-encrypt.hcl
> systemctl restart consul ; sleep 5
> ```
> Do mgr-1 → mgr-2 → mgr-3 → workers, waiting for `consul members` to show all
> `alive` between each.
> **EXPECTED:** keyring converges; all 6 stay `alive`.
> **VERIFY:** `consul keyring -list` shows the key present on `[6/6]` members.

> **Step 5.4.2 — Issue per-node TLS certs + enable Consul mTLS, hard-cut HTTP→HTTPS**
> **WHERE:** issue on `vault-1`; place on each node.
> **WHY:** mTLS for internal RPC + Raft (`verify_server_hostname` needs the magic
> `server.nexus-lab.consul` SAN), server-TLS for the HTTPS API on `:8501`. **HTTP
> `:8500` must be hard-cut via a systemd `-http-port=-1` CLI flag** — setting
> `ports.http=-1` in an HCL drop-in does **not** override the value `consul.hcl`
> already set (config-dir merge *adds*, never *overrides* scalars — per
> `feedback_consul_hcl_ports_merge_no_override`). Roll sequentially.
> **WHAT (per node — issue on vault-1, substitute host/IPs):**
> ```bash
> # On vault-1: issue this node's leaf (full chain into server.crt)
> ISSUED=$(vault write -format=json pki_int/issue/consul-server \
>   common_name="<host>.consul.nexus.lab" \
>   alt_names="<host>,<host>.nexus.lab,server.nexus-lab.consul,localhost" \
>   ip_sans="<vmnet10>,<vmnet11>,127.0.0.1" ttl=2160h)
> echo "$ISSUED" | jq -r '.data.certificate, .data.issuing_ca' > /tmp/consul-server.crt
> echo "$ISSUED" | jq -r '.data.private_key' > /tmp/consul-server.key
> echo "$ISSUED" | jq -r '.data.issuing_ca'  > /tmp/consul-ca.pem
> # scp the three files to <host>:/etc/consul.d/tls/{server.crt,server.key,ca.pem}
> ```
> ```bash
> # On the node (files placed, owned consul:consul, key 0600):
> cat > /etc/consul.d/20-tls.hcl <<'EOF'
> tls {
>   defaults {
>     ca_file   = "/etc/consul.d/tls/ca.pem"
>     cert_file = "/etc/consul.d/tls/server.crt"
>     key_file  = "/etc/consul.d/tls/server.key"
>     verify_incoming = false
>     verify_outgoing = true
>   }
>   internal_rpc { verify_incoming = true  verify_server_hostname = true }
>   https        { verify_incoming = false }
> }
> ports { https = 8501 }
> EOF
> chown consul:consul /etc/consul.d/20-tls.hcl ; chmod 640 /etc/consul.d/20-tls.hcl
> # Hard-cut HTTP via a systemd drop-in CLI flag (HCL ports.http override is ignored)
> install -d /etc/systemd/system/consul.service.d
> cat > /etc/systemd/system/consul.service.d/10-http-off.conf <<'EOF'
> [Service]
> ExecStart=
> ExecStart=/usr/local/bin/consul agent -config-dir=/etc/consul.d/ -http-port=-1
> EOF
> systemctl daemon-reload ; systemctl restart consul ; sleep 5
> ```
> **EXPECTED:** Consul serves HTTPS on `:8501`; `:8500` refused.
> **VERIFY:** `consul members -http-addr=https://127.0.0.1:8501 -ca-file=/etc/consul.d/tls/ca.pem`
> → 6 alive; `curl -s http://127.0.0.1:8500/v1/status/leader` → connection refused.

> **Step 5.4.3 — Bootstrap Consul ACLs (default-deny + per-agent tokens)**
> **WHERE:** mgr-1 (bootstrap), then each node.
> **WHY:** lock the API: `default_policy=deny`, every agent carries a token. The
> bootstrap mints the management token (save to Vault KV). **Do NOT set
> `acl.tokens.default`** — an anonymous request under deny returns 200+empty (not
> 403); verify with `/v1/agent/self` (per `feedback_consul_acl_anon_filtered_not_403`).
> **WHAT:**
> ```bash
> # mgr-1: enable ACLs in deny-mode on each node first (drop-in), restart, then bootstrap
> # On EACH node:
> cat > /etc/consul.d/30-acl.hcl <<'EOF'
> acl {
>   enabled        = true
>   default_policy = "deny"
>   down_policy    = "extend-cache"
>   enable_token_persistence = true
> }
> EOF
> chown consul:consul /etc/consul.d/30-acl.hcl ; chmod 640 /etc/consul.d/30-acl.hcl
> systemctl restart consul ; sleep 5
>
> # On mgr-1 only — bootstrap (one-time), save the SecretID to Vault KV:
> export CONSUL_HTTP_ADDR=https://127.0.0.1:8501 CONSUL_CACERT=/etc/consul.d/tls/ca.pem
> consul acl bootstrap -format=json | tee /root/consul-bootstrap.json
> MGMT=$(jq -r .SecretID /root/consul-bootstrap.json)
> # store it: vault kv put nexus/swarm/consul-bootstrap-token management_token=$MGMT
>
> # Create an agent policy + a token per node; set each node's agent token:
> export CONSUL_HTTP_TOKEN=$MGMT
> consul acl policy create -name agent-policy -rules 'node_prefix "" { policy = "write" } service_prefix "" { policy = "read" }'
> for h in swarm-manager-1 swarm-manager-2 swarm-manager-3 swarm-worker-1 swarm-worker-2 swarm-worker-3; do
>   T=$(consul acl token create -description "$h agent" -policy-name agent-policy -format=json | jq -r .SecretID)
>   echo "$h -> $T"   # set on that node: consul acl set-agent-token agent "$T"
> done
> ```
> **EXPECTED:** bootstrap succeeds once; tokens created.
> **VERIFY:** anonymous `curl --cacert … https://127.0.0.1:8501/v1/agent/self` →
> **403**; with `-H "X-Consul-Token: $MGMT"` → 200.

### 5.5 — Harden Nomad (mTLS → ACL → Consul-HTTPS → Vault)

> **Step 5.5.1 — Issue Nomad TLS certs + enable mTLS (PARALLEL big-bang restart)**
> **WHERE:** issue on `vault-1`; place on each node; restart **all at once**.
> **WHY:** mTLS for Nomad RPC + Raft + HTTPS API. **The restart must be parallel,
> not rolling:** flipping `tls.rpc=true` makes a node reject *plain* peers, so a
> sequential restart isolates the first node and Raft can't elect (per
> `feedback_nomad_tls_rolling_restart_must_be_parallel`). The cert needs the
> `server.global.nomad` SAN for `verify_server_hostname`.
> **WHAT (per node — issue on vault-1):**
> ```bash
> ISSUED=$(vault write -format=json pki_int/issue/nomad-server \
>   common_name="server.global.nomad" \
>   alt_names="<host>,<host>.nexus.lab,client.global.nomad,localhost" \
>   ip_sans="<vmnet10>,<vmnet11>,127.0.0.1" ttl=2160h)
> # -> /etc/nomad.d/tls/{server.crt (leaf+issuing_ca), server.key, ca.pem}
> ```
> ```bash
> # On EACH node:
> cat > /etc/nomad.d/tls.hcl <<'EOF'
> tls {
>   http = true
>   rpc  = true
>   ca_file   = "/etc/nomad.d/tls/ca.pem"
>   cert_file = "/etc/nomad.d/tls/server.crt"
>   key_file  = "/etc/nomad.d/tls/server.key"
>   verify_server_hostname = true
>   verify_https_client    = false
> }
> EOF
> chmod 640 /etc/nomad.d/tls.hcl
> ```
> Then, from the build host, restart **all 6 in parallel**:
> ```powershell
> '111','112','113','131','132','133' | ForEach-Object -ThrottleLimit 6 -Parallel {
>   ssh nexusadmin@192.168.70.$_ 'sudo systemctl restart nomad'
> }
> ```
> **EXPECTED:** all nodes come back on TLS together; Raft re-elects.
> **VERIFY (set `NOMAD_ADDR=https://127.0.0.1:4646 NOMAD_CACERT=…`):**
> `nomad server members` → 3 alive.

> **Step 5.5.2 — Bootstrap Nomad ACLs**
> **WHERE:** mgr-1, then each node.
> **WHY:** lock the Nomad API. **Note:** Nomad's `acl{}` block has **no agent
> `token` field** — agent identity is the mTLS cert; ACL tokens are for operators
> only (per `feedback_nomad_acl_no_agent_token_in_config`).
> **WHAT:**
> ```bash
> # On EACH node: enable ACLs
> echo 'acl { enabled = true }' > /etc/nomad.d/acl.hcl ; chmod 640 /etc/nomad.d/acl.hcl
> systemctl restart nomad ; sleep 5
> # On mgr-1: bootstrap once, save the mgmt token
> export NOMAD_ADDR=https://127.0.0.1:4646 NOMAD_CACERT=/etc/nomad.d/tls/ca.pem
> nomad acl bootstrap -json | tee /root/nomad-bootstrap.json   # -> .SecretID = mgmt token
> # vault kv put nexus/swarm/nomad-bootstrap-token management_token=<SecretID>
> ```
> **EXPECTED:** bootstrap succeeds once.
> **VERIFY:** `NOMAD_TOKEN=<mgmt> nomad acl token self` → an accessor with
> `management` type.

> **Step 5.5.3 — Rewire Nomad → Consul over HTTPS (scheme-less address)**
> **WHERE:** each node, root shell; sequential rolling restart.
> **WHY:** Consul now refuses plain `:8500`, so point Nomad at Consul's HTTPS
> `:8501` with a token. **The `address` must be scheme-less** (`127.0.0.1:8501`,
> *not* `https://…`) with `ssl=true` — a scheme prefix fails boot with "too many
> colons" (per `feedback_nomad_consul_address_scheme_less`). Use the node's Consul
> agent token from §5.4.3. Replace the firstboot `consul{address=127.0.0.1:8500}`.
> **WHAT (per node):**
> ```bash
> # Remove the plain-HTTP consul stanza from nomad.hcl, then add the HTTPS one:
> sed -i '/consul {/,/}/d' /etc/nomad.d/nomad.hcl
> cat > /etc/nomad.d/40-consul.hcl <<EOF
> consul {
>   address   = "127.0.0.1:8501"
>   ssl       = true
>   ca_file   = "/etc/consul.d/tls/ca.pem"
>   cert_file = "/etc/nomad.d/tls/server.crt"
>   key_file  = "/etc/nomad.d/tls/server.key"
>   token     = "<this node's Consul agent token>"
> }
> EOF
> chmod 640 /etc/nomad.d/40-consul.hcl
> systemctl restart nomad ; sleep 5
> ```
> **WHAT (workers only — pin the servers):** once Consul HTTP is hard-cut, a
> worker's Consul-based server discovery can silently break on restart; pin the
> managers explicitly (per `feedback_nomad_workers_need_explicit_servers`):
> ```bash
> cat > /etc/nomad.d/41-client-servers.hcl <<'EOF'
> client { servers = ["192.168.10.111:4647","192.168.10.112:4647","192.168.10.113:4647"] }
> EOF
> chmod 640 /etc/nomad.d/41-client-servers.hcl ; systemctl restart nomad
> ```
> **EXPECTED:** Nomad reconnects to Consul over HTTPS.
> **VERIFY:** `curl -s --cacert … https://127.0.0.1:4646/v1/agent/self | jq '.config.Consuls // .Consul'`
> shows the `:8501` address + no errors in `journalctl -u nomad`.

> **Step 5.5.4 — Nomad–Vault integration (managers)**
> **WHERE:** mgr-1 (Vault role) + each manager.
> **WHY:** lets Nomad jobs pull secrets from Vault. Create a `nomad-cluster` token
> role in Vault, mint a periodic token, and load a `vault{}` stanza on the servers.
> **WHAT (on vault-1, root token):**
> ```bash
> vault write auth/token/roles/nomad-cluster \
>   allowed_policies="nexus-operator" disallowed_policies="" \
>   token_explicit_max_ttl=0 orphan=true token_period=259200 renewable=true
> vault token create -role=nomad-cluster -format=json | jq -r .auth.client_token   # -> periodic token
> ```
> **WHAT (on each manager):**
> ```bash
> cat > /etc/nomad.d/50-vault.hcl <<'EOF'
> vault {
>   enabled          = true
>   address          = "https://vault-1.nexus.lab:8200"
>   ca_file          = "/usr/local/share/ca-certificates/nexus-vault-pki-root.crt"
>   create_from_role = "nomad-cluster"
>   token            = "<periodic token>"
> }
> EOF
> chmod 640 /etc/nomad.d/50-vault.hcl ; systemctl restart nomad ; sleep 5
> ```
> **EXPECTED:** Nomad reports Vault connected.
> **VERIFY:** `curl -s --cacert … https://127.0.0.1:4646/v1/agent/self | jq '.config.Vaults'`
> shows `Enabled: true`; `journalctl -u nomad | grep -i vault` shows a successful
> token renew.

### 5.6 — Deploy Portainer CE as a Swarm service

> **Step 5.6.1 — Mount the NFSv4 `/data` share on all 3 managers**
> **WHERE:** each manager, root shell.
> **WHY:** Portainer's single Server replica can land on any manager; the shared
> NFS `/data` (from Guide 01) means a reschedule keeps its BoltDB state. The
> server's `fsid=0` export is the NFSv4 **pseudo-root**, so mount via `:/` (per
> `feedback_nfsv4_fsid0_pseudo_root`).
> **WHAT (on mgr-1/2/3):**
> ```bash
> apt-get install -y nfs-common
> install -d -m0755 /var/lib/portainer-data
> echo '192.168.70.1:/  /var/lib/portainer-data  nfs4  rw,hard,bg,_netdev,vers=4.2,sec=sys  0  0' >> /etc/fstab
> mount -a
> ```
> **EXPECTED:** the share mounts.
> **VERIFY:** `findmnt /var/lib/portainer-data` shows the `nfs4` mount;
> `touch /var/lib/portainer-data/.t && rm /var/lib/portainer-data/.t` succeeds.

> **Step 5.6.2 — Issue the Portainer TLS cert + DNS record + admin password**
> **WHERE:** issue on `vault-1`; place on each manager; DNS on the gateway.
> **WHY:** the UI serves HTTPS with a PKI cert (CN `portainer.nexus.lab`, IP-SANs
> per manager); the gateway publishes a **round-robin** `portainer.nexus.lab`
> A-record to all three managers; the admin password is a bcrypt hash Portainer
> reads at first boot.
> **WHAT (vault-1 — one cert, SANs cover all 3 managers):**
> ```bash
> ISSUED=$(vault write -format=json pki_int/issue/portainer-server \
>   common_name=portainer.nexus.lab \
>   ip_sans='192.168.70.111,192.168.70.112,192.168.70.113,127.0.0.1' ttl=2160h)
> # place on EACH manager: /etc/portainer/tls/{server.crt (leaf+issuing_ca), server.key}
> ```
> ```bash
> # Admin password (bcrypt) on EACH manager:
> NEW=$(openssl rand -base64 24 | tr -dc 'a-zA-Z0-9' | head -c 24)
> htpasswd -nbB admin "$NEW" | cut -d: -f2 > /etc/portainer/admin-password.txt    # bcrypt hash
> echo "Portainer admin password: $NEW"     # save it
> ```
> ```bash
> # On nexus-gateway: round-robin A-record to the 3 managers
> echo 'host-record=portainer.nexus.lab,192.168.70.111,192.168.70.112,192.168.70.113' \
>   | sudo tee /etc/dnsmasq.d/portainer-record.conf
> sudo dnsmasq --test && sudo systemctl reload dnsmasq
> ```
> **EXPECTED:** cert + password file on each manager; DNS record live.
> **VERIFY:** host `Resolve-DnsName -Server 192.168.70.1 portainer.nexus.lab` →
> three A records.

> **Step 5.6.3 — Deploy the Portainer stack**
> **WHERE:** mgr-1, root shell.
> **WHY:** Portainer as a Swarm stack — Server (1 replica, manager-pinned,
> HTTPS:9443, bind-mounting the NFS `/data` + TLS + password file) and Agent
> (global, one per node, for full-cluster visibility).
> **WHAT:**
> ```bash
> cat > /tmp/portainer-stack.yml <<'EOF'
> version: "3.8"
> services:
>   server:
>     image: portainer/portainer-ce:2.21.4
>     command: [--ssl, --sslcert, /certs/server.crt, --sslkey, /certs/server.key, --admin-password-file, /run/secrets/admin-pw]
>     ports:
>       - { target: 9443, published: 9443, protocol: tcp, mode: ingress }
>       - { target: 8000, published: 8000, protocol: tcp, mode: ingress }
>     volumes:
>       - /var/lib/portainer-data:/data
>       - /etc/portainer/tls:/certs:ro
>       - /etc/portainer/admin-password.txt:/run/secrets/admin-pw:ro
>     networks: [agent_network]
>     deploy:
>       mode: replicated
>       replicas: 1
>       placement: { constraints: ["node.role == manager"] }
>       restart_policy: { condition: on-failure, delay: 5s, max_attempts: 3 }
>   agent:
>     image: portainer/agent:2.21.4
>     environment: { AGENT_CLUSTER_ADDR: tasks.agent }
>     volumes:
>       - /var/run/docker.sock:/var/run/docker.sock
>       - /var/lib/docker/volumes:/var/lib/docker/volumes
>     networks: [agent_network]
>     deploy: { mode: global, restart_policy: { condition: on-failure, delay: 5s } }
> networks:
>   agent_network: { driver: overlay, attachable: true }
> EOF
> docker stack deploy --with-registry-auth --prune -c /tmp/portainer-stack.yml portainer
> ```
> **EXPECTED:** Swarm schedules the stack; server converges 1/1, agent 6/6.
> **VERIFY:** `docker service ls --filter label=com.docker.stack.namespace=portainer`
> → `portainer_server 1/1`, `portainer_agent 6/6`;
> `curl -sk -o /dev/null -w '%{http_code}' https://127.0.0.1:9443/api/system/status` → `200`.

---

## 6. Validation — by-hand acceptance smoke

From the **host** (TLS args via the stock root CA bundle, `~/.nexus/vault-ca-bundle.crt`).

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | Swarm 6 nodes | `ssh …@111 'docker node ls'` | 6 `Ready`/`Active`, 3 managers |
| 2 | Consul 6 members | `ssh …@111 'CONSUL_HTTP_ADDR=https://127.0.0.1:8501 CONSUL_CACERT=… consul members'` | 6 alive (3 server + 3 client) |
| 3 | Consul HTTP hard-cut | `ssh …@111 'curl -s http://127.0.0.1:8500/v1/status/leader'` | connection refused |
| 4 | Consul ACL deny works | host: `curl --cacert … https://192.168.70.111:8501/v1/agent/self` | `403` (no token) |
| 5 | Nomad 3 servers | `ssh …@111 'NOMAD_ADDR=https://127.0.0.1:4646 NOMAD_CACERT=… nomad server members'` | 3 alive |
| 6 | Nomad→Consul HTTPS healthy | `ssh …@111 'journalctl -u nomad \| grep -i consul \| tail'` | no connection errors |
| 7 | Nomad→Vault enabled | `curl …4646/v1/agent/self \| jq .config.Vaults` | `Enabled: true` |
| 8 | Off-cluster TLS chain validates | host: `curl.exe --cacert ~/.nexus/vault-ca-bundle.crt --ssl-no-revoke -o $null -w '%{http_code}' https://192.168.70.111:8501/v1/status/leader` | `200` (full chain on wire) |
| 9 | Portainer up | host: `curl.exe --cacert … --ssl-no-revoke -o $null -w '%{http_code}' https://portainer.nexus.lab:9443/api/system/status` | `200` |
| 10 | Portainer state survives reschedule | `ssh …@111 'docker service update --force portainer_server'`; re-check #9 after it lands on another manager | still `200`, same admin login |

**1–9 green ⇒ Guide 05 satisfied.** 10 proves the NFS-backed state. (8 is the
off-cluster proof the full leaf+intermediate chain is served — the stock bundle
has only the root.)

---

## 7. Teardown / reset

```bash
# On any manager: remove the Portainer stack + leave the swarm on every node.
ssh nexusadmin@192.168.70.111 'sudo docker stack rm portainer'
for ip in 131 132 133 113 112 111; do ssh nexusadmin@192.168.70.$ip 'sudo docker swarm leave --force'; done
```
Then stop/delete the 6 VMs (Guide 00 §7 pattern). The gateway NFS export + dnsmasq
record belong to Guide 01 — leave them unless wiping that too.

> **Cold-rebuild prerequisite** (per `feedback_cold_rebuild_stale_kv_tokens`):
> before re-applying after a teardown, **wipe the stale bootstrap tokens** from
> Vault KV — the new cluster mints fresh Consul/Nomad mgmt tokens, and a stale one
> in KV makes ACL verification fail ("expected 6 alive, got 0"):
> ```bash
> vault kv metadata delete nexus/swarm/consul-bootstrap-token
> vault kv metadata delete nexus/swarm/nomad-bootstrap-token
> # (gossip key is preserved — it's not bootstrap state)
> ```

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| After `nft -f`, Swarm/Portainer ports unreachable | `nft -f` atomically drops Docker's iptables-nft tables | `systemctl restart docker` after **every** `nft -f` (per `feedback_nftables_flush_ruleset_wipes_docker`). |
| `host:9443` reachable on the manager but not from the build host | `inet filter forward` policy=drop with no `docker_gwbridge` accepts | Add the `forward`-chain accepts (§5.1.5) + restart docker. |
| Consul still serves `:8500` after the TLS drop-in | HCL config-dir merge does NOT override `ports.http` from `consul.hcl` | Use the systemd `-http-port=-1` CLI flag drop-in (§5.4.2). See `feedback_consul_hcl_ports_merge_no_override`. |
| Nomad drop-in (`tls.hcl`/`acl.hcl`) seems ignored | stock unit uses `-config=/etc/nomad.d/nomad.hcl` (single file) | Unit must use `-config=/etc/nomad.d/` (directory). §5.1.4 / `feedback_nomad_systemd_unit_single_file_config`. |
| Nomad Raft can't elect after enabling TLS | sequential restart isolated the first TLS node from plain peers | Restart **all servers in parallel** (§5.5.1). `feedback_nomad_tls_rolling_restart_must_be_parallel`. |
| Nomad won't boot: "too many colons in address" | `consul.address` has an `https://` scheme | Use scheme-less `127.0.0.1:8501` + `ssl = true` (§5.5.3). `feedback_nomad_consul_address_scheme_less`. |
| Workers' Nomad loses the servers after a restart | Consul HTTP hard-cut broke cached discovery | Pin `client.servers` explicitly (§5.5.3). `feedback_nomad_workers_need_explicit_servers`. |
| Tried to set an agent token in Nomad `acl{}` | Nomad `acl{}` has no `token` field — agent identity is the mTLS cert | Don't; ACL tokens are operator-only. `feedback_nomad_acl_no_agent_token_in_config`. |
| Consul anonymous probe returns 200 + empty (looks "open") | under deny, filtered endpoints return 200+empty, not 403 | Verify deny with `/v1/agent/self` (returns 403). Never set `acl.tokens.default`. `feedback_consul_acl_anon_filtered_not_403`. |
| `consul`/probe says config file MISSING though it's there | `/etc/consul.d/` is `0750 root:consul`; `nexusadmin` can't traverse | `sudo` every probe of `/etc/consul.d/`. `feedback_sudo_required_for_consul_etc_traverse`. |
| Portainer `mount.nfs4: No such file or directory` | `fsid=0` makes the export the pseudo-root | Mount via `192.168.70.1:/` not `:/srv/nfs/portainer-data` (§5.6.1). `feedback_nfsv4_fsid0_pseudo_root`. |
| Cold rebuild: ACL "expected 6 alive, got 0" | stale bootstrap token in Vault KV from the prior cluster | Wipe `swarm/consul-bootstrap-token` + `swarm/nomad-bootstrap-token` before re-apply (§7). |
| Off-cluster TLS = `PartialChain` / curl 60 | server presents leaf-only; build host has only the root | Concatenate **leaf + issuing_ca** into `server.crt` (every issue step does). |

---

### Cross-references

- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (swarm `.111`–`.113`/`.131`–`.133`)
- **Automated equivalents:** `nexus-infra-swarm-nomad/packer/swarm-node/` + `terraform/envs/swarm-nomad/role-overlay-*.tf`
- **Scaffolding pattern reused:** [`04-foundation-vault-pki-ldap.md`](./04-foundation-vault-pki-ldap.md) Part D
- **Gateway NFS export consumed:** [`01-foundation-nexus-gateway.md`](./01-foundation-nexus-gateway.md) §5.8
- **Previous guide:** [`04-foundation-vault-pki-ldap.md`](./04-foundation-vault-pki-ldap.md)
- **Next guide:** Guide 06 — Kafka ecosystem (2 KRaft clusters + MirrorMaker 2 DR). See [`INDEX.md`](../INDEX.md).
