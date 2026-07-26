# Resilient `amqsevt` event-monitor service wrapper — design spec

- **Epic:** `logical-minds-foundry/.github#122`
- **Design task:** `logical-minds-foundry/.github#123`
- **Follow-on brainstorm task:** `logical-minds-foundry/.github#124`
- **Documentation-review task:** `logical-minds-foundry/mq-resiliency-lab-for-linux#758`
- **Builds on:** `#31` (event monitoring → JSON → Loki, the mechanism), `#515`
  (the `amqsevt` collector SERVICE), `#114` (rollout of the collector to every arm)
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-07-21

## 1. Problem & motivation

Epics `#31`/`#515`/`#114` gave every lab queue manager an event-monitoring
collector: IBM's `amqsevt` sample runs as an MQ `SERVICE` object
(`CONTROL(QMGR)`), draining `SYSTEM.ADMIN.*.EVENT` destructively to JSON on
journald, where the journald → alloy → Loki → Grafana pipeline picks it up. The
SERVICE's `STARTCMD` runs a host-local wrapper,
`ansible/roles/mq-event-monitor/templates/run.sh.j2`, which today does exactly one
thing (`run.sh.j2:29`):

```bash
exec stdbuf -oL {{ mq_event_amqsevt_bin }} -m "$1" -o json_compact > >(exec logger …)
```

That `exec` is deliberate and gives a **clean stop for free**: it replaces the
shell so the process the QM tracks (`MQ_SERVER_PID`) *is* `amqsevt`, and the
SERVICE's `STOPCMD` — `kill MQ_SERVER_PID` — stops the collector cleanly on QM
shutdown. The trade-off is that the model provides **zero crash resilience**.

`amqsevt` is a **non-transactional** MQ sample that parses PCF event messages. A
corrupted or unexpected message can crash it. When it dies, the `exec`'d process
is gone, the SERVICE ends, and event monitoring **silently stops** until the next
QM restart. Unlike systemd, an MQ SERVICE object has **no built-in restart** — the
QM starts the `STARTCMD` once and does not resurrect it. So the current design's
robustness rests entirely on `amqsevt` never crashing, which for a PCF-parsing
sample program is not a safe assumption in a resiliency lab.

This epic makes the event-monitor service **self-healing across `amqsevt`
crashes** without losing the clean-stop guarantee — and proves both behaviours
with real, captured lab evidence in an internal engineering report.

## 2. Doctrine & principles

- **Resilience is the lab's whole subject.** A monitoring collector that silently
  dies on the first malformed event is exactly the failure mode this lab exists to
  study and eliminate. The collector must recover itself.
- **Do not lose the clean-stop guarantee.** The current `exec` model's one virtue —
  a QM stop cleanly stops the collector, no orphan, no leaked handle — is a hard
  requirement of the new design, not something to trade away for restart.
- **Keep the wrapper inside the bash envelope.** A restart loop is trivial in shell.
  The design deliberately avoids in-script signal handling (`trap` + backgrounding),
  which is the point at which a shell script has outgrown its language. The chosen
  mechanism uses only portable, decades-old Unix process-group semantics — no
  advanced signal plumbing in the script itself. (See the shell-vs-real-language
  rule in §3.)
- **Read the source before guessing.** `amqsevt` is one of the rare IBM samples
  shipped with readable C source. Its exit codes, signal handling, queue-open mode,
  and handle-close behaviour are facts to be *read and then proven*, not reverse-
  engineered from black-box observation.
- **Evidence, not assertion.** The deliverable is not just a working script but a
  repeatable validation procedure and an internal report containing real log-file
  evidence of each behaviour under test.

## 3. Design — the wrapper mechanism

### 3.1 Chosen: bash loop + process-group kill

`run.sh` becomes a minimal supervision loop:

- It runs `amqsevt` in the **foreground** (not backgrounded) inside a `while` loop,
  in its own session/process group so the **wrapper is the group leader** —
  `PGID == wrapper PID`. The committed launch form is the whole script wrapped in
  `setsid`, from the SERVICE `STARTCMD`:
  `setsid -w -- /path/to/run.sh <QM>` (util-linux `setsid`, present on RHEL and
  Ubuntu). `--` stops `setsid` parsing `run.sh`'s own arguments; `-w` keeps the
  QM-tracked process alive for the service's lifetime in the edge case below.
