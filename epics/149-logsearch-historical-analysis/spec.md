# `logsearch` — historical full-text log-analysis tier — design spec

**Epic:** `logical-minds-foundry/.github#149`
**Status:** draft (spec) — pushback review applied 2026-07-29
**Scope:** lab core infrastructure (stack-agnostic; no MQ coupling)
**Depends on:** the existing log pipeline — the log-streaming design
([`2026-06-12-lab-log-streaming-design.md`](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/blob/develop/docs/specs/2026-06-12-lab-log-streaming-design.md))
and the MQ JSON-logging design
([`2026-06-19-mq-json-logging-design.md`](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/blob/develop/docs/specs/2026-06-19-mq-json-logging-design.md)).

---

## 1. Problem & motivation

The lab already has an open-source log aggregator: Grafana **Alloy → Loki →
Grafana**. Alloy is the sole collector (`ansible/roles/alloy/templates/config.alloy.j2`),
MQ diagnostics are already single-line JSON in the journal, and everything is
queryable in Grafana with LogQL. That pipeline is excellent at one thing and
structurally bad at another.

- **What Loki does well — the 1st and 2nd order view.** *"Is it up, how loaded,
  what is happening right now?"* Loki is label-indexed and Grafana-native, so
  live-tail and recent-window queries are fast and ergonomic. A dashboard is
  optimized for the real-time / recent view: even a time-series panel is a
  *window*, looked at from altitude.
- **What Loki does badly — the 3rd and 4th order view.** *"How often has this
  happened, when, and is there a pattern?"* Loki's label model is not a full-text
  inverted index, and its home is a dashboard, not an investigation surface.
  Questions like *"how frequently have we seen channel retries over this run,"*
  *"what time of day does this class of error cluster,"* or *"has this recurred,
  and how often"* are exactly what a dashboard is not for. Answering them is a
  full-text-search + aggregation problem over a long-horizon corpus — a different
  engine class.

This epic **adds** that engine class as a bolt-on tier **alongside** Loki (never
replacing it): a dedicated, full-text, historical log-analysis platform with a
Kibana-class investigation UI. It sits *downstream* of the existing collection —
it receives logs indirectly from Alloy and does not touch how MQ emits them — so
it is additive and does not disturb the metrics/live-log stack.

### Why this belongs in the lab

The lab's thesis is to build a **reproducible mini-enterprise** that demonstrates
how enterprise functionality is *actually* implemented. Real shops run a
Splunk/Elasticsearch-class tier for historical log investigation; the lab has not
modelled it. This tier closes that gap and is deliberately built as **stack-agnostic
core infrastructure** (a sibling of `obs`), so it rides unchanged into future labs
(e.g. a database-resiliency lab) — the reusable asset is the generic lab core, not
an MQ-specific component.

### Which side of the boundary this sits on

`logsearch` is **instrumentation**, not instrumented system. The security-boundary
tenet (§10) governs: MQ and everything that touches it is secured and HA/DR-modelled
maximally; the telemetry plane that carries emitted logs is deliberately simple. Two
consequences settled during design:

- **The store is single-node.** We do not make the analysis tier itself HA/DR. That
  is the trap the lab already rejected once (the multi-month MQ-exporter-HA battle):
  making the observability tier resilient buys the *demonstration* little and costs a
  great deal. Replicating OpenSearch across sites is the same trap in new clothes.
- **The valuable resiliency story rides on the *pipeline*, not the store.** *"Do the
  logs stay complete and correctly captured across an MQ DR failover?"* is a real,
  in-scope scenario — but it is verified by querying an ordinary single-node store
  *after* the failover, because the components that fail over with MQ (journald /
  Alloy on the MQ nodes) are what is under test. It needs no A/B `logsearch`. It is a
  named follow-on scenario (§12), not v1 machinery.

---

## 2. Goals & success criteria

**Goal.** Stand up a full-text, long-horizon log-analysis tier that ingests the
lab's existing structured-log corpus, keeps it at the build-volume tier of
ephemerality, and is usable by hand through an authentic investigation UI.

**Success criteria.** After a fresh **cold rebuild** and a lab run of some
duration:

1. A dedicated `logsearch` node runs OpenSearch + OpenSearch Dashboards and is
   reachable on the management plane.
