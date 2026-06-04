# Guide 20 — Observability · Grafana LGTM (Prometheus + Loki + Tempo + Grafana + OTel)

> **Mirrors:** `nexus-infra-observability` (a **separate repo**) — the 6 Packer
> templates (`obs-{prom,loki,tempo,grafana,grafana-pg,otel-collector}-node`) + the 5
> Terraform envs (`obs-{prom,loki,tempo,grafana,otel}`) + the shared
> `nexus_observability` role (Vector fleet shipper) — Phase 0.I / ADR-0038, tier
> `01-foundation` extension. **The biggest guide: 14 nodes + 2 VRRP VIPs**, the
> platform's metrics/logs/traces/dashboards plane.

> 🔗 **Depends on Guide 16 (MinIO).** Loki + Tempo store their chunks/blocks in MinIO
> (`s3://loki`, `s3://tempo` via dedicated tenants). MinIO must be **powered on**.
> Also the last infra tier before the sharding guides (21–22).

---

## 1. Overview & purpose

The **Grafana LGTM** observability stack — **L**oki (logs) + **G**rafana
(dashboards) + **T**empo (traces) + **M**etrics (Prometheus), plus Alertmanager and
an OpenTelemetry ingest tier. **14 nodes, six roles + a fleet shipper:**

- **Metrics — `prom-1/2` (`.170/.171`)** — two Prometheus servers, each scraping
  every target (Grafana de-dups at query time), with an **Alertmanager gossip mesh**
  between them (`:9094` on the backplane). HA by *redundant scrape*, not sharding.
- **Logs — `loki-1/2/3` (`.172/.173/.174`)** — Loki in **simple-scalable** mode, a
  3-node `memberlist` ring, chunks + index in MinIO `s3://loki`.
- **Traces — `tempo-1/2/3` (`.175/.176/.177`)** — Tempo **scalable-single-binary**,
  3-node ring, blocks in MinIO `s3://tempo`, OTLP receivers on `:4317/:4318`.
- **Dashboards — `grafana-1/2` (`.178/.179`)** — active-active Grafana over a
  **shared PostgreSQL** state DB, fronted by a keepalived **VRRP VIP**
  `grafana.nexus.lab` (`.184`). The one human-facing UI; the LGTM datasources point
  at the engines above.
- **Grafana state DB — `grafana-pg-1/2` (`.180/.181`)** — a dedicated PG 17
  master-replica HA pair (streaming replication + VRRP VIP `grafana-db.nexus.lab`
  `.185`) holding Grafana's dashboards/users/orgs (so the two app nodes share state).
- **Ingest — `otel-collector-1/2` (`.182/.183`)** — an active-active OpenTelemetry
  Collector pair (round-robin `otel.nexus.lab`, no VIP) that receives app telemetry
  (OTLP) and fans it out: traces→Tempo, metrics→Prometheus, logs→Loki.
- **Fleet shipper — Vector** — on every fleet node, shipping journald/system logs
  straight to the Loki HA pair.

**Why LGTM + OTel:** one coherent plane for the three pillars (metrics, logs,
traces), all queryable through one Grafana, all durable on the shared MinIO object
store. **Why it matters:** this is how the whole platform is observed — every other
tier's `node_exporter`, logs, and app traces land here.

---

## 2. Component primer

- **Prometheus + Alertmanager.** Prometheus pulls (scrapes) `/metrics` from targets
  into a TSDB + evaluates alert rules; Alertmanager dedups/groups/routes the alerts.
  Two Proms both scrape everything (**HA by redundancy**); their two Alertmanagers
  **gossip** so an alert fires once. *Why two Proms not a cluster:* Prometheus has no
  native clustering — redundant scrape + Grafana de-dup is the canonical HA pattern.
  *Otherwise:* Thanos/Cortex/Mimir (heavier; deferred), VictoriaMetrics (retired here).
- **Loki.** A log aggregation system that indexes only *labels* (not full text) and
  stores compressed chunks in object storage. **Simple-scalable** = all components in
  each node, coordinated by a `memberlist` ring. *Why:* cheap, Grafana-native log
  search. *Otherwise:* Elasticsearch/OpenSearch (far heavier).
- **Tempo.** A trace backend — stores spans in object storage, queried by trace ID
  (and from logs via derived fields). **scalable-single-binary** = the multi-node
  mode that actually joins a ring. *Why:* cheap distributed tracing, Grafana-native.
  *Otherwise:* Jaeger (retired here — Tempo replaces it).
- **Grafana + shared PG.** The dashboard/exploration UI. Run **active-active**, both
  app nodes read/write the **same PostgreSQL** (dashboards, users, orgs) so either
  serves identical state; a VRRP VIP is the single URL. *Why external PG not local
  SQLite:* SQLite can't be shared → no HA. *Otherwise:* a single Grafana (no HA).
- **PostgreSQL HA (Grafana state).** The same streaming-replication + keepalived-VIP
  pattern as **Guide 17/19's PG pairs** — here holding Grafana's state. *Why a
  dedicated pair:* isolate the dashboard DB.
- **OpenTelemetry Collector.** A vendor-neutral telemetry pipeline: OTLP receivers →
  processors (batch, memory-limit, attributes) → exporters (Tempo/Prom/Loki). Apps
  emit OTLP to **one** endpoint; the Collector routes the three signals. *Why:* one
  ingest contract for all apps; swap backends without touching app code. *Otherwise:*
  per-backend SDK exporters in every app.
- **Vector.** A lightweight log shipper on every node; tails journald/system logs and
  pushes to Loki. *Why a separate shipper:* infra logs don't need the OTel hop —
  Vector → Loki directly. *Otherwise:* Promtail (Loki's own agent; Vector is more
  general + the platform standard).
- **memberlist ring.** A gossip-based membership protocol Loki + Tempo use to form
  their clusters (vs. an external KV like Consul/etcd). *Why:* no extra coordination
  service for the LGTM data tier.

---

## 3. Prerequisites

| # | Requirement | One-command verify |
|---|---|---|
| 1 | **Guide 16 (MinIO) built + POWERED ON**, with `loki` + `tempo` tenants/buckets | `mc ls nexuslocal/loki && mc ls nexuslocal/tempo` on `minio-1` |
| 2 | **Foundation alive** (Guides 00–04) — Vault PKI + KV; gateway DNS | `vault status` on `vault-1` → `Sealed: false` |
| 3 | **CA bundle** on the build host (`~/.nexus/vault-ca-bundle.crt`) | `Test-Path ~/.nexus/vault-ca-bundle.crt` → `True` |
| 4 | **14 `deb13` nodes** baselined (Guide 00), dual-NIC static `.170–.183` | the 14 answer `:22`; firstboot mapped `NEXUS_ROLE` / `NEXUS_PG_ROLE` |
| 5 | Egress to upstream release sites (Prometheus/Grafana/Loki/Tempo/OTel/Vector) | `curl -sI https://github.com/grafana/loki/releases | head -1` |

> **Versions:** Prometheus **2.55.1** + Alertmanager **0.28.0**, Loki **3.5.1**,
> Tempo **2.7.2**, Grafana OSS **11.6.3**, OTel Collector contrib **0.117.0**, Vector
> **0.50.0**, PostgreSQL **17**. Front doors: RR DNS `prometheus/loki/tempo/otel`
> `.nexus.lab`; VRRP VIPs `grafana.nexus.lab .184` + `grafana-db.nexus.lab .185`.

