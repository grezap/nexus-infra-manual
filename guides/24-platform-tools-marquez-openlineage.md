# Guide 24 — Platform tools · Marquez (OpenLineage data-lineage backend, PostgreSQL HA)

> **Mirrors:** `nexus-infra-platform-tools` (a **separate repo**) — the 2 Packer templates
> (`platform-marquez-node` + `platform-marquez-pg-node`) + the `platform-marquez` Terraform env
> overlays (`…-nftables-backplane`, `…-vault-agents`, `…-tls`, `…-pg-replication`, `…-compose`,
> `…-bootstrap`) — Phase **0.Q.1** / **ADR-0043**, tier `09-platform`. The **data-lineage** backend —
> the first tool in the platform-tools tier, and the sink for enhancement **E16**.

> The 24th guide, and the first of the **platform-tools** tier (`09-platform`) — self-hosted
> developer/data-platform services that sit *alongside* the data plane. Marquez is the **OpenLineage**
> backend: pipelines emit run/job/dataset events and it renders the "if I change this table, what breaks
> downstream?" graph. Marquez ships as **containers**, so the app node runs Docker + compose (mirroring
> the Harbor docker-on-VM precedent in this same tier, Guide 19), backed by a **dedicated PostgreSQL 17
> HA pair** behind a VRRP VIP. Self-contained — depends only on the foundation (gateway + Vault); **no**
> Kafka, MinIO, or lakehouse dependency, deliberately (a lineage backend that can't start until the
> lakehouse is up would be useless exactly when you need it).

---

## 1. Overview & purpose

**Marquez** is the reference implementation of the **OpenLineage** standard — a metadata service that
records *runs* (a pipeline execution), *jobs* (the transform), and *datasets* (inputs/outputs), and
exposes the lineage graph over a REST API + web UI. DataFlow Studio (Phase 1) is the first emitter:
its `curation` and `warehouse-sink` jobs POST run events, and Marquez renders
`oltp.* → dfs.* → dwh.*` (Guide 23 §9.3).

**3 VMs + 1 VRRP VIP, two node types:**

| Node type | Count | What it runs |
|---|---|---|
| `marquez` (app) | 1 | Docker CE + compose: Marquez **API** (:5000) + **admin** (:5001) + **web** (:3000), all behind an **nginx TLS front door** on :443 carrying the node's Vault-PKI leaf → `https://marquez.nexus.lab` |
| `marquez-pg` (datastore) | 2 | PostgreSQL 17 **streaming replication** (primary + hot standby) + **keepalived VRRP VIP** `marquez-db.nexus.lab .136` that follows the PG primary |

The app node is stateless: all state lives in `marquez-db`, so the API reaches PG **through the VIP**
(`marquez-db.nexus.lab:5432`) and follows a datastore failover without reconfiguration. The app node is
a deliberate SPOF (§9 / ADR-0031) — acceptable because OpenLineage emission is best-effort, so a down
Marquez never fails a pipeline run.

---

## 2. Component primer

- **OpenLineage** — an open standard for lineage metadata. A `RunEvent` is `{eventType (START/COMPLETE/
  FAIL), eventTime, producer, run:{runId}, job:{namespace,name}, inputs:[…], outputs:[…]}`, POSTed to
  `/api/v1/lineage`. *Why a standard:* emitters (Spark, dbt, Airflow, .NET) all speak it, so the backend
  is emitter-agnostic.
- **Marquez** — the OpenLineage backend. A **Dropwizard** (Java) service (`api`) + a React UI (`web`),
  reading/writing a PostgreSQL metadata DB. Published as plain container images
  (`marquezproject/marquez`, `marquezproject/marquez-web`), pinned to **0.51.1** — never `:latest` (an
  upstream bump silently changes schema-migration behaviour on a cold rebuild).
- **nginx TLS terminator** — Marquez has no built-in TLS; the front-door container terminates HTTPS on
  :443 with the node's Vault-PKI leaf and reverse-proxies `/api/*` → api:5000 and `/` → web:3000. *Why
  a front door:* one leaf, one port, one hostname (`marquez.nexus.lab`) instead of exposing three plain
  HTTP ports.
- **PostgreSQL 17 streaming replication** — a primary + a hot standby fed by WAL streaming; the standby
  is read-only until promoted. *Why dedicated (not co-located on `registry-pg`):* lineage metadata is a
  first-class store with its own growth/retention profile; sharing would couple the lineage plane's
  availability to the image registry's.