2. Alloy fans out the **same** structured corpus it already sends to Loki to a
   **second sink** on the `logsearch` node; the Loki path is unchanged.
3. An operator opens OpenSearch Dashboards **Discover**, full-text-searches across
   the accrued corpus, and runs a **generic** time-bucketed aggregation (e.g.
   events-per-hour by severity) that returns non-empty against **real** data.
   *(Demonstrating a specific interesting pattern — channel-retry frequency — is
   follow-on work that induces the condition; see §11–§12.)*
4. The corpus lives on a **dedicated fixed-size persistent volume** backed by
   `build/state/logsearch/` and **survives a `logsearch` node cold-rebuild**
   (`vagrant destroy logsearch && … up` re-attaches the existing indices). It is
   destroyed only by a full `build/` (macOS) / `/vergil` (cloud) purge — the
   build-volume tier of ephemerality, and no more.
5. `mqlab logsearch status` reports tier health **including disk-used and any
   read-only-index state**, so a full or wedged tier is **loud, not silent**;
   `mqlab logsearch open` prints the Dashboards URL.
6. Everything renders/provisions from topology, is unit-tested where it is pure
   code, and passes `vrg-container-run -- vrg-validate`.

**Explicit non-goal for v1:** no seeded queries, saved searches, dashboards, or
investigation runbook. v1 proves the tier stands up, persists at the build-volume
tier, and is usable *by hand*. Deciding *which* investigations matter, and building
them, is the epic's follow-on brainstorm (`mq-resiliency-lab-for-linux#819`).

---

## 3. Approach & alternatives considered

**Chosen: OpenSearch + OpenSearch Dashboards** (Apache-2.0 forks of Elasticsearch
7.10 and Kibana). It is the *authentic* article: a true inverted index, the full
query DSL, and the Dashboards **Discover** investigation surface (field extraction,
drill-down, saved searches, aggregations). Two decisive reasons:

1. **Fidelity to the thesis.** The lab exists to show how enterprise log
   investigation is really done; this is the near-identical OSS twin of the
   Elasticsearch/Kibana stack real shops run, with direct skill transfer.
2. **Footprint is affordable.** The heaviest option (JVM; ~3–5 GB), but the lab
   already gives `obs` its own node, and cloud instances make RAM a non-issue. A
   dedicated node keeps the JVM from starving the metrics stack.

**Rejected:**

- **VictoriaLogs** (single Go binary, LogsQL) — ~10× lighter and strong at the
  "how often / when" analytics, but its UI is not Kibana-class and LogsQL is not
  the Elastic DSL, so no fidelity and no skill transfer. *Named fallback* if the
  local VM footprint proves intolerable.
- **Quickwit** (Rust, ES-compatible API, sub-second full text) — real inverted
  index at a fraction of the weight, but no first-class investigation UI; splits
  the difference and fully satisfies neither goal.
- **Graylog** — Splunk-like UX but needs OpenSearch **+ MongoDB + Graylog server**
  (three services); heaviest overall, only worth it if the investigation *workflow
  UI* were the headline.
- **SigNoz / ClickHouse** — excellent trend analytics, but pulls toward a
  metrics/traces platform that overlaps Prometheus.

---

## 4. Architecture & data flow

The collection layer does not change. Alloy stays the **sole collector** and gains
**one fan-out sink**:

```
                                  ┌──────────────► Loki  ──► Grafana   (1st/2nd order: live/recent)
journald (MQ JSON) ─┐             │                                    UNCHANGED
mqweb messages.log ─┼─► Alloy ────┤
cluster/app logs   ─┘  (collector)│
                                  └──────────────► OpenSearch ──► OpenSearch Dashboards
                                     NEW second sink              (3rd/4th order: historical/search)
                                                  │
                                                  └── path.data → dedicated fixed-size volume
                                                                  (build/state/logsearch/, tier-2 durable)
```

This honours the JSON-logging design's explicit principle that **transport is a
swappable boundary**: the Loki path is untouched, and the new export is purely
additive. `logsearch` receives logs *indirectly*; it never touches MQ or how MQ
emits diagnostics.

**Stack-agnostic.** The `logsearch` tier makes no MQ-specific assumption. It
indexes whatever structured stream Alloy forwards; field mappings are derived from
the log envelope, not from MQ semantics.

