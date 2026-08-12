# File-sink variant of the resilient MQ event collector — design spec

**Epic:** `logical-minds-foundry/.github#152`
**Status:** draft (spec)
**Family:** event monitoring (`#31` enable/collect, `#122` resilient wrapper,
`#110`/`#114` reference & rollout)
**Role touched:** `ansible/roles/mq-event-monitor`
**Validation arm:** Native HA RHEL (`nha-rhel-*`) — samples/`amqsevt` present, and
where the resilient wrapper was originally validated (`#760`/`#122`).

---

## 1. Problem & motivation

The two 2026-07-28 event-monitoring reports —
[`2026-07-28-mq-event-monitor-resilient-service.md`](../../../../mq-resiliency-lab-for-linux/docs/reports/2026-07-28-mq-event-monitor-resilient-service.md)
and its companion
[`2026-07-28-mq-event-monitoring-to-file.md`](../../../../mq-resiliency-lab-for-linux/docs/reports/2026-07-28-mq-event-monitoring-to-file.md) —
ship a **fully tested, evidence-backed** resilient collector for the
**syslog/journald** sink. Both then relegate the **file** sink — the variant a
real consumer actually deploys — to a footnote:

> *"If you need a file instead, `run.sh` is a one-line change — redirect
> `amqsevt`'s stdout to a file rather than piping it through `logger`."*
> (resilient-service report, "Choosing the sink")

This inverts the point of the testing. Handing someone an aggressively tested
script and then telling them to hand-edit one line to produce the variant they
run means **the delivered artifact is no longer the tested artifact**. It is the
software-engineering equivalent of signing off a fully security-reviewed document
and then saying "just change these two fields before you use it." The change may
be small, but the moment it is made by hand at the point of use, every claim the
testing earned no longer applies to what is actually running.

The lab already implements the resilient wrapper as the `mq-event-monitor` role,
but only for the **syslog** sink (`run.sh.j2` pipes `amqsevt` to `logger`; the
`mq-events` journald tag feeds the lab's journald→Alloy pipeline). The file sink
exists nowhere as a tested artifact — only as that footnote.

## 2. Goals & non-goals

### Goals

1. Make the **file** sink a first-class, **tested** path in the `mq-event-monitor`
   role, selected by a role variable.
2. Keep the **selection logic in the management layer (Ansible)**: the sink is
   chosen at **template-render time**, so the script that lands on the box — and
   that the report quotes verbatim — is **single-purpose**, with no runtime
   sink-branching and no `logger` dependency. The delivered file script does
   exactly one thing: append events to a file, resiliently.
3. Prove the file variant on the live lab against the **two resilience corner
   cases** the wrapper was designed to handle, plus verify the file output.
4. Publish a report documenting the **exact tested script + service definition +
   install steps + captured evidence** — install as written, no one-line edits —
   and retire the "one-line change" footnote in the two 2026-07-28 reports.

**Non-goals** (documented as the consumer's responsibility, exactly as today's
file report already frames them — not implemented or tested here):

- File **rotation** (must be copy-truncate; the collector holds the file open with
  no reopen-on-signal).
- **Checkpointed forwarding** of the `.json` file by a downstream agent.
- **Lossless / transactional** delivery — `amqsevt` consumes destructively and
  non-transactionally; that caveat remains a caveat.
- Changing the **syslog** path or the journald→Alloy pipeline in any way. Syslog
  stays the lab default.

## 3. Design

### 3.1 Selectable sink — the selection lives in Ansible

Add a role variable `mq_event_sink` with values `syslog` (default) and `file`.
`run.sh.j2` gains a **render-time** Jinja branch (`{% if mq_event_sink == 'file'
%} … {% else %} … {% endif %}`) that wraps **only the three lines that differ
between the two sinks**:

- the definition of `say()` (its output destination — `logger` vs. stderr),
- the `amqsevt` invocation's **stdout** sink (the `logger` process substitution
  vs. `>> "$DATA_FILE"`), and
