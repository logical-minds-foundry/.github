# Event monitoring as a de-facto standard on every queue manager — design spec

- **Epic:** `logical-minds-foundry/.github#114`
- **Design task:** `logical-minds-foundry/.github#115`
- **Follow-on brainstorm task:** `logical-minds-foundry/.github#116`
- **Documentation-review task:** `logical-minds-foundry/mq-resiliency-lab-for-linux#718`
- **Builds on:** `#31` (event monitoring → JSON → Loki — the mechanism), `#514`
  (enable event classes), `#515` (the `amqsevt` collector SERVICE)
- **Sibling / prior epic:** observability stack (`logical-minds-foundry/.github#8`)
- **Unblocks:** `#110` (Native HA event-JSON capture)
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-07-20

## 1. Problem & motivation

Epic `#31` gave the lab an instrumentation-event stream: enable the Multiplatforms
event classes on a queue manager, and run IBM's `amqsevt` sample as an MQ `SERVICE`
object that drains `SYSTEM.ADMIN.*.EVENT` destructively to JSON on journald
(tagged `mq-events`), where the existing journald → alloy → Loki → Grafana pipeline
picks it up. It is pure configuration — no custom PCF-parsing application — a clean
demonstration of the lab's glass-box thesis.

But `#31` only ever **deployed to one queue manager.** Both halves — the event-class
enable (`#514`) and the collector SERVICE (`#515`) — live exclusively in the
`mq-qmgr` ansible role, and `mq-qmgr` is applied only to `hosts: svc` in
`site-distributed-shared.yml`. That is the shared Business-B counterparty `SVCQM`
on `svc-sim`. The four canonical `#350` stack queue managers —
`NHARAPP` (nativeha-rhel), `NHAUAPP` (nativeha-ubuntu), `PCMKAPP` (pcmk-ubuntu),
`RDQMAPP` (rdqm-rhel) — get **neither**: their QM-creation roles (`mq-nativeha`,
`mq-pcmk-qmgr`) and the rdqm script path carry no event enablement and no
collector. Verified live: on `NHARAPP`, every class is `DISABLED` except
`STRSTPEV`, `MQ.EVENT.MONITOR` does not exist, and `journalctl -t mq-events` is
empty.

`#514`/`#515` were built as a proof-of-concept on `SVCQM`; the follow-on to make it
standard everywhere was never tracked and never happened. This is precisely the
drift the epic framework exists to catch — a small-scale success whose large-scale
follow-on is quietly lost.

## 2. Doctrine & principles

- **Event monitoring is a de-facto standard, not a feature.** Every queue manager
  in the lab should emit its instrumentation-event stream by construction, so the
  observability framework covers all of them uniformly.
- **Single source of truth.** The enable + collector configuration lives in exactly
  one place; every QM-creation path consumes it by inclusion. There is no second
  copy to forget — adding a future arm is one `include_role`. This is the direct
  structural answer to the drift that motivated the epic.
- **Ride the existing pattern.** Every QM-creation role already
  `include_role: mq-diag-logging` (the `#282` JSON-diagnostics + journald drop-in).
  Event monitoring rides that same seam — one more shared cross-cutting include.
- **The SERVICE object travels; the wrapper file does not.** The collector is an MQ
  SERVICE with `CONTROL(QMGR)`, so it is part of the QM's object definitions and
  starts, stops, and fails over **with** the QM — across HA failover and DR/CRR
  cutover alike. Defining it once per logical QM is sufficient; site-B/DR needs no
  separate handling. **But** the SERVICE's `STARTCMD` runs a host-local `run.sh`
  wrapper, which is **not** part of the QM and does **not** replicate — so the
  wrapper must be installed on **every node the QM can activate on**, or the
  collector fails to start after a failover. This asymmetry — object once on the
  active node, wrapper on every node — is the structural driver of the role's two
  entry points (§3.1).
- **No bespoke code, no new plumbing.** Everything is MQSC + an existing role. The
  journald drop-in, alloy, and `amqsevt` are already present on every arm node.

## 3. Deliverables

A single generic mechanism plus its wiring into every QM-creation path:

### 3.1 Consolidated `mq-event-monitor` role, two entry points

Extend the existing `mq-event-monitor` role (today: define + start the collector
SERVICE, `#515`) to **also** enable the event classes (`#514`:
`ALTER QMGR …EV(ENABLED)` and the per-queue performance-event step), and split it
into two entry points along the object-vs-file asymmetry of §2:

