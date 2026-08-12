# Work-edition Grafana dashboards — design spec

- **Epic:** `logical-minds-foundry/.github#169`
- **Design task:** `logical-minds-foundry/.github#170`
- **Phase 0 child (COMPLETE):** `logical-minds-foundry/.github#173` — observability-config
  verification & grant model; delivered by #943/#944 and live-verified on a full cold rebuild
  (§8). The exporter now publishes the full catalog the boards bind to.
- **Depends on (external):** `logical-minds-foundry/.github#79` (observability / collector
  extraction — Wave 2), the **LogSearch project** (logs + events → Elasticsearch — Wave 1 ES
  switch)
- **Related:** cockpit-dashboards-extraction-goal (#313), component-extraction-roadmap (#368),
  dashboards-lead-with-time-series (#489/#521), collectors-stdlib-only-non-MQI (#79); the
  Phase-0 fixes that surfaced during verification — grafana token (#935), exporter authz (#936)
- **Status:** architecture approved; prior-art + live verification done; **Phase 0
  (observability config, #173) COMPLETE — exporter verified live at 3172 series / 238 metric
  names / 49 queues (was ~118); Wave 1a panel co-development is the next action**
- **Date:** 2026-08-05 (updated 2026-08-07)

## 1. Problem & motivation

The author has Grafana access at work, where IBM MQ is already monitored with the
**`mq_prometheus` exporter into Prometheus** and where logs and events are standardized on
**Elasticsearch**. What is *not* available at work: a development environment, Claude Code, or
any real development tooling — and, by corporate policy, no ability to import code (no `pip`,
no `uv`, no unit-tested repositories). Pasting a single `.py` file and generating a `.pi` file
is possible; running Claude at work is possible.

Under those constraints, the highest-leverage way to add value is to ship **high-quality MQ
dashboards**. The plan is to develop them *here*, in the lab — where we have full tooling — and
carry the rendered JSON forward (email to self → import through the Grafana UI at work). A
Grafana dashboard is JSON configuration, not application code; whether the hand-carry is
acceptable under work policy is a governance question for the author's management, out of scope
here. This design assumes it is and is built to make the hand-off trivial regardless.

This is deliberately unusual work: little traditional software development, and a lot of
iterative, interactive dashboard tuning. We optimize for **getting each board ~80% right and
making the iteration loop cheap**, not for perfecting every panel up front.

## 2. Goals & non-goals

### Goals

- Three reusable, portable **work-edition** boards: a **QM view**, a **Queue/channel view**,
  and an **Infrastructure / HA-DR view (Native HA CRR only)**.
- Portability with near-zero manual editing at work: import, pick a datasource, pick a QM.
- A signal-in-the-noise design language shared across all three boards.
- A fast render → export → import → tweak iteration loop.
- **Prior-art citations shipped as first-class documentation** (see §3.1).

### Non-goals

- Disturbing the lab's existing object-driven boards. The work edition is a separate family.
- RDQM and Pacemaker/DRBD infrastructure views — work does not run those technologies. Backlog
  for a possible follow-on epic; explicitly out of scope here.
- Pulling the collector extraction (#79) or LogSearch *into* this epic — they are external
  dependencies (§7).
- Finalizing the panel/metric set in this document (§8).

## 3. Design principles

### 3.1 Prior art is a shipped, cited credibility artifact

Engineering-led, AI-assisted work is easily dismissed as "just AI." The antidote is visible
scaffolding: **every non-obvious metric or panel choice traces to checkable prior art** — IBM
MQ documentation, the `mq_prometheus` / `mq-metric-samples` authors' own reference dashboards,
and the IBM MQ user community (IMWUC, MQGem, MQ Technical Conference). Each board's companion
doc cites the reference, the specific claim it supports, and the link, **separating data (what
a source says) from judgment (our reasoning on top)**. Prior-art research is not a private,
throwaway input; it is a deliverable that demonstrates a scientific, serious engineering
approach grounded in the documented history of how MQ operators actually monitor MQ.

### 3.2 Portability by *contract*, not by copy

Every data source that is not already identical at work gets a **written contract** that is
simultaneously (a) the deliverable of the upstream project producing the data and (b) the thing
that makes a board portable. Board queries bind to the contract; the contract is what is
carried to work. Nothing hardcodes a datasource UID, a QM name, or an undocumented field.

- **Prometheus schema note** (`ibmmq_*`) — *not* contract-free. The whole reusability
  mechanism hinges on one query (`label_values(ibmmq_qmgr_status, qmgr)`), and the exact metric
  names and label keys emitted by `mq_prometheus` depend on the **mq-metric-samples version**
  and its **config** (e.g. `monitoredQueues` gates whether some series exist at all). So the
  Prometheus path gets a short **schema note**: the exact metric names + label keys the boards
  bind to, plus the exporter version and config they assume. This turns "assumed identical" into
  a one-time verified check at work (import → confirm `$qmgr` populates → done) and gives a
  concrete list to sanity-check against work's exporter before the JSON is even emailed. Panels
  reference a `$datasource` variable; the per-site difference is which Prometheus the user picks.
  **Phase 0 (#173) produced this note's backing data:** the verified live catalog — 238
  `ibmmq_*` metric names spanning the qmgr / queue / channel / `nha` classes, under the fire-hose
  config (EXTENDED queue class, `useObjectStatus`, wildcard `monitoredQueues`, the MQMon
  namespace grant) — captured in `phase0-observability-config.md`. The schema note is now a
  concrete, checkable list rather than a promise.
- **Metric contract** (Infra board) — exact metric names, labels, source CLI commands
  (`dspmq -o nativeha` and the CRR status commands), parsing rules, and output format for the
  `cluster_nha_*` / CRR link·lag·role signals. Produced by the collector extraction (#79) and
  handed to Claude-at-work as prose to regenerate a faithful collector `.py`. **The contract
  ships with golden sample output** — a literal captured block of the collector's `/metrics`
  (representative lines per series, with labels and a sample value, taken from the lab) — so the
  regenerated work collector can be **diffed against the golden block**: match = faithful,
  mismatch = caught *before* the infra board is trusted. This is the cheapest fidelity gate that
  survives the no-code-import constraint and honours the "never ship a silently-empty panel"
  stance (§6). The board's PromQL binds to these names. **Naming is the central portability risk
  for this board:** there is no standard for Native HA / CRR collector metric names — each
  implementation picks its own — so the board binds to the **lab collector's names** and
  **translates in-board** for the first iterations, converging later via either a namespace
  agreed with the work operators or this documented contract. The contract's job is to make that
  reconciliation a diff, not a guess.
- **ES doc/field contract** (logs + events) — index/data-stream name and the field names the
  panels query (`@timestamp`, `severity`, `qmgr`, `object`, `reason_code`, `message`). Produced
  by the LogSearch project. The board's Lucene queries bind to it.

Concrete deliverables: **three boards + three contract documents** (Prometheus schema note,
Native HA CRR metric contract with golden sample output, ES doc/field contract) **+ per-board
prior-art citations.**

### 3.3 Iteration as a first-class requirement

The boards will be tuned aggressively and interactively once live; v1 is a strong starting
point, not a frozen spec. The design prioritizes a cheap loop: edit generator → render portable
JSON → email → import → tweak live → fold the good tweaks back into the generator.

## 4. Datasource strategy

Two primary datasources at work, both selected through template variables:

- **Prometheus** (`$datasource`, type `prometheus`) — metrics, `ibmmq_*` and `cluster_nha_*`.
- **Elasticsearch** (`$logs`, type `elasticsearch`) — logs and MQ events.

The lab's Alloy/Loki stack stays in place — good infrastructure, worth keeping — but is
**lab-only**, serving the lab's own object-driven boards. The **work-edition** log/event panels
are built **once, against Elasticsearch** (Wave 1b), with **no Loki placeholder**: a Loki panel
(LogQL, Loki datasource type) and an ES panel (Lucene, ES datasource type, different field
mapping) share almost nothing, so a Loki-first step would mean building the log/event panels
twice *and* shipping a work board whose log section cannot function (work has no Loki). LogSearch
is being built **in parallel now** (~1–2 days out), so Wave 1b lands close behind 1a. The author
expects the Elasticsearch form to translate to work more easily than the Alloy/Loki form.

## 5. Reusability — variable-driven, one board per view

Each board is a single reusable artifact driven by dropdowns, not a per-object board:

- `$datasource` — pick work's Prometheus on import.
- `$logs` — pick work's Elasticsearch (work-edition log/event panels are ES-only, Wave 1b).
- `$qmgr` — `label_values(ibmmq_qmgr_status, qmgr)`; one board serves every QM.
- `$queue`, `$channel` — multi-select `label_values(...)` scoped to `$qmgr` (Queue/channel
  board).
- `$level` — log/event severity toggle (reuses the lab's existing pattern), defaults to error.

This answers both portability (no names to search-and-replace) and "concise but advanced." It
is a deliberate departure from the lab's object-driven convention (#178), which remains correct
for the lab's fixed topology; the work edition serves arbitrary QMs by dropdown.

## 6. Shared signal-in-the-noise model

Applied identically on all three boards so they read as one system:

1. **Lead with a compact status band** — a few pills, calm/green when normal, red/amber on
   breach. A glance answers "fine?" vs "look here."
2. **Then trend graphs, not stat tiles** — time-series lead, so the *derivative* and the *when*
   are visible (put-vs-get = the depth derivative; CRR lag over time = RPO risk). Per the lab's
   dashboards-lead-with-time-series convention.
3. **Thresholds encode "error" in the panel** — depth% red above threshold, channel status red
   when stopped, DLQ depth red when > 0, quorum red when lost.
4. **The Attention + Inventory pair** — the canonical realization of signal-in-the-noise, used
   on every board that lists objects (queues, channels, replicas):
   - a **hero "Attention" table** whose *query itself* filters to breaching objects (e.g. PromQL
     `... > threshold`), so healthy objects don't appear at all — **empty = healthy**, the
     literal signal in the noise; and
   - a **collapsible "Inventory" table** below it — *all* objects, threshold-colored, sorted
     worst-first — the full census kept one click away for investigation.

   Grafana tables don't filter rows by threshold natively, so "empty = healthy" is achieved by
   pushing the filter into the query, not by row-coloring. The two panels are complementary, not
   alternatives: Attention answers "is anything wrong?", Inventory answers "show me everything."
5. **The event/log feed defaults to error-severity** (`$level`) — signal, not a firehose.

## 7. The three boards (purpose + structure; panel detail deferred)

Panel-level detail — exact metrics, channel-type scenarios, thresholds — is **deliberately
deferred** to the research → co-development phase (§10). What follows is the approved *structure*.

### 7.1 QM view — Wave 1a metrics (pure `ibmmq_*`) + Wave 1b ES feed

"Is this queue manager healthy, and is it trending toward trouble?" QM-scoped by design; all
per-queue/per-channel breach detail lives on the Queue/channel board, one drill-link away.

- **① Status band** *(1a)***:** QM status · Uptime · Services (initiator ∧ command server ∧
  listeners) · Connections.
- **② Trend band** *(1a)***:** message rate · recovery-log % · connections over time.
- **③ Attention** *(1a)***:** services detail (which piece is down).
- **④ Event / error feed** *(1b, ES)*: QM event feed + error-log feed, severity-filtered.
- **Drill-down seam:** a data link carries `$qmgr` → the Queue/channel board.

### 7.2 Queue/channel view — Wave 1a metrics (pure `ibmmq_*`) + Wave 1b ES feed

"Which queues and channels on `$qmgr` are in trouble, and is anything backing up?" The
signal-in-the-noise showcase.

- **① Status band** *(1a)***:** queues in trouble (count) · channels not running (count) · DLQ
  depth (red > 0) · oldest message age.
- **② Attention + Inventory** *(1a)* — per §6.4, for both queues and channels: a query-filtered
  **Attention** table (only breaching, **empty = healthy**) as the hero, plus a collapsible
  **Inventory** table (all objects, threshold-colored, worst-first). Channel Attention floats
  Retrying/Stopped/in-doubt.
- **③ Trend band** *(1a)***:** depth over time · put-vs-get on one graph (the leading indicator)
  · channel throughput · backout rate.
- **④ Event/log feed** *(1b, ES)*: channel and performance events, severity-filtered.

**Verification — done (2026-08-06, live lab; see research brief §Live-exporter verification):**
depth% inputs, oldest-message-age and uncommitted are present **gauges**; put/get and the
qmgr-interval series are all **counters** (→ `rate()` everywhere); channel status/squash are
gauges (raw + value-map). **Two signals are NOT stock series** — per-queue **backout** and
**in-doubt** — so both come from the **ES event feed (Wave 1b)**, not Prometheus. The live run
also showed the exporter is **under-collecting** (see §8 Phase 0).

### 7.3 Infrastructure / HA-DR view — Wave 2 (Native HA CRR only)

The reliability substrate *below* the QM: quorum, in-sync replicas, and cross-region
replication (CRR) link, lag, and role. Keyed by QM name but deliberately not about the QM's
messaging. Bindings are `cluster_nha_*` / CRR custom-collector metrics — **not** stock
`ibmmq_*` — so this board is gated on the collector extraction (#79). Scope is **Native HA CRR
only**; RDQM and Pacemaker/DRBD are out of scope (§2). A coworker has already extracted some
Native HA CRR data at work, which is a starting point for the collector's prose contract.
Replica/component state uses the same **Attention + Inventory** pair (§6.4). Fidelity of the
work-regenerated collector is verified against the **golden sample output** shipped with the
metric contract (§3.2) — no code import, just a diff. Board work can begin **now** against the
embedded lab collector's live metrics; the exact metric names (non-standard across
implementations) are translated in-board initially and converge via the §3.2 naming contract.
Extraction (#79) is the *completion* gate, not a prerequisite to start.

## 8. Wave phasing & dependency graph

| Wave | Board(s) | Metrics | Logs/events | Prerequisite |
|------|----------|---------|-------------|--------------|
| **0 — DONE** | *(no board)* observability-config verification & grant model (**#173**, delivered by #943/#944) | — | — | complete — the verified foundation the boards stand on |
| **1a — after 0** | QM view · Queue/channel view (metric panels) | `ibmmq_*` | — | Phase 0 complete + Prometheus schema note (§3.2) verified against work's exporter |
| **1b — days** | QM · Queue/channel **ES event/log feed** | — | ES (built once, no Loki) | LogSearch (in parallel now, ~1–2 days) + ES doc/field contract |
| **2 — startable now** | Infra / HA-DR (Native HA CRR) | embedded-collector CRR metrics (live in the lab now) | ES event feed | **startable against the embedded collector's live metrics**; extraction (#79) + the metric-name contract gate *completion*, not start |
| **backlog** | RDQM / Pacemaker infra | — | — | work does not run these — future follow-on epic |

**Phase 0 — observability-config verification & grant model (#173, COMPLETE).** The initial
verification (2026-08-06) proved the exporter was *under-collecting* (~118 series; `useStatus`
off; limited `monitoredQueues`) and surfaced two authz bugs (#935, #936). Phase 0 then reviewed
the `mq_prometheus` configuration **top-down** and delivered, via **#943/#944**: the QM
fire-hose (`MON*(HIGH)`/`STATCHL(HIGH)`), the MQMon least-privilege grant model as MQSC
`SET AUTHREC` (documenting the previously-undocumented permission set), and the exporter YAML
migration enabling the EXTENDED queue class. **Live-verified on a full cold rebuild (2026-08-07):
452 → 3172 series, 4 → 49 queues (45 `SYSTEM.*`), EXTENDED class publishing.** In hindsight this
was effectively its own epic that got folded in here — captured as a lesson in the #171
retrospective — but either way it is now the verified foundation Wave 1a builds on.

**External dependencies (not children of this epic):**

- **LogSearch project** (**imminent** — code done + deployed; in cold-boot/sanity wrap-up) —
  ships lab logs + MQ events to Elasticsearch and defines the ES doc/field contract. Unblocks
  **Wave 1b**, which is high-priority: work standardizes on Elasticsearch, so the ES form is what
  makes the prototypes work-relevant (the reason work adoption was held until it lands).
- **Collector extraction** (#79, `mq-resiliency-observability`) — the HA/DR collectors already
  run **embedded in the lab and publish their metrics live**, so Wave-2 board work can *start*
  against them now. Extraction into a standalone, exportable form + the documented metric
  contract is the **completion** gate (portability + the naming reconciliation, §3.2), not a
  start gate.
- **Corporate-policy hand-off** — code cannot be imported at work, so the collector's real
  deliverable is a **detailed prose spec in Markdown** precise enough that Claude-at-work
  regenerates a faithful collector `.py` from it alone. The CLI collectors are simple (they
  shell out to `dspmq` / CRR status commands), so this is not expected to be a major blocker.
  Acknowledged as pragmatic, not ideal engineering.

## 9. Production model

**Purpose-built work-edition generators** (e.g. `workqmboard.py` / `workflowboard.py`), reusing
the lab's tested panel primitives (`_stat`, `_timeseries`, `_ds`, `_logs_panel`) but free to
diverge in structure toward the reusable, dropdown-driven layout. This keeps the lab's
object-driven boards untouched while the work edition is redesigned freely. Once the shapes
settle, folding the portable render mode back into the main generators is a clean follow-up
(converging toward the extraction roadmap, #368). Hand-authored JSON is avoided except as a
throwaway spike to feel out a layout. Rendered output carries no hardcoded UIDs or names; a
render target (e.g. `mqlab render --portable`) emits the work editions ready to email and
import.

## 10. Research

**Part A — prior-art (DONE).** A cited prior-art pass produced the research brief
(`research/prior-art.md`, sources S1–S25, DATA vs. judgment) and was augmented with the
2026-08-06 live-exporter verification (§7.2). It covered:

- The `mq_prometheus` / `mq-metric-samples` project's own **reference Grafana dashboards** — the
  most directly relevant prior art (identical exporter).
- **IBM MQ monitoring documentation** — statistics/accounting, channel status attributes, queue
  monitoring, the event-message catalog.
- **IBM MQ user-community best practice** (IMWUC, MQGem, MQ Technical Conference) — what to
  monitor and the channel-type scenarios (SVRCONN vs SENDER/RECEIVER vs CLUSSDR health).
- **Common alerting rules** — DLQ > 0, queue-depth-high events, channel retry/stopped,
  in-doubt, oldest-message-age.

**Part B — observability-config verification (Phase 0, COMPLETE).** Prior-art told us *what* to
monitor; Part B verified the lab actually *publishes and secures* it. It reviewed the
`mq_prometheus` configuration top-down for maximum coverage and produced the complete,
documented grant model, delivered via **#943/#944** and live-verified on a cold rebuild (§8).
The data set is confirmed complete and correctly secured, so Wave 1a panel co-development can
begin.

## 11. Open questions (deferred to co-development, after Phase 0)

The exact metric set per board; channel-type scenarios and their signals; threshold values;
which signals earn a status-band pill vs a trend vs an attention-table row; and the final ES
doc/field and metric contracts. These are derived from the §10 research **and the Phase 0
config verification (#173)**, then co-developed interactively — not decided here.