- **keepalived VRRP VIP** — `marquez-db.nexus.lab .136` floats to whichever PG node is primary; the
  `vrrp_script` probes PG, so the VIP follows the leader. **vrid 74** (the registry tier holds 73 on the
  same L2 — a collision silently merges the two VRRP domains). **Unicast** peers (VMnet11 drops
  multicast), `BACKUP` + `nopreempt`.

---

## 3. Prerequisites

| Prerequisite | Why | Verify |
|---|---|---|
| `nexus-gateway` (`.1`) alive | DHCP reservations (`.127/.134/.135`), DNS (`marquez` / `marquez-db`), NAT egress (pull images from Docker Hub) | `ssh nexusadmin@192.168.70.1 'systemctl is-active dnsmasq'` |
| Vault HA (`vault-1/2/3` + `vault-transit`) **unsealed** | PKI leaves + KV creds | `vault status` → `Sealed false` |
| Foundation + security applied (below) | pins the 3 IPs/DNS + the `platform-tools-server` PKI role/policies/AppRoles/KV | — |
| Build host | `vmrun.exe` at the **non-(x86)** path; Packer ≥ 1.11, Terraform ≥ 1.9; ISO at `H:\VMS\ISO\debian-13.5.0-amd64-netinst.iso` | — |

**Two cross-repo applies must run FIRST** (both in `nexus-infra-vmware`, both idempotent):

```powershell
pwsh -File scripts/foundation.ps1 apply   # dhcp-host pins .127/.134/.135 + DNS marquez / marquez-db
pwsh -File scripts/security.ps1   apply   # platform-tools-server PKI role, 3 policies, 3 AppRole sidecars, KV seeds
```

> ⚠️ **A `-Vars` partial apply is a FULL variable override.** Any `enable_*` toggle you don't pass
> reverts to its default — on `foundation`, applying without
> `enable_platform_tools_dhcp_reservations=true` on a lab that had it enabled **removes the Marquez
> pins**. Pass every reservation you care about, or pass none. (memory:
> `feedback_terraform_partial_apply_destroys_resources`.)