- **Host-prep (`main`)** — assert preconditions (amqsevt present, journald drop-in
  present) and install the `run.sh` wrapper. Runs on **every** node the QM can
  activate on.
- **MQSC (`service`)** — enable the event classes, set per-queue performance events,
  define + start the SERVICE. Runs **once, on the active instance**; the objects
  replicate and travel.

This mirrors the split `mq-diag-logging` already uses (`tasks_from: system` on all
nodes vs `tasks_from: qmini` per-QM). A standalone QM (SVCQM) includes both halves
on its one host; an HA arm includes host-prep on every node and the MQSC half on
the active node.

### 3.2 Wiring into every QM-creation path, `mq-qmgr` de-duplicated

- `mq-nativeha` → `include_role: mq-event-monitor` (covers nativeha-rhel **and**
  nativeha-ubuntu — one OS-agnostic role)
- `mq-pcmk-qmgr` → `include_role: mq-event-monitor` (pcmk-ubuntu)
- rdqm path (`site-rdqm.yml`, gated `when: rdqm_is_active`) →
  `include_role: mq-event-monitor` (rdqm-rhel)
- `mq-qmgr` → replace its inline `#514` enable tasks with
  `include_role: mq-event-monitor` (SVCQM), removing the now-duplicated block

## 4. Scope — the queue managers covered

| Stack | QM | Creation seam | Change |
|---|---|---|---|
| nativeha-rhel | `NHARAPP` | `mq-nativeha` role | add include |
| nativeha-ubuntu | `NHAUAPP` | `mq-nativeha` role (same) | (covered by the same edit) |
| pcmk-ubuntu | `PCMKAPP` | `mq-pcmk-qmgr` role | add include |
| rdqm-rhel | `RDQMAPP` | `rdqm-qm-create.sh` + `site-rdqm.yml` include | add include |
| shared | `SVCQM` | `mq-qmgr` role | refactor to include (re-verify) |

Three code seams to change (`mq-nativeha`, `mq-pcmk-qmgr`, the rdqm path) plus one
to refactor (`mq-qmgr`). Because the collector travels with the QM, the site-B/DR
instances of each are covered without separate work.

## 5. Approach & data flow

Unchanged from `#31`: on the QM host, the `amqsevt` collector runs in bindings
mode as a `CONTROL(QMGR) SERVTYPE(SERVER)` SERVICE, draining `SYSTEM.ADMIN.*.EVENT`
destructively and emitting JSON to `logger -t mq-events` → journald. Alloy relabels
the tag to `unit="mq-events"` and ships it to Loki on obs; Grafana reads it. This
epic changes **only which QMs run that configuration** — from one to all — by
factoring the enable + define into the shared role and including it everywhere.

**Where each half runs.** The **host-prep** half (wrapper + asserts) runs on every
node in the arm's group, alongside the arm's other per-node prep (it rides the same
place `mq-diag-logging` is included). The **MQSC** half (enable + SERVICE-define) is
a QM-level operation, so it runs once against the node where the QM is live: the
active Native HA instance, the Pacemaker resource owner, or `when: rdqm_is_active`.
Each seam already carries that active-node gating for its other QM-level includes, so
the role rides it rather than inventing new host-selection logic.

## 6. Component boundaries & isolation

- **`mq-event-monitor` role** — *what:* enable event classes + define/start the
  collector on one QM. *Interface:* included with `qmgr_name` (and any per-arm
  perf-queue list) in scope; asserts amqsevt + journald drop-in. *Depends on:*
  `mq-diag-logging` having run (drop-in), the MQ samples (amqsevt). This is the
  whole surface — no consumer reads its internals.
- **QM-creation roles / paths** — *what:* create the QM, then include
  `mq-event-monitor`. Their only new responsibility is the one include, on the
  correct (active) node.
- **`mq-qmgr`** — loses its inline event block; gains the include. Behaviour for
  `SVCQM` is unchanged (verified by acceptance), but the logic is no longer a
  private copy.

## 7. Implementation tasks (preview; finalized in the plan)

- **T1 — Consolidate + refactor.** Fold `#514` enable (+ per-queue perf) into
  `mq-event-monitor`; refactor `mq-qmgr` to include it; delete the inline block.
  Acceptance: `SVCQM` still gets events + a running collector.
- **T2 — nativeha.** Include the role in `mq-nativeha`. Acceptance: `NHARAPP` and
  `NHAUAPP` show events + a running collector, events reach journald/Loki.