**Lab-self-observability-ready.** The node bakes in `node-exporter` like every
other lab node, so the `logsearch` node's own health and disk-space appear in the
lab's metrics fleet with no rework, ready for the lab-health board that the
follow-on brainstorm (§12) will design.

---

## 5. The `logsearch` node

A new dedicated node, provisioned exactly as `obs` is (`lab/topology.yaml` declares
`obs` as `obs-ubuntu2404`, cpus 2 / memory 4096, running the whole metrics stack
baked in):

- **Node.** A new `logsearch` entry in `lab/topology.yaml`, on the management
  plane, sized **~2 CPU / 6–8 GB** (OpenSearch JVM heap + Dashboards). **Single
  node** — no A/B, no replication (see §1, §10).
- **Box.** A new baked fat box `logsearch-ubuntu2404` via
  `lab/boxes/build-fatbox.sh --box logsearch-ubuntu2404` (mirroring
  `--box obs-ubuntu2404` and `ansible/bake-obs.yml`), with OpenSearch +
  OpenSearch Dashboards + node-exporter + Alloy baked in.
- **Host prerequisites baked into the role/box.** `vm.max_map_count = 262144`
  (OpenSearch refuses to start otherwise) and the OpenSearch **version pinned** via
  the repo's version-manifest mechanism — so a routine re-bake never installs a
  newer major that cannot open the persisted indices on the durable volume (§7).
- **Rationale.** Its own node (a) prevents the JVM from contending with
  Prometheus/Grafana/Loki on `obs`, and (b) keeps the tier a clean, liftable unit —
  consistent with the OSS-component-boundary principle (configure only what it
  owns; never mutate across the boundary). Collapsing tiers onto shared VMs later
  is a topology edit, not a redesign.

---

## 6. Ingest — Alloy fan-out

Alloy already collects the corpus: journald with a syslog-identifier relabel for
MQ-core JSON, plus a conditional tail of mqweb's `messages.log` on QM-bearing nodes
(`ansible/observability.yml`, `ansible/roles/alloy/`). The tier adds a second
**write** of that same collected stream to OpenSearch.

**Connector — the one real feasibility risk, de-risked first.** Whether Alloy can
write to OpenSearch natively is unverified and must not be assumed (Alloy embeds a
*curated* subset of OpenTelemetry Collector components). This is the **first
implementation task**, a spike that gates the rest, with a **guaranteed fallback
named now** so the architecture cannot dead-end:

1. **Alloy-native** — an `otelcol.exporter.*` (elasticsearch/opensearch) if the
   Alloy build bundles it. Cleanest: single collector, no new component.
2. **Fallback: OpenSearch Data Prepper** on the `logsearch` node — OpenSearch's own
   first-party ingestion; Alloy exports OTLP to it. Definitely works, so the path
   is guaranteed. Cost: a second pipeline component alongside Alloy.

Whichever wins, the collection topology stays clean and the Loki path is untouched.

**Index model.** Time-based **daily indices** `logs-YYYY.MM.DD` (via an index
template) with **`number_of_replicas: 0`** — the correct setting for a single-node
cluster, so health reads **green** honestly rather than a permanent replica-starved
**yellow**. Daily indices also make a future retention policy a per-index drop.

---

## 7. Storage & persistence — the build-volume tier

The corpus sits at the **build-volume tier of ephemerality**: it survives node/VM
cold-rebuilds (so you can iterate on the `logsearch` node without losing history),
and is destroyed only by an intentional full `build/` (macOS) / `/vergil` (cloud)
purge. As durable as we want, and no more — the same gitignored, unmanaged,
purge-at-whim convenience that makes the baked boxes stable.

**Mechanism: a dedicated fixed-size persistent block volume.** OpenSearch runs
*inside* the `logsearch` guest, and the lab deliberately disables synced folders
(`lab/Vagrantfile:26`), so the in-guest process cannot see the host `build/` tree.
The corpus therefore lives on a **dedicated libvirt data volume**:

- Backed by `build/state/logsearch/` (resolved via **`mqlab build path state`**,
  never a hardcoded `build/<X>` path) — the *irreplaceable, shared* bucket that
  survives a routine `mqlab build clean` and is dropped only under the explicit
  `--yes-destroy-state` guard
  ([`docs/development/build-layout.md`](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/blob/develop/docs/development/build-layout.md)),
  in the same host-durable family as `build/state/boxes/` and
  `build/state/snapshots/`.