- The SERVICE definition changes its stop from targeting a single PID to targeting
  the **whole process group**:
  `STOPCMD('/bin/kill') STOPARG('-TERM -- -+MQ_SERVER_PID+')`. The `+MQ_SERVER_PID+`
  insert **must** carry the `+` delimiters — MQ substitutes it (even embedded in a
  larger argument) only when delimited; the bare `-MQ_SERVER_PID` is passed literally.
  The **negative PID** is the classic Unix idiom: `kill -TERM -<pgid>` signals every
  process in the group — both the wrapper *and* the `amqsevt` child — in one call; the
  `--` is for procps-ng `/bin/kill`, so `-<pid>` reads as a process group, not an option.
- No `trap` in the script. On `STOPCMD`, the wrapper receives `SIGTERM` via the
  group signal and terminates by its **default disposition**; it does not loop.
  This is what keeps the shell "dumb" and inside the bash envelope.

Distinguishing the two exit paths falls out of the mechanism, with no exit-code
logic:

| Event | What happens | Wrapper behaviour |
|---|---|---|
| Clean stop (`STOPCMD`) | Group `SIGTERM` hits wrapper **and** child | Wrapper dies by default; no restart |
| `amqsevt` crash | Child exits; wrapper is **still alive** | Loop body returns → log, sleep, re-run |

The child's exit status is used only for the log line, never for control flow.