- **T3 — pcmk.** Include the role in `mq-pcmk-qmgr` (pcmk-ubuntu). Acceptance:
  `PCMKAPP` likewise, and the SERVICE survives a Pacemaker failover.
- **T4 — rdqm (the disproportionate-effort task).** Include the role in the rdqm
  path. Unlike T2/T3 this seam is **script-based** (`rdqm-qm-create.sh` +
  `site-rdqm.yml` includes), not a clean role include, and was **recently rewritten**
  to IBM's coordinated one-`crtmqm`-per-site model (`#561`) with reworked active-node
  resolution (`#582`) and a string of follow-on fixes (`#559`, `#591`, `#593`). It is
  the most-churned, most-fragile arm. The plan must give it its own careful step:
  the exact include point, the active-node gating (`when: rdqm_is_active`), and
  re-provision idempotency, all validated against the **current** script — not
  assumed to mirror the role-based arms. Acceptance: `RDQMAPP` likewise, and the
  SERVICE survives an RDQM failover.

Ordering: **T1 first** (it establishes the consolidated role the others include).
**T2 and T3** (the clean role-based arms) proceed next and independently — landing
them first proves the consolidated role on real QMs. **T4 last**, deliberately, so
it builds on a role already exercised by T2/T3 rather than being the first live test
of it. T4 is explicitly the largest, riskiest task of the four, not an equal peer.

## 8. Acceptance criteria

- **Live end-to-end proof per distinct QM-creation mechanism** — not a full
  bring-up of all four stacks. On a freshly provisioned stack, the target QM has all
  event classes enabled (`CMDEV(NODISPLAY)` per `#514`), `MQ.EVENT.MONITOR` defined
  with `CONTROL(QMGR)` and `STATUS(RUNNING)`, and `journalctl -t mq-events` showing
  JSON — the same state `SVCQM` has today. This is demonstrated **live** on each
  distinct code path: one `mq-nativeha` arm (proves both nativeha-rhel and
  nativeha-ubuntu, which share the role — the twin is confirmed by config-inclusion),
  `mq-pcmk-qmgr` (pcmk-ubuntu), and the rdqm path (rdqm-rhel). `SVCQM` is already
  live. Every code path gets real proof without standing up every stack.
- `SVCQM` is unchanged post-refactor (no regression from de-duplicating `mq-qmgr`).
- The enable + collector configuration exists in exactly one role; no arm carries a
  private copy.
- A representative forced event (e.g. an authority failure) appears in the Grafana
  `mq-events` stream for at least one arm QM (proof the full path works end to end).
- `vrg-validate` passes.

## 9. Out of scope

- **`pcmk-rhel`** — being decommissioned (an unfinished OSS-Pacemaker-on-RHEL
  proof-of-concept); it has no MQ queue manager to instrument.
- **A standing regression guardrail / validation gate** — the chosen path is to
  wire event monitoring in everywhere; an active check that fails when coverage
  drifts is a candidate follow-on (`#116`), not part of this epic.
- **A broader "standard QM" umbrella role** unifying diag-logging + events +
  monitoring stats (brainstorm Approach C) — candidate follow-on (`#116`).
- **Any change to shipping, storage, or visualization** of the event JSON — that
  boundary stays with `#31` / `#8`. This epic only changes coverage.

## 10. Open questions (resolve at build time)

- **Per-arm performance-event queues.** `#514`'s per-queue perf step loops over
  `mq_event_perf_queues` (empty by default). Decide per arm whether any app queue
  should carry depth thresholds, or leave the safe no-op. Resolve in the plan/build.
- **rdqm include placement.** Confirm the exact point in `site-rdqm.yml` (after QM
  creation, `when: rdqm_is_active`) where the include is both correct and
  idempotent on re-provision.
- **Failover survival evidence.** T3/T4 acceptance asserts the SERVICE survives
  failover; decide whether that is a live-lab demonstration or an inspection of the
  replicated QM object set. Resolve in the plan.
- **Event volume across the fleet (accepted, watch).** Enabling all classes on ~5
  QMs — several under HA/DR channel churn — funnels more `mq-events` into one Loki
  than the single-`SVCQM` proof-of-concept did. This is **accepted for a lab**:
  `CMDEV(NODISPLAY)` (`#514`) already prevents the `mq_prometheus` exporter's
  DISPLAY/Inquire-PCF poll-flood, and event volume is otherwise modest. Watch for
  noise and revisit only if it bites; no volume-control work is in scope here.