> **By-hand divergence:** read KV with `vault kv get` (no Vault Agent); issue certs
> with the `vault` CLI from the **`observability-server`** PKI role (its
> `allowed_domains` covers all 14 hostnames + the 5 RR DNS names + the 2 VIP DNS +
> localhost). Each service's TLS lands in its own `/etc/nexus-<svc>/tls/` (PKCS#8
> keys). The Grafana PG HA pair is the **same pattern as Guide 17 §5.4 / Guide 19
> §5.4** — referenced, not re-derived.

---

## 4. Target topology

| Node | Role | VMnet11 | VMnet10 | RAM | Ports |
|---|---|---|---|---|---|
| `prom-1` | Prometheus + Alertmanager | `.170` | `.10.170` | 4 GB | 9090 Prom · 9093 AM · 9094 AM-mesh |
| `prom-2` | Prometheus + Alertmanager | `.171` | `.10.171` | 4 GB | 9090 · 9093 · 9094 |
| `loki-1/2/3` | Loki simple-scalable | `.172/.173/.174` | `.10.172–.174` | 4 GB | 3100 HTTPS · 9095 gRPC · 7946 memberlist |
| `tempo-1/2/3` | Tempo scalable | `.175/.176/.177` | `.10.175–.177` | 4 GB | 3200 HTTP · 9095 gRPC · 4317/4318 OTLP · 7946 |
| `grafana-1/2` | Grafana HA | `.178/.179` | `.10.178/.179` | 3 GB | 3000 HTTPS |
| `grafana-pg-1/2` | Grafana state PG HA | `.180/.181` | `.10.180/.181` | 2 GB | 5432 |
| `otel-collector-1/2` | OTel Collector | `.182/.183` | `.10.182/.183` | 2 GB | 4317 gRPC · 4318 HTTP · 13133 health |
| **VIP** | `grafana.nexus.lab` | **`.184`** | — | — | 3000 → active Grafana |
| **VIP** | `grafana-db.nexus.lab` | **`.185`** | — | — | 5432 → PG primary |

