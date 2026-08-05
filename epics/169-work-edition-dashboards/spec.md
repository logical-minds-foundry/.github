# Work-edition Grafana dashboards — design spec

- **Epic:** `logical-minds-foundry/.github#169`
- **Design task:** `logical-minds-foundry/.github#170`
- **Depends on (external):** `logical-minds-foundry/.github#79` (observability / collector
  extraction — Wave 2), the **LogSearch project** (logs + events → Elasticsearch — Wave 1 ES
  switch)
- **Related:** cockpit-dashboards-extraction-goal (#313), component-extraction-roadmap (#368),
  dashboards-lead-with-time-series (#489/#521), collectors-stdlib-only-non-MQI (#79)
- **Status:** design (brainstorm output); architecture approved, panel-level detail deferred to
  a research → co-development phase
- **Date:** 2026-08-05

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

**Goals**

- Three reusable, portable **work-edition** boards: a **QM view**, a **Queue/channel view**,
  and an **Infrastructure / HA-DR view (Native HA CRR only)**.
- Portability with near-zero manual editing at work: import, pick a datasource, pick a QM.
- A signal-in-the-noise design language shared across all three boards.
- A fast render → export → import → tweak iteration loop.
- **Prior-art citations shipped as first-class documentation** (see §3.1).

**Non-goals**

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

- **Prometheus / `ibmmq_*`** — no contract needed. Identical exporter and schema at work.
  Panels reference a `$datasource` variable; the only per-site difference is which Prometheus
  the user picks on import.
- **Metric contract** (Infra board) — exact metric names, labels, source CLI commands
  (`dspmq -o nativeha` and the CRR status commands), parsing rules, and output format for the
  `cluster_nha_*` / CRR link·lag·role signals. Produced by the collector extraction (#79) and
  handed to Claude-at-work as prose to regenerate a faithful collector `.py`. The board's
  PromQL binds to these names.
- **ES doc/field contract** (logs + events) — index/data-stream name and the field names the
  panels query (`@timestamp`, `severity`, `qmgr`, `object`, `reason_code`, `message`). Produced
  by the LogSearch project. The board's Lucene queries bind to it.

Concrete deliverables: **three boards + two short contract documents + per-board prior-art
citations.**

### 3.3 Iteration as a first-class requirement

The boards will be tuned aggressively and interactively once live; v1 is a strong starting
point, not a frozen spec. The design prioritizes a cheap loop: edit generator → render portable
JSON → email → import → tweak live → fold the good tweaks back into the generator.

## 4. Datasource strategy

Two primary datasources at work, both selected through template variables:

- **Prometheus** (`$datasource`, type `prometheus`) — metrics, `ibmmq_*` and `cluster_nha_*`.
- **Elasticsearch** (`$logs`, type `elasticsearch`) — logs and MQ events.

The lab's Alloy/Loki stack stays in place — good infrastructure, worth keeping — but is
**lab-only**. Log/event panels are developed first against Loki as a *placeholder*, then
switched to Elasticsearch once LogSearch lands. The author expects the Elasticsearch form to
translate to work more easily than the Alloy/Loki form.

## 5. Reusability — variable-driven, one board per view

Each board is a single reusable artifact driven by dropdowns, not a per-object board:

- `$datasource` — pick work's Prometheus on import.
- `$logs` — pick work's Elasticsearch (Loki in the lab until LogSearch lands).
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
4. **An "attention" table shows only breaching objects, worst-first** — the literal signal in
   the noise. Empty-of-red = healthy.
5. **The event/log feed defaults to error-severity** (`$level`) — signal, not a firehose.

## 7. The three boards (purpose + structure; panel detail deferred)

Panel-level detail — exact metrics, channel-type scenarios, thresholds — is **deliberately
deferred** to the research → co-development phase (§8). What follows is the approved *structure*.

### 7.1 QM view — Wave 1 (buildable now, pure `ibmmq_*`)

"Is this queue manager healthy, and is it trending toward trouble?" QM-scoped by design; all
per-queue/per-channel breach detail lives on the Queue/channel board, one drill-link away.

- **① Status band:** QM status · Uptime · Services (initiator ∧ command server ∧ listeners) ·
  Connections.
- **② Trend band:** message rate · recovery-log % · connections over time.
- **③ Attention:** services detail (which piece is down) · QM event feed · error-log feed.
- **Drill-down seam:** a data link carries `$qmgr` → the Queue/channel board.

### 7.2 Queue/channel view — Wave 1 (buildable now, pure `ibmmq_*`)

"Which queues and channels on `$qmgr` are in trouble, and is anything backing up?" The
signal-in-the-noise showcase.

- **① Status band:** queues in trouble (count) · channels not running (count) · DLQ depth
  (red > 0) · oldest message age.
- **② Attention tables:** queues sorted by depth% desc, threshold-colored; channels by status
  (Retrying/Stopped/in-doubt float up).
- **③ Trend band:** depth over time · put-vs-get on one graph (the leading indicator) · channel
  throughput · backout rate.
- **④ Event/log feed:** channel and performance events, severity-filtered.

**Verification caveat:** several bindings — `maxdepth`/depth%, oldest-message-age, per-queue
backout — must be verified against a live exporter's `/metrics` before commit. The
`mq_prometheus` exporter does not export everything, and some signals may require exporter
`monitoredQueues` config or a computed expression. Any signal not actually available will be
flagged, never shipped as a silently-empty panel.

### 7.3 Infrastructure / HA-DR view — Wave 2 (Native HA CRR only)

The reliability substrate *below* the QM: quorum, in-sync replicas, and cross-region
replication (CRR) link, lag, and role. Keyed by QM name but deliberately not about the QM's
messaging. Bindings are `cluster_nha_*` / CRR custom-collector metrics — **not** stock
`ibmmq_*` — so this board is gated on the collector extraction (#79). Scope is **Native HA CRR
only**; RDQM and Pacemaker/DRBD are out of scope (§2). A coworker has already extracted some
Native HA CRR data at work, which is a starting point for the collector's prose contract.

## 8. Wave phasing & dependency graph

| Wave | Board(s) | Metrics | Logs/events | Prerequisite |
|------|----------|---------|-------------|--------------|
| **1 — now** | QM view · Queue/channel view | `ibmmq_*` (ready) | Loki placeholder → ES | LogSearch (~1–2 days) for the ES switch |
| **2 — next** | Infra / HA-DR (Native HA CRR) | `cluster_nha_*` / CRR collectors | ES event feed | collector extraction (#79) + prose-spec regen at work |
| **backlog** | RDQM / Pacemaker infra | — | — | work does not run these — future follow-on epic |

**External dependencies (not children of this epic):**

- **LogSearch project** (just kicked off, ~1–2 days) — ships lab logs + MQ events to
  Elasticsearch and defines the ES doc/field contract. Unblocks the Wave-1 ES switch.
- **Collector extraction** (#79, `mq-resiliency-observability`) — finish extracting the HA/DR
  collectors into a standalone, exportable form and produce the metric contract. The Wave-2
  infra-board tasks are `Blocked-by` this landing.
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

## 10. Prior-art research — the immediate next action

Before panel-level co-development, a **prior-art research pass** produces a cited research brief
(data vs. judgment, checkable links) that seeds panel selection and is **committed into the epic
docs** (per §3.1). Scope:

- The `mq_prometheus` / `mq-metric-samples` project's own **reference Grafana dashboards** — the
  most directly relevant prior art (identical exporter).
- **IBM MQ monitoring documentation** — statistics/accounting, channel status attributes, queue
  monitoring, the event-message catalog.
- **IBM MQ user-community best practice** (IMWUC, MQGem, MQ Technical Conference) — what to
  monitor and the channel-type scenarios (SVRCONN vs SENDER/RECEIVER vs CLUSSDR health).
- **Common alerting rules** — DLQ > 0, queue-depth-high events, channel retry/stopped,
  in-doubt, oldest-message-age.

## 11. Open questions (deferred to co-development)

The exact metric set per board; channel-type scenarios and their signals; threshold values;
which signals earn a status-band pill vs a trend vs an attention-table row; and the final ES
doc/field and metric contracts. These are derived from the §10 research and co-developed
interactively, not decided here.