- Attached to the guest as a **real block device** (native filesystem semantics and
  performance) and mounted at OpenSearch's `path.data`. **Not** a synced
  folder / virtiofs mount: OpenSearch is mmap-heavy and explicitly warns against
  passthrough/shared filesystems for its data path, so a block volume is the correct
  (and only safe) choice for a data store.
- **Kept across `vagrant destroy`** so a node cold-rebuild re-attaches the existing
  indices. Adding `logsearch/` as a new `state/` subdir is a documented change to the
  bucket inventory in `build-layout.md` (picked up by the doc-review bookend, `#818`).

**Fixed size is a feature, not a limitation.** The volume is sized up front, which
**bounds** how much of `build/` this can ever consume — directly addressing the
disk-limit pain the lab has hit before. It caps the footprint, and OpenSearch's disk
**flood-stage watermark** (default 95%) acts as a clean guardrail against a bounded
ceiling rather than a runaway across the whole host disk.

**Retention is out of scope for v1.** Within the fixed volume the corpus accumulates
by default; automated retention/GC is a named follow-on (`#819`), modelled on
baked-image GC. v1's obligations are only: **do not lose data on a node rebuild**,
**bound the footprint** (fixed volume), and **fail loud** when full (§5, §9) — never
silently wedge.

---

## 8. Provisioning

Ansible, not shell (native modules give idempotency and fail-loud for free), and a
structural mirror of the observability provisioning:

- Roles: `opensearch` and `opensearch-dashboards` under `ansible/roles/`,
  peers of the existing `loki` / `grafana` / `prometheus` / `alloy` roles. The
  `opensearch` role owns: the `vm.max_map_count` sysctl, the pinned version, the
  data-dir mount on the persistent volume, and the index template
  (`number_of_replicas: 0`, daily indices).
- Play: `ansible/site-logsearch.yml` (configure half) + `ansible/bake-logsearch.yml`
  (bake half), mirroring `site-obs.yml` / `bake-obs.yml`.
- The Alloy fan-out sink is added to the existing `alloy` role's config template,
  gated so only the intended sources are duplicated to OpenSearch.
- The persistent-volume attach + keep-on-`destroy` behaviour is wired through the
  topology/`Vagrantfile` (vagrant-libvirt additional-disk lifecycle) — confirmed by
  a spike (§13) since the lab has no existing keep-on-destroy volume precedent.

---

## 9. CLI surface

A new `mqlab logsearch` command group (a `src/mqlab/logsearch.py` module wired into
`src/mqlab/cli.py`, peer of `obslog.py` / `dashboard.py`), minimal in v1:

- `mqlab logsearch status` — tier health: node reachable, OpenSearch cluster status
  (**green** expected on a single node with `replicas: 0`; red / unreachable is the
  only failure), Dashboards up, index/doc counts, **disk-used on the data volume,
  and any read-only-index state** so a full tier is loud.
- `mqlab logsearch open` (a.k.a. `url`) — print the OpenSearch Dashboards URL.

Error vocabulary follows the layered convention: `mqlab` messages name
fully-qualified `mqlab` commands; OpenSearch's own tooling speaks for itself. No
`uv run` in any runtime path — companion tools are invoked by bare name via `$PATH`.

---

## 10. Security posture

**v1: management-plane-only, no hardening.** The tier is exposed only on the mgmt
plane (the same model as `obs`), and the OpenSearch admin credential is
**runtime-injected**, following the existing `MQWEB_ADMIN_*` pattern
(`OPENSEARCH_ADMIN_*` / the OpenSearch initial-admin-password env var), never
committed. Recent OpenSearch requires a bootstrapped admin password; the plan wires
it through the same runtime-injection seam the mqweb credential uses. No TLS/RBAC
hardening in v1.

**The design tenet this records (the reason the above is correct):**

