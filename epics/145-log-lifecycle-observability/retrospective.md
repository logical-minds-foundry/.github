# Native HA log-lifecycle observability — Retrospective

> Backward-looking record partnering `spec.md` and `plan.md`. Read the three in
> order: what we set out to do → how we planned it → how it actually went.

## §0 At a glance

We set out to make the **replicated/linear log lifecycle** of the lab's Native HA
arms **observable and teachable** — enabling `LOGGEREV` on the linear-equivalent
Native HA arms via declare-and-verify, surfacing the log lifecycle on the cockpit,
and embodying the **monitor-the-automation** model (observe that automatic log
management is healthy; do not hand-roll archiving). All of it shipped, and — the
part that matters most — it is **proven end-to-end on a cold-rebuilt live arm**,
not just in unit tests.

**Work delivered**

| PR | Issue | What it did |
|----|-------|-------------|
| .github#148 | #146 | Spec + plan published under `epics/145-log-lifecycle-observability/` |
| #978 | #808 | **Spike** (gating): S1–S4 answered on a live arm; confirmed the architecture + corrected 3 plan assumptions |
| #979 | #809 | Log-type-aware `LOGGEREV` via declare-and-verify (verify `qm.ini` `LogType`, retire the #721 note) |
| #980 | #810 | Non-MQI `loglifecycle.py` collector deployed by the existing `nativeha-state` role/timer |
| #981 | #811 | Time-series, instance-aware log-health band on the per-QM cockpit board |
| #982 | #812 | Live-lab induce-and-assert validation playbook |
| #983 | #813 | Operator runbook for the Native HA log lifecycle (versioned site docs) |
| #986 | #985 | Fix: induce accumulates instead of clearing per iteration (spun off from #815) |
| #988 | #806 | Doc-review sweep — cross-referenced the work across the site guides |
| _(pending)_ | #147 | This retrospective |

**Validation gates (no PR — closed by evidence comment):** #814 cold-rebuild — **PASS**
(one-pass provision, LOGGEREV enabled, collector emitting on all three instances,
timer healthy); #815 live-lab induce-and-assert — **PASS** on re-run after #985.

**By the numbers**
- **Repos touched:** 2 — `mq-resiliency-lab-for-linux` (all code + site docs), `logical-minds-foundry/.github` (spec/plan + this retrospective).
- **Child tasks:** 12 — 2 docs bookends, 1 spike, 5 implementation tasks, 2 validation gates, 1 doc-review bookend, 1 spun-off fix.
- **PRs merged:** 9 (8 in the lab repo + the spec/plan PR), retrospective pending.
- **Releases cut:** none (develop-integration work).
- **Span:** opened 2026-07-28 → closing 2026-08-10 (~13 days).

## §1 How the plan evolved

The plan carried no formal "Evolution during execution" log; this narrative is
synthesized from the execution record. The plan's spine — **spike → Tasks 2/3/4 →
Task 5 → Task 6** — held. What changed was mostly *detail the spike corrected* and
*one bug the live-lab gate caught*:

- **The spike de-risked exactly what it was meant to — and paid for itself.** It
  **confirmed** the load-bearing architecture (LOGGEREV *is* accepted on the
  replicated QM; logger events *do* ride the existing `amqsevt` → journald
  pipeline) while **correcting three guessed mechanics** that would otherwise have
  become bugs downstream: `DISPLAY QMGR LOGTYPE` does not exist (→ verify against
  `qm.ini` `LogType`), the real log path is `/var/mqm/log/<QM>/active/` (not
  `…/qmgrs/<QM>/active`), and `rcdmqimg -t all` is not a valid invocation. All
  three flowed cleanly into Tasks 2–4.
- **A false alarm became a finding.** The spike's first read was `CURDEPTH(0)` on
  the logger-event queue — seemingly "no events." The cause was the lab's own
  `mq-event-monitor` *draining the queue in real time* (`IPPROCS(1)`); the events
  were flowing to journald all along. Channel A was fine — the check just had to
  look at the sink, not the depth.
- **The strict 1→2→3→4→5→6 chain relaxed to a partial order** once the spike
  landed: Tasks 2 and 3 were independent and ran in parallel; Task 4 followed 3
  (it consumes the collector's metric names); Tasks 5 and 6 ran in parallel (the
  runbook was blocked only by the spike). This shortened wall-clock without
  loosening any real dependency.
- **Two small design collisions, resolved in-task:** (a) #810's "import the
  role-detection helper" collided with the deploy-verbatim invariant (the nodes
  have no `mqlab` package) — resolved with a `TYPE_CHECKING`-guarded dual-context
  import; (b) #811's media-image-recency panel had no collector metric behind it,
  so it ships as a graceful no-data sentinel rather than a fabricated series.