- the file variant's post-run `cat "$ERRLOG" >&2` flush (see 3.2) — absent in the
  syslog variant.

Everything else — the setsid group-leader model, the `while` restart loop,
`HEALTHY_RUN`, the `OPEN_RETRY` bounded give-up, **the per-run temp `ERRLOG`
capture and its `head -1` root-cause reason-code grep**, and all lifecycle
logging structure — is **shared, single-source**. This is the
critical property: the resilience core (especially the 2042 fast-restart logic)
**cannot drift** between two copies, because there is only one copy.

Because Jinja resolves at template time, the **rendered file** contains none of
the branch: on a `file`-sink host, `run.sh` has no `logger`, no syslog tag, and no
`if sink == …` — it is a clean, single-purpose file writer. (Alternative
considered and rejected: two separate full template files. That duplicates the
resilience core and invites exactly the drift this design prevents.)

**Regression guard (acceptance criterion on the impl task).** The refactor edits
the template that also renders the **syslog** script currently deployed on every
arm and feeding the journald→Alloy pipeline. The impl is therefore not done until
the **syslog-rendered `run.sh` is byte-for-byte identical** before and after the
change: render the template with `mq_event_sink: syslog` on the pre-change and
post-change template and `diff` them — the diff must be empty. This lets the
shared core be refactored fearlessly without disturbing the shipping path.

### 3.2 The rendered file script — behavioural contract

The `file`-variant `run.sh` (what the report quotes verbatim):

- **Events → `.json` data file.** `amqsevt … -o json_compact` stdout is appended
  in-script to `$DATA_FILE` (`>> "$DATA_FILE"`). The wrapper owns `amqsevt`'s
  stdout, so it redirects it directly; `-o json_compact` keeps one JSON object per
  line (JSONL) for a line-oriented consumer.
- **Diagnostics → `.error` file.** `say()` writes to **stderr**; the MQ `SERVICE`
  redirects the wrapper's `STDOUT`/`STDERR` to `$ERROR_FILE` (see 3.3), so wrapper
  lifecycle lines land in `.error`. `amqsevt`'s own stderr is also captured to
  `.error`, so the MQ reason code on a failed run (e.g. `…MQRC_OBJECT_IN_USE
  [2042]`) is visible there.
- **Reason enrichment is retained, using the *same* logic as the syslog variant.**
  `amqsevt` stderr is captured to the **per-run temp `ERRLOG`** (overwritten each
  loop), and the reason is extracted with the identical `grep … | head -1`
  root-cause pass — so the extraction grabs the root `MQOPEN … 2042` before the
  shutdown-path `MQCLOSE` noise, and the designed **fail-twice-then-succeed 2042
  heartbeat** reads byte-for-byte the same as syslog. After each run the file
  variant additionally flushes that run's raw diagnostics into `.error` with
  `cat "$ERRLOG" >&2` (the wrapper's stderr is the service's `.error` redirect),
  so `.error` carries both the wrapper lifecycle (`say`) and the raw `amqsevt`
  diagnostics for an operator, while the reason logic stays shared and
  deterministic. The temp `ERRLOG` remains an implementation detail of the loop,
  not a delivered file.
- **No `logger`, no syslog tag, no `--size`.** Those belong only to the syslog
  variant. The file script is dependency-free beyond `bash` + `util-linux` /
  `procps-ng` + the MQ sample.

The **stop semantics are unchanged and sink-independent**: the script is still
launched under `setsid` as its own process-group leader, and the `SERVICE`
STOPCMD group-kill reaps the wrapper and `amqsevt` together. No orphan on stop;
self-restart on crash.

### 3.3 Sink-aware SERVICE definition (`tasks/service.yml`)

The `DEFINE SERVICE` is rendered per sink:

- **`syslog` (unchanged):** `STARTCMD('/usr/bin/setsid') STARTARG('-w --
  …/run.sh <QM>')`, no `STDOUT`/`STDERR` (events go to journald via `logger`
  inside the script), `DESCR` naming the journald sink.
- **`file`:** same `STARTCMD`/`STARTARG` and the same group-kill `STOPCMD`/
  `STOPARG` (`-TERM -- -+MQ_SERVER_PID+`), **plus** `STDOUT('$ERROR_FILE')
  STDERR('$ERROR_FILE')` so the wrapper's lifecycle and `amqsevt`'s diagnostics
  are redirected to `.error`, while events reach `.json` via the in-script
  redirect. `DESCR` names the file sink.

The `+MQ_SERVER_PID+` insert is still the `setsid`/`run.sh` group-leader pid, so
the negative-pid group kill is identical across sinks.

### 3.4 Variables (`defaults/main.yml`)

- `mq_event_sink: syslog` — sink selector.
- `mq_event_file_dir: /var/mqm/event-monitor` — directory for the file-sink
  outputs (created mqm-owned by host-prep when `sink == file`).
- `mq_event_data_file: "{{ mq_event_file_dir }}/{{ qmgr_name }}.events.json"`
- `mq_event_error_file: "{{ mq_event_file_dir }}/{{ qmgr_name }}.error"`

`mq_event_syslog_tag` and `mq_event_logger_max_size` remain, used only when
`sink == syslog`. Host-prep (`tasks/main.yml`) creates the file-sink directory
(mqm-owned) only in the `file` case; the `amqsevt`-present and journald-drop-in
asserts are re-scoped so the journald-rate-limit assert applies to the syslog
sink (it is the sink that can be rate-dropped), not the file sink.

## 4. Validation

Provision the **Native HA RHEL arm** with `mq_event_sink: file` and run, on the
live queue manager:

1. **File output.** Force representative events (per the existing event-generation
   reference) and confirm: each line in `$DATA_FILE` parses as standalone JSON
   (JSONL); the `SYSTEM.ADMIN.*.EVENT` queue depth rises then drains to zero
   (destructive drain, not browse); `$ERROR_FILE` carries wrapper lifecycle lines
   and is otherwise a diagnostic catch (normally quiet).
2. **Corner case 1 — clean stop, no orphan.** `STOP SERVICE(MQ.EVENT.MONITOR)`;
   assert no surviving `run.sh` or `amqsevt` under `mqm` (`AMQ8732I`, group kill
   reaped both).
3. **Corner case 2 — crash recovery through the 2042 window.** `kill -9` the
   `amqsevt` child; watch the wrapper fast-fail on 2042 a couple of times, then
   land a healthy run once the queue manager reaps the stale exclusive handle;
   `DISPLAY SVSTATUS` reads `RUNNING` again and the `.json` feed resumes. The
   reason annotation (`…2042…`) is visible in `.error`.

The existing `tools/validate-event-monitor-wrapper.sh` /
`docs/reference/event-monitor-wrapper-validation.md` are **parameterized by
sink** — one harness runs the same clean-stop + 2042-recovery assertions against
either sink (reading `.json`/`.error` for `file`, journald for `syslog`), mirroring
the sink-selectable role rather than duplicating the scaffolding in a sibling
script. Evidence is captured as report assets under `docs/reports/assets/`.

The file-variant validation closes on `Outcome: SUCCESS` only when **all four**
asserts pass:

1. **JSONL well-formed** — sampled lines of `$DATA_FILE` each parse as standalone
   JSON.
2. **Destructive drain** — a forced event's `SYSTEM.ADMIN.*.EVENT` queue depth
   rises then drains to **0** (drain, not browse).
3. **Clean stop, no orphan** — after `STOP SERVICE`, no `run.sh` or `amqsevt`
   survives under `mqm` (`AMQ8732I`).
