# Native HA log-lifecycle observability — design spec

**Epic:** `logical-minds-foundry/.github#145`
**Status:** draft (spec)
**Arms in scope:** the two Native HA arms (`nha-rhel-*`, `nha-ubuntu-*`)
**Arms touched only by a verified negative:** RDQM, PCMK, single-QM

---

## 1. Problem & motivation

IBM MQ has **three** log types, not two
([Types of logging, 9.4](https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=lost-types-logging)):
circular, linear, and **replicated** — the last *"used only by Native HA
configurations."* The same page states, verbatim:

> *"A replicated log is equivalent to a linear log with automatic log management
> and automatic media images enabled."*

That single sentence is the root of this epic. The lab's two Native HA arms are
created with `crtmqm -lr` (`ansible/roles/mq-nativeha/tasks/main.yml:63`,
`ansible/roles/mq-nativeha-spike/tasks/main.yml:112`), so their queue managers
run a replicated log and are therefore **linear-equivalent** for every
operational purpose: media images, log-extent create/delete/reuse (AMQ7490),
log-disk growth, and — critically — the validity of the `LOGGEREV` queue-manager
attribute. The RDQM, PCMK, and single-QM arms use the `crtmqm` default
(**circular**); RDQM's "replication" is DRBD block-level, so its MQ log is plain
circular.

The lab currently disables `LOGGEREV` on **every** arm. This was a deliberate,
correct-at-the-time fix in #721: `ALTER QMGR ... LOGGEREV(ENABLED)` is rejected
**wholesale** (`AMQ8518E`) on a circular queue manager, aborting the entire
`ALTER` so that *no* event class applied. The role
(`ansible/roles/mq-event-monitor/tasks/service.yml:13-16`) dropped the clause and
left an explicit note:

> *"LOGGEREV(ENABLED) dropped … it is only valid on a linear-logging queue
> manager, and this lab is circular-logging everywhere … **Revisit only if linear
> logging is ever adopted.**"*

**That revisit condition is already met and we missed it.** The Native HA arms
are linear-equivalent. So today the lab unconditionally disables `LOGGEREV`
exactly where it is valid, and exactly where it is the natural backbone of
log-lifecycle observability. This epic corrects that, makes the
replicated/linear log lifecycle observable, and — because this is a genuinely new
operational layer for the operator — makes it legible and teachable.

## 2. Doctrine & principles

- **Monitor the automation; do not hand-roll it.** A replicated log is
  automatic-log-management + automatic-media-images by default. MQ drives extent
  reclaim/reuse and media images. The lab's job is to *observe that the
  automation is healthy* and *teach what it is doing* — not to build manual
  archiving. (See §5 for the "why not manual" decision.)
- **Declare and verify what you do not own.** The role declares the intended log
  type per arm and asserts it against the live queue manager's actual log type,
  failing loud on drift — never silently trusting either side.
- **Fail loud, must-apply.** Every MQSC step that must apply asserts it did
  (`AMQ8005I`), never merely "does not error" — the discipline already in
  `mq-event-monitor` after #721.
- **Collectors are stdlib-only / non-MQI.** The log-health collector shells out
  to `du`/`df`/filesystem like the existing HA/DR collectors — no PyMQI, no
  compiled deps.
- **Dashboards lead with time-series.** The log-health panel group leads with
  trend panels, not point-in-time tiles, per the lab's dashboard convention.
- **Legibility is a deliverable.** The operator runbook and the panel group exist
  to build operator intuition for a new layer, not to decorate it.

## 3. Grounding: logging in the lab today

| Arm | `crtmqm` | Log type | `LOGGEREV` today | Correct target |
|---|---|---|---|---|
| Native HA RHEL (`nha-rhel-*`) | `-lr` | replicated (linear-equiv) | disabled (omitted) | **enabled** |
| Native HA Ubuntu (`nha-ubuntu-*`) | `-lr` | replicated (linear-equiv) | disabled (omitted) | **enabled** |
| RDQM (`rdqm-a/b-*`) | `-sx` (no `-lr`/`-ll`) | circular | disabled (omitted) | verified-omitted |
| PCMK (`pcmk-a/b-*`) | plain (`-md/-ld`) | circular | disabled (omitted) | verified-omitted |
| Single / plain (`mq-qmgr`) | plain | circular | disabled (omitted) | verified-omitted |