**Credentials — the field name is `value`** (the registry convention, *not* observability's `password`):

| Secret | Vault path | Field |
|---|---|---|
| Marquez DB password | `nexus/platform-tools/marquez/db-password` | `value` |
| PG replication password | `nexus/platform-tools/marquez/replication-password` | `value` |
| PG superuser password | `nexus/platform-tools/marquez/superuser-password` | `value` |

---

## 4. Target topology

| Node | Role | VMnet11 | VMnet10 | vCPU/RAM/disk | Ports |
|---|---|---|---|---|---|
| `marquez` | Marquez app (Docker CE + compose) | `.70.127` | `.10.127` | 2 / 4 GB / 40 GB | 443 (nginx TLS) · 5000 API · 5001 admin · 3000 web |
| `marquez-pg-1` | PostgreSQL 17 **primary** | `.70.134` | `.10.134` | 2 / 2 GB / 60 GB | 5432 |
| `marquez-pg-2` | PostgreSQL 17 **replica** | `.70.135` | `.10.135` | 2 / 2 GB / 60 GB | 5432 |
| `marquez-db.nexus.lab` | keepalived VRRP VIP (no VM) | `.70.136` | — | — | 5432 |

MACs `:E0`–`:E2`. Client/UI/HTTPS ride **VMnet11**; PG streaming replication + keepalived ride the
**VMnet10 backplane**. Images pinned `marquezproject/marquez:0.51.1` + `…/marquez-web:0.51.1` + nginx
`1.27-alpine`.

---

## 5. Step-by-step build

> The repo does this with 2 Packer templates + one `marquez.ps1 apply` (7 ordered overlays). By hand the
> order is the same: **foundation/security first**, then the two node types, then backplane → TLS → PG
> HA → VIP → compose → lineage round-trip.

### 5.0 — Foundation: KV seeds, PKI role, VIP DNS

Done by `security.ps1 apply`; by hand:

- **KV seeds** (mount `nexus`, field `value`): `platform-tools/marquez/{db,replication,superuser}-password`.
- **PKI role `platform-tools-server`** on `pki_int` — allowed_domains include `marquez.nexus.lab`,
  `marquez-db.nexus.lab`, the three hostnames + the VIP; `server_flag=true`, key PKCS#8. Each node's
  Vault-agent renders its leaf from this role.
- **DNS** (on the gateway `dnsmasq`): `marquez → .127`, `marquez-db → .136` (the VIP).

### 5.1 — Build the two node types

One Packer template **per engine** — canon. ~20–40 min each; run sequentially:

```powershell
cd packer/platform-marquez-pg-node ; packer init . ; packer build .   # PostgreSQL 17 baked
cd ../platform-marquez-node        ; packer init . ; packer build .   # Docker CE + compose baked
```

By hand each is a Debian 13 base (Guide 00) plus: the **pg** node — PostgreSQL 17 (`apt install
postgresql-17`), the `repluser` role, `pg_hba` for the backplane, a firstboot unit; the **app** node —
Docker CE + the compose plugin, the `marquez` group, `/etc/nexus-marquez` (`0750 root:marquez`).

> If a Packer build hangs at preseed with no console progress, the host's **`VMnetDHCP`** service
> stopped silently — `Start-Service VMnetDHCP` from an **elevated** shell (memory:
> `feedback_vmware_dhcp_service_stopped`).

### 5.2 — nftables backplane + the Docker chain-reapply drop-in

Each node's `/etc/nftables.conf` input chain (policy drop) accepts: `lo`, established/related, ICMP,
SSH + node_exporter from `192.168.70.0/24` on `nic0`, and the **VMnet10 backplane** (`nic1`,
`192.168.10.0/24`) for PG replication. The app node additionally accepts `443/3000/5000/5001` from
`192.168.70.0/24`, and the PG nodes accept VRRP (`ip protocol vrrp`) + `5432` from the 3 platform IPs +
the VIP.

**The critical trap:** `nft -f` begins with `flush ruleset`, which **wipes Docker's own chains** — the
published-port DNAT vanishes and the stack loses external networking. Handle it with an
`nftables.service` drop-in that restarts Docker after every reload, and `iifname "docker0"` +
`iifname "br*"` accepts in **both** input and forward:

```ini
# /etc/systemd/system/nftables.service.d/10-reapply-docker.conf
[Service]
ExecStartPost=-/bin/systemctl try-restart docker.service
```

Use `iifname` (runtime string match), not `iif docker0` — the latter fails `nft -c -f` at bake because
`docker0` doesn't exist yet. (memory: `feedback_nftables_flush_ruleset_wipes_docker`,
`feedback_cluster_template_nftables_backplane`.)

### 5.3 — PostgreSQL 17 HA pair (primary + streaming replica)

On **`marquez-pg-1` (primary, `.10.134`)** — enable replication + create the roles:

```bash
# postgresql.conf: wal_level=replica, max_wal_senders>=4, hot_standby=on, listen on the backplane
sudo -u postgres psql -c "CREATE ROLE repluser WITH REPLICATION LOGIN PASSWORD '<replication-password>';"
sudo -u postgres psql -c "CREATE ROLE marquez  WITH LOGIN PASSWORD '<db-password>';"
sudo -u postgres psql -c "CREATE DATABASE marquez OWNER marquez;"
# pg_hba.conf: host replication repluser 192.168.10.135/32 scram-sha-256  (+ the app's marquez user)
sudo systemctl reload postgresql@17-main
```

On **`marquez-pg-2` (replica, `.10.135`)** — base-backup from the primary over the backplane:

```bash
sudo systemctl stop postgresql@17-main
sudo rm -rf /var/lib/postgresql/17/main
echo "192.168.10.134:5432:replication:repluser:<replication-password>" | sudo tee /var/lib/postgresql/.pgpass
sudo chown postgres:postgres /var/lib/postgresql/.pgpass ; sudo chmod 0600 /var/lib/postgresql/.pgpass
sudo -u postgres env PGPASSWORD='<replication-password>' \
  pg_basebackup -h 192.168.10.134 -p 5432 -U repluser -D /var/lib/postgresql/17/main -Fp -Xs -P -R
sudo systemctl start postgresql@17-main
sudo -u postgres psql -tAc 'select pg_is_in_recovery();'   # → t
```

`-R` writes `standby.signal` + `primary_conninfo`, so the node comes up as a hot standby. **Verify on
the primary:** `sudo -u postgres psql -c 'select client_addr,state from pg_stat_replication;'` →
`192.168.10.135 | streaming`. (This exact `pg_basebackup … -R` is also the recover-from-split runbook —
if a failover leaves the standby orphaned on its own timeline, re-run it to rejoin.)