4. **2042 crash recovery** — after `kill -9` of the `amqsevt` child, `SVSTATUS`
   returns to `RUNNING` and `$DATA_FILE` resumes, with the root-cause `…2042…`
   reason visible in `.error`.

This is a **live-lab `validation` task** (the corner cases are induced conditions
with asserted observable outcomes), gated on the file variant being **deployed**
onto the arm — hence the `impl → deployment → validation` ordering below.

## 5. The report

New report `docs/reports/2026-07-29-mq-event-monitor-file-sink-resilient.md`:

- The **exact rendered file-sink `run.sh`** and the **exact `DEFINE SERVICE`**,
  quoted as installed — no "change this line" caveats.
- Install steps (samples present, enable classes, drop the script, define+start
  the service), a Quick start that is the whole setup.
- The captured **test evidence** for the two corner cases + file-output check,
  with the same validation-provenance framing as the sibling reports.
- The retained non-goals (rotation, forwarding, lossless) as the consumer's
  follow-on requirements — carried over verbatim in intent from the existing file
  report, since they are unchanged.
- **The HA-failover host-local stranding window, stated explicitly** (required
  caveat). The `.json`/`.error` files are **host-local**; on failover any events
  written on the old node but not yet forwarded are **stranded** — the new node
  starts a fresh file. Keeping the downstream forwarder current bounds the window,
  but it cannot close it. This is a file-sink–specific window that the syslog sink
  does **not** have (syslog inherits the platform's forwarding), and it is one of
  the concrete reasons **syslog is the lab default and standing recommendation** —
  stated plainly so a reader deploying on a Native HA arm (the arm we validate on)
  is not surprised by it.

The two 2026-07-28 reports are amended: the "`run.sh` is a one-line change"
footnote is **replaced** with a pointer to this report's tested file variant. No
other content in those reports changes.

## 6. Task breakdown (filed from the plan)

Ordering: **impl → deployment → validation → report**, under epic `#152`.

1. **Impl (member repo).** Role change: `mq_event_sink` selector + file-path vars;
   render-time Jinja branch in `run.sh.j2`; sink-aware `service.yml`; host-prep
   directory creation and re-scoped asserts. Closed by a same-repo PR to
   `develop`.
2. **Deployment (member repo, `--kind deployment`, blocked-by impl).** Provision
   the Native HA RHEL arm with `mq_event_sink: file` so the variant is deployed
   and usable for validation.
3. **Validation (member repo, `--kind validation`, blocked-by deployment).** The
   two corner cases + file-output verification on the live arm; closes on
   `Outcome: SUCCESS`.
4. **Report (member repo, blocked-by validation).** The new report + footnote
   fixes, carrying the captured evidence. Closed by a same-repo PR.

**Bookends (already seeded):** documentation task `.github#153` (this spec +
plan), documentation-review task `mq-resiliency-lab-for-linux#825`, retrospective
task `.github#154`.

## 7. References

- Lab role: `ansible/roles/mq-event-monitor/{templates/run.sh.j2,tasks/service.yml,tasks/main.yml,defaults/main.yml}`
- Reports: `docs/reports/2026-07-28-mq-event-monitor-resilient-service.md`,
  `docs/reports/2026-07-28-mq-event-monitoring-to-file.md`,
  `docs/reports/2026-07-20-mq-service-stdout-open-mode-evidence.md`
- Harness: `tools/validate-event-monitor-wrapper.sh`,
  `docs/reference/event-monitor-wrapper-validation.md`
- Prior art: `#31` (event monitoring), `#122` (resilient wrapper), `#514`/`#515`
  (enable + SERVICE define), `#760` (setsid / 2042 evidence), `#784`/`#786` (stop
  semantics).
- IBM MQ 9.4: *amqsevt*, *DEFINE SERVICE*, *Replaceable inserts on service
  definitions*, *2042 (RC2042) MQRC_OBJECT_IN_USE*.
