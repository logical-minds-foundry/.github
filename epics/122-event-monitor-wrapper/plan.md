# Resilient `amqsevt` Event-Monitor Wrapper — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the MQ event-monitor SERVICE self-heal across `amqsevt` crashes without losing the clean-stop guarantee, and prove both behaviours with captured lab evidence in an internal report.

**Architecture:** Replace the current `exec amqsevt` launcher (`ansible/roles/mq-event-monitor/templates/run.sh.j2:29`) with a minimal bash supervision loop launched under `setsid -w --` so the wrapper is its own process-group leader; change the SERVICE stop to a process-group kill (`STOPARG('-TERM -MQ_SERVER_PID')`) so a QM stop reaps both the wrapper and `amqsevt`, while an `amqsevt` crash leaves the wrapper alive to log-sleep-restart. A research phase first characterises MQ's spawn topology, the `amqsevt` source, and the queue-manager stale-handle reap window, so the mechanism is proven rather than assumed; a Python fallback is documented if the process-group identity does not hold.

**Tech Stack:** Bash + util-linux `setsid`, IBM MQ 9.4 (`amqsevt` SERVICE, `runmqsc` `DISPLAY QSTATUS TYPE(HANDLE)`), systemd journald + `logger -t mq-events`, Ansible for provisioning, the existing alloy→Loki→Grafana pipeline. Validation: `vrg-container-run -- vrg-validate` + live-lab evidence capture.

## Global Constraints