### 5.4 — keepalived: the VRRP VIP that follows the PG primary

Both PG nodes run keepalived; the VIP `.136` lands wherever PG is primary. `/etc/keepalived/keepalived.conf`:

```
vrrp_script chk_pg {
  script "/usr/lib/postgresql/17/bin/pg_isready -q -h 127.0.0.1 -p 5432"   # the VERSIONED path
  interval 2 ; fall 2 ; rise 2
}
vrrp_instance marquez_db {
  state BACKUP ; nopreempt        # nopreempt: no thundering failback when a node returns
  interface nic0 ; virtual_router_id 74      # 74 — registry holds 73 on this L2
  unicast_src_ip 192.168.70.134   # peer's 192.168.70.135 in unicast_peer (VMnet11 drops multicast)
  authentication { auth_type PASS ; auth_pass <vrrp-secret> }
  virtual_ipaddress { 192.168.70.136/24 dev nic0 }
  track_script { chk_pg }
}
```

`pg_isready` **must** be the versioned `/usr/lib/postgresql/17/bin/…` path (the unversioned wrapper
isn't on keepalived's `PATH`). **Verify:** `ip -4 -o addr show nic0 | grep 136` returns on **exactly
one** node.

### 5.5 — The Marquez docker-compose stack (app node)

On **`marquez` (`.127`)**, author `/etc/nexus-marquez/` (all `0640/0750 root:marquez`) and `up -d`:

**`marquez.yml`** (Dropwizard) — points the JDBC at the **VIP** with `verify-ca`, DB password from Vault
KV (never on the build host):

```
db:
  url: jdbc:postgresql://marquez-db.nexus.lab:5432/marquez?ssl=true&sslmode=verify-ca&sslrootcert=/etc/nexus-marquez/tls/ca.crt
  user: marquez
  password: <from Vault KV nexus/platform-tools/marquez/db-password>
```

**`nginx.conf`** — the TLS front door (`upstream api → api:5000`, `web → web:3000`; leaf
`/etc/nexus-marquez/tls/marquez.{crt,key}`):

```nginx
server {
  listen 443 ssl; server_name _;
  ssl_certificate /etc/nexus-marquez/tls/marquez.crt; ssl_certificate_key /etc/nexus-marquez/tls/marquez.key;
  ssl_protocols TLSv1.2 TLSv1.3; client_max_body_size 32m;
  location /api/ { proxy_pass http://marquez_api; proxy_set_header Host $host; proxy_set_header X-Forwarded-Proto https; }
  location /    { proxy_pass http://marquez_web; proxy_set_header Host $host; }
}
```

**`docker-compose.yml`** — three pinned services; the api reads its config via **`MARQUEZ_CONFIG` env**
(the entrypoint runs `java -jar marquez.jar server $MARQUEZ_CONFIG` and **ignores `command:` args** — a
`command:[--config,…]` is silently discarded and it falls back to the baked-in dev config), and
`extra_hosts` pins the VIP inside the container so JDBC hostname verification is deterministic:

```yaml
name: nexus-marquez
services:
  api:
    image: marquezproject/marquez:0.51.1
    environment: [ "MARQUEZ_CONFIG=/opt/marquez/marquez.yml" ]
    extra_hosts: [ "marquez-db.nexus.lab:192.168.70.136" ]
    volumes:
      - /etc/nexus-marquez/marquez.yml:/opt/marquez/marquez.yml:ro
      - /etc/nexus-marquez/tls/ca.crt:/etc/nexus-marquez/tls/ca.crt:ro
    ports: [ "5000:5000", "5001:5001" ]
  web:
    image: marquezproject/marquez-web:0.51.1
    ports: [ "3000:3000" ]
  nginx:
    image: nginx:1.27-alpine
    volumes:
      - /etc/nexus-marquez/nginx.conf:/etc/nginx/nginx.conf:ro
      - /etc/nexus-marquez/tls:/etc/nexus-marquez/tls:ro
    ports: [ "443:443" ]
```

```bash
# run as root from the project dir — the 0750 dir denies an unprivileged `cd`
sudo docker compose --project-directory /etc/nexus-marquez -f /etc/nexus-marquez/docker-compose.yml up -d
```

Wait for the admin healthcheck: `curl -sf http://127.0.0.1:5001/healthcheck` → 200.

### 5.6 — OpenLineage round-trip (exit gate)