**The one environment-dependent assumption (the checkpoint).** `setsid` only forks
when the process MQ spawned is *already* a process-group leader; otherwise it calls
`setsid(2)` in place and `exec`s `run.sh`, giving the clean identity
`MQ_SERVER_PID == PGID == run.sh` with `amqsevt` as its child in that group — the
case the group-kill relies on. If MQ *does* spawn the STARTCMD as a group leader,
`setsid` forks and the QM-tracked PID diverges from the group leader (here `-w`
prevents the parent exiting and orphaning the collector, but `-TERM -- -+MQ_SERVER_PID+`
would no longer target the child's session). Which case MQ actually gives us is
determined empirically first (§5, A0), and the identity is asserted as a **named
pre-flight checkpoint** (§5, A2 / §6, B2): **`MQ_SERVER_PID == PGID` and the QM is
tracking the live leader (not an orphan) ⇒ the `setsid -w --` design ships;
otherwise ⇒ escalate to the §3.4 Python fallback.**

### 3.2 Rejected: `trap` + background

The alternative — `trap` `SIGTERM` in the shell, background `amqsevt`, forward the
signal to it, `wait` — is rejected. It puts real signal handling into a shell
script, which is precisely the smell the §2 rule names. It is more code, more
fragile (`wait`/job-control interactions), and buys nothing the process-group
approach does not deliver more simply.

### 3.3 Shell-vs-real-language rule (why bash is the right call *here*)

The governing rule: a shell script that runs past roughly a screen page, **or**
that needs advanced Unix functionality such as signal handling, has outgrown bash
and should be a real, tightly-C-library-integrated language (Python today, Perl
historically) — which additionally lets it be built as a properly unit-, integration-,
and security-tested tool. This problem sits **deliberately inside** the bash
envelope: a short loop using only process-group semantics that are portable across
any modern Linux, with no in-script signal handling. That is why bash is chosen
over escalation, not merely tolerated.

### 3.4 Fallback: stdlib-only Python wrapper

If the checkpoint (§3.1, §5 A0/A2, §6 B2) proves the process-group approach
**cannot** guarantee both processes die in the real MQ-SERVICE launch context —
i.e. `MQ_SERVER_PID != PGID` because MQ spawns the STARTCMD as a group leader and
`setsid` is forced to fork, or the negative-PID `STOPARG` does not
substitute/dispatch as expected — the design escalates to a small **stdlib-only
Python** wrapper doing the supervision and signal handling properly (matching the
lab's existing stdlib-only collector precedent). This fallback is **documented,
not built** unless the research forces it. Crossing into Python is exactly the
threshold the §3.3 rule describes.

## 4. Restart & logging policy

- **Fixed ~10 s sleep, infinite retry on crash, no backoff.** The cost of a restart
  is low and the workload is not a tight loop, so exponential backoff solves a
  problem this design does not have. The ~10 s interval is chosen to clear the
  *typical* stale-handle reap window measured in §5 (A3).
- **The sleep only avoids 2042; the retry *handles* it — and the retry is
  required, not defensive.** The queue-manager reap of a stale exclusive handle is
  **non-deterministic**: a fixed sleep can only reduce how often a restart hits
  `MQRC_OBJECT_IN_USE` (2042), never guarantee avoidance — a slow or wedged QM can
  push reap past any fixed margin. So the loop must **handle** a 2042 open failure,
  not merely try to sidestep it. A 2042 exit is just another non-zero exit the loop
  retries, so no bespoke 2042 code path is needed — the generic loop *is* the
  handling.
- **Bounded open-retry, then a loud failure (open question — the bound value).** A
  2042 is *either* a stale handle that will eventually reap *or* a queue genuinely
  held by a live exclusive owner that never will; the wrapper cannot tell them
  apart at failure time. So the open-retry is bounded (a long window — order ~30 min,
  value resolved at build from A3) after which the wrapper **exits non-zero** so the
  SERVICE goes down visibly rather than retrying into the void — fail-loud, caught
  by monitoring. Whether the bound is finite or infinite is an open question (§12).
- **Lifecycle logging to the existing journald tag, via an explicit `logger`.** Each
  iteration logs its state — start, `amqsevt exited rc=N`, `open failed rc=2042,
  retrying`, `restarting in 10s` — under the same `mq_event_syslog_tag`, beside the
  events. These lines use a direct `logger -t <tag>` call, **not** `amqsevt`'s
  stdout pipe (which is dead exactly when the wrapper needs to speak).
- **The restart heartbeat is the monitoring signal, by design.** A persistent
  crash-loop emits a steady "restarting" beat every ~10 s; catching that is the job
  of the observability stack, not of cleverness in the loop. If a truly poison event
  makes `amqsevt` crash on every restart, the loop keeps the collector alive for all
  *other* events and makes the failure loud and continuous rather than silent.
- **Poison messages are out of scope.** Because `amqsevt` is non-transactional, a
  message that crashes the parse is simply lost — an accepted corner case, to be
  confirmed by reading the source (§5, A1), not solved here. Making event capture
  transactional (and thus needing poison-message handling) is a different problem.

## 5. Research to answer — read → understand → prove

Discipline: for anything with published IBM guidance, **read and understand it
first, then prove it in the lab** — do not experiment blind where documentation
exists.

- **A0 — MQ's spawn topology (do this first; it shapes the `setsid` wiring).**
  Inspect the currently-running collector left by `#114` on a live arm QM —
  `ps -o pid,ppid,pgid,sid,cmd` on the `amqsevt`/`run.sh` process and its ancestry —
  to answer the one question `setsid`'s behaviour turns on: **does MQ start the
  SERVICE `STARTCMD` as a process-group/session leader, or not?** Not-a-leader ⇒
  `setsid` execs in place and the clean `MQ_SERVER_PID == PGID` identity holds
  (expected); already-a-leader ⇒ `setsid` is forced to fork and we take the §3.4
  fallback. Also confirm `setsid` (and its `--`/`-w` handling) on the actual lab
  RHEL 9 / Ubuntu util-linux builds, since `--` was verified only on the dev VM's
  2.39.3.
- **A1 — Read `amqsevt.c`.** Pull the sample source off a lab box into the **shared
  build tree** (the re-fetchable side, alongside the `ibm-docs` refs; located via
  `mqlab build path`, never a hardcoded `build/<X>` path) so it is easy to browse.
  Determine: how it opens the event queues (shared vs exclusive input), its own
  signal handling, its exit codes, whether it closes handles on exit, and whether it
  logs or silently drops an unparsable message. Exclusive input is the *expected*
  and *desirable* finding — event ordering matters, and a shared-input drainer that
  let a second reader interleave (or emit out-of-sequence crash/restart events)
  would be genuinely ugly, with timestamp metadata not always able to rescue the
  sort. A1's result **branches B3** (§6).
- **A2 — Exit & queue-handle behaviour under signals (incl. the checkpoint).** For
  `SIGTERM(15)`, `SIGINT(2)`, `SIGHUP(1)`, `SIGKILL(9)`, and the **real SERVICE
  `STOPCMD` path**: does `amqsevt` exit cleanly, and are the event-queue handles
  closed cleanly or left open? A non-clean exit (notably `SIGKILL`) is the natural
  generator of the 2042 scenario in A3/B3. This step also runs the **named
  pre-flight checkpoint** from §3.1: assert `MQ_SERVER_PID == PGID` and that the QM
  tracks the live leader (not an orphan) before the wrapper mechanism is declared
  viable.
- **A3 — Stale-handle reap timing (`MQRC_OBJECT_IN_USE` / 2042).** How long the
  queue manager takes to reap a stale exclusive-input handle after the holder is
  `kill -9`'d, so a fresh open can succeed. **Literature review first** (IBM docs +
  blogs — this behaviour is commonly discussed), then a measured experiment. The
  result informs the *typical* reap window (the ~10 s sleep in §4) — but note the
  reap is **non-deterministic**, so this characterises the typical case, it does not
  bound the worst case (hence the required retry, §4).

## 6. Validation matrix — the report's proof set

Each item is demonstrated live on a lab queue manager and its evidence captured:

- **B1 — Normal start.** SERVICE starts; wrapper **and** `amqsevt` are both running;
  events reach journald.
- **B2 — Clean stop via `STOPCMD` (carries the checkpoint).** Both the wrapper and
  `amqsevt` die; **no orphaned** process; **no restart**; the SERVICE shows
  `STOPPED`. This is the regression the current `exec` model guarantees and the new
  design must not lose. Records the §3.1 checkpoint evidence — `MQ_SERVER_PID` vs the
  wrapper PGID, and that the group-kill reaped both processes.
- **B3 — Crash recovery, and 2042 recovery (branches on A1).** `kill -9 amqsevt`;
  the wrapper logs the exit, sleeps, and restarts it; events resume — captured as
  **real log-file evidence** of the self-healing. The 2042 arm branches on A1's
  open-mode finding:
  - *Exclusive input (expected):* the harness **deliberately forces the collision** —
    restart `amqsevt` faster than the measured reap window, or have the harness hold
    the handle open — to produce real `rc=2042`-then-recovery log evidence,
    decoupling the demonstration from the production sleep value.
  - *Shared input:* 2042 cannot arise against `amqsevt`; the report **documents it as
    non-applicable**, records the retry as general defensive robustness, and still
    reports the A3 reap-timing characterisation.

**Failover is out of scope.** Epic `#114` already proved a `CONTROL(QMGR)` SERVICE
travels with the QM across failover; re-testing it here would only re-test startup
unless the entire B-matrix were re-run post-failover, which is redundant.

## 7. Deliverables & where they live

1. **Resilient wrapper.** Rewritten `run.sh.j2` (loop + `setsid`) and the SERVICE
   definition change (`STOPCMD`/`STOPARG` to the process-group form) in the
   `mq-event-monitor` role, plus any new role variables (e.g. sleep interval).
2. **Internal engineering report** under
   `mq-resiliency-lab-for-linux/docs/reports/YYYY-MM-DD-*.md`, with an `assets/`
   evidence directory — matching the existing 2026-07-20 report pattern. It records
   the A-research findings (including the `amqsevt.c` reading and the 2042 timing)
   and the B-matrix evidence.
3. **Validation runbook + script** — a clear, repeatable harness an AI lab agent can
   run to reproduce the A/B matrix and gather evidence. A runnable procedure with
   captured output, **not** mechanized into CI.

## 8. Component boundaries & isolation

- **`run.sh` wrapper** — *what:* supervise one `amqsevt` process, restart it on
  crash, die cleanly on a group stop. *Interface:* invoked by the SERVICE `STARTCMD`
  with the QM name as `STARTARG`; emits events + lifecycle lines to journald.
  *Depends on:* `setsid`, `amqsevt`, `logger`, the journald drop-in — all already
  present on every arm node. No consumer reads its internals.
- **SERVICE definition (`service.yml`)** — *what:* define/start the collector with a
  process-group-aware `STOPCMD`. Its only change is the stop form; the enable/verify
  logic is unchanged.
- **Validation harness** — *what:* induce each A/B condition and capture evidence.
  *Interface:* a documented sequence of commands runnable on a live lab QM; reads the
  QM/service state and journald, writes evidence files. Depends on a deployed
  resilient wrapper. It **observes**; it does not modify the role.

## 9. Implementation tasks (preview; finalized in the plan)

- **T1 — Characterization research (A0–A3).** Inspect MQ's spawn topology (A0)
  first, read `amqsevt.c` (into the shared build tree), run the signal/exit
  experiments + the §3.1 checkpoint, do the 2042 lit-review + timing experiment.
  Output: the research findings section of the report; confirms (or refutes) the
  `setsid` mechanism and informs the ~10 s sleep. Gates the wrapper design.
- **T2 — Resilient wrapper.** Rewrite `run.sh.j2` (loop + `setsid`) and the SERVICE
  `STOPCMD`/`STOPARG`; add role variables. Acceptance: `vrg-validate`; wrapper +
  `amqsevt` come up and clean-stop on a QM.
- **T3 — Validation runbook + harness.** The repeatable B-matrix procedure/script an
  AI lab agent can run, plus the report scaffold.
- **T4 — Live-lab validation run (operational `validation` task).** Deploy the
  wrapper to a lab QM and run the A/B harness on the live lab, capturing evidence;
  record `Outcome: SUCCESS` and land the completed report. Gates epic closure.

A **`deployment`** operational task may precede T4 if the plan needs the wrapper
explicitly *deployed and usable* on a lab QM before the validation run. Ordering:
T1 → T2 (T1's findings confirm the design) → T3 → T4. Both operational tasks and
their `Blocked-by` links are seeded from the plan (`epic-create` step 9).

## 10. Acceptance criteria

- The event-monitor SERVICE **survives an `amqsevt` crash**: after `kill -9
  amqsevt`, the wrapper restarts it and events resume, with captured log evidence
  (B3). Where A1 confirms exclusive-open, this includes forced recovery through a
  2042 stale-handle window; where A1 finds shared-open, 2042 is documented as
  non-applicable and the reap timing (A3) is reported instead.
- A QM `STOPCMD` **cleanly stops the whole collector**: both wrapper and `amqsevt`
  terminate, no orphan, SERVICE `STOPPED` (B2) — no regression from the `exec` model.
- Normal start brings up wrapper + `amqsevt` with events reaching journald (B1),
  proven on a **cold-rebuilt** stack (one-pass provisioning of the new wrapper).
- The `setsid -w --` mechanism is **proven, not assumed**: the report records the
  A0 spawn-topology finding and the A2/B2 checkpoint evidence (`MQ_SERVER_PID == PGID`,
  group-kill reaped both), or documents the escalation to the §3.4 Python fallback.
- The internal report exists with the A-research findings (MQ spawn topology, the
  `amqsevt.c` reading, and the measured 2042 reap timing) and the B-matrix evidence,
  and a repeatable validation runbook accompanies it.
- `amqsevt.c` is present in the shared build tree for browsing.
- `vrg-validate` passes.

## 11. Out of scope

- **Failover survival** — already proven by `#114` (see §6).
- **Poison-message / transactional event capture** — `amqsevt` is non-transactional
  by design; lost-on-crash messages are an accepted corner case (§4).
- **Mechanizing the validation into CI** — the deliverable is a runnable, agent-
  operable procedure with captured evidence, not an automated gate.
- **The Python fallback wrapper** — documented as the escalation path (§3.4); built
  only if the research forces it, in which case it is its own follow-on.
- **Any change to event shipping, storage, or visualization** — that boundary stays
  with `#31`/`#8`.

## 12. Open questions (resolve at build time)

- **Does `setsid -w -- run.sh` + negative-PID `STOPARG` work in the MQ-SERVICE launch
  context?** The core mechanism assumption, and it turns on MQ's spawn topology (A0):
  if MQ starts the STARTCMD as a group leader, `setsid` forks and `MQ_SERVER_PID`
  diverges from the group leader. Confirmed or refuted by the A0 inspection + the
  A2/B2 checkpoint; the §3.4 Python fallback is the contingency. Whether
  `MQ_SERVER_PID` substitutes correctly inside a `-TERM -- -+MQ_SERVER_PID+` `STOPARG`
  is part of this. **Resolved (build time):** A0 shows MQ does *not* spawn the STARTCMD
  as a group leader, so `setsid` execs in place and `MQ_SERVER_PID == PGID` holds; MQ
  substitutes the insert only when delimited as `+MQ_SERVER_PID+` (the bare token is
  passed literally), and `--` is required for procps-ng `/bin/kill`. Proven live — full
  B-matrix PASS on nativeha-ubuntu (T7), no Python fallback needed.
- **Open-retry bound: finite (~30 min) or infinite?** The 2042/open-failure retry
  needs a policy: retry a long bounded window then exit loud (surfaces a genuinely
  held queue), or retry indefinitely (relies purely on monitoring). Resolve at build
  from the A3 findings; the value (~30 min) is a starting point, not a decision.
- **Logger pipeline across restarts.** The current wrapper pipes `amqsevt` to a
  `logger` via process substitution. Decide whether the loop re-establishes the
  `logger` per iteration or holds one `logger` for the wrapper's life; validate no
  events are dropped or duplicated across a restart. (Distinct from the wrapper's own
  lifecycle lines, which use a direct `logger -t` call per §4.)
- **`amqsevt` event-queue open mode.** Whether it opens the event queues for
  exclusive input determines whether 2042 can arise at all and branches B3; resolved
  by A1. Exclusive is expected (event ordering).
- **Which lab QM hosts the experiments**, and whether the run rides a dedicated
  cold rebuild or an existing stack. Resolve in the plan.