- **The headline deviation — #815 caught a real bug that static validation did
  not.** The Task-5 playbook (#812) passed `vrg-validate` (ansible-lint, 100%
  coverage) but **failed its first live run**: the induce step never rolled a log
  extent. Two compounding defects — a per-iteration `CLEAR` that released the log
  before it could grow, and `base64 … | fold -w 4096` that (because `base64`
  wraps at 76 columns) was actually putting **76-byte** messages, not 4 KB. A
  controlled retain-don't-clear churn confirmed the mechanism, #985 fixed both,
  and the #815 re-run passed with the roll happening on the first pass. This is
  the epic's clearest lesson (see §2).

## §2 Lessons learned

- **A gating spike on a genuinely uncertain assumption is worth it.** Task 1
  returned three plan corrections and one architecture confirmation *before any
  role code was written*. Cheap insurance against expensive rework.
- **Static green is not "it works."** #812 was lint-clean and 100%-covered yet did
  nothing useful live. Live-lab validation (#815) is not ceremony — it is the only
  gate that catches "the playbook runs successfully and asserts nothing." Keep the
  live gate mandatory for anything whose whole job is to *observe real behaviour*.
- **`CURDEPTH(0)` is a false negative when a consumer holds the queue.** Check
  `IPPROCS`/the sink, not depth, before concluding "no events."
- **The monitor-the-automation thesis held up empirically.** Automatic log
  management and automatic media images are on by default on the replicated arms;
  the right observables are filesystem extent counts + logger events + the
  error-log watermarks (`AMQ7467I/7468I/7490I`) — never hand-managed archiving.

## §3 Compromises & tradeoffs

- **Media-image recency ships coarse.** The source (`mediaLogExtentName` /
  `MEDIALOG` / `AMQ7468I`) advances only on the automatic-image schedule and is
  pinned on a young QM; the collector does not emit a recency metric. The board
  therefore shows an honest no-data sentinel (`or vector(-1)`) rather than a fake
  reactive panel. Debt: a real `media_image_age` collector metric is a clean
  follow-on.
- **Reclaim health is surfaced, not fixed.** The spike observed automatic reclaim
  *not occurring in-window* (`MEDIALOG` pinned at extent 0; `AMQ7490I` = 55
  created / 0 reused / 0 deleted; extent count rising). We characterized this as
  expected pre-first-automatic-image early-life and **made it visible on the
  cockpit** rather than engineering around it — consistent with the epic's
  monitor-the-automation stance.
- **The induce fix was reactive.** It surfaced in validation, not review. We took
  the spin-off fix task (#985) rather than reopening the merged #812 branch —
  correct per the freeze discipline, but a sharper review of the induce shell
  would have caught the 76-byte-message bug earlier.

## §4 New problems & opportunities

- **#985 — induce accumulate-not-clear + real message sizing.** Spun off from the
  #815 failure; **fixed and closed.**
- **Media-image-age collector metric.** The board already carries the panel
  scaffold with a sentinel; a collector metric would make it real. *Logged, not
  yet acted on.*
- **Reclaim-health alerting.** The "monotonic extent rise, 0 reuse, pinned
  `MEDIALOG`" pattern is a natural alert threshold — deliberately **out of scope**
  here (the epic deferred alerting). *Logged.*

## §5 What's next

- Alerting/thresholds on the log-health band (the epic's explicit deferred
  follow-on).
- A `media_image_age` (or equivalent) collector metric to promote the media-image
  panel from a sentinel to a live gauge.
- No forward-looking brainstorm was accrued during this epic; none is queued.

## Appendix A — Operational notes (validation phase)

The terminal validation was run against a **freshly cold-rebuilt** arm so the
merged roles applied from scratch:

1. `mqlab teardown nativeha-rhel` (destroys the arm; commons too, as it was the
   last stack up).
2. `mqlab bootstrap nativeha-rhel` — box **REUSE** (no re-bake), net → vms →
   provision → observe, provisioned from `develop` at the Tasks-2/3 merge.
3. **#814** assertions on the fresh arm: `DISPLAY QMGR LOGGEREV` → `ENABLED`;
   `qm.ini LogType=REPLICATED`; `mqlab_log_*` present on all three instances
   (`sample_stale=0`); `lab-nativeha-state.timer` active.
4. **#815**: `ANSIBLE_CONFIG=ansible/ansible.cfg ansible-playbook
   ansible/validate-nativeha-log-lifecycle.yml` → induce rolls on pass 1, three
   asserts pass, negative-path drift guardrail aborts loud and is rescued.

**Gotchas:** after a teardown, re-registering the cached boxes into libvirt adds
~9 min before VM boot; the `net` phase idempotency fix (#974, pre-existing) is
what lets a re-bootstrap reuse the already-defined shared networks without
colliding.