- **Validation is one command only:** `vrg-container-run -- vrg-validate`. Do not run individual linters/formatters. (repo CLAUDE.md)
- **Git/GitHub via wrappers:** `vrg-git`, `vrg-gh`, commit with `vrg-commit --type <t> --scope <s> --message <m>`. Raw `git`/`gh` are denied. No direct commits to `develop`/`main`; work on `feature/<issue>-<slug>`.
- **Implementation repo:** `logical-minds-foundry/mq-resiliency-lab-for-linux`. All paths below are relative to it.
- **Build-tree paths via `mqlab build path`** — never hardcode a `build/<X>` path. `amqsevt.c` lands in the shared, re-fetchable side (beside the `ibm-docs` refs).
- **The wrapper stays in the bash envelope** — a short loop using only process-group semantics; **no in-script `trap`/signal handling**. Escalate to the stdlib-only Python fallback *only* if the A0/B2 checkpoint (`MQ_SERVER_PID == PGID`) fails.
- **Committed launch form:** `setsid -w -- {{ mq_event_run_dir }}/run.sh <QM>`. `--` stops `setsid` arg-parsing; `-w` keeps the QM-tracked process alive if `setsid` is forced to fork.
- **Fixed ~10 s sleep, infinite retry on crash; bounded open-retry then loud non-zero exit.** The retry is required (reap is non-deterministic), not defensive. Wrapper lifecycle lines go to journald via an explicit `logger -t {{ mq_event_syslog_tag }}` call, not the `amqsevt` stdout pipe.
- **Do not lose the clean-stop regression** — a QM `STOPCMD` must reap both wrapper and `amqsevt`, no orphan (the one virtue of today's `exec` model).
- **Worked-example target: the standalone `SVCQM` on the `svc` host** (single host, already running, no failover complexity) — any single arm QM works; the choice is confirmed at run time.

---

### Task 1: Characterization research (A0–A3) → findings report

**No code change. Deliverable:** the research-findings sections of the internal report at `docs/reports/2026-07-21-mq-event-monitor-wrapper-resilience.md` (use the actual run date), plus `amqsevt.c` in the shared build tree. This task **gates T2**: A0 decides `setsid`-viable vs Python-fallback, A1's open mode branches the 2042 work, A3 informs the sleep value. Every finding is recorded verbatim (command + output) as evidence.

**Files:**
- Create: `docs/reports/2026-07-21-mq-event-monitor-wrapper-resilience.md` (research sections; B-matrix sections filled in Task 4)
- Create: `docs/reports/assets/mq-event-monitor-wrapper/` (evidence: `ps` captures, `runmqsc` handle dumps, reap-timing log)
- Fetch (not committed): `amqsevt.c` into `$(mqlab build path …)` shared tree

**Interfaces:**
- Produces (consumed by T2/T4): **A0 decision** — `setsid` execs-in-place (mechanism viable) vs forced-fork (fallback); **A1 open mode** — exclusive vs shared input on the event queues; **A3 reap window** — typical and observed-max seconds for a stale exclusive handle to clear.
- Consumes: the running `#114` collector on `SVCQM` (`svc` host).

- [ ] **Step 1: A0 — capture MQ's spawn topology of the running collector**

On the live QM host, dump the process table for the current `run.sh`/`amqsevt` and its ancestry, and compare PID/PGID/SID:

```bash
cd ansible
uv run --project .. ansible svc -b -m shell -a "ps -eo pid,ppid,pgid,sid,comm | grep -E 'amqsevt|run.sh|amqzsc|runmqsc' ; echo '---' ; pgrep -a amqsevt"
```

Record whether the `run.sh`/`amqsevt` process is a **process-group leader** (`PGID == PID`) or a **session leader** (`SID == PID`). Save the raw output to `docs/reports/assets/mq-event-monitor-wrapper/a0-topology.txt`.

- [ ] **Step 2: A0 — decide the mechanism, and confirm `setsid --`/`-w` on the lab OS**

```bash
uv run --project .. ansible svc -b -m shell -a "setsid --version; setsid -w -- /bin/echo setsid-double-dash-ok"
```

Expected: version prints; `setsid-double-dash-ok` prints (proving `--` on the lab util-linux). **Decision rule:** if A0 Step 1 shows the STARTCMD process is *not* already a group leader → `setsid` will exec in place → `MQ_SERVER_PID == PGID` holds → **mechanism viable, proceed to T2**. If it *is* already a group leader → `setsid` forks → record this and flag the **Python fallback** (§3.4 of the spec) for T2. Write the decision into the report's "A0 — spawn topology" section.

- [ ] **Step 3: A1 — bring `amqsevt.c` into the shared build tree and read it**

```bash
# land amqsevt.c in the shared, re-fetchable bucket (beside the ibm-docs refs); no hardcoded build/<X>
DEST="$(uv run mqlab build path cache)/amqsevt"
mkdir -p "$DEST"
cd ansible
uv run --project .. ansible svc -b -m fetch -a "src=/opt/mqm/samp/amqsevt.c dest=$DEST/ flat=yes"
ls -l "$DEST/amqsevt.c"
```

Read `amqsevt.c` and record, in the report's "A1 — amqsevt source" section: the `MQOPEN` options on the event queues (look for `MQOO_INPUT_EXCLUSIVE` vs `MQOO_INPUT_SHARED` / `MQOO_INPUT_AS_Q_DEF`), any `signal()`/`sigaction()` handlers, the `exit()` codes, whether it `MQCLOSE`s handles before exit, and how it treats an unparsable message (log-and-continue vs abort). **Branch:** exclusive input ⇒ 2042 is reachable (B3 exclusive arm); shared input ⇒ record 2042 as non-applicable.

- [ ] **Step 4: A1 — corroborate the open mode from the live handle**

Cross-check the source reading against the running collector's actual open options:

```bash
cd ansible
uv run --project .. ansible svc -b -m shell -a "su - mqm -c 'echo \"DISPLAY QSTATUS(SYSTEM.ADMIN.QMGR.EVENT) TYPE(HANDLE) OPENOPTS APPLTAG\" | runmqsc SVCQM'"
```

Expected: a handle held by `amqsevt` with `OPENOPTS(...)` listing `MQOO_INPUT_EXCLUSIVE` or `MQOO_INPUT_SHARED`. Save to `assets/mq-event-monitor-wrapper/a1-openopts.txt`. Confirm it agrees with the source. Record the confirmed open mode as the A1 decision.

- [ ] **Step 5: A2 — signal/exit behaviour of `amqsevt` (pre-wrapper)**

Run `amqsevt` directly (outside the SERVICE) so signals can be sent cleanly, and observe exit code + whether the event-queue handle is left open after each signal. For each of `SIGINT(2)`, `SIGTERM(15)`, `SIGHUP(1)`, `SIGKILL(9)`:

```bash
cd ansible
uv run --project .. ansible svc -b -m shell -a "su - mqm -c '
  /opt/mqm/samp/bin/amqsevt -m SVCQM -o json_compact >/tmp/ev.out 2>&1 & P=\$!; sleep 3;
  echo \"handle-before:\"; echo \"DISPLAY QSTATUS(SYSTEM.ADMIN.QMGR.EVENT) TYPE(HANDLE) APPLTAG\" | runmqsc SVCQM | grep -c APPLTAG;
  kill -TERM \$P; sleep 2; echo \"exit=\$?\";
  echo \"handle-after:\"; echo \"DISPLAY QSTATUS(SYSTEM.ADMIN.QMGR.EVENT) TYPE(HANDLE) APPLTAG\" | runmqsc SVCQM | grep -c APPLTAG'"
```

Repeat with `kill -INT`, `kill -HUP`, `kill -KILL`. Record per signal: does `amqsevt` exit cleanly, and is the handle count 0 (closed) or >0 (left open/stale) immediately after. `SIGKILL` is expected to leave a stale handle — the A3 input. Save each run to `assets/mq-event-monitor-wrapper/a2-<signal>.txt`.

- [ ] **Step 6: A3 — literature review of the 2042 reap window (read first)**

Using the IBM-docs cache tool, capture the canonical guidance on `MQRC_OBJECT_IN_USE` (2042) and stale-handle / connection cleanup timing:

```bash
python3 tools/ibm_doc_cache.py "https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=codes-2042-07fa-rc2042-mqrc-object-in-use"
```

Read it (and any linked connection-cleanup / `DISCINT` / channel-reap material). Summarise in the report's "A3 — reap timing (literature)" subsection what IBM says governs when a dead connection's exclusive handle is released, and cite `content.txt` with the `source_url` from `meta.json`. This frames the experiment; do not experiment blind.

- [ ] **Step 7: A3 — measure the reap window experimentally**

Force a stale exclusive handle and poll until the queue manager releases it:

```bash
cd ansible
uv run --project .. ansible svc -b -m shell -a "su - mqm -c '
  /opt/mqm/samp/bin/amqsevt -m SVCQM -o json_compact >/dev/null 2>&1 & P=\$!; sleep 3;
  kill -9 \$P; T0=\$(date +%s);
  while :; do
    N=\$(echo \"DISPLAY QSTATUS(SYSTEM.ADMIN.QMGR.EVENT) TYPE(HANDLE) APPLTAG\" | runmqsc SVCQM | grep -c APPLTAG);
    echo \"\$(( \$(date +%s) - T0 ))s handles=\$N\";
    [ \"\$N\" -eq 0 ] && break; sleep 2;
  done'"
```

Record the elapsed seconds until `handles=0` (the reap time). Repeat 3–5× to characterise typical vs observed-max, and note it is **non-deterministic** (the justification for the required retry, not just the sleep). Save to `assets/mq-event-monitor-wrapper/a3-reap-timing.txt`. This value informs T2's `mq_event_sleep_secs` default (choose ~typical-reap, and the open-retry bound in Task 2).

- [ ] **Step 8: Commit the research findings + report ready**

```bash
vrg-git add docs/reports/2026-07-21-mq-event-monitor-wrapper-resilience.md docs/reports/assets/mq-event-monitor-wrapper
vrg-commit --type docs --scope events \
  --message "characterization research (amqsevt exit/handle behaviour, 2042 reap timing) for the resilient wrapper (.github#122)" \
  --body "A0 MQ spawn topology + setsid decision; A1 amqsevt open mode (source + live OPENOPTS); A2 signal/exit + handle behaviour; A3 2042 reap-window lit review + measurement. Findings gate the wrapper design. Refs .github#122."
vrg-pr-workflow report-ready --issue 760 --title "docs(events): amqsevt behaviour + 2042 reap-timing research for the resilient wrapper (.github#122)" --summary "Characterise amqsevt exit/handle behaviour under signals, its event-queue open mode, MQ's SERVICE spawn topology (setsid viability), and the queue-manager stale-handle reap window; captured as evidence in the internal report." --notes "Gates T2: A0 fixes setsid-vs-Python, A1 branches the 2042 work, A3 sets the sleep/retry bound. amqsevt.c staged in the shared build tree."
```

---

### Task 2: Resilient wrapper — `run.sh.j2` loop + process-group `STOPCMD`

**Blocked-by T1** (needs the A0 mechanism decision and the A3 sleep/retry values). Deliverable: the new wrapper + SERVICE definition, `vrg-validate` green, and a live start + clean-stop smoke test on `SVCQM`.

**Files:**
- Modify: `ansible/roles/mq-event-monitor/templates/run.sh.j2` (exec → supervision loop under `setsid`)
- Modify: `ansible/roles/mq-event-monitor/tasks/service.yml` (SERVICE `STARTCMD`/`STOPCMD`/`STOPARG`)
- Modify: `ansible/roles/mq-event-monitor/defaults/main.yml` (add `mq_event_sleep_secs`, `mq_event_open_retry_secs`)

**Interfaces:**
- Consumes: T1's A0 decision (mechanism), A3 reap window (sleep/bound values), existing role vars (`mq_event_amqsevt_bin`, `mq_event_syslog_tag`, `mq_event_logger_max_size`, `mq_event_run_dir`, `mq_event_service_name`).
- Produces (consumed by T3/T4): a `run.sh` that supervises `amqsevt`, restarts on crash, dies cleanly on a group `SIGTERM`; a SERVICE whose `STOPCMD` reaps the whole group.

- [ ] **Step 1: Add the sleep + open-retry defaults**

Append to `ansible/roles/mq-event-monitor/defaults/main.yml`:

```yaml
# Resilient wrapper (.github#122). Crash-restart sleep: long enough to clear the *typical*
# stale-handle reap window (measured in the #122 research), short enough to recover promptly.
mq_event_sleep_secs: 10
# Bounded open-retry: how long to keep retrying a failing MQOPEN (e.g. 2042 stale handle)
# before the wrapper exits non-zero (loud) so the SERVICE goes down visibly. 0 = retry forever.
mq_event_open_retry_secs: 1800
```

- [ ] **Step 2: Rewrite `run.sh.j2` as a supervision loop**

Replace the whole body of `ansible/roles/mq-event-monitor/templates/run.sh.j2` (currently the single `exec …` line + its header) with:

```bash
#!/bin/bash
# MQ event collector launcher (.github#122, epic .github#31). Started by the queue manager as
# an MQ SERVICE object (CONTROL(QMGR)) via `setsid -w -- run.sh <QM>`, so run.sh is its own
# process-group leader (PGID == PID). The SERVICE STOPCMD sends `kill -TERM -MQ_SERVER_PID`
# (negative PID => whole group), reaping BOTH this wrapper and amqsevt. On a group SIGTERM the
# wrapper terminates by default disposition (no trap) and does NOT restart. When amqsevt exits
# on its own (crash), the wrapper is still alive: it logs, sleeps, and re-runs — the resilience
# an MQ SERVICE object does not provide itself. See epics/122-event-monitor-wrapper/spec.md.
#
# $1 is the queue-manager name (the SERVICE STARTARG).
set -u
QM="$1"
TAG="{{ mq_event_syslog_tag }}"
SLEEP="{{ mq_event_sleep_secs }}"
OPEN_RETRY="{{ mq_event_open_retry_secs }}"   # 0 = forever
say() { logger -t "$TAG" "run.sh: $*"; }       # wrapper lifecycle lines -> journald (own logger)

say "starting event collector for $QM (pid $$, pgid $(ps -o pgid= -p $$ | tr -d ' '))"
open_deadline=0
while true; do
  # Destructive drain to json_compact -> per-run logger (one JSON object per journald entry).
  stdbuf -oL {{ mq_event_amqsevt_bin }} -m "$QM" -o json_compact \
    > >(exec logger --size {{ mq_event_logger_max_size }} -t "$TAG")
  rc=$?
  # rc 2042 surfaces as a non-zero amqsevt exit; treat a *persistent* open failure as fatal.
  if [ "$rc" -ne 0 ]; then
    now=$(date +%s)
    [ "$open_deadline" -eq 0 ] && open_deadline=$(( now + OPEN_RETRY ))
    if [ "$OPEN_RETRY" -ne 0 ] && [ "$now" -ge "$open_deadline" ]; then
      say "amqsevt exited rc=$rc and open kept failing past ${OPEN_RETRY}s — giving up (loud)"
      exit "$rc"
    fi
    say "amqsevt exited rc=$rc; restarting in ${SLEEP}s"
  else
    open_deadline=0
    say "amqsevt exited cleanly (rc=0); restarting in ${SLEEP}s"
  fi
  sleep "$SLEEP"
done
```

Note: the `open_deadline` only advances on consecutive failures and resets after any run that is not a failed open — a single crash restarts promptly; only a *stuck* open trips the bound. (Refine the rc→"is-an-open-failure" test in Step 4 if T1/A1 shows `amqsevt` uses a distinct exit code for 2042.)

- [ ] **Step 3: Change the SERVICE to launch under `setsid` and stop the whole group**

In `ansible/roles/mq-event-monitor/tasks/service.yml`, change the `define + start` task's `printf` so `STARTCMD` wraps `run.sh` in `setsid -w --` and `STOPCMD`/`STOPARG` target the process group. Replace the `DEFINE SERVICE(...)` line's `STARTCMD`/`STOPCMD`/`STOPARG` clauses:

```yaml
    printf "DEFINE SERVICE(%s) REPLACE CONTROL(QMGR) SERVTYPE(SERVER) STARTCMD('/usr/bin/setsid') STARTARG('-w -- %s/run.sh %s') STOPCMD('/bin/kill') STOPARG('-TERM -MQ_SERVER_PID') DESCR('Drain SYSTEM.ADMIN.*.EVENT to JSON on journald (mq-events); self-healing wrapper (.github#122)')\nSTART SERVICE(%s)\n" \
      "{{ mq_event_service_name }}" "{{ mq_event_run_dir }}" "{{ qmgr_name }}" "{{ mq_event_service_name }}" \
```

(Confirm `/usr/bin/setsid` is the path on the lab OS from T1 Step 2; adjust if it resolves elsewhere.)

- [ ] **Step 4: Static validation**

```bash
vrg-container-run -- vrg-validate
```

Expected: PASS (yamllint/ansible-lint clean; the `run.sh.j2` template renders).

- [ ] **Step 5: Live smoke — start + clean-stop on SVCQM**

A fast developer smoke via re-provision (the authoritative one-pass/cold-rebuild proof of B1 is the Task 4 deployment). Re-provision SVCQM's event monitor, then prove start and clean-stop:

```bash
cd ansible
uv run --project .. ansible-playbook site-distributed-shared.yml --limit svc
# both wrapper (run.sh under setsid) and amqsevt present, and PGID identity holds:
uv run --project .. ansible svc -b -m shell -a "ps -eo pid,pgid,sid,comm | grep -E 'run.sh|amqsevt'"
# clean stop reaps BOTH (no orphan):
uv run --project .. ansible svc -b -m shell -a "su - mqm -c 'echo \"STOP SERVICE(MQ.EVENT.MONITOR)\" | runmqsc SVCQM'; sleep 3; ps -eo pid,comm | grep -E 'run.sh|amqsevt' || echo 'NO ORPHAN — both reaped'"
```

Expected: with the SERVICE running, `run.sh` and `amqsevt` share one PGID equal to `run.sh`'s PID (the checkpoint); after `STOP SERVICE`, **neither** process remains (`NO ORPHAN — both reaped`). If an orphan remains or the PGID identity fails, **stop and escalate to the Python fallback** (spec §3.4) — do not paper over it.

- [ ] **Step 6: Commit + report ready**

```bash
vrg-git add ansible/roles/mq-event-monitor/templates/run.sh.j2 ansible/roles/mq-event-monitor/tasks/service.yml ansible/roles/mq-event-monitor/defaults/main.yml
vrg-commit --type feat --scope events \
  --message "self-healing amqsevt wrapper: setsid supervision loop + process-group STOPCMD (.github#122)" \
  --body "run.sh becomes a crash-restart loop launched under 'setsid -w --' so it leads its own process group; SERVICE STOPCMD sends 'kill -TERM -MQ_SERVER_PID' to reap wrapper+amqsevt together. Bounded open-retry on a stuck 2042, then loud exit. Refs .github#122."
vrg-pr-workflow report-ready --issue 761 --title "feat(events): self-healing amqsevt event-monitor wrapper (.github#122)" --summary "Replace the exec-amqsevt launcher with a setsid-led supervision loop that restarts amqsevt on crash and dies cleanly on a process-group STOPCMD, preserving the clean-stop guarantee." --notes "Live-smoked on SVCQM: PGID identity holds, STOP SERVICE reaps both processes with no orphan. Blocked-by T1; escalates to the Python fallback only if the checkpoint fails."
```

---

### Task 3: Validation runbook + harness + report scaffold

**Blocked-by T2** (the harness drives the deployed wrapper). Deliverable: a repeatable B-matrix procedure and a script an AI lab agent can run to gather evidence, plus the report's B-matrix scaffold. No production-role change.

**Files:**
- Create: `docs/reference/event-monitor-wrapper-validation.md` (the human/agent-runnable procedure)
- Create: `tools/validate-event-monitor-wrapper.sh` (the evidence-gathering harness)
- Modify: `docs/reports/2026-07-21-mq-event-monitor-wrapper-resilience.md` (add the empty B1/B2/B3 evidence sections)

**Interfaces:**
- Consumes: T2's wrapper/SERVICE behaviour; the QM name + host as harness parameters.
- Produces (consumed by T4): `validate-event-monitor-wrapper.sh <host-group> <QM>` writing evidence files under `docs/reports/assets/mq-event-monitor-wrapper/b*/`, and a runbook prose describing each check + expected result.

- [ ] **Step 1: Write the harness script (B1/B2/B3 evidence capture)**

Create `tools/validate-event-monitor-wrapper.sh` — a Bash harness that, given a host group and QM, runs each B-matrix scenario and writes captured output to an evidence dir. It must: (B1) provision + assert `run.sh`+`amqsevt` up with shared PGID and JSON in `journalctl -t mq-events`; (B2) `STOP SERVICE` then assert no orphan + SERVICE `STOPPED` + record `MQ_SERVER_PID` vs PGID; (B3) `kill -9` the `amqsevt` child and assert the wrapper logs `amqsevt exited` and a fresh `amqsevt` reappears within `~2×sleep`, capturing the journald lines; and — only if T1/A1 found **exclusive** open — force a tight restart (kill `amqsevt`, immediately `START` a second `amqsevt`) to capture a real `rc=2042`-then-recovery sequence. Each scenario echoes `PASS`/`FAIL` and writes raw output under `docs/reports/assets/mq-event-monitor-wrapper/b<N>/`. Keep every lab action behind `uv run --project ../.. ansible <group> -b -m shell -a "…"` so it runs from the repo, and print a one-line summary table at the end. Fail loud: any scenario that cannot assert its expected end state exits non-zero.

- [ ] **Step 2: Write the runbook**

Create `docs/reference/event-monitor-wrapper-validation.md` describing, per scenario, the induced condition, the exact command, the expected observable result, and where the evidence lands — written so an AI lab agent (or a human) can execute it unaided. Cross-reference the harness script as the mechanised form and the report as where evidence is filed. Include the A1 branch note (2042 arm runs only for exclusive-open).

- [ ] **Step 3: Add the B-matrix scaffold to the report**

In `docs/reports/2026-07-21-mq-event-monitor-wrapper-resilience.md`, add empty `## B1 — normal start`, `## B2 — clean stop (checkpoint)`, `## B3 — crash recovery (+2042 if exclusive)` sections with a one-line description and an "evidence: see assets/…/b<N>/" pointer, to be filled by the Task 4 run.

- [ ] **Step 4: Static validation**

```bash
vrg-container-run -- vrg-validate
```

Expected: PASS (shellcheck via the pipeline on the new `.sh`, markdown clean). Fix any shellcheck findings inline.

- [ ] **Step 5: Commit + report ready**

```bash
vrg-git add tools/validate-event-monitor-wrapper.sh docs/reference/event-monitor-wrapper-validation.md docs/reports/2026-07-21-mq-event-monitor-wrapper-resilience.md
vrg-commit --type docs --scope events \
  --message "validation runbook + evidence harness for the resilient event-monitor wrapper (.github#122)" \
  --body "Repeatable B-matrix procedure (runbook) and an agent-runnable harness that induces normal-start / clean-stop / crash-recovery (+2042 if exclusive) and captures evidence into the report assets. Refs .github#122."
vrg-pr-workflow report-ready --issue 762 --title "docs(events): validation runbook + evidence harness for the wrapper (.github#122)" --summary "Add a repeatable B-matrix runbook and a Bash harness an AI lab agent can run to induce and capture normal-start, clean-stop, and crash-recovery (plus 2042 for exclusive-open) evidence." --notes "Blocked-by T2. Consumed by the Task 4 live validation run."
```

---

### Task 4 (operational `deployment`): cold-rebuild the target stack with the resilient wrapper

**Blocked-by T2 (merged).** Not PR-workable — run via `issue-deploy`; closes only on `Outcome: SUCCESS`. This one action does two jobs: it is the **cold-rebuild acceptance gate** for the provisioning change (spec §10 — one-pass provisioning of the new wrapper = B1 proven on a cold-built stack) *and* it makes the wrapper deployed-and-usable for the Task 5 validation run. Autonomy boundary: agent-safe deploy steps only (a full cold rebuild via the lab tooling); no release step.

- [ ] **Step 1: Precondition self-check** — T2 merged to `develop` (the new `run.sh.j2` + SERVICE `setsid`/group-`STOPCMD` present). If unmet: comment "blocked: preconditions not met" and stop.
- [ ] **Step 2: Cold-rebuild the target stack** carrying the merged wrapper (worked example `svc`), from a clean VM via the lab bring-up path (`vrg-vm rebuild …` / `uv run mqlab bootstrap svc`) — proving one-pass provisioning of the new wrapper.
- [ ] **Step 3: Assert B1 on the cold-built stack** — `run.sh` (under `setsid`) + `amqsevt` up sharing one PGID, SERVICE `RUNNING`, JSON in `journalctl -t mq-events`. Any failure ⇒ `Outcome: FAILURE`, stays open.
- [ ] **Step 4: Record `Outcome: SUCCESS`** with the one-pass evidence (the spec §10 cold-rebuild + B1 proof).

---

### Task 5 (operational `validation`): live-lab B2/B3 run + finalize the report

**Blocked-by Task 4 (deployed on a cold-built stack) and T3 (harness).** Not PR-workable — run via `issue-validate`; closes only on `Outcome: SUCCESS` recorded as a comment. Deliverable: the executed B2/B3 matrix with captured evidence, and the finalized internal report merged.

- [ ] **Step 1: Precondition self-check** — Task 4 closed (wrapper deployed on the cold-built stack) and `tools/validate-event-monitor-wrapper.sh` exists on `develop`. If unmet: comment "blocked: preconditions not met" and stop.
- [ ] **Step 2: Run the harness** on the cold-built QM (worked example `svc` / `SVCQM`): `bash tools/validate-event-monitor-wrapper.sh svc SVCQM`. Capture the summary table.
- [ ] **Step 3: Assert every scenario PASS** — B1 re-confirmed, B2 clean stop (no orphan, SERVICE STOPPED, checkpoint `MQ_SERVER_PID == PGID`), B3 crash recovery (wrapper restarts amqsevt; for exclusive-open, a captured `rc=2042`-then-recovery). Any FAIL ⇒ `Outcome: FAILURE`, task stays open.
- [ ] **Step 4: Fill the report's B-matrix sections** from the captured evidence and land the completed report (a same-repo docs PR references this task; the human submits it).
- [ ] **Step 5: Record `Outcome: SUCCESS`** as a comment with the evidence pointers.

---

## Self-Review

**Spec coverage:**
- A0 spawn topology + setsid decision (spec §5 A0, §3.1 checkpoint) → T1 Steps 1–2. ✅
- A1 amqsevt source read + open mode, branches 2042 (spec §5 A1, §6 B3) → T1 Steps 3–4. ✅
- A2 signal/exit + handle behaviour (spec §5 A2) → T1 Step 5. ✅
- A3 reap timing, read-then-prove (spec §5 A3) → T1 Steps 6–7. ✅
- amqsevt.c into the shared build tree (spec §7, §10) → T1 Step 3. ✅
- Resilient wrapper: setsid loop + process-group STOPCMD, no trap (spec §3.1) → T2 Steps 2–3. ✅
- Fixed sleep + bounded open-retry then loud exit; own logger for lifecycle lines (spec §4) → T2 Steps 1–2. ✅
- Clean-stop regression preserved (spec §2, §6 B2, acceptance) → T2 Step 5 (smoke), T5 Step 3. ✅
- Python fallback only on checkpoint failure (spec §3.4) → T2 Step 5 + report-ready notes. ✅
- Validation runbook + agent-runnable harness (spec §7, §6) → Task 3. ✅
- Cold-rebuild one-pass acceptance + B1 (spec §10, standing cold-rebuild gate) → Task 4 (deployment). ✅
- Live B-matrix evidence + internal report (spec §6, §7, §10) → Task 3 (scaffold) + Task 5 (run/finalize). ✅
- Failover out of scope (spec §6, §11) → no task. ✅

**Placeholder scan:** `report-ready` carries the real task numbers (T1 #760, T2 #761, T3 #762; T4 #763 deployment, T5 #764 validation are operational, run via `issue-deploy`/`issue-validate`); the only soft value is the report date `2026-07-21` (set to the actual run date). No TBD/TODO/"handle errors". T3 Step 1 describes the harness by its exact required behaviours + I/O contract rather than a full script body, because the concrete `ps`/`runmqsc`/`kill` calls it wires together are already given verbatim in T1 and T2 — it is assembly of proven commands, not new logic. ✅

**Consistency:** role vars named identically to the existing role (`mq_event_amqsevt_bin`, `mq_event_syslog_tag`, `mq_event_logger_max_size`, `mq_event_run_dir`, `mq_event_service_name`) plus the two new defaults (`mq_event_sleep_secs`, `mq_event_open_retry_secs`) used verbatim in `run.sh.j2`; the harness/report evidence dir `docs/reports/assets/mq-event-monitor-wrapper/` is one path throughout; `SVCQM`/`svc` is the single worked example across T1/T2/T4. ✅

**Open questions carried from the spec (resolve at build):** setsid exec-vs-fork (T1 Step 1–2 decides, T2 Step 5 proves); open-retry bound value (`mq_event_open_retry_secs` default 1800, revisited from A3); logger pipeline across restarts (T2 Step 2 uses a per-run `logger`; T4/B3 confirms no drop/dup across a restart); exclusive-vs-shared open (T1 branches); which QM hosts the run (worked example `SVCQM`; confirmed at T4).
```
