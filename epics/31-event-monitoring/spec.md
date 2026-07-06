# MQ instrumentation event monitoring → JSON → Loki — design spec

- **Epic:** `logical-minds-foundry/.github#31`
- **Design task:** `logical-minds-foundry/.github#32`
- **Promoted from:** `mq-resiliency-lab-for-linux#365`
- **Sibling epic:** `logical-minds-foundry/.github#8` (Lab observability stack)
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-07-06

## 1. Problem & motivation

The lab has **no instrumentation-event logging today**. IBM MQ continuously
emits rich instrumentation events — authority failures, queue full/high, channel
start/stop/error, config and command activity, logger and QM start/stop — as
**binary PCF messages** on the `SYSTEM.ADMIN.*.EVENT` queues. Historically, to
read them you had to *write the program that parsed PCF* — the "I had to be the
app that parsed those messages" problem.

IBM's supplied `amqsevt` sample removes that entirely: it reads the event queues
and formats each message, including a structured **JSON** mode. So a complete
event-logging capability across the whole architecture becomes **config, not
code** — a thin collector plus pipeline reuse, with zero bespoke PCF parsing.
That is a clean demonstration of the lab's glass-box, "simple supportable
tooling" thesis, and it fills a real gap: during an HA/DR drill or a live
triage, the operator currently cannot *see* MQ's own account of what the queue
manager is doing.

This work was filed as task `#365` under the observability epic (`#8`) and is
now promoted to its own epic because it has several genuinely distinct layers —
enabling + collecting events, publishing them as JSON on the pipeline, and
consuming them on dashboards — each its own PR.

## 2. Doctrine & principles

- **Config, not code.** The headline is that event logging needs *no*
  PCF-parsing application. The collector is IBM's own sample wrapped as a
  service; everything else is declarative MQSC and pipeline config. Any drift
  toward writing a bespoke event processor is a failure of the thesis.
- **Production-proposable.** This is pitched as an actual production pattern, not
  a lab toy. The binary must be non-special (already on the box), the collector
  must survive failover the way a real deployment would, and the consumption
  model must be correct under load — no footguns dressed up as safety.
- **Ride the existing rails.** journald → Alloy → Loki → Grafana is already live
  for logs (`#282`, `#194`). Events join that pipeline with minimal new wiring;
  they are *separated from* logs by one low-cardinality label, not by a parallel
  stack.
- **No silent failures.** Events are drained and processed exactly once. Nothing
  accumulates until it is silently dropped. (This rules out browse-mode — see
  §5.) This principle also reaches into journald: the event stream **depends on
  the `mq-diag-logging` rate-limit drop-in** (`10-mq.conf`,
  `RateLimitIntervalSec=0`), which already disables journald rate limiting on MQ
  nodes so bursty MQ output is never silently rate-dropped. With all event
  classes enabled, `mq-events` can burst too — so the collector's nodes **must**
  carry that drop-in.
- **Travels with the queue manager.** Under HA/DR the QM floats; the event
  collector must be wherever the QM is active, without VIP juggling or
  live-instance coordination. The MQ `SERVICE` object gives this for free.
- **Complementary, not a replacement.** This is a *second* MQ→journald stream
  alongside the diagnostic-log stream (`mq-diag-logging`, `#308`/`#282`), not a
  replacement for it.

## 3. Architecture overview

```
                 (per queue manager)
  ┌──────────────────────────────────────────────┐
  │  Queue manager (bindings)                      │
  │                                                │
  │  ALTER QMGR  ...EV(ENABLED)   ── emits ──▶ SYSTEM.ADMIN.*.EVENT (PCF)
  │                                                 │
  │  DEFINE SERVICE(MQ.EVENT.MONITOR)               │
  │    CONTROL(QMGR)  SERVTYPE(SERVER)              ▼
  │    amqsevt -o json  ─ destructive drain ─▶ (reads & removes events)
  │              │                                 │
  │              └── stdout | logger -t mq-events ─┘
  └────────────────────────┼───────────────────────┘
                           ▼
                   journald (tag: mq-events)      ← beside MQ core logs (ibm-mq)
                           ▼
                   Alloy  (relabel tag → unit="mq-events")
                           ▼
                   Loki   (stream {unit="mq-events"}, JSON body parsed query-side)
                           ▼
                   Grafana:  • dedicated events feed  • per-object event filter
```