POST a run event and read it back through the front door (SSH-local-curl on the node against its own
CA — the world-readable `/etc/ssl/certs/platform-tools-ca.pem`, since `/etc/nexus-marquez/tls/ca.crt` is
root-only):

```bash
CA=/etc/ssl/certs/platform-tools-ca.pem
CURL="curl -sS --cacert $CA --resolve marquez.nexus.lab:443:127.0.0.1"
NOW=$(date -u +%Y-%m-%dT%H:%M:%S.000Z)
printf '{"eventType":"COMPLETE","eventTime":"%s","producer":"guide-24","run":{"runId":"11111111-1111-1111-1111-111111111111"},"job":{"namespace":"nexus-lineage","name":"hello"},"inputs":[{"namespace":"nexus-lineage","name":"raw.orders"}],"outputs":[{"namespace":"nexus-lineage","name":"curated.orders"}]}' "$NOW" \
  | $CURL -o /dev/null -w '%{http_code}\n' -X POST https://marquez.nexus.lab/api/v1/lineage -H 'Content-Type: application/json' -d @-   # → 201
$CURL "https://marquez.nexus.lab/api/v1/namespaces/nexus-lineage/jobs" | grep -o '"name":"[^"]*"'   # → hello
```

---

## 6. Validation — by-hand acceptance smoke (demo / playbook)

The repo ships `smoke-0.Q.1.ps1` (7 sections). By hand:

1. **Reachability** — ping `.127/.134/.135`.
2. **Services** — `docker compose … ps` shows api/web/nginx **Up** (run it with
   `--project-directory`, not a bare `cd` — a bare `cd` as `nexusadmin` on the `0750` dir returns empty
   and reads as a false "not running"); PG active on both datastore nodes.
3. **Marquez API/admin/web/front-door** — `:5000/api/v1/namespaces` 200, `:5001/healthcheck` 200,
   `:443` via the front door 200.
4. **PG streaming replication** — `pg_stat_replication.state = streaming` on the primary.
5. **VIP bound + TLS through the VIP** — `psql 'host=marquez-db.nexus.lab sslmode=verify-full'` from
   `marquez-pg-2`. This is the check that earns its keep: it's the **only** one that proves *both* leaves
   carry the VIP in their SANs — a VIP failover with a narrowed SAN set is otherwise silent until the day
   you fail over.
6. **Cert SANs** — the app leaf carries `marquez.nexus.lab` + the IP SAN `.70.127`; both PG leaves carry
   `marquez-db.nexus.lab` + `.136`.
7. **OpenLineage round-trip** — §5.6.

**The full data-flow demo** (the "see the data" playbook): `nexus-infra-platform-tools/scripts/
marquez-lineage-demo.ps1` emits a 2-job / 3-dataset graph (`raw.orders → curate-orders →
curated.orders → load-dwh → dwh.fact_order`) and reads it back. **The real emitter** is DataFlow Studio
(`dataflow-studio/scripts/dfs-lineage-demo.ps1`, Guide 23 §9.3) — 2 jobs + 29 datasets.

---

## 7. Teardown / reset

```powershell
pwsh -File scripts/marquez.ps1 destroy   # or: vmrun stop soft ×3, then deleteVM (Guide 00 §7)
# gateway: remove the marquez dnsmasq records + reload; nothing else to reconcile.
```

There is **no external state** — the lineage DB is the tier's own, so a rebuild is genuinely from zero
(unlike the registry tier, whose blobs live in MinIO). Cold-rebuild = the same sequence with `destroy`
in front: `foundation apply → security apply → packer build ×2 → marquez.ps1 apply → smoke-0.Q.1`
(**PROVEN 2026-07-21**: destroy 19 → from-zero apply 19, API healthy first try, zero transients, smoke
ALL GREEN).

---

## 8. Troubleshooting

**The 5 apply-time transients from the first build — all fixed in source, so a replay hits none:**

