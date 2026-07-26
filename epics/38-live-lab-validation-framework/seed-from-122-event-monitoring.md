# Seed material for #38 (Live-lab validation framework) — from epic #122

> **Status: seed / input only.** #38 runs its own brainstorm → spec → plan at
> initiation; this is proven material to draw from, **not** #38's design. Handoff
> produced by epic #122's closing brainstorm (#124).

## Why this exists

Epic #122 (resilient `amqsevt` event-monitor wrapper) shipped, and in the process
produced a **working, lab-proven instance of this epic's named first deliverable,
`validate event-monitoring`.** Rather than let that sit unread until #38 is picked
up — the exact "knowledge existed but wasn't surfaced at the decision" failure that
cost real time inside #122 (a correct triage, #101, sat unread for days) — it is
handed off here, and pointed at from the #38 issue body.

## The proven contract (#38's `arrange → induce → assert observable → report`), demonstrated

`tools/validate-event-monitor-wrapper.sh` (in `mq-resiliency-lab-for-linux`) already
runs exactly this shape today:

| #38 contract stage | What the harness does |
|---|---|
| **arrange** | resolve the active node; confirm the collector + `amqsevt` are up |
| **induce** | `kill -9` the collector; `STOP SERVICE` |
| **assert observable** | no orphan; restart occurs; `PID == PGID`; JSON events in journald |
| **report** | PASS/FAIL per scenario + captured evidence; fail-loud; leaves the lab healthy |

Proven live on nativeha-ubuntu (`NHAUAPP`): full B-matrix PASS, T7/#785.

## Reference artifacts (all in `mq-resiliency-lab-for-linux`)

- Harness: `tools/validate-event-monitor-wrapper.sh`
- Engineering report: `docs/reports/2026-07-21-mq-event-monitor-wrapper-resilience.md`
- Procedure (per-arm active-node resolution + how to run + where evidence goes):
  `docs/reference/event-monitor-wrapper-validation.md`

## Design surface for a *full* `validate event-monitoring` (material, not prescription)

- **Conditions to induce:** channel start/stop; a channel forced into retry; a queue
  pushed past `QDEPTHHI`; an MQSC command issued; the collector killed.
- **Observables to assert:** the matching event JSON reaches journald with the
  expected `eventSource` / `eventType`; a drained event queue's depth returns to
  zero (destructive drain, not browse); the service travels on HA failover; the
  collector self-heals through the `MQRC_OBJECT_IN_USE` (2042) reap window.
- **Cross-cutting:** per-arm active-node resolution (Native HA / RDQM / Pacemaker /
  standalone — see the procedure doc); a leave-lab-healthy discipline; fail-loud
  everywhere.

## Standing caveat (load-bearing)

`amqsevt` gets events **non-transactionally** (`MQGMO_NO_SYNCPOINT`). A behavior
validator can therefore assert **process recovery** and **feed liveness**, but **not
zero event-loss across a crash** — an in-flight event at crash time is simply gone.
Exact loss accounting belongs to the **DR/HA loss-quantification tenant (#44)**, not
to this behavior validator. If event-loss ever becomes a real requirement, the answer
is a better collector than the sample, not more scaffolding around it.