Four layers, described below. Layers 0–2 are the "get events into the pipeline"
half; layer 3 is consumption.

## 4. Layer 0 — the binary (non-special, already present)

`amqsevt` is an **IBM-supplied sample, shipped prebuilt**:

- On a **server install** it is the single binary `/opt/mqm/samp/bin/amqsevt`,
  with source `amqsevta.c` alongside — usable as-is, no compilation.
- It is also included in the **redistributable MQ client**, so the same collector
  is deployable on a host without a full MQ install.

Our QM hosts already run **full MQ**, so **the binary is already on the box** the
moment MQ is installed — nothing to fetch, build, package, or special-case. This
is the direct answer to "how do you get it?": you already have it.

> **Build-time verification.** Confirm the exact path and the default
> event-queue set against **IBM Docs 9.4** via the repo's docs-cache tool
> (`tools/ibm_doc_cache.py`), per the IBM-docs fetch approach. Sources:
> IBM Docs 9.4 *"Sample program to monitor instrumentation events (amqsevt)"*;
> Mark Taylor, *"Formatting MQ Events as JSON."*

## 5. Layer 1 — enable + collect (declarative MQSC, per QM)

This entire layer is MQSC, living where the exporter's `MONQ/STATMQI` gate
already lives (the `mq-qmgr` role and the per-arm site playbooks), so it is
**cold-boot reproducible** and survives a rebuild.

### 5.1 Event gate — enable everything to start

`ALTER QMGR` enables **all** instrumentation-event classes. Starting broad is
deliberate: show the full range first, then pare back once real volume is known
(volume is expected to be modest — the lab does not run the workload that would
generate high event rates).

| QMGR attribute | Class | Notes |
|---|---|---|
| `AUTHOREV(ENABLED)`  | Authority | Authorization failures — clean security demo |
| `CHADEV(ENABLED)`    | Channel auto-def | Auto-definition of receiver/server-connection channels |
| `CHLEV(ENABLED)`     | Channel | Start/stop/error — highest value during failover/reconnect drills |
| `STRSTPEV(ENABLED)`  | Start/stop | QM start/stop — narrates HA/DR cutovers |
| `LOGGEREV(ENABLED)`  | Logger | Recovery-log events — ties to the storage/replication story |
| `PERFMEV(ENABLED)`   | Performance | Queue depth high/full, service interval — **also needs per-queue thresholds, see below** |
| `CONFIGEV(ENABLED)`  | Config | Object create/alter/delete audit — chatty, kept for the demo |
| `CMDEV(NODISPLAY)`   | Command | **`NODISPLAY`, not `ENABLED`** — see §5.2 |
| `INHIBTEV(ENABLED)`  | Inhibit | Get/put inhibited |
| `LOCALEV(ENABLED)`   | Local | e.g. unknown object / alias-base-queue errors |
| `REMOTEEV(ENABLED)`  | Remote | Remote-queue resolution errors |
| `SSLEV(ENABLED)`     | TLS | Certificate / TLS-handshake events |

> **Platform note (confirmed against IBM Docs 9.4, ALTER QMGR reference).** The set
> above is the complete Multiplatforms event-class list. `BRIDGEEV` (IMS-bridge
> events) is **z/OS-only** (footnote 2, "Valid only on z/OS") and is deliberately
> excluded — issuing it on a Linux QM would make `runmqsc` reject the whole `ALTER`.
> There is **no `COMMEV`** attribute; earlier drafts listed one in error.