> MAC block `:B2–:BF` (14 hosts; the 2 VIPs float, no MAC). All inter-component
> traffic (AM mesh, Loki/Tempo memberlist, Prom scrape, PG replication) rides the
> **VMnet10 backplane**; client/UI/OTLP ride VMnet11. VMs under
> `H:\VMS\NexusPlatform\01-foundation\<node>\`.

---

## 5. Step-by-step build

> **WHERE:** node steps as `nexusadmin` → `sudo -i` (root). `vault` runs on
> **`vault-1`**. Build order: metrics → logs → traces → Grafana state PG → Grafana →
> OTel → fleet Vector. Each engine reads its KV secrets with `vault kv get` and
> serves HTTPS with a leaf from the `observability-server` PKI role.

### 5.0 — Foundation: KV seeds, MinIO tenants, PKI role, DNS

> **Step 5.0.1 — Seed KV + create the MinIO Loki/Tempo tenants + the PKI role**
> **WHERE:** `vault-1` (KV + PKI); `minio-1` (tenants).
> **WHY:** the obs services read passwords + S3 keys from KV; Loki/Tempo need
> dedicated MinIO tenants (scoped service accounts + buckets, like Guide 16 §5.7.2);
> the `observability-server` PKI role issues every leaf.
> **WHAT (on vault-1 — KV + PKI):**
> ```bash
> export VAULT_ADDR=https://127.0.0.1:8200 VAULT_CACERT=$HOME/.nexus/vault-ca-bundle.crt
> # web-auth (Prom/AM basic-auth), grafana admin/session/db, S3 keys for loki/tempo
> vault kv put nexus/observability/prometheus/web-auth-password    value="$(openssl rand -hex 16)" password="$(openssl rand -hex 16)"
> vault kv put nexus/observability/grafana/admin-password          value="$(openssl rand -hex 16)"
> vault kv put nexus/observability/grafana/session-key             value="$(openssl rand -hex 32)"
> vault kv put nexus/observability/grafana-pg/superuser-password   value="$(openssl rand -hex 24)"
> vault kv put nexus/observability/grafana-pg/replication-password value="$(openssl rand -hex 24)"
> vault kv put nexus/observability/grafana-pg/grafana-db-password  value="$(openssl rand -hex 24)"
> vault kv put nexus/observability/loki/s3-access-key   value="nexus-loki-app"
> vault kv put nexus/observability/loki/s3-secret-key   value="$(openssl rand -base64 30)"
> vault kv put nexus/observability/tempo/s3-access-key  value="nexus-tempo-app"
> vault kv put nexus/observability/tempo/s3-secret-key  value="$(openssl rand -base64 30)"
> # the PKI role (32-entry allowed_domains; one role issues all 14 leaves)
> vault write pki_int/roles/observability-server \
>   allowed_domains='nexus.lab,prometheus.nexus.lab,alertmanager.nexus.lab,loki.nexus.lab,tempo.nexus.lab,otel.nexus.lab,grafana.nexus.lab,grafana-db.nexus.lab,prom-1,prom-2,loki-1,loki-2,loki-3,tempo-1,tempo-2,tempo-3,grafana-1,grafana-2,grafana-pg-1,grafana-pg-2,otel-collector-1,otel-collector-2,localhost' \
>   allow_subdomains=true allow_bare_domains=true allow_ip_sans=true enforce_hostnames=false \
>   server_flag=true client_flag=true key_type=rsa key_bits=2048 ttl=2160h max_ttl=2160h
> ```
> **WHAT (on minio-1 — Loki + Tempo tenants, like Guide 16 §5.7.2):**
> ```bash
> # for each of loki + tempo: mc mb the bucket, create a scoped policy (s3:* on that bucket), add the user
> for t in loki tempo; do
>   AK=$(vault kv get -field=value nexus/observability/$t/s3-access-key)
>   SK=$(vault kv get -field=value nexus/observability/$t/s3-secret-key)
>   sudo mc mb --ignore-existing nexuslocal/$t
>   printf '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["s3:*"],"Resource":["arn:aws:s3:::%s","arn:aws:s3:::%s/*"]}]}' "$t" "$t" > /tmp/$t.json
>   sudo mc admin policy create nexuslocal $t-tenant /tmp/$t.json 2>/dev/null || true
>   sudo mc admin user add nexuslocal "$AK" "$SK" 2>/dev/null || true
>   sudo mc admin policy attach nexuslocal $t-tenant --user "$AK" 2>/dev/null || true
> done
> ```
> **VERIFY:** `vault read pki_int/roles/observability-server`; `mc ls nexuslocal | grep -E 'loki|tempo'`.

> **Step 5.0.2 — Publish the RR DNS + VIP records (gateway)**
> **WHERE:** `nexus-gateway` (`.70.1`), root shell.
> **WHY:** the front doors — round-robin for the symmetric services, VIP A-records
> for the two keepalived VIPs.
> **WHAT:**
> ```bash
> cat > /etc/dnsmasq-observability.hosts <<'EOF'
> 192.168.70.170 prometheus.nexus.lab
> 192.168.70.171 prometheus.nexus.lab
> 192.168.70.170 alertmanager.nexus.lab
> 192.168.70.171 alertmanager.nexus.lab
> 192.168.70.172 loki.nexus.lab
> 192.168.70.173 loki.nexus.lab
> 192.168.70.174 loki.nexus.lab
> 192.168.70.175 tempo.nexus.lab
> 192.168.70.176 tempo.nexus.lab
> 192.168.70.177 tempo.nexus.lab
> 192.168.70.182 otel.nexus.lab
> 192.168.70.183 otel.nexus.lab
> 192.168.70.184 grafana.nexus.lab
> 192.168.70.185 grafana-db.nexus.lab
> EOF
> echo 'addn-hosts=/etc/dnsmasq-observability.hosts' > /etc/dnsmasq.d/observability-records.conf
> dnsmasq --test && systemctl reload dnsmasq
> ```
> **VERIFY:** `dig @192.168.70.1 prometheus.nexus.lab +short` → `.170` + `.171`;
> `dig @192.168.70.1 grafana.nexus.lab +short` → `.184`.

> **Step 5.0.3 — Issue + place a leaf cert on every obs node (the shared pattern)**
> **WHERE:** issue on `vault-1`; place on each node.
> **WHY:** every service serves HTTPS with a per-host leaf from `observability-server`.
> Use the **same by-hand issuance as Guides 16–19** (`vault write -format=json
> pki_int/issue/observability-server …` → split into `server.crt`/`server.key`
> (PKCS#8)/`ca.crt`). SANs add the node's RR DNS / VIP name (e.g. loki nodes add
> `loki.nexus.lab`; grafana-pg nodes add `grafana-db.nexus.lab` + IP-SAN `.185`).
> Place under each role's `/etc/nexus-<svc>/tls/` (e.g. `/etc/nexus-prometheus/tls`,
> `/etc/nexus-loki/tls`, …), owner `root:<svc-group>`, key `0640` (PG key **`0600`**).
> Also drop the chain world-readable at `/etc/ssl/certs/obs-ca.pem`.
> **WHAT (example — `loki-1`; repeat per node with that node's names/IPs):**
> ```bash
> # on vault-1:
> vault write -format=json pki_int/issue/observability-server \
>   common_name=loki-1.nexus.lab \
>   alt_names='loki-1,loki-1.nexus.lab,loki.nexus.lab,localhost' \
>   ip_sans='192.168.10.172,192.168.70.172,127.0.0.1' ttl=2160h > /tmp/loki-1.json
> # place on loki-1 (split leaf+intermediate -> server.crt; PKCS#8 -> server.key; chain -> ca.crt + /etc/ssl/certs/obs-ca.pem)
> ```
> **VERIFY (each node):** `openssl x509 -in /etc/nexus-<svc>/tls/server.crt -noout -ext subjectAltName`
> → the node's RR DNS / VIP name present.

### 5.1 — Metrics: Prometheus HA + Alertmanager mesh (`prom-1/2`)

> **Step 5.1.1 — Install Prometheus 2.55.1 + Alertmanager 0.28.0 (both nodes)**
> **WHERE:** `prom-1` + `prom-2`, root shell.
> **WHY:** two static binaries + system users. ⚠️ the Debian preseed already
> installed `prometheus-node-exporter`, which **created the `prometheus` user** — do
> **not** re-create it (a `usermod` on the in-use account fails, T1); just assert it
> exists.
> **WHAT (both nodes):**
> ```bash
> getent passwd prometheus >/dev/null || { echo "expected prometheus user from node-exporter"; exit 1; }
> getent group alertmanager >/dev/null || groupadd --system alertmanager
> getent passwd alertmanager >/dev/null || useradd --system --gid alertmanager --no-create-home --shell /usr/sbin/nologin alertmanager
> # binaries
> curl -fSL https://github.com/prometheus/prometheus/releases/download/v2.55.1/prometheus-2.55.1.linux-amd64.tar.gz | tar xz -C /tmp
> install -m0755 /tmp/prometheus-2.55.1.linux-amd64/prometheus /usr/local/bin/prometheus
> curl -fSL https://github.com/prometheus/alertmanager/releases/download/v0.28.0/alertmanager-0.28.0.linux-amd64.tar.gz | tar xz -C /tmp
> install -m0755 /tmp/alertmanager-0.28.0.linux-amd64/alertmanager /usr/local/bin/alertmanager
> install -d -o prometheus   -g prometheus   -m0750 /var/lib/nexus-prometheus   /etc/nexus-prometheus/tls
> install -d -o alertmanager -g alertmanager -m0750 /var/lib/nexus-alertmanager /etc/nexus-alertmanager/tls
> ```
> Install the two systemd units (verbatim from the repo — Prometheus with
> `--web.config.file`, `--web.enable-remote-write-receiver` so OTel can push, TSDB
> path/retention; Alertmanager with `--cluster.listen-address=0.0.0.0:9094`,
> `--cluster.advertise-address=${NEXUS_VMNET10_IP}:9094`, `--cluster.peer=${NEXUS_AM_PEER}`,
> `EnvironmentFile=/etc/nexus-alertmanager/cluster.env`). Leave both **disabled**.
> **VERIFY:** `prometheus --version` → `2.55.1`; `alertmanager --version` → `0.28.0`.

> **Step 5.1.2 — Render `prometheus.yml` + `web.yml` + `alertmanager.yml` + `cluster.env`**
> **WHERE:** each prom node, root shell.
> **WHY:** Prometheus scrapes both Proms + both AMs + the foundation node_exporters
> over HTTPS; AM has a null receiver (v0.1) + the gossip mesh. ⚠️ `web.yml` is
> **TLS-only, no `basic_auth_users`** — basic-auth would gate `/-/ready` and break
> the readiness probe (T6). The `cluster.env` `NEXUS_AM_PEER` is the *other* node's
> backplane IP + `:9094` (brace it — `${peer}:9094` — to avoid the PowerShell
> scope-qualifier trap, T5; by hand just write the literal).
> **WHAT (on `prom-1` — `prometheus.yml`):**
> ```bash
> cat > /etc/nexus-prometheus/prometheus.yml <<'EOF'
> global:
>   scrape_interval:     30s
>   evaluation_interval: 30s
>   external_labels:
>     cluster: nexus-observability
>     replica: prom-1
> alerting:
>   alertmanagers:
>     - scheme: https
>       tls_config:
>         ca_file: /etc/nexus-prometheus/tls/ca.crt
>         cert_file: /etc/nexus-prometheus/tls/server.crt
>         key_file: /etc/nexus-prometheus/tls/server.key
>       static_configs:
>         - targets: ['prom-1.nexus.lab:9093', 'prom-2.nexus.lab:9093']
> scrape_configs:
>   - job_name: prometheus
>     scheme: https
>     tls_config: { ca_file: /etc/nexus-prometheus/tls/ca.crt, cert_file: /etc/nexus-prometheus/tls/server.crt, key_file: /etc/nexus-prometheus/tls/server.key }
>     static_configs: [ { targets: ['prom-1.nexus.lab:9090','prom-2.nexus.lab:9090'] } ]
>   - job_name: alertmanager
>     scheme: https
>     tls_config: { ca_file: /etc/nexus-prometheus/tls/ca.crt, cert_file: /etc/nexus-prometheus/tls/server.crt, key_file: /etc/nexus-prometheus/tls/server.key }
>     static_configs: [ { targets: ['prom-1.nexus.lab:9093','prom-2.nexus.lab:9093'] } ]
>   - job_name: node_exporter
>     static_configs:
>       - targets: ['prom-1.nexus.lab:9100','prom-2.nexus.lab:9100','vault-1.nexus.lab:9100','vault-2.nexus.lab:9100','vault-3.nexus.lab:9100','vault-transit.nexus.lab:9100','nexus-gateway.nexus.lab:9100']
> EOF
> # web.yml (TLS-only; NO basic_auth_users -- T6)
> cat > /etc/nexus-prometheus/web.yml <<'EOF'
> tls_server_config:
>   cert_file: /etc/nexus-prometheus/tls/server.crt
>   key_file:  /etc/nexus-prometheus/tls/server.key
>   client_auth_type: NoClientCert
> EOF
> # alertmanager.yml (null receiver v0.1) + its web.yml (same TLS-only shape under /etc/nexus-alertmanager/)
> cat > /etc/nexus-alertmanager/alertmanager.yml <<'EOF'
> global: { resolve_timeout: 5m }
> route: { group_by: ['alertname','cluster'], group_wait: 30s, group_interval: 5m, repeat_interval: 12h, receiver: 'null' }
> receivers: [ { name: 'null' } ]
> inhibit_rules: []
> EOF
> cp /etc/nexus-prometheus/web.yml /etc/nexus-alertmanager/web.yml
> sed -i 's#nexus-prometheus#nexus-alertmanager#g' /etc/nexus-alertmanager/web.yml
> # cluster.env -- self backplane IP + the PEER's backplane IP:9094 (on prom-1 the peer is .10.171)
> printf 'NEXUS_VMNET10_IP=192.168.10.170\nNEXUS_AM_PEER=192.168.10.171:9094\n' > /etc/nexus-alertmanager/cluster.env
> chown root:prometheus /etc/nexus-prometheus/{prometheus.yml,web.yml}
> chown root:alertmanager /etc/nexus-alertmanager/{alertmanager.yml,web.yml,cluster.env}
> chmod 0640 /etc/nexus-prometheus/*.yml /etc/nexus-alertmanager/*.yml /etc/nexus-alertmanager/cluster.env
> ```
> **On `prom-2`:** identical but `replica: prom-2`, `NEXUS_VMNET10_IP=192.168.10.171`,
> `NEXUS_AM_PEER=192.168.10.170:9094`.
> **VERIFY:** `promtool check config /etc/nexus-prometheus/prometheus.yml` (if installed) or `grep node_exporter prometheus.yml`.

> **Step 5.1.3 — Start both (parallel) + verify the AM mesh formed**
> **WHERE:** both prom nodes (start), then either.
> **WHY:** start the two Alertmanagers close together so the gossip mesh forms; the
> mesh should show **2 peers**.
> **WHAT (both nodes):**
> ```bash
> systemctl daemon-reload
> systemctl enable --now nexus-prometheus nexus-alertmanager
> ```
> **EXPECTED:** both `/-/ready` return 200; the AM mesh has 2 peers.
> **VERIFY:** `curl -fsS -k https://localhost:9090/-/ready`; `curl -sk https://localhost:9093/api/v2/status | jq '.cluster.peers | length'` → `2` (T7: use `jq`, not inline python).

### 5.2 — Logs: Loki simple-scalable on MinIO (`loki-1/2/3`)

> **Step 5.2.1 — Install Loki 3.5.1 + render `loki.yaml` (all 3 nodes)**
> **WHERE:** `loki-1/2/3`, root shell.
> **WHY:** one binary + the `memberlist` ring across the 3 backplane IPs; chunks +
> index in MinIO `s3://loki`. ⚠️ gRPC `:9095` stays **plain-text** on the backplane
> (enabling `grpc_tls_config` without matching per-component client TLS breaks
> distributor→ingester writes, T10); TLS is on the client-facing `:3100`. The
> `ingester.chunk_idle_period=30s` makes small streams visible sub-minute (T12).
> **WHAT (on each Loki node — set `SELF` to this node's backplane IP `.10.172/.173/.174`):**
> ```bash
> getent group loki >/dev/null || groupadd --system loki
> getent passwd loki >/dev/null || useradd --system --gid loki --no-create-home --shell /usr/sbin/nologin loki
> curl -fSL https://github.com/grafana/loki/releases/download/v3.5.1/loki-linux-amd64.zip -o /tmp/loki.zip
> unzip -o /tmp/loki.zip -d /opt/loki && mv /opt/loki/loki-linux-amd64 /opt/loki/loki && chmod 0755 /opt/loki/loki
> install -d -o loki -g loki -m0750 /var/lib/nexus-loki /etc/nexus-loki/tls
> SELF=192.168.10.172   # <-- this node's backplane IP
> AK=$(vault kv get -field=value nexus/observability/loki/s3-access-key)
> SK=$(vault kv get -field=value nexus/observability/loki/s3-secret-key)
> cat > /etc/nexus-loki/loki.yaml <<EOF
> auth_enabled: false
> server:
>   http_listen_address: 0.0.0.0
>   http_listen_port: 3100
>   grpc_listen_address: 0.0.0.0
>   grpc_listen_port: 9095
>   http_tls_config: { cert_file: /etc/nexus-loki/tls/server.crt, key_file: /etc/nexus-loki/tls/server.key, client_auth_type: NoClientCert }
> common:
>   instance_addr: $SELF
>   path_prefix: /var/lib/nexus-loki
>   replication_factor: 3
>   ring: { kvstore: { store: memberlist } }
>   storage:
>     s3: { endpoint: https://minio.nexus.lab:9000, bucketnames: loki, access_key_id: $AK, secret_access_key: $SK, s3forcepathstyle: true, insecure: false }
> memberlist:
>   bind_addr: ["$SELF"]
>   bind_port: 7946
>   join_members: ["192.168.10.172:7946","192.168.10.173:7946","192.168.10.174:7946"]
> schema_config:
>   configs:
>     - { from: 2024-01-01, store: tsdb, object_store: s3, schema: v13, index: { prefix: index_, period: 24h } }
> storage_config:
>   tsdb_shipper: { active_index_directory: /var/lib/nexus-loki/tsdb-shipper-active, cache_location: /var/lib/nexus-loki/tsdb-shipper-cache, cache_ttl: 24h }
>   aws: { endpoint: https://minio.nexus.lab:9000, bucketnames: loki, access_key_id: $AK, secret_access_key: $SK, s3forcepathstyle: true, insecure: false }
> compactor: { working_directory: /var/lib/nexus-loki/compactor, retention_enabled: true, delete_request_store: s3 }
> ingester:
>   chunk_idle_period: 30s
>   max_chunk_age: 2h
>   wal: { enabled: true, dir: /var/lib/nexus-loki/wal }
> limits_config: { reject_old_samples: true, reject_old_samples_max_age: 168h, retention_period: 168h, allow_structured_metadata: true }
> EOF
> chown root:loki /etc/nexus-loki/loki.yaml ; chmod 0640 /etc/nexus-loki/loki.yaml
> ```
> Install the unit `nexus-loki.service`
> (`ExecStart=/opt/loki/loki -config.file=/etc/nexus-loki/loki.yaml -log.level=info`).
> **VERIFY:** after start, `curl -sk https://localhost:3100/ready` → `ready`.

> **Step 5.2.2 — Start all 3 + verify the memberlist ring (3 members)**
> **WHERE:** each Loki node.
> **WHY:** the ring must show 3 members (the deterministic liveness signal — cross-node
> *query* visibility has a 1–6 min Loki-intrinsic latency floor, so the data-plane
> round-trip is **not** the gate, T13).
> **WHAT (each node):** `systemctl daemon-reload && systemctl enable --now nexus-loki`
> **VERIFY:** `curl -sk https://localhost:3100/metrics | grep loki_build_info`;
> the ring page (`/ring`) lists 3 members.

### 5.3 — Traces: Tempo scalable-single-binary on MinIO (`tempo-1/2/3`)

> **Step 5.3.1 — Install Tempo 2.7.2 + render `tempo.yaml` (all 3 nodes)**
> **WHERE:** `tempo-1/2/3`, root shell.
> **WHY:** ⚠️ the service **must** run `-target=scalable-single-binary` or the
> `memberlist:` block is ignored (single-binary mode uses an in-memory ring, T15).
> Every ring component needs `instance_interface_names: ["nic1"]` + `instance_addr`
> (Tempo defaults to `eth0/en0`, which don't exist here — T16), and the querier needs
> an explicit `frontend_worker.frontend_address: 127.0.0.1:9095` (T18). Blocks in
> MinIO `s3://tempo`; OTLP receivers on `:4317/:4318`.
> **WHAT (on each Tempo node — set `SELF` to `.10.175/.176/.177`):**
> ```bash
> getent group tempo >/dev/null || groupadd --system tempo
> getent passwd tempo >/dev/null || useradd --system --gid tempo --no-create-home --shell /usr/sbin/nologin tempo
> curl -fSL https://github.com/grafana/tempo/releases/download/v2.7.2/tempo_2.7.2_linux_amd64.tar.gz | tar xz -C /opt/tempo tempo
> chmod 0755 /opt/tempo/tempo
> install -d -o tempo -g tempo -m0750 /var/lib/nexus-tempo /etc/nexus-tempo/tls
> SELF=192.168.10.175   # <-- this node's backplane IP
> AK=$(vault kv get -field=value nexus/observability/tempo/s3-access-key)
> SK=$(vault kv get -field=value nexus/observability/tempo/s3-secret-key)
> cat > /etc/nexus-tempo/tempo.yaml <<EOF
> server:
>   http_listen_address: 0.0.0.0
>   http_listen_port: 3200
>   grpc_listen_address: 0.0.0.0
>   grpc_listen_port: 9095
>   http_tls_config: { cert_file: /etc/nexus-tempo/tls/server.crt, key_file: /etc/nexus-tempo/tls/server.key, client_auth_type: NoClientCert }
> distributor:
>   receivers:
>     otlp:
>       protocols:
>         grpc: { endpoint: 0.0.0.0:4317, tls: { cert_file: /etc/nexus-tempo/tls/server.crt, key_file: /etc/nexus-tempo/tls/server.key } }
>         http: { endpoint: 0.0.0.0:4318, tls: { cert_file: /etc/nexus-tempo/tls/server.crt, key_file: /etc/nexus-tempo/tls/server.key } }
> ingester:
>   lifecycler:
>     address: $SELF
>     interface_names: ["nic1"]
>     ring: { kvstore: { store: memberlist }, replication_factor: 3 }
>   max_block_duration: 5m
> querier:
>   frontend_worker: { frontend_address: 127.0.0.1:9095 }
> memberlist:
>   bind_addr: ["$SELF"]
>   bind_port: 7946
>   join_members: ["192.168.10.175:7946","192.168.10.176:7946","192.168.10.177:7946"]
> compactor:
>   ring: { instance_interface_names: ["nic1"], instance_addr: $SELF, kvstore: { store: memberlist } }
>   compaction: { block_retention: 168h }
> storage:
>   trace:
>     backend: s3
>     s3: { endpoint: https://minio.nexus.lab:9000, bucket: tempo, access_key: $AK, secret_key: $SK, insecure: false, forcepathstyle: true }
>     wal: { path: /var/lib/nexus-tempo/wal }
>     local: { path: /var/lib/nexus-tempo/blocks }
> metrics_generator:
>   ring: { instance_interface_names: ["nic1"], instance_addr: $SELF, kvstore: { store: memberlist } }
>   storage: { path: /var/lib/nexus-tempo/generator-wal }
> EOF
> chown root:tempo /etc/nexus-tempo/tempo.yaml ; chmod 0640 /etc/nexus-tempo/tempo.yaml
> ```
> Install `nexus-tempo.service`
> (`ExecStart=/opt/tempo/tempo -config.file=/etc/nexus-tempo/tempo.yaml -target=scalable-single-binary`).
> **VERIFY:** after start, `curl -sk https://localhost:3200/ready` → `ready`; `/memberlist` shows 3 members.

### 5.4 — Grafana state DB: PG 17 HA pair (`grafana-pg-1/2`, VIP `.185`)

> **Step 5.4.1 — Build the PG master-replica pair (same as Guide 17/19 §5.4)**
> **WHERE:** `grafana-pg-1` (primary) + `grafana-pg-2` (replica), root shell.
> **WHY:** the dashboard state DB. Build it **exactly like Guide 17 §5.4 / Guide 19
> §5.4** — PGDG install (bookworm fallback for `libicu72`/`libldap-2.5-0`), primary
> conf.d + `pg_hba` + `repluser` + `grafana` role + `grafana` DB, replica
> `pg_basebackup -R` (with `.pgpass`), keepalived VRRP VIP `.185` (`virtual_router_id
> 85`, the versioned `pg_isready` check). ⚠️ **The replica must ALSO write the
> `pg_hba.conf` replication block** — `pg_basebackup` copies the DATA dir but
> `pg_hba.conf` lives in the CONFIG dir, so without this the promoted replica can't
> accept the old primary as a standby after a failover (T24). The PG cert SAN includes
> `grafana-db.nexus.lab` + IP-SAN `.185`.
> **WHAT (deltas from Guide 17 §5.4 — the names):** DB `grafana`, role `grafana`,
> conf.d `/etc/postgresql/17/main/conf.d/nexus-grafana.conf`, `pg_hba` marker
> `NEXUS-GRAFANA-HBA`, VIP `.185`, KV `nexus/observability/grafana-pg/*`, cert dir
> `/etc/nexus-grafana-pg/tls/`. Write the `NEXUS-GRAFANA-HBA` block on **both** nodes
> (T24):
> ```
> # NEXUS-GRAFANA-HBA  (on BOTH primary and replica)
> host    replication   repluser   192.168.10.0/24   scram-sha-256
> hostssl grafana       grafana    192.168.70.0/24   scram-sha-256
> hostssl all           postgres   192.168.70.0/24   scram-sha-256
> ```
> **VERIFY:** `psql -tAc "SELECT count(*) FROM pg_stat_replication"` → `1` (primary);
> `pg_is_in_recovery()` → `t` (replica); VIP `.185` on exactly one node;
> `psql -tAc "SELECT 1 FROM pg_database WHERE datname='grafana'"` → `1`.

### 5.5 — Dashboards: Grafana HA over shared PG (`grafana-1/2`, VIP `.184`)

> **Step 5.5.1 — Install Grafana 11.6.3 (both nodes)**
> **WHERE:** `grafana-1` + `grafana-2`, root shell.
> **WHY:** Grafana OSS from `apt.grafana.com`, run active-active over the shared PG.
> **WHAT (both nodes):**
> ```bash
> apt-get install -y apt-transport-https software-properties-common
> mkdir -p /etc/apt/keyrings
> curl -fsSL https://apt.grafana.com/gpg.key | gpg --dearmor > /etc/apt/keyrings/grafana.gpg
> echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" > /etc/apt/sources.list.d/grafana.list
> apt-get update && apt-get install -y grafana=11.6.3
> apt-get install -y keepalived
> install -d -o root -g grafana -m0750 /etc/nexus-grafana/tls
> systemctl disable --now grafana-server keepalived 2>/dev/null || true
> ```
> **VERIFY:** `grafana-server -v` → `11.6.3`.

> **Step 5.5.2 — Render `grafana.ini` + datasources + keepalived; start (each node)**
> **WHERE:** `grafana-1` (prio 110) + `grafana-2` (prio 100), root shell.
> **WHY:** HTTPS Grafana over the shared PG (`verify-full` to `grafana-db.nexus.lab`),
> the LGTM datasources (Prometheus basic-auth, Loki, Tempo — with a derived field
> linking Loki→Tempo by trace ID), and the keepalived VIP `.184` (`virtual_router_id
> 84`, `chk_grafana` curls `/api/health`). ⚠️ import the obs CA into the **system
> trust** (`update-ca-certificates`) so Grafana's datasource HTTPS clients validate;
> and any apply-time `--cacert /etc/nexus-grafana/tls/ca.crt` curl must be **sudo**'d
> (the `0750 root:grafana` dir isn't `nexusadmin`-traversable, T21) and the health
> check must tolerate Grafana's pretty-printed `"database": "ok"` (whitespace — use
> `jq`, T23).
> **WHAT (set `SELF`/`PEER`/`PRIO` per node):**
> ```bash
> SELF=192.168.70.178 ; PEER=192.168.70.179 ; PRIO=110   # grafana-1 (grafana-2: SELF .179, PEER .178, PRIO 100)
> ADMINPW=$(vault kv get -field=value nexus/observability/grafana/admin-password)
> SESSION=$(vault kv get -field=value nexus/observability/grafana/session-key)
> GRAFPW=$(vault kv get -field=value nexus/observability/grafana-pg/grafana-db-password)
> PROMPW=$(vault kv get -field=password nexus/observability/prometheus/web-auth-password)
> cat > /etc/grafana/grafana.ini <<EOF
> [paths]
> data = /var/lib/grafana
> provisioning = /etc/grafana/provisioning
> [server]
> protocol = https
> http_addr = 0.0.0.0
> http_port = 3000
> domain = grafana.nexus.lab
> root_url = https://grafana.nexus.lab:3000/
> cert_file = /etc/nexus-grafana/tls/server.crt
> cert_key  = /etc/nexus-grafana/tls/server.key
> [database]
> type = postgres
> host = grafana-db.nexus.lab:5432
> name = grafana
> user = grafana
> password = $GRAFPW
> ssl_mode = verify-full
> ca_cert_path = /etc/nexus-grafana/tls/ca.crt
> [security]
> admin_user = admin
> admin_password = $ADMINPW
> secret_key = $SESSION
> cookie_secure = true
> [analytics]
> reporting_enabled = false
> check_for_updates = false
> EOF
> chown root:grafana /etc/grafana/grafana.ini ; chmod 0640 /etc/grafana/grafana.ini
> # datasources
> install -d -m0750 -o root -g grafana /etc/grafana/provisioning/datasources
> cat > /etc/grafana/provisioning/datasources/nexus-obs.yaml <<EOF
> apiVersion: 1
> deleteDatasources:
>   - { name: Prometheus, orgId: 1 }
>   - { name: Loki, orgId: 1 }
>   - { name: Tempo, orgId: 1 }
> datasources:
>   - name: Prometheus
>     type: prometheus
>     access: proxy
>     url: https://prometheus.nexus.lab:9090
>     isDefault: true
>     basicAuth: true
>     basicAuthUser: admin
>     secureJsonData: { basicAuthPassword: $PROMPW }
>     jsonData: { tlsAuthWithCACert: true, tlsSkipVerify: false }
>   - name: Loki
>     type: loki
>     access: proxy
>     url: https://loki.nexus.lab:3100
>     jsonData:
>       tlsSkipVerify: false
>       derivedFields:
>         - { name: TraceID, matcherRegex: 'traceID=(\w+)', url: '\${__value.raw}', datasourceUid: tempo }
>   - name: Tempo
>     type: tempo
>     uid: tempo
>     access: proxy
>     url: https://tempo.nexus.lab:3200
>     jsonData: { tlsSkipVerify: false }
> EOF
> chown root:grafana /etc/grafana/provisioning/datasources/nexus-obs.yaml ; chmod 0640 /etc/grafana/provisioning/datasources/nexus-obs.yaml
> # trust the obs CA system-wide (datasource HTTPS validation)
> cp /etc/nexus-grafana/tls/ca.crt /usr/local/share/ca-certificates/nexus-obs-ca.crt
> update-ca-certificates
> # keepalived VIP .184 (chk_grafana curls /api/health; keepalived runs as root)
> cat > /usr/local/sbin/nexus-grafana-check.sh <<'EOS'
> #!/bin/bash
> exec /usr/bin/curl -fsS --max-time 4 --cacert /etc/nexus-grafana/tls/ca.crt --resolve grafana.nexus.lab:3000:127.0.0.1 https://grafana.nexus.lab:3000/api/health -o /dev/null
> EOS
> chmod 0755 /usr/local/sbin/nexus-grafana-check.sh
> cat > /etc/keepalived/keepalived.conf <<EOF
> global_defs { script_user root }
> vrrp_script chk_grafana { script "/usr/local/sbin/nexus-grafana-check.sh" ; interval 5 ; fall 3 ; rise 2 }
> vrrp_instance VI_GRAFANA {
>   state BACKUP
>   nopreempt
>   interface nic0
>   virtual_router_id 84
>   priority $PRIO
>   advert_int 1
>   unicast_src_ip $SELF
>   unicast_peer { $PEER }
>   authentication { auth_type PASS ; auth_pass grafanvr }
>   virtual_ipaddress { 192.168.70.184/24 dev nic0 }
>   track_script { chk_grafana }
> }
> EOF
> systemctl daemon-reload ; systemctl enable --now grafana-server keepalived
> ```
> **EXPECTED:** both nodes serve `/api/health` with `database=ok`; the VIP binds on one.
> **VERIFY:** `sudo curl -fsS --cacert /etc/nexus-grafana/tls/ca.crt --resolve grafana.nexus.lab:3000:127.0.0.1 https://grafana.nexus.lab:3000/api/health | jq -e '.database=="ok"'`;
> `ip addr show nic0 | grep .184` on exactly one node.

### 5.6 — Ingest: OTel Collector pair (`otel-collector-1/2`, RR `otel.nexus.lab`)

> **Step 5.6.1 — Install OTel Collector contrib 0.117.0 + render `config.yaml` (both)**
> **WHERE:** `otel-collector-1` + `otel-collector-2`, root shell.
> **WHY:** the active-active ingest tier — OTLP receivers (mTLS) → processors →
> three exporters (Tempo/Prom/Loki). ⚠️ the user/group **must** be `otel` (the shared
> firstboot maps `IDENTITY_GROUP=otel`; the upstream `otelcol` name breaks the
> firstboot `chown`, T30). Loki uses its **native OTLP** endpoint
> (`https://loki.nexus.lab:3100/otlp`; the deprecated `loki` exporter was removed in
> Collector 0.86+). Prom basic-auth from KV.
> **WHAT (both nodes — `$hostName` = `otel-collector-1`/`-2`):**
> ```bash
> getent group otel >/dev/null || groupadd --system otel
> getent passwd otel >/dev/null || useradd --system --gid otel --no-create-home --shell /usr/sbin/nologin otel
> curl -fSL https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.117.0/otelcol-contrib_0.117.0_linux_amd64.tar.gz | tar xz -C /opt/otel-collector otelcol-contrib
> chmod 0755 /opt/otel-collector/otelcol-contrib
> install -d -o root -g otel -m0750 /etc/nexus-otel-collector/tls
> PROMPW=$(vault kv get -field=password nexus/observability/prometheus/web-auth-password)
> cat > /etc/nexus-otel-collector/config.yaml <<EOF
> receivers:
>   otlp:
>     protocols:
>       grpc: { endpoint: 0.0.0.0:4317, tls: { cert_file: /etc/nexus-otel-collector/tls/server.crt, key_file: /etc/nexus-otel-collector/tls/server.key, ca_file: /etc/nexus-otel-collector/tls/ca.crt } }
>       http: { endpoint: 0.0.0.0:4318, tls: { cert_file: /etc/nexus-otel-collector/tls/server.crt, key_file: /etc/nexus-otel-collector/tls/server.key, ca_file: /etc/nexus-otel-collector/tls/ca.crt } }
> processors:
>   memory_limiter: { check_interval: 5s, limit_mib: 400, spike_limit_mib: 100 }
>   batch: { timeout: 5s, send_batch_size: 1024 }
>   attributes: { actions: [ { key: nexus_collector, value: $(hostname), action: upsert }, { key: fleet, value: nexusplatform, action: upsert } ] }
> exporters:
>   otlp/tempo:
>     endpoint: tempo.nexus.lab:4317
>     tls: { ca_file: /etc/nexus-otel-collector/tls/ca.crt, insecure: false }
>     retry_on_failure: { enabled: true, initial_interval: 5s, max_interval: 60s, max_elapsed_time: 300s }
>   otlphttp/loki:
>     endpoint: https://loki.nexus.lab:3100/otlp
>     tls: { ca_file: /etc/nexus-otel-collector/tls/ca.crt, insecure: false }
>     retry_on_failure: { enabled: true }
>   prometheusremotewrite:
>     endpoint: https://prometheus.nexus.lab:9090/api/v1/write
>     tls: { ca_file: /etc/nexus-otel-collector/tls/ca.crt, insecure: false }
>     auth: { authenticator: basicauth/prom }
>     retry_on_failure: { enabled: true }
> extensions:
>   basicauth/prom: { client_auth: { username: admin, password: $PROMPW } }
>   health_check: { endpoint: 127.0.0.1:13133 }
> service:
>   extensions: [basicauth/prom, health_check]
>   pipelines:
>     traces:  { receivers: [otlp], processors: [memory_limiter, attributes, batch], exporters: [otlp/tempo] }
>     metrics: { receivers: [otlp], processors: [memory_limiter, attributes, batch], exporters: [prometheusremotewrite] }
>     logs:    { receivers: [otlp], processors: [memory_limiter, attributes, batch], exporters: [otlphttp/loki] }
> EOF
> chown root:otel /etc/nexus-otel-collector/config.yaml ; chmod 0640 /etc/nexus-otel-collector/config.yaml
> ```
> Install `nexus-otel-collector.service` (`User=otel`, `Group=otel`,
> `ExecStart=/opt/otel-collector/otelcol-contrib --config=/etc/nexus-otel-collector/config.yaml`),
> then `systemctl enable --now nexus-otel-collector`.
> **EXPECTED:** `/health` (`127.0.0.1:13133`) returns 200; OTLP `:4317/:4318` listening.
> **VERIFY:** `curl -fsS http://127.0.0.1:13133/`; `ss -ltn | grep -E '4317|4318'`;
> `dig @192.168.70.1 otel.nexus.lab +short` → `.182` + `.183`.

### 5.7 — Fleet log shipping: Vector → Loki (every fleet node)

> **Step 5.7.1 — Install Vector 0.50.0 + the Loki sink on each fleet node**
> **WHERE:** every running Linux fleet node, root shell.
> **WHY:** ship journald/system logs straight to the Loki HA pair (infra logs don't
> need the OTel hop). ⚠️ Vector's `loki` sink needs `encoding` (T34) + an explicit
> `data_dir` matching the created dir (T35); some nodes need a defensive resolv.conf
> write before the install curl (T36).
> **WHAT (on each node):**
> ```bash
> getent hosts packages.timber.io >/dev/null || echo "nameserver 192.168.70.1" > /etc/resolv.conf   # T36
> curl -fsSL https://sh.vector.dev | bash -s -- -y --prefix /opt/vector || true
> install -d -o root -g root -m0755 /var/lib/nexus-vector /etc/nexus-vector
> cat > /etc/nexus-vector/vector.yaml <<'EOF'
> data_dir: /var/lib/nexus-vector
> sources:
>   journald: { type: journald }
> sinks:
>   loki:
>     type: loki
>     inputs: [journald]
>     endpoint: https://loki.nexus.lab:3100
>     encoding: { codec: json }
>     labels: { fleet: nexusplatform, host: "{{ host }}" }
>     tls: { verify_certificate: false }
> EOF
> # install nexus-vector.service (ExecStart=/opt/vector/bin/vector --config /etc/nexus-vector/vector.yaml)
> systemctl daemon-reload ; systemctl enable --now nexus-vector
> ```
> **EXPECTED:** `nexus-vector.service` active, shipping to Loki.
> **VERIFY:** `systemctl is-active nexus-vector` → `active`; `vector validate /etc/nexus-vector/vector.yaml`.

---

## 6. Validation — by-hand acceptance smoke (demo / playbook)

Condensed from `smoke-0.I.1..6.ps1`. Per-node SSH probes from the **build host**.

- **Input:** the 14 nodes up; MinIO on; rings formed; VIPs bound; Grafana healthy.
- **Where observed:** SSH to each node / `curl` on each engine / `psql` on grafana-pg
  / `dig` on the gateway / the Grafana UI.
- **Proves:** a working LGTM plane — metrics, logs, traces, dashboards, ingest.
- **Prerequisites:** Guides 00–04 + 16 (powered on) alive; §5 complete.

| # | Check | Command | Pass criteria |
|---|---|---|---|
| 1 | 14 nodes reachable | `ssh …@170..183 'echo ok'` | all `ok` |
| 2 | Prometheus HA | `curl -k https://localhost:9090/-/ready` (both prom) | `200` |
| 3 | Alertmanager mesh | `…:9093/api/v2/status \| jq '.cluster.peers\|length'` | `2` |
| 4 | Prom scrape targets up | `…:9090/api/v1/targets` | foundation node_exporters `up` |
| 5 | Loki ring | `…:3100/metrics \| grep loki_build_info`; `/ring` | 3 members |
| 6 | Tempo ring | `…:3200/ready`; `/memberlist` | ready; 3 members |
| 7 | MinIO tenants | `mc ls nexuslocal/loki && mc ls nexuslocal/tempo` | buckets present |
| 8 | Grafana PG replication | `SELECT count(*) FROM pg_stat_replication` (pg-1) | `1` |
| 9 | Grafana PG VIP | `ip addr show nic0 \| grep .185` (each pg) | exactly one |
| 10 | Grafana HA health | `curl -k …/api/health \| jq .database` (both) | `"ok"` (shared PG) |
| 11 | Grafana VIP | `ip addr show nic0 \| grep .184` (each grafana) | exactly one |
| 12 | Datasources | `…/api/datasources` (admin) | Prometheus + Loki + Tempo |
| 13 | OTel health + OTLP | `curl 127.0.0.1:13133`; `ss -ltn \| grep 4317` | 200; listening (both) |
| 14 | RR + VIP DNS | `dig prometheus/loki/tempo/otel/grafana/grafana-db.nexus.lab +short` | correct IPs/VIPs |
| 15 | Fleet Vector | `systemctl is-active nexus-vector` (fleet nodes) | `active` |
| 16 | **VIP failover** (chaos) | stop active Grafana / PG primary; VIP + service move | UI/DB stays up |

**1–15 green ⇒ Guide 20 satisfied.** 16 is the HA payoff (both VIPs). **This
completes the Observability tier (Phase 0.I) and the infrastructure layer's
non-sharding tiers.**

---

## 7. Teardown / reset

```bash
for ip in 182 183; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-otel-collector'; done
for ip in 178 179; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now grafana-server keepalived'; done
for ip in 180 181; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now keepalived postgresql@17-main'; done
for ip in 175 176 177; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-tempo'; done
for ip in 172 173 174; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-loki'; done
for ip in 170 171; do ssh nexusadmin@192.168.70.$ip 'sudo systemctl disable --now nexus-prometheus nexus-alertmanager'; done
# fleet: per node 'sudo systemctl disable --now nexus-vector'
# gateway: rm /etc/dnsmasq.d/observability-records.conf /etc/dnsmasq-observability.hosts ; systemctl reload dnsmasq
# then vmrun stop + deleteVM each of the 14 (Guide 00 §7).
```

> **Loki logs + Tempo traces persist in MinIO** (`s3://loki`, `s3://tempo`); Grafana
> dashboards persist in the grafana-pg `grafana` DB. Wipe by emptying the buckets +
> dropping the DB. The Vault KV creds + the MinIO tenants survive (sticky).

---

## 8. Troubleshooting (representative; full ledger = handbook §3.A–§3.F, 37 transients)

| # | Symptom | Cause | Fix |
|---|---|---|---|
| **T1** | `prometheus` user creation fails (`usermod: user is currently used`) | the preseed's `prometheus-node-exporter` already created + is using the `prometheus` user | assert it exists (`getent`), don't re-create (§5.1.1). |
| **T5** | Alertmanager: `split host/port for peer : missing port` | `$peer:9094` parsed as a PowerShell scope-qualifier → empty | brace `${peer}:9094`; by hand write the literal `…171:9094` (§5.1.2). |
| **T6** | Prom `/-/ready` returns 401 | `basic_auth_users` in `web.yml` gates *all* endpoints incl. liveness | drop `basic_auth_users` — TLS-only at the wire; Grafana adds human auth (§5.1.2). |
| **T10** | Loki distributor→ingester `error reading server preface: EOF` | `grpc_tls_config` on the gRPC listener without matching per-component client TLS | leave gRPC `:9095` plain-text on the backplane; TLS on `:3100` (§5.2.1). |
| **T12** | Loki push→query round-trip times out | default `chunk_idle_period=30m` delays small-stream flush | set `ingester.chunk_idle_period=30s` + WAL (§5.2.1); cross-node query has a 1–6 min floor (T13) — gate on the **ring**, not the round-trip. |
| **T15** | Tempo `memberlist` ignored / ring count 0 | default `-target=single-binary` uses an in-memory ring | run `-target=scalable-single-binary` (§5.3.1). |
| **T16** | Tempo: `no useable address found for interfaces [eth0 en0]` | Tempo auto-detects `eth0/en0`; our NICs are `nic0/nic1` | `instance_interface_names: ["nic1"]` + `instance_addr` on every ring (§5.3.1). |
| **T18** | Tempo: `querier: frontend worker address not specified` | scalable mode needs an explicit frontend address | `querier.frontend_worker.frontend_address: 127.0.0.1:9095` (§5.3.1). |
| **T21** | Grafana `/api/health` apply-check fails (`TLS handshake error … EOF`) | the `--cacert` curl ran as `nexusadmin`, who can't traverse `0750 root:grafana` | `sudo`-prefix the health curl; keepalived's check runs as root already (§5.5.2). |
| **T23** | Grafana health check never matches | grep for `"database":"ok"` misses Grafana's pretty-printed `"database": "ok"` | parse with `jq -e '.database=="ok"'` (whitespace-tolerant) (§5.5.2). |
| **T24** | After a PG failover the old primary can't rejoin | `pg_basebackup` copies DATA not `pg_hba.conf`; the replica had no replication rule | write the `NEXUS-GRAFANA-HBA` block on **both** nodes from day one (§5.4.1). |
| **T30** | OTel firstboot fails (`chown: invalid group 'root:otel'`) | the role created group `otelcol`; the shared firstboot expects `otel` | use user/group **`otel`** everywhere (role + service + chown) (§5.6.1). |
| **T32** | OTel: `exporters::otlp/tempo requires non-empty endpoint` (`endpoint: :4317`) | `$tempoDns:4317` parsed as a scope-qualifier → empty | brace `${tempoDns}:4317`; by hand write the literal `tempo.nexus.lab:4317` (§5.6.1). |
| **T34** | Vector crashloops `missing field 'encoding'` | the `loki` sink needs `encoding` | `encoding: { codec: json }` (+ `data_dir` matching the created dir, T35) (§5.7.1). |
| **—** | Backplane down (ring/replication can't form) | VMware left nic1 NO-CARRIER at power-on | reconnect the 2nd NIC + `systemctl restart systemd-networkd` (as Guides 16–19). |

> **CI note (for the automated repo):** always `terraform fmt -recursive` before push
> — hand-written overlays drift the `=` alignment and fail `packer-validate`.

---

### Cross-references

- **0.I architecture:** memory `project_nexus_infra_observability_phase`; ADR-0038
  (Grafana LGTM on MinIO); ADR-0025 (LB-tier VIP HA); ADR-0031 (RR DNS, no VIP for writes)
- **Network canon:** `nexus-platform-plan/docs/infra/network.md` (obs `.170–.183` +
  VIPs `.184/.185`, MAC `:B2–:BF`); `vms.yaml`
- **Automated equivalents:** `nexus-infra-observability/packer/obs-*-node/` +
  `terraform/envs/obs-{prom,loki,tempo,grafana,otel}/role-overlay-*.tf` + the shared
  `nexus_observability` role (Vector)
- **Smoke mirror:** `nexus-infra-observability/scripts/smoke-0.I.{1..6}.ps1`
- **Depends on:** [`16-lakehouse-minio.md`](./16-lakehouse-minio.md) (the `loki` + `tempo` S3 tenants)
- **Related PG HA pattern:** [`17-lakehouse-iceberg-nessie.md`](./17-lakehouse-iceberg-nessie.md) §5.4 + [`19-registry-harbor-ha.md`](./19-registry-harbor-ha.md) §5.4 (the streaming-repl + keepalived base the Grafana PG reuses)
- **Transients:** [[feedback_cluster_template_nftables_backplane]] · [[powershell-url-scope-qualifier]] · [[keepalived-check-versioned-binary]]
- **Previous guide:** [`19-registry-harbor-ha.md`](./19-registry-harbor-ha.md)
- **Next guide:** Guide 21 — Sharding · Vitess (MySQL). See [`INDEX.md`](../INDEX.md). **(Observability complete; the last two guides are the sharding tiers.)**