> **The security bar is the instrumented system's interface, not the telemetry
> pipeline.** Every path that *touches MQ* — MQ-to-MQ channels, app-to-QM
> connections, MQ writing its own diagnostics — is secured maximally (TLS, real
> auth), because that is the production behaviour the lab exists to demonstrate
> faithfully. Once MQ has *securely emitted* a log line, it has crossed into the
> lab's deliberately-**unhardened** telemetry plane (journald → Alloy → Loki /
> `logsearch`) — the same lab that keeps its secrets in plain view under `build/`.
> Hardening the analysis tier would simulate the wrong thing.

Whether the analysis tier nonetheless warrants *some* security demonstration is
itself a named follow-on (`#819`), not a v1 blocker.

---

## 11. Testing & validation

- **Unit tests** for all pure code: topology rendering of the `logsearch` node,
  the `mqlab logsearch` command logic (status parsing, health/disk interpretation,
  URL construction), any index naming/template helper.
- **Cold-rebuild acceptance gate.** This is bring-up/provisioning work, so per the
  lab's standing rule it is accepted only after a **full VM cold rebuild** proves
  one-pass provisioning — lint-green is not done. This is the epic's `validation`
  operational task (seeded at plan time, blocked-by the implementation tasks). It
  asserts:
  1. **Baseline (always true on a healthy build):** the node is up and reachable,
     ingestion is *flowing* (doc count rising), the corpus is full-text-searchable,
     and a generic time-bucketed aggregation returns non-empty.
  2. **Persistence:** `vagrant destroy logsearch && … up` **re-attaches** the
     existing corpus (the durable volume survives the node rebuild).
  3. **Loud-not-silent:** `mqlab logsearch status` surfaces disk-used and would
     report a read-only/full state rather than wedging silently.
  Demonstrating a *specific induced pattern* (e.g. channel-retry frequency) is
  deferred to the follow-on, which fits the induce-and-assert shape of the live-lab
  validation framework (epic #38).
- `vrg-container-run -- vrg-validate` is the only validation command.

---

## 12. Scope boundary — named follow-ons (not v1)

Tracked under the epic's follow-on brainstorm (`mq-resiliency-lab-for-linux#819`):

1. **Query/investigation catalog** — enumerate the typical questions
   (channel-retry frequency, problem recurrence, time-of-day patterns) → implement
   as saved searches/filters.
2. **Obs + log dashboards side-by-side** — the integrated operator surface.
3. **Security-posture review** — does the tier warrant any security demo, or is
   unhardened correct?
4. **Retention & disk-space management** — modelled on baked-image GC (within the
   fixed volume).
5. **Synthetic-corpus extraction** — harvest real accrued scenarios into a curated,
   known-answer dataset for future PoCs.
6. **Cross-instance central aggregation** — one `logsearch` fed by many cloud
   instances (each store is self-contained in v1; central aggregation is designed-for,
   built later).
7. **Lab self-observability** — a lab-health board showing the lab's *own*
   infrastructure (`obs`, `logsearch`, and per-node disk-space across the fleet).
8. **Logging pipeline across an MQ DR failover** — verify logs stay complete and
   correctly captured when the instrumented system fails over (single-node store
   suffices; induce-and-assert per epic #38).

**Explicitly not in this epic, and not a follow-on of it — a possible *separate
future epic*:** a *resilient logging tier* (HA/DR of the store itself). Real
enterprises run HA SIEM, but doing so here repeats the rejected instrumentation-HA
trap; if ever wanted, it is its own initiative, not scope creep on standing up v1.

---

## 13. Open questions

1. **Alloy → OpenSearch connector** (§6) — native `otelcol` exporter vs. Data
   Prepper fallback. Resolved by the first (gating) spike; the fallback guarantees
   the path cannot dead-end.
2. **Persistent-volume keep-on-`destroy`** (§7, §8) — confirm the vagrant-libvirt
   additional-disk approach that survives `vagrant destroy`; the lab has no existing
   precedent, so this is a spike.
3. **Volume size** — the fixed allocation for `build/state/logsearch/`; sized for a
   useful multi-day corpus while bounding build-disk consumption. Confirmed
   empirically.
4. **Node sizing** (§5) — 6 GB vs 8 GB heap headroom; confirmed during the
   cold-rebuild validation.
5. **Which nodes' logs fan out in v1** — the full fleet, or an initial subset
   matching the mqweb-tail gating already in `observability.yml`. Leaning: the same
   source set Loki already receives, for parity.