**Performance events** additionally require **per-queue** thresholds to fire:
set `QDPMAXEV(ENABLED)` / `QDPHIEV(ENABLED)` (and, where wanted, `QDPLOEV`,
`QSVCIEV`/`QSVCINT`) on the app queues. Enabling `PERFMEV` at the QMGR alone
produces nothing without these — the gate must set both.

### 5.2 The `CMDEV(NODISPLAY)` decision

`CMDEV` accepts `ENABLED` or `NODISPLAY`. Watching **admin/mutating commands**
(`ALTER`, `DEFINE`, `DELETE`, `STOP CHANNEL`, …) scroll past a dashboard during a
live triage is a genuinely useful observability moment — an operator can *see*
what is being changed as it happens. But plain `ENABLED` also emits an event for
every `DISPLAY`/PCF-inquire, and **the MQ exporter polls the QM with those
constantly**. Under `ENABLED` the human-run commands would be buried under the
exporter's poll flood. `CMDEV(NODISPLAY)` keeps exactly the signal (mutating
commands) and drops the poll noise. This can be revisited when tuning.

### 5.3 The collector — an MQ `SERVICE` object

The collector is defined **in MQSC as a queue-manager service**, not run as an
external daemon or client. The service points at a **small role-managed wrapper
script**, not an inline shell pipeline — cramming `sh -c '… | logger'` with
nested quotes into an MQSC `STARTARG` is a known quoting minefield in this repo
(`mq-pcmk-qmgr` already resorts to `'"'"'`-style escaping to get quotes through
`runmqsc`). A wrapper script sidesteps that, is independently testable, and is
the natural home for the buffering fix (§5.3 build-time note). It is a *launcher*,
not a PCF parser, so the "config, not code" thesis holds.

```
DEFINE SERVICE(MQ.EVENT.MONITOR) REPLACE +
  CONTROL(QMGR)  SERVTYPE(SERVER) +
  STARTCMD('/opt/mq-event-monitor/run.sh') +
  STARTARG('<QM>') +
  STOPCMD('/bin/kill')  STOPARG('<server-pid-token>') +
  DESCR('Drain SYSTEM.ADMIN.*.EVENT to JSON on journald')
```

where the role-templated `run.sh` is essentially:

```sh
#!/bin/sh
exec /opt/mqm/samp/bin/amqsevt -m "$1" -o json | logger -t mq-events
```

Key properties:

- **`CONTROL(QMGR)`** — the service starts when the QM starts and stops when the
  QM ends. Under HA/DR, when the QM activates on a node, the QM starts the
  collector *there*. **The collector travels with the queue manager** — no VIP,
  no client channel, no CCDT, no "which instance is live" logic. This is the
  central win of the SERVICE approach over a client collector.
- **Bindings mode, local** — `amqsevt` connects to its own QM directly (`-m
  <QM>`, no `-c`). It starts after the QM is up, so the event queues exist.
- **Destructive drain** (default MQI get, **no `-b`**) — the collector *is* the
  event consumer and processes each event exactly once. Events are removed as
  read, so the event queues never climb to `MAXDEPTH` and MQ never silently
  discards. This is the real and only mechanism for reading these events. (A
  future multi-consumer need is met by routing events to a **topic** — `amqsevt
  -t` — not by browsing.)
- **`STOPCMD`/`STOPARG`** — on QM shutdown, MQ terminates the service via its
  server-PID token so the collector (and its `logger` pipe) exits cleanly.