| # | Symptom | Fix |
|---|---|---|
| T0 | `Retrieving ISO … 404` after ~5s | Debian archived the 13.5.0 point release off the live mirror → `iso_url` default → the local `H:/VMS/ISO/` copy (fleet-wide tripwire; every deb13 template shared it) |
| T1 | `cd: /etc/nexus-marquez: Permission denied` → stack never starts | the dir is `0750 root:marquez`; the compose step's bare `cd` runs as `nexusadmin` → `sudo docker compose --project-directory …` (no unprivileged traversal). Same fix in the smoke gate (T4) |
| T2 | API crash-loops `Connection to localhost:5432 refused`; rendered JDBC lost the db name | the URL host used the braced `$${dbDns}` but the db name used bare `$marquezDb`; the `?` after it breaks PowerShell's unbraced-var parse → brace it `$${marquezDb}` |
| T3 | API *still* crash-loops on `localhost:5432` with a correct `marquez.yml` | the Marquez entrypoint **ignores `command:` args** and reads `MARQUEZ_CONFIG` → pass the config via the env var, drop the dead `command:` |
| T4 | smoke reports all 3 services "not running" while every endpoint is 200 | same bare-`cd` false-fail as T1, in the gate → `--project-directory` |

**Operational gotchas (found live):**

- **`:443` times out right after power-on, even from the node's own localhost, though `docker ps` shows
  the containers "Up".** The VM was **suspended** (not powered off) — on resume the containers are stale
  (the API logs are frozen at suspend-time) and Docker's published-port DNAT is gone (the boot-time
  `nft -f` wiped it). Fix: **`sudo systemctl restart docker`** on `.127` — the API is 200 within ~30s.
  (Seen 2026-07-25 during the DataFlow Studio 3F bring-up.)
- **Windows `curl --cacert … https://192.168.70.127/…` fails (exit 60).** schannel can't consume a PEM
  `--cacert` the way Linux curl does — a *curl* limitation, not a chain problem. The front-door leaf
  chains to the NexusPlatform root and carries an IP SAN, so a proper client (a .NET `HttpClient` with a
  custom root-trust callback, or on-node curl with `--resolve`) validates it by IP fine.
- **Read-back `curl: (77) error setting certificate file`.** `/etc/nexus-marquez/tls/ca.crt` is
  `0640 root:marquez`; SSH-local-curl runs as `nexusadmin`. Use the world-readable
  `/etc/ssl/certs/platform-tools-ca.pem`.

**Traps already encoded in source** (don't "simplify" them out): versioned `pg_isready` in the
`vrrp_script`; DB-user key mode `0600`; VRRP **unicast** + **vrid 74**; `pg_basebackup` copies data not
config (the replica writes its own `pg_hba`); MAC-keyed `.link` (an `en*` match hits both NICs); the
conditional `$altNames` (the app node's `alt` is legitimately empty and an empty `alt` emits
`host,host.nexus.lab,,localhost`, which Vault rejects).

---

## 9. Production tuning — Marquez + its PostgreSQL

- **Marquez API** — size the Dropwizard `server.*` thread pools + the JDBC pool for your emitter fan-in;
  the default is fine for a lab. Watch the admin `:5001/metrics` for pool saturation.
- **PostgreSQL** — the lineage DB is write-heavy (every run writes run/job/dataset/facet rows). Tune
  `shared_buffers` / `checkpoint_*` / `wal_compression` as for any OLTP PG; keep the standby's
  `hot_standby_feedback=on` if you ever read from it.
- **Lineage retention** — Marquez writes lineage rows **forever**; there is no retention policy in
  0.Q.1 (called out in ADR-0043 as a real risk, not designed away). At scale add a periodic prune of old
  run/facet rows.
- **HA the app node** — it's a single VM (SPOF). State is fully external in `marquez-db`, so a second
  app node behind round-robin DNS (`marquez.nexus.lab`) is a cheap add (ADR-0031) — the pattern the
  registry tier's Harbor app pair already uses (Guide 19).
- **SSO** — Harbor gets Vault OIDC; Marquez 0.Q.1 does not. Deferred, not forgotten.

---

### Cross-references

- **Repo handbook** — `nexus-infra-platform-tools/docs/handbook.md` (§0–§3: the apply-based from-zero
  replay + the transient ledger this guide's §8 mirrors).
- **ADR-0043** (`nexus-platform-plan`) — why a dedicated first-class Marquez tier (not docker-on-obs),
  the dedicated PG, the phase-ID (`0.Q`).
- **Guide 19** (Harbor) — the docker-on-VM + dedicated-PG-HA precedent in this same `09-platform` tier.
- **Guide 23 §9.3** (Connect & Observe Cookbook) — connecting DataFlow Studio's OpenLineage emitter +
  reading the graph back; the real `oltp.* → dfs.* → dwh.*` lineage.
- **DataFlow Studio** — `dataflow-studio/docs/handbook.md` §1.8c + `scripts/dfs-lineage-demo.ps1`: the
  first real emitter (E16).
