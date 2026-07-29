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
4. The corpus is durable at the **build-volume tier via host-side snapshots**:
   `mqlab logsearch snapshot` captures a consistent copy to `build/state/logsearch/`
   (host-durable on `/vergil` / macOS `build/`), and bring-up **auto-restores** the
   latest snapshot if one exists. So a `logsearch` node cold-rebuild — or a full
   `vrg-vm rebuild` — preserves history **provided a snapshot was taken first**
   (automatable into graceful teardown). A rebuild with no prior snapshot starts
   empty: an accepted, explicit tradeoff. History dies only on a `build/` / `/vergil`
   purge. *(The live corpus runs on the guest's ordinary ephemeral disk; durability
   is the snapshot, not a persistent volume — see §7 and #386/#397.)*
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
                          path.data → guest disk (ephemeral) ── mqlab logsearch snapshot ──►
                                                  build/state/logsearch/ (host-durable, auto-restored on bring-up)
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
- **Data location.** The live corpus is OpenSearch's `path.data` on the guest's own
  **ephemeral** disk (the libvirt default pool, wiped on rebuild like every other
  guest volume — #386/#397). Durability is the host-side snapshot (§7), not a
  persistent volume.
- **Host prerequisites baked into the role/box.** `vm.max_map_count = 262144`
  (OpenSearch refuses to start otherwise) and the OpenSearch **version pinned** via
  the repo's version-manifest mechanism — so a routine re-bake never installs a
  version that cannot **restore** an existing snapshot (OpenSearch refuses to restore
  a snapshot taken by a newer version; a pin keeps the snapshot/restore contract
  stable — §7).
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

## 7. Storage & persistence — host-side snapshot/restore

The corpus sits at the **build-volume tier of ephemerality**, but the lab's
architecture dictates *how* it gets there, and it is not a live persistent volume.

**Why not a persistent volume (the reality check).** OpenSearch runs *inside* the
`logsearch` guest. Guest volumes live in libvirt's default pool on the **ephemeral
boot disk** and are wiped on rebuild; the lab **tried** redirecting persistent VM
storage onto `/vergil` and **reverted it** — #376 *"put wipe-on-rebuild overlays on a
never-wiped disk, orphaning volumes and breaking rebuilds,"* reverted in **#386**
(`bb5bcd7`) and documented in **#397** (`97d2539`): *"the VM image pool lives on the
ephemeral boot disk, not /vergil."* And synced folders are disabled
(`lab/Vagrantfile:26`), so the guest cannot mount `build/` either. A keep-on-`destroy`
data volume is therefore the exact mechanism the lab already abandoned. We do not
resurrect it.

**Mechanism: host-side snapshot/restore** — the same idiom the lab already uses to
persist in-guest state (`lab/scripts/lab-snapshot.sh` writes golden VM state to
`build/state/snapshots/`), and the one path that reaches host-durable storage without
a live mount:

- The **live corpus** runs on the guest's ordinary ephemeral `path.data`.
- **`mqlab logsearch snapshot`** captures a **consistent** copy and lands it under
  **`build/state/logsearch/`** on the host (resolved via **`mqlab build path state`**,
  never a hardcoded `build/<X>` path) — the *irreplaceable, shared* bucket
  ([`docs/development/build-layout.md`](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/blob/develop/docs/development/build-layout.md)),
  in the same host-durable family as `build/state/snapshots/`. Transport is
  **Ansible** (fetch guest → host), consistent with the Vagrantfile's *"Ansible owns
  file transport"* — no synced folder.
- **Bring-up auto-restores** the latest snapshot if one exists in
  `build/state/logsearch/` (the `site-logsearch.yml` configure half stages it back to
  the guest and restores before/at OpenSearch start).

**The durability contract (accepted tradeoff).** Preserving history across a rebuild
requires a **snapshot action first** — which we automate into the graceful
shutdown/rebuild path. Blow the node (or the whole dev box) away *without* snapshotting
and it comes back empty; that is acceptable and explicit. History dies only on a
`build/` / `/vergil` purge. Point-in-time, not continuous — bounded loss is covered by
auto-snapshot on graceful teardown plus an optional periodic snapshot.

**Snapshot mechanism (spike, §13).** Either OpenSearch's native snapshot API (register
a filesystem repository, `_snapshot`) or a cold copy of a *stopped* `path.data`; the
spike picks one on evidence. Adding `logsearch/` as a new `state/` subdir is a
documented change to the bucket inventory in `build-layout.md` (doc-review bookend
`#818`).

**Disk safety — two bounded surfaces.** (1) The **live** corpus is bounded by the
guest data disk; OpenSearch's disk **flood-stage watermark** (default 95%) is the
guardrail, surfaced loudly by `mqlab logsearch status` (§9) rather than wedging
silently. (2) The **snapshots** in `build/` are bounded by keeping the last *N* — but
snapshot GC / retention is a named follow-on (`#819`), modelled on baked-image GC.
v1's only obligations: **restore what was snapshotted**, and **fail loud** when the
live disk fills.

---

## 8. Provisioning

Ansible, not shell (native modules give idempotency and fail-loud for free), and a
structural mirror of the observability provisioning:

- Roles: `opensearch` and `opensearch-dashboards` under `ansible/roles/`,
  peers of the existing `loki` / `grafana` / `prometheus` / `alloy` roles, each split
  along the same bake/configure line as `loki` (`tasks/install.yml` +
  `tasks/configure.yml` + `tasks/main.yml`). The `opensearch` role owns: the
  `vm.max_map_count` sysctl, the pinned version, `path.data` on the guest disk, the
  index template (`number_of_replicas: 0`, daily indices), the snapshot filesystem
  repository, and the **restore-on-bring-up** step.
- Play: `ansible/site-logsearch.yml` (configure half — enable+start, plus stage &
  auto-restore the latest host snapshot if present) + `ansible/bake-logsearch.yml`
  (bake half — binaries + static config, inert), mirroring `site-obs.yml` /
  `bake-obs.yml`.
- The Alloy fan-out sink is added to the existing `alloy` role's config template,
  gated so only the intended sources are duplicated to OpenSearch.
- Snapshot/restore transport is **Ansible fetch/copy** between the guest snapshot
  repository and `build/state/logsearch/` on the control host — no synced folder, no
  keep-on-`destroy` volume (which #386/#397 ruled out).

---

## 9. CLI surface

A new `mqlab logsearch` command group (a `src/mqlab/logsearch.py` module wired into
`src/mqlab/cli.py`, peer of `obslog.py` / `dashboard.py`), minimal in v1:

- `mqlab logsearch status` — tier health: node reachable, OpenSearch cluster status
  (**green** expected on a single node with `replicas: 0`; red / unreachable is the
  only failure), Dashboards up, index/doc counts, **disk-used on `path.data`, and any
  read-only-index state** so a full tier is loud.
- `mqlab logsearch open` (a.k.a. `url`) — print the OpenSearch Dashboards URL.
- `mqlab logsearch snapshot` — capture a consistent snapshot to `build/state/logsearch/`
  (the durable, host-side copy; automatable into graceful teardown).
- `mqlab logsearch restore [--snapshot <id>]` — restore the latest (or a named)
  host snapshot into the live store. *(Bring-up already auto-restores the latest via
  `site-logsearch.yml`; this is the manual/explicit verb.)*

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
  2. **Snapshot/restore round-trip:** `mqlab logsearch snapshot` →
     `vagrant destroy logsearch && … up` → bring-up **auto-restores** → the corpus is
     present again. (And, as the documented tradeoff, a destroy with *no* prior
     snapshot comes back empty — the accepted behaviour, not a failure.)
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
2. **Snapshot mechanism** (§7) — OpenSearch native snapshot API (register a
   filesystem repository) vs. a cold copy of a *stopped* `path.data`. Spike to pick;
   both land in `build/state/logsearch/` via Ansible.
3. **Node / guest-disk sizing** (§5, §7) — heap headroom (6 vs 8 GB) and the guest
   `path.data` disk size that bounds the live corpus before the flood watermark.
   Confirmed empirically during the cold-rebuild validation.
4. **Which nodes' logs fan out in v1** — the full fleet, or an initial subset
   matching the mqweb-tail gating already in `observability.yml`. Leaning: the same
   source set Loki already receives, for parity.