> **Build-time verifications.** (1) `amqsevt`/`logger` output buffering: confirm
> events reach journald promptly under a pipe (line-buffered); if stdio
> block-buffers, wrap the `amqsevt` invocation in `run.sh` with `stdbuf -oL`.
> (2) Confirm exact MQSC `SERVICE` `STOPARG` token for the server PID against
> IBM Docs 9.4. (3) Confirm the service's `amqsevt` picks up the standard default
> event-queue set (or list queues explicitly with `-q`). (4) **Verify actual
> failover behavior per arm.** A Native HA takeover promotes a replica — it is
> *not* a `strmqm` — so `STRSTPEV` may not fire the way it does on a cold
> pcmk/RDQM start, and events in-flight across the takeover may duplicate or gap
> as the newly-active instance's service starts and drains the replicated queue.
> Do not assume uniform event continuity across the arms; observe each.

### 5.4 Output to journald via `logger`

The service pipes `amqsevt` JSON through `logger -t mq-events`, so events land in
**journald tagged `mq-events`** — right next to the MQ core diagnostic logs,
which arrive tagged `ibm-mq` (via MQ's native Syslog service, `mq-diag-logging`).
One sink, two identifiers. This matches the intended "events in the same place as
the syslog" mental model and gives clean symmetry: `logs = ibm-mq`,
`events = mq-events`.

## 6. Layer 2 — pipeline (ride existing rails)

journald → Alloy → Loki is already wired and running on the QM-bearing nodes
(Alloy already ships the journal and already tails mqweb's `messages.log`).

- **Alloy relabel.** Alloy's journal relabel already maps the `ibm-mq` syslog
  identifier → `unit="ibm-mq"`. Add the analogous mapping for the `mq-events`
  identifier → **`unit="mq-events"`**. This is the same one-rule pattern already
  in `config.alloy.j2`; no new source or exporter.
- **The `unit` label is the whole separation mechanism.** Because events carry a
  distinct low-cardinality `unit`, they are a **separate Loki stream** from the
  logs by construction — the operator's "show events separately from logs"
  requirement falls out for free, with no schema work.
- **Query-side JSON.** The event body (`eventSource`, `eventType`, `eventReason`,
  `eventCreation`, `eventData.*`) is parsed in LogQL (`| json`), so every event
  field — including the object name — is filterable without indexing cost.

## 7. Layer 3 — consume (dashboards)

Two deliverables, both pure Grafana/Loki (no new plumbing):

1. **Dedicated events feed.** A Grafana panel (or a small board) scoped to
   `{unit="mq-events"}`, filterable by `eventType` and source QM. It is distinct
   from the log panels by virtue of the label — the operator sees an event stream,
   not log lines.
2. **Per-object event filtering.** `amqsevt` JSON carries the affected object's
   name in `eventData` (e.g. queue name, channel name). On a queue or channel
   dashboard, an events panel filters
   `{unit="mq-events"} | json | queueName="$queue"` (or `channelName="$channel"`),
   driven by the board's existing object template variable. This is the
   "show me events for *this* object" feature — a LogQL filter, nothing more.

**Separation cuts both ways — the log panels must exclude events.** The `unit`
label separates the two streams in Loki, but only if the *log* panels are written
not to sweep in `mq-events`. Commit `#440` *wildcarded the logs `unit` selector*
on the messaging board, so a broad wildcard (`unit=~".+"`, `ibm-.*`, …) would
**bleed events into the log panels** — defeating the whole "events separate from
logs" requirement. So the dashboards task must, on the *log* side, either
negative-match (`unit!="mq-events"`) or positively scope to the log units, while
the events feed positive-matches `{unit="mq-events"}`. Clean separation is a
property of *both* panels, not just the new one.

> **Naming alignment.** Panels/queries must not hardcode a QM name; the QM is a
> template variable from a single source, consistent with the cockpit-dashboard
> de-hardcoding direction (`#313`/`#351`).

## 8. Component boundaries & isolation

- **MQSC gate** — owns *enabling* events and defining the service. Declarative,
  idempotent, in the qmgr role/create path. Verifiable by a cold rebuild.
- **Collector (`SERVICE`)** — owns *draining* events to JSON on journald.
  Interface in: the event queues. Interface out: journald tag `mq-events`. No
  knowledge of Loki or Grafana. Testable by asserting JSON on journald.
- **Pipeline (Alloy relabel)** — owns *labeling* the stream. Interface in:
  journald tag. Interface out: Loki `unit="mq-events"`. Testable by a LogQL
  query returning parsed fields.
- **Dashboards** — own *presentation*. Interface in: the Loki stream. No
  knowledge of how events are produced. Testable by eye against a live feed.

Each layer can be understood, changed, and verified without reading the others'
internals.

## 9. Implementation tasks (preview; finalized in the plan)

1. **MQSC event gate** — enable all classes + `CMDEV(NODISPLAY)` + per-queue perf
   thresholds, across all arms; cold-rebuild verified.
2. **Collector `SERVICE`** — the `DEFINE SERVICE` + role-shipped `run.sh` wrapper
   (`amqsevt … | logger`) wiring (new `mq-event-monitor` role or folded into
   `mq-qmgr`); verify it travels across a failover, per arm.
3. **Pipeline label + verify** — Alloy relabel for `mq-events`; prove event JSON
   reaches Loki queryable by its fields.
4. **Dashboards** — dedicated events feed + per-object (queue/channel) event
   filter, **and** the log-panel exclusion so events don't bleed into the log
   views.

Plus epic bookends: documentation (this spec + the plan, `#32`), follow-on
brainstorm (`#33`), and the docs-review gate (`#34`).

## 10. Acceptance criteria

- [ ] Instrumentation events are enabled on the lab QMs **declaratively** (MQSC
      in the qmgr role/create block), surviving a cold rebuild.
- [ ] An `amqsevt -o json` collector runs as an **MQ `SERVICE` object**
      (`CONTROL(QMGR)`) that **travels with the QM across a failover**, draining
      events **destructively** and emitting JSON to journald tagged `mq-events`.
- [ ] Event JSON reaches Loki as the `{unit="mq-events"}` stream and is
      **queryable by its JSON fields** (`eventType`, `eventReason`, source QM,
      object name).
- [ ] A **dedicated Grafana panel/board** shows the live event feed, visually
      distinct from logs.
- [ ] The **log panels exclude `unit="mq-events"`** (no bleed) — separation holds
      on both the log side and the event side, not just the new panel.
- [ ] A queue and/or channel dashboard shows **events filtered to that object**.
- [ ] `CMDEV(NODISPLAY)` demonstrably surfaces human-run admin commands **without**
      the exporter's DISPLAY-poll noise.
- [ ] Demonstrates the thesis: the collector is a thin wrapper around `amqsevt` +
      config — **no bespoke PCF-parsing code**.
- [ ] `vrg-container-run -- vrg-validate` passes.

## 11. Out of scope

- **Not** replacing the `mq-diag-logging` diagnostic-log stream (`#308`/`#282`) —
  this is a complementary event stream.
- **Alerting/automation** off events — a follow-up once the feed exists (`#33`).
- Per-event-class **dashboard design** beyond the first live feed + per-object
  filter.
- The **exporter's** connection model — it stays client-mode as-is; whether it
  should also become a QM-concurrent `SERVICE` (mirroring this collector) is an
  explicitly parked question for the follow-on brainstorm (`#33`).

## 12. Open questions (resolve at build time)

- Exact `amqsevt` path and the default event-queue set — confirm against IBM
  Docs 9.4 (§4, §5.3).
- `SERVICE` `STOPARG` server-PID token syntax — confirm against IBM Docs 9.4.
- Output buffering under the `logger` pipe — verify prompt delivery; `stdbuf -oL`
  if needed (§5.3).
- Which queues get `QDPMAXEV`/`QDPHIEV` thresholds (which app queues are worth
  performance events) — decide during the gate task.
- Event volume once everything is enabled — measure, then pare back the class set
  if needed (feeds `#33`).