The `LOGGEREV` clause was removed from the single `ALTER QMGR` in
`mq-event-monitor/tasks/service.yml`; the remaining 11 event classes apply
cleanly. Logger events, if enabled, would land on `SYSTEM.ADMIN.LOGGER.EVENT` and
are consumable by the existing `amqsevt` wrapper the event-monitor role already
runs.

## 4. The design

### 4.1 Spike first — de-risk before building (gates everything)

A short spike on one live Native HA arm, findings captured as an engineering
note, answers four questions that the rest of the epic assumes:

- **S1 — Acceptance.** Is `ALTER QMGR LOGGEREV(ENABLED)` actually *accepted* on a
  replicated-log queue manager? The "equivalent to linear" claim is IBM's; we
  verify it on the live lab rather than trust it.
- **S2 — Which events fire.** Under automatic log management, which logger events
  are generated, and what extent watermarks do they carry (archive / current /
  media-recovery / restart-recovery)? This determines what the event channel can
  actually surface.
- **S3 — Pollable-without-MQI signals.** What log-health signals are available by
  shelling out (filesystem extent counts under the log dir, `du`/`df` on the log
  path) vs. only via the event stream? This determines the collector's scope.
- **S4 — The circular negative.** Is `LOGGEREV(DISABLED)` even *accepted* on a
  circular queue manager, or is the attribute wholesale-rejected regardless of
  value (#721 says it is "only valid on a linear-logging queue manager")? If
  rejected, the circular "verified negative" is **verify-and-omit** (assert the
  QM is circular and never emit the clause), not an explicit `DISABLED` `ALTER`.

**S1 and S4 can change the shape of §4.2; S2 and S3 can change §4.3.** The spike
is the first task and its findings are applied forward.

### 4.2 Correctness core — log-type-aware `LOGGEREV` (declare-and-verify)

- **Declared intent.** A per-arm log-type declaration (e.g. `mq_log_type:
  replicated | circular`) lives with the arm's group/role configuration — one
  source of truth, explicit and readable.
- **Conditioned clause.** `mq-event-monitor` conditions the `LOGGEREV` clause on
  the declared type: **enabled** on the Native HA (replicated) arms; on circular
  arms the clause is **omitted** (or set per the S4 finding), never a silent gap.
- **Verify against reality.** Before acting, the role asserts the declared type
  matches the live queue manager's *actual* log type and **fails loud** on any
  drift. This mirrors the role's existing must-apply/`AMQ8005I` assertion and
  retires the #721 "revisit if linear is ever adopted" note.
- **Must-apply, per arm.** On the Native HA arms the enabling `ALTER` must return
  `rc 0` + `AMQ8005I`; anything else is a real failure and surfaces. On circular
  arms the verified negative is asserted, so the lab *proves* it made the right
  choice on every arm rather than leaving `LOGGEREV` absent by accident.

### 4.3 Observability — two channels into one panel group

**Channel A — logger events → the `mq-event-monitor` `amqsevt` pipeline.** With
`LOGGEREV(ENABLED)` on the Native HA arms, logger events (extent-created, media
image) reach `SYSTEM.ADMIN.LOGGER.EVENT` and flow through the existing wrapper
into the event stream. No new consumer is built; the existing pipeline gains a
new event class it already knows how to carry.

**Channel B — log-health metrics → a stdlib collector.** A new collector, in the
same non-MQI style as the HA/DR collectors, shells out to `du`/`df`/filesystem to
emit: log-disk usage %, active/inactive extent counts, and enough to render a
fill trend. Metric names/types are verified on a live exporter before the panel
is wired (rate vs. gauge per `# TYPE`).

**The panel.** A dedicated, **time-series-led** log-health panel group on the
Native HA arms' cockpit: log-disk % with fill trend, extent counts over time,
media-image recency, and a logger-event log. This is the deliverable that makes
the lifecycle legible at a glance. QM name is a variable, not hardcoded (per the
dashboard de-hardcoding direction).

### 4.4 Live-lab validation

An induce-and-assert playbook under the **live-lab validation framework**
(epic `#38`): drive log churn / force an extent roll on a Native HA arm, then
assert **both** that the logger event fires **and** that the collector/panel
reflects it. Turns "it should work" into a repeatable, human-out-of-the-loop
proof.

### 4.5 Operator runbook (first-class, versioned site docs)

A teaching-oriented guide in `docs/site/…`: what automatic log management and
automatic media images actually do; extent create/delete/reuse and AMQ7490; how
to read the log-health panel; and the one real constraint — **automatic log
management without archiving means a backup queue manager is not supported**
([Types of logging, 9.4](https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=lost-types-logging)).
That constraint is documented, not engineered around: Native HA obsoletes the
backup-QM pattern by construction.

## 5. Binding decisions (explicit)

1. **Maintenance model = monitor-the-automation.** No manual log management, no
   hand-rolled archiving. *Judgment (not IBM-sourced):* greenfield 2026 Native HA
   has essentially no reason to manage extents by hand — the backup-QM reason is
   obsoleted by Native HA, and archiving (the one still-real use) is event-driven.
2. **No archiving.** Automatic-log-management-without-archiving; the backup-QM
   constraint is documented (§4.5), not engineered around.
3. **Circular arms get a *verified* negative,** not a silent omission — the exact
   form (verify-and-omit vs. explicit `DISABLED`) is set by spike finding S4.
4. **Collector stays stdlib / CLI, non-MQI,** consistent with the HA/DR
   collectors.
5. **Alerting/thresholds are out of scope** — a noted follow-on if the alerting
   layer is ever added, not a seeded task.
6. **Everything targets the Native HA arms;** no changes to RDQM/PCMK/single log
   configuration beyond the verified negative.

## 6. Scope & non-goals

**In scope:** log-type-aware `LOGGEREV`; the de-risking spike; the stdlib
log-health collector; the time-series cockpit panel group on the Native HA arms;
logger events through the existing event pipeline; a live-lab induce-and-assert
validation playbook; the operator runbook.

**Non-goals:** manual log management / archiving tooling; alerting or threshold
signals; any change to RDQM/PCMK/single log configuration; touching the
`crtmqm -lr` formation itself (the replicated log already exists — this epic
observes and correctly configures around it).

## 7. Verification & acceptance

- **Spike note** recorded (S1–S4 answered) before dependent tasks proceed.
- **Log-type-aware `LOGGEREV`:** on the Native HA arms the enabling `ALTER`
  returns `AMQ8005I`; the declared-vs-actual log-type assertion passes; a forced
  drift fails loud. On circular arms the verified negative holds.
- **Observability:** the collector emits log-health metrics on a live Native HA
  arm; the panel group renders them time-series-led; logger events appear in the
  event stream.
- **Live-lab validation:** the induce-and-assert playbook drives log churn and
  asserts both the event and the panel reflect it — `Outcome: SUCCESS`.
- **Cold rebuild:** the log-type-aware changes + the collector come up one-pass on
  a cold-rebuilt Native HA arm (the lab's standing cold-rebuild acceptance gate).
- **Runbook** merged into the versioned site docs and reviewed by the
  documentation-review bookend.

## 8. Relationships

- **Builds on #114 / #122** — event monitoring as a de-facto standard on every
  QM, and the resilient `amqsevt` wrapper this epic reuses as Channel A.
- **Plugs into #38** — the live-lab validation framework (the induce-and-assert
  playbook).
- **Feeds #79 / #128** — the observability-extraction and observability ad-hoc
  epics; the log-health collector + panel are extraction candidates (QM name as a
  variable, per the cockpit de-hardcoding direction).
- **Retires the #721 note** in `mq-event-monitor/tasks/service.yml`.

## 9. Task breakdown

Indicative; finalized in the plan.

1. **Spike** — verify S1–S4 on a live Native HA arm; record an engineering note.
2. **Log-type-aware `LOGGEREV`** — per-arm declaration + declare-and-verify in
   `mq-event-monitor`; retire the #721 note.
3. **Log-health collector** — stdlib/non-MQI; log-disk %, extent counts, fill
   trend; metric names/types verified on a live exporter.
4. **Cockpit log-health panel group** — time-series-led, Native HA arms, QM name
   as a variable.
5. **Live-lab validation playbook** — induce-and-assert under #38.
6. **Operator runbook** — versioned site docs.
7. **Operational gates** (seeded at plan time): cold-rebuild validation; the
   live-lab validation run.
