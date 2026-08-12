# Minimal, reliable OpenSearch log-search tier — spec

## 1. Problem and context

The lab's `logsearch` node runs the **full OpenSearch 3.8.0 bundle** — core plus
roughly fifteen default plugins (security, ML Commons, SQL, Anomaly Detection,
k-NN with native libraries, Alerting, Index State Management, Observability,
Reports, Notifications, and more). On the node's two vCPUs those plugins churn
through initialization for more than twelve minutes — load average was observed
at 170 — and OpenSearch never binds its HTTP port `:9200`. Because OpenSearch
never becomes reachable, Data Prepper cannot reach its sink and never binds its
OTLP port `:21892`, so `ansible/site-logsearch.yml`'s readiness wait fails and the
`observe` phase aborts.

This became critical when `logsearch` was promoted to a **core observability
layer** (#1018, merged): every `mqlab bootstrap` now boots and provisions the
tier, and the observe phase renders the Alloy fan-out gate and runs
`site-logsearch.yml` **before** Alloy is configured. A broken `logsearch` now
blocks **all** observability, and it is a hard blocker for the Grafana logs and
events dashboards (epic `logical-minds-foundry/.github#169`, Wave 1b).

The tier has exactly one job: forward all logs and events
(Alloy → Data Prepper OTLP → OpenSearch), make them searchable, and serve them to
Grafana. Everything else the bundle carries is dead weight.

## 2. Goal

Make the `logsearch` tier cold-boot **reliably and fast** on the lab's existing
node by running OpenSearch **core-only**, so `mqlab bootstrap` brings the whole
log and event pipeline up clean and searchable — and prove it with **repeated**
cold boots, the gap that let the current breakage slip through.

## 3. Design decisions (approved)

1. **OpenSearch → the "min" (core-only) distribution.** The min tarball is core
   OpenSearch with **zero plugins**. It was verified available for 3.8.0 at
   `https://artifacts.opensearch.org/releases/core/opensearch/3.8.0/opensearch-min-3.8.0-linux-{arm64,x64}.tar.gz`
   (HTTP 200). Core alone covers every requirement: Data Prepper writes documents
   over REST, operators full-text search them, and Grafana reads them over REST.
2. **Plaintext.** The min distribution ships no security plugin, so OpenSearch
   runs plaintext HTTP on `:9200` on the internal management network — no TLS, no
   auth. This is a lab on a private network; plaintext is the intended, simpler
   posture and removes the bundle's forced `https` + `admin:admin`.
3. **Data Prepper — keep.** It remains the Alloy → OpenSearch connector (#939).
   Its OpenSearch sink is repointed to plaintext `http`/no-auth.
4. **OpenSearch Dashboards — keep.** Dashboards' "Discover" view is a valuable way
   to confirm that the *expected* logs and events actually landed in OpenSearch
   while the Grafana boards are being built. It is repointed at the now-plaintext
   OpenSearch. It is dropped only if it later proves too expensive on the node.
5. **Grafana OpenSearch datasource** is **created** (it does not exist today —
   Grafana provisions only Prometheus and Loki) as a plaintext `http`/no-auth
   datasource, so Grafana can query the store and #169 Wave 1b has a datasource
   to build log/event panels on.
6. **Node sizing.** Keep the node at **2 vCPU / 6 GB** initially. Measure startup
   and steady state; bump vCPU only if core-min plus Data Prepper plus Dashboards
   still cannot fit. Footprint reduction is the primary lever, not hardware.

## 4. Changes by component

### 4.1 OpenSearch role (`ansible/roles/opensearch`)

- `defaults/main.yml`: change `opensearch_pkg` / `opensearch_url` to the **min**
  artifact (`opensearch-min-<version>-linux-<arch>` under `/releases/core/…`).
  The unpacked directory name differs from the bundle's, so the install and
  symlink tasks that reference `opensearch-{{ opensearch_version }}` must track
  the min directory name.
- `tasks/install.yml` / `tasks/configure.yml`: remove the security configuration
  from `opensearch.yml` and the per-run admin-password drop-in
  (`opensearch.service.d/admin.conf`). With no security plugin present, the demo
  security bootstrap and TLS settings are gone; `opensearch.yml` keeps only the
  core settings (single-node discovery, `network.host`, http port, cluster/node
  name, data and snapshot paths).
- Heap stays a tunable (`opensearch_heap`), sized to leave headroom for Data
  Prepper and Dashboards on a 6 GB node.

### 4.2 Data Prepper role (`ansible/roles/data-prepper`)

- Fix the malformed systemd `Environment=` line
  (`tasks/install.yml:95`): `Environment=JAVA_OPTS=-Xms{{ heap }} -Xmx{{ heap }}`
  is **unquoted**, so systemd splits it on the space into `JAVA_OPTS=-Xms…` and a
  bogus `-Xmx…` (rejected: "Invalid environment assignment, ignoring: -Xmx1g").
  Quote the value: `Environment="JAVA_OPTS=…"`.
- Repoint the OpenSearch sink to plaintext `http://<logsearch-mgmt-ip>:9200` with
  no credentials.

### 4.3 OpenSearch Dashboards role (`ansible/roles/opensearch-dashboards`)

- Repoint at the plaintext OpenSearch endpoint; drop TLS and credentials from its
  configuration.

### 4.4 Grafana OpenSearch datasource

- **Create** the OpenSearch datasource — it does not exist yet.
  `ansible/roles/grafana/templates/datasource.yml.j2` provisions only Prometheus
  and Loki. Add a third entry: the OpenSearch datasource type, plaintext
  `http://<logsearch-mgmt-ip>:9200`, no TLS, no credentials, and a **pinned
  `uid`** so #169 Wave 1b's log/event panels can reference it the same way the
  cockpit boards pin `{type: prometheus, uid: prometheus}` and `{type: loki,
  uid: loki}`. This is the "serve them to Grafana" leg of the tier and the
  prerequisite the Wave 1b dashboards build on.

### 4.5 Baked box (`lab/boxes/build-fatbox.sh`, `BAKE=logsearch`)

- Rebuild `logsearch-ubuntu2404` with the min distribution baked in, so a cold
  boot reuses a min-based box rather than the bundle.

### 4.6 Readiness waits (`ansible/site-logsearch.yml`)

- Re-tune the OpenSearch and Data Prepper readiness waits to the new, fast reality
  (core OpenSearch should bind `:9200` in roughly one to two minutes). Keep them
  **fail-loud** — a tier that does not come up must still abort the observe phase.

### 4.7 Node sizing (`lab/topology.yaml`)

- Keep the `logsearch` node at 2 vCPU / 6 GB for the first cut. If measurement
  shows core-min plus Data Prepper plus Dashboards cannot fit, bump vCPU (the
  host has ample cores) and record the change.

## 5. Acceptance criteria

A cold `mqlab bootstrap nativeha-ubuntu --no-dr` brings `logsearch` up in a single
pass:

- OpenSearch binds `:9200` within roughly one to two minutes.
- Data Prepper binds its OTLP port `:21892`.
- A probe log line is searchable **both** directly in OpenSearch (REST query) and
  via Grafana's OpenSearch datasource.
- The obs guest's disk stays flat — no Alloy "sending queue is full" loop, no
  disk-fill.
- The observe phase completes and `mqlab bootstrap` exits zero.
- A **repeat** cold boot reproduces all of the above (reliability, not a one-off).

## 6. Verification

The acceptance is a live check, not a unit test, so it is carried by a
**cold-rebuild validation task** (infra/provisioning epics carry one by default):
tear down, `bootstrap --no-dr`, assert the criteria in §5, then repeat the cold
boot and assert again. Unit-testable pieces (role defaults, template rendering,
the data-prepper unit quoting) are covered by `vrg-validate` in the member repo.

## 7. Non-goals and deferred

- **TLS / auth.** Out of scope; plaintext on the private management net is the
  intended lab posture. Revisit only if the lab's trust model changes.
- **Retention / index-lifecycle (ISM).** The min distribution has no ISM plugin.
  Defer log retention; revisit when disk growth bites, either by adding *only* the
  ISM plugin or by a small "delete indices older than N days" job.
- **Dropping Data Prepper.** Alloy could in principle write to OpenSearch directly
  via the OpenTelemetry `elasticsearch` exporter (OpenSearch is ES-API
  compatible), removing a whole JVM — but #939 deliberately routed through Data
  Prepper. Treat any such change as a **spike**, not a commitment in this epic.
- **Dropping Dashboards.** Kept for now (§3.4); drop only if it proves too heavy.

## 8. Prior art and references

- OpenSearch distributions: the **bundle** (`/releases/bundle/…`, core plus
  default plugins) versus the **min** (`/releases/core/…`, core only). This epic
  switches to min.
- `#939` — the decision to route Alloy → OpenSearch through Data Prepper.
- `#1018` — logsearch promoted to a core observability layer (booted and
  provisioned on every bootstrap, before Alloy is configured).
- `logical-minds-foundry/.github#169` Wave 1b — the Grafana logs and events
  dashboards this epic unblocks.

## 9. Risks and open questions

- **Sizing after strip-down.** Even core-only, three services (OpenSearch, Data
  Prepper, Dashboards) share a 6 GB / 2 vCPU node. If startup is still marginal,
  the fallback is a vCPU bump (§4.7); the further fallback is dropping Dashboards
  (§3.4).
- **Min directory naming.** The min tarball's unpacked directory name must be
  reconciled with the install/symlink tasks that hard-reference the bundle name.
- **Config drift.** Removing security touches `opensearch.yml`, the Data Prepper
  sink, Dashboards, and the Grafana datasource together; they must move as one so
  no consumer is left dialing `https`/auth.
