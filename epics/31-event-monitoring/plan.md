# MQ instrumentation event monitoring — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give the lab an instrumentation-event stream — distinct from its diagnostic logs — by draining MQ's `SYSTEM.ADMIN.*.EVENT` queues to JSON with `amqsevt` and riding the existing journald → Alloy → Loki → Grafana pipeline.

**Architecture:** Four layers, each an independent PR. Enable events + define the collector in MQSC (in the qmgr create path); run `amqsevt -o json` as an MQ `SERVICE` object (`CONTROL(QMGR)`, bindings, destructive drain) whose start command is a role-shipped wrapper piping to `logger -t mq-events`; add one Alloy relabel rule so events land as `unit="mq-events"` in Loki; surface a Grafana events feed + per-object filter, and stop the existing log panels from wildcard-matching the new stream.

**Tech Stack:** IBM MQ 9.4 (`amqsevt`, MQSC `SERVICE`), Ansible (roles + per-arm site playbooks), systemd journald + `logger`, Grafana Alloy (River config), Loki/LogQL, Python 3 dashboard generators (`mqlab/*board.py`) with pytest.

- **Design spec:** `epics/31-event-monitoring/spec.md` (this directory).
- **Epic:** `logical-minds-foundry/.github#31`. **Docs task:** `#32`.
- **Implementation repo:** `logical-minds-foundry/mq-resiliency-lab-for-linux` (all paths below are relative to that repo, on a feature branch per task).

## Global Constraints

- **Validation is one command only:** `vrg-container-run -- vrg-validate`. Do not run individual linters/formatters. (repo CLAUDE.md)
- **Git/GitHub via wrappers:** `vrg-git`, `vrg-gh`, commit with `vrg-commit --type <t> --scope <s> --message <m>`. Raw `git`/`gh` are denied. No direct commits to `develop`/`main`; work on `feature/<issue>-<slug>`.
- **Cold-rebuild acceptance gate:** any lab bring-up / provisioning change (Tasks 1–3) is accepted only after a full VM cold rebuild proves it one-pass. Lint-green ≠ done.
- **Declarative + reproducible:** MQSC lives in the qmgr create path / per-arm site playbooks so it survives a cold boot. No hand-editing live VMs.
- **No bespoke PCF code:** the collector is `amqsevt` + config + a launcher wrapper. Nothing parses PCF.
- **No silent failures:** destructive drain (no `-b`); the collector's nodes must carry the `mq-diag-logging` journald rate-limit drop-in (`10-mq.conf`).
- **QM name is a variable from one source** in any dashboard code — never hardcode a QM name (`#313`/`#351`).
- **IBM Docs at build time:** confirm flags/paths against IBM Docs 9.4 via `tools/ibm_doc_cache.py` (browser-UA cache); do not trust WebFetch on `ibm.com/docs`.

---

### Task 1: MQSC event gate — enable all event classes + per-queue performance thresholds

Turn on every instrumentation-event class on the QMs, declaratively, in the same MQSC surface as the existing `MONQ/STATMQI` exporter gate, so it survives a cold rebuild. Performance events additionally need per-queue thresholds to fire.

**Files:**
- Modify: `ansible/roles/mq-qmgr/tasks/main.yml` (append a new MQSC task after the `enable QM monitoring for the prometheus exporter` task at lines 65–76)
- Reference (same pattern, do not duplicate the gate into these unless an arm bypasses `mq-qmgr`): `ansible/site-nativeha.yml:66`, `ansible/site-nativeha-ubuntu.yml:57`, `ansible/roles/mq-pcmk-qmgr/tasks/main.yml:80`

**Interfaces:**
- Consumes: `{{ qmgr_name }}` (already in scope in `mq-qmgr`), `/opt/mqm/bin/runmqsc`.
- Produces: QMs emitting PCF events onto `SYSTEM.ADMIN.*.EVENT`. Task 2's collector depends on these queues being populated.

- [ ] **Step 1: Confirm the class/attribute list against IBM Docs 9.4**

Run: `python3 tools/ibm_doc_cache.py "https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=qmgr-alter-qmgr"` and read `content.txt`.
Confirm the exact spelling/values of: `AUTHOREV, CHLEV, CONFIGEV, INHIBTEV, LOCALEV, REMOTEEV, LOGGEREV, PERFMEV, STRSTPEV, COMMEV` (all `ENABLED`) and `CMDEV(NODISPLAY)`; and that per-queue `QDPMAXEV`/`QDPHIEV` are `ALTER QLOCAL` attributes.

- [ ] **Step 2: Add the event-gate MQSC task**

In `ansible/roles/mq-qmgr/tasks/main.yml`, append after line 76:

```yaml
# Enable ALL instrumentation-event classes (#31). CMDEV(NODISPLAY) captures human
# admin/mutating commands for the events dashboard WITHOUT the exporter's DISPLAY-poll
# flood. Events land on SYSTEM.ADMIN.*.EVENT; the MQ.EVENT.MONITOR service drains them.
- name: enable instrumentation events (#31)
  ansible.builtin.shell: |
    set -o pipefail
    printf "ALTER QMGR AUTHOREV(ENABLED) CHLEV(ENABLED) CONFIGEV(ENABLED) INHIBTEV(ENABLED) LOCALEV(ENABLED) REMOTEEV(ENABLED) LOGGEREV(ENABLED) PERFMEV(ENABLED) STRSTPEV(ENABLED) COMMEV(ENABLED) CMDEV(NODISPLAY)\n" \
      | /opt/mqm/bin/runmqsc {{ qmgr_name }}
  args:
    executable: /bin/bash
  become: true
  become_user: mqm
  register: evgate
  changed_when: "'AMQ8005' in evgate.stdout"
  failed_when: evgate.rc not in [0, 10]
```

- [ ] **Step 3: Add per-queue performance thresholds on the app queues**

Performance events (`PERFMEV`) fire only when a queue has depth thresholds set. Add, after the task above, an MQSC task that sets `QDPMAXEV(ENABLED) QDPHIEV(ENABLED) QDEPTHHI(80)` on the arm's application queue(s). Use the queue name already defined for the stack (grep the arm's MQSC for `DEFINE QLOCAL` to get the exact names — e.g. `PCMK.SVC.REQUEST`); do not invent queue names.

```yaml
- name: enable per-queue performance events on the app queue(s) (#31)
  ansible.builtin.shell: |
    set -o pipefail
    printf "ALTER QLOCAL({{ item }}) QDPMAXEV(ENABLED) QDPHIEV(ENABLED) QDEPTHHI(80)\n" \
      | /opt/mqm/bin/runmqsc {{ qmgr_name }}
  args:
    executable: /bin/bash
  become: true
  become_user: mqm
  loop: "{{ mq_event_perf_queues | default([]) }}"
  register: perfq
  changed_when: "'AMQ8008' in perfq.stdout"
  failed_when: perfq.rc not in [0, 10]
```

Define `mq_event_perf_queues` in `ansible/roles/mq-qmgr/defaults/main.yml` as `[]` (empty = no perf queues by default), and set the real list per-arm at the site-playbook call site. Empty-loop is a safe no-op where an arm has no curated app queue.

- [ ] **Step 4: Validate**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS (yamllint/ansible-lint clean).

- [ ] **Step 5: Lab verification (cold-rebuild gate)**

After a cold rebuild of one arm, on a QM host:
Run: `su mqm -c 'echo "DISPLAY QMGR CMDEV CHLEV PERFMEV" | /opt/mqm/bin/runmqsc <QM>'`
Expected: `CMDEV(NODISPLAY) CHLEV(ENABLED) PERFMEV(ENABLED)`.
Then generate an event (e.g. `STOP CHANNEL` / start-stop a channel) and confirm depth on the event queue:
Run: `su mqm -c 'echo "DISPLAY QLOCAL(SYSTEM.ADMIN.CHANNEL.EVENT) CURDEPTH" | /opt/mqm/bin/runmqsc <QM>'`
Expected: `CURDEPTH` > 0 (an event was written).

- [ ] **Step 6: Commit**

```bash
vrg-commit --type feat --scope obs --message "enable all MQ instrumentation events on the QMs (#31)" \
  --body "ALTER QMGR turns on every event class + CMDEV(NODISPLAY); per-queue QDPMAXEV/QDPHIEV via mq_event_perf_queues. Lands in the qmgr create path so it survives a cold rebuild. Task 1 of epic .github#31."
```

---

### Task 2: Collector — `amqsevt` as an MQ `SERVICE` object with a wrapper script

Deploy `amqsevt -o json` as a queue-manager service that travels with the QM across failover, draining events destructively and emitting JSON to journald tagged `mq-events`. The service points at a role-shipped wrapper script (never an inline `sh -c '…|logger'` — that is an MQSC quoting minefield this repo has already lost once, see `mq-pcmk-qmgr`).

**Files:**
- Create: `ansible/roles/mq-event-monitor/tasks/main.yml`
- Create: `ansible/roles/mq-event-monitor/templates/run.sh.j2`
- Create: `ansible/roles/mq-event-monitor/defaults/main.yml`
- Modify: `ansible/roles/mq-qmgr/tasks/main.yml` (include `mq-event-monitor` after the event gate, mirroring how `mq-diag-logging` is included at lines 3–6)

**Interfaces:**
- Consumes: `{{ qmgr_name }}`; the event queues populated by Task 1; the `mq-diag-logging` journald rate-limit drop-in (assert it is present).
- Produces: journald entries with `SYSLOG_IDENTIFIER=mq-events` carrying one JSON object per event. Task 3 relabels these into Loki.

- [ ] **Step 1: Confirm `amqsevt` path, default event-queue set, and `SERVICE` STOPARG token**

Run: `python3 tools/ibm_doc_cache.py "https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=applications-amqsevt-sample-monitor-instrumentation-events"` and read `content.txt`.
Confirm: the binary path (`/opt/mqm/samp/bin/amqsevt`), that no `-q` means the standard default event-queue set, and the `-o json` flag. Then confirm the MQSC `SERVICE` server-PID substitution token (used in `STOPARG`) via `topic=reference-define-service`.

- [ ] **Step 2: Write the wrapper script template**

`ansible/roles/mq-event-monitor/templates/run.sh.j2`:

```sh
#!/bin/sh
# Launcher for the MQ event collector (#31). Started by the QM as an MQ SERVICE
# object (CONTROL(QMGR)), so it travels with the queue manager across failover.
# Destructive drain (no -b): this IS the single event consumer, processed once.
# JSON to journald tagged 'mq-events', beside the MQ core logs (tag 'ibm-mq').
# stdbuf -oL keeps the pipe line-buffered so events reach journald promptly.
exec stdbuf -oL {{ mq_event_amqsevt_bin }} -m "$1" -o json | logger -t {{ mq_event_syslog_tag }}
```

- [ ] **Step 3: Write role defaults**

`ansible/roles/mq-event-monitor/defaults/main.yml`:

```yaml
---
# MQ event collector (#31)
mq_event_amqsevt_bin: /opt/mqm/samp/bin/amqsevt
mq_event_syslog_tag: mq-events
mq_event_run_dir: /opt/mq-event-monitor
mq_event_service_name: MQ.EVENT.MONITOR
```

- [ ] **Step 4: Write the role tasks — ship the wrapper, assert the rate-limit drop-in, define the service**

`ansible/roles/mq-event-monitor/tasks/main.yml`:

```yaml
---
# MQ event collector (#31): amqsevt as an MQ SERVICE object. Travels with the QM.
- name: assert the journald rate-limit drop-in is present (no silent event loss)
  ansible.builtin.stat:
    path: /etc/systemd/journald.conf.d/10-mq.conf
  register: rl_dropin

- name: fail loudly if the rate-limit drop-in is missing
  ansible.builtin.fail:
    msg: >-
      /etc/systemd/journald.conf.d/10-mq.conf is absent — mq-diag-logging must run
      before mq-event-monitor, else 'everything on' event bursts are silently rate-dropped.
  when: not rl_dropin.stat.exists

- name: run dir for the collector wrapper
  ansible.builtin.file:
    path: "{{ mq_event_run_dir }}"
    state: directory
    mode: "0755"
  become: true

- name: install the collector wrapper script
  ansible.builtin.template:
    src: run.sh.j2
    dest: "{{ mq_event_run_dir }}/run.sh"
    mode: "0755"
  become: true

- name: define the MQ.EVENT.MONITOR service (idempotent)
  ansible.builtin.shell: |
    set -o pipefail
    printf "DEFINE SERVICE(%s) REPLACE CONTROL(QMGR) SERVTYPE(SERVER) STARTCMD('%s/run.sh') STARTARG('%s') STOPCMD('/bin/kill') STOPARG('$MQ_SERVER_PID') DESCR('Drain SYSTEM.ADMIN.*.EVENT to JSON on journald')\nSTART SERVICE(%s)\n" \
      "{{ mq_event_service_name }}" "{{ mq_event_run_dir }}" "{{ qmgr_name }}" "{{ mq_event_service_name }}" \
      | /opt/mqm/bin/runmqsc {{ qmgr_name }}
  args:
    executable: /bin/bash
  become: true
  become_user: mqm
  register: svc
  changed_when: "'AMQ8153' in svc.stdout or 'AMQ8005' in svc.stdout"
  failed_when: svc.rc not in [0, 10]
```

> Note: `$MQ_SERVER_PID` is MQ's own substitution token expanded by the QM when it runs `STOPCMD` — confirm the exact token spelling in Step 1 and adjust the `printf` (it must reach `runmqsc` literally, so it is single-quoted in the MQSC text, not shell-expanded).

- [ ] **Step 5: Wire the role into the qmgr create path**

In `ansible/roles/mq-qmgr/tasks/main.yml`, after the Task 1 event-gate tasks, add:

```yaml
- name: deploy the MQ event collector service (#31)
  ansible.builtin.include_role:
    name: mq-event-monitor
```

- [ ] **Step 6: Validate**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

- [ ] **Step 7: Lab verification (cold-rebuild gate) + failover, per arm**

After a cold rebuild:
Run: `su mqm -c 'echo "DISPLAY SERVICE(MQ.EVENT.MONITOR) STATUS" | /opt/mqm/bin/runmqsc <QM>'` → service present.
Run: `journalctl -t mq-events -n 5 -o cat` → JSON event objects appear.
Then verify it travels: fail the QM over to its partner and re-run `journalctl -t mq-events` on the newly-active node → events resume there. **Observe per arm** — a Native HA takeover is not a `strmqm`, so watch for `STRSTPEV` differences and any duplicate/gap across the takeover (spec §5.3 note 4). Record actual behavior per arm.

- [ ] **Step 8: Commit**

```bash
vrg-commit --type feat --scope obs --message "add mq-event-monitor: amqsevt as a QM SERVICE draining events to JSON (#31)" \
  --body "New role ships a run.sh wrapper (amqsevt -o json | logger -t mq-events) launched by an MQ SERVICE object with CONTROL(QMGR), so the collector travels with the QM across failover. Destructive drain, single consumer. Fails loud if the journald rate-limit drop-in is absent. Task 2 of epic .github#31."
```

---

### Task 3: Pipeline — Alloy relabel so events land as `unit="mq-events"`

Add one relabel rule so the `mq-events` syslog identifier becomes `unit="mq-events"` in Loki, exactly mirroring how `ibm-mq` is already produced. This is the entire "events are a separate stream" mechanism.

**Files:**
- Modify: `ansible/roles/alloy/templates/config.alloy.j2` (the `loki.relabel "journal"` block, lines 5–23)

**Interfaces:**
- Consumes: journald entries with `SYSLOG_IDENTIFIER=mq-events` from Task 2.
- Produces: Loki stream `{unit="mq-events"}` with the JSON body parseable via `| json`. Task 4 queries this.

- [ ] **Step 1: Confirm the existing relabel already covers `mq-events`**

Read `config.alloy.j2:16–22`. The second rule already fills `unit` from `__journal__syslog_identifier` when `systemd_unit` is empty — so `logger -t mq-events` (which has no systemd unit) will *already* produce `unit="mq-events"`. Decide: the relabel likely needs **no change**; the value of this task is making that explicit and proving it end-to-end.

- [ ] **Step 2: Make the intent explicit in the template comment**

Update the comment at `config.alloy.j2:13–15` to name both identifiers, so a future reader knows `mq-events` is intentional:

```
  // #282/#31: MQ logs and events via syslog have no systemd unit. Fill `unit` from the
  // syslog identifier ONLY when systemd_unit is empty. MQ core logs → unit="ibm-mq";
  // the MQ.EVENT.MONITOR collector → unit="mq-events" (a separate Loki stream from logs).
```

(No rule change unless Step 4 shows events are mis-labeled.)

- [ ] **Step 3: Validate**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

- [ ] **Step 4: Lab verification — events queryable by JSON field**

After the observe pass ships Alloy, query Loki (via the obs box `logcli` or Grafana Explore):
Run: `logcli query '{unit="mq-events"} | json | line_format "{{.eventType}} {{.eventReason}}"' --limit 10`
Expected: rows with parsed `eventType`/`eventReason`. Confirm a source-QM field and an object-name field (e.g. `queueName`/`channelName`) are present in `| json` output — Task 4 filters on them.

- [ ] **Step 5: Commit**

```bash
vrg-commit --type feat --scope obs --message "label the MQ event stream as unit=mq-events in Alloy (#31)" \
  --body "The existing syslog-identifier relabel already maps mq-events → unit=mq-events; make the intent explicit and prove events reach Loki queryable by JSON fields. Task 3 of epic .github#31."
```

---

### Task 4: Dashboards — events feed, per-object filter, and log-panel exclusion

Add a dedicated Grafana events feed (`{unit="mq-events"}`), a per-object events filter on the queue/channel views, and — critically — stop the existing cluster log panels from wildcard-matching the new stream. `clusterboard.py` currently leaks: `log_row` (~L605) uses `unit=~"...|.*mq.*"` and the nativeha timeseries (~L735) uses `unit=~"...|mq-.*"`, both of which match `mq-events`.

**Files:**
- Modify: `src/mqlab/clusterboard.py` (`log_row` ~L603–606; nativeha logs `sel` ~L735)
- Modify: `src/mqlab/messagingboard.py` (add a per-object events panel; its log panel at `mq-app-requester.*`/`mq-svc-responder.*` is positively scoped and does NOT leak — leave that selector as-is)
- Test: `tests/test_clusterboard.py`, `tests/test_messagingboard.py`

**Interfaces:**
- Consumes: the Loki `{unit="mq-events"}` stream (Task 3) with JSON fields `eventType`, `eventReason`, object name (`queueName`/`channelName`).
- Produces: rendered dashboard JSON (no downstream consumer).

- [ ] **Step 1: Write the failing test — log panels must exclude events**

In `tests/test_clusterboard.py`, add:

```python
def test_cluster_log_panels_exclude_the_event_stream():
    """The wildcard log selectors must not sweep in unit=mq-events (#31)."""
    p = log_row("loki-uid", 0)
    expr = p["targets"][0]["expr"]
    assert 'unit!="mq-events"' in expr
```

Add the analogous assertion for the nativeha logs timeseries panel (use the existing test that reaches that panel as a template for how it's constructed).

- [ ] **Step 2: Run it to confirm it fails**

Run: `uv run pytest tests/test_clusterboard.py::test_cluster_log_panels_exclude_the_event_stream -v`
Expected: FAIL (`unit!="mq-events"` not in expr).

- [ ] **Step 3: Add the exclusion to both leaky selectors**

In `src/mqlab/clusterboard.py` `log_row`, change the selector to append the exclusion:

```python
    sel = '{host=~"pcmk-.*|san-.*", unit=~"corosync.*|pacemaker.*|drbd.*|.*mq.*", unit!="mq-events"} |~ `${level}`'
```

And in the nativeha logs `sel` (~L735):

```python
    sel = f'{{host=~"{prefix}-.*", unit=~".*mqmonitor.*|.*amq.*|.*ibmmq.*|mq-.*", unit!="mq-events"}} |~ `${{level}}`'
```

- [ ] **Step 4: Run to confirm green**

Run: `uv run pytest tests/test_clusterboard.py -v`
Expected: PASS (new test + existing log-row tests still green).

- [ ] **Step 5: Write the failing test — a per-object events panel on the messaging board**

In `tests/test_messagingboard.py`, add a test asserting the board includes a `logs`-type panel whose expr is `{unit="mq-events"}` filtered by the per-stack object variable, e.g.:

```python
def test_board_has_an_events_panel_filtered_to_the_stack_objects():
    board = messaging_board(_stack_fixture())  # reuse the module's existing fixture helper
    panels = _all_panels(board)
    ev = [p for p in panels if p["type"] == "logs" and 'unit="mq-events"' in p["targets"][0]["expr"]]
    assert ev, "expected a mq-events panel"
    expr = ev[0]["targets"][0]["expr"]
    assert "| json" in expr and ("queueName" in expr or "channelName" in expr)
```

Match the test to the module's actual constructor/fixture names (mirror the neighbouring `test_logs_panel_covers_both_app_and_svc_units` at `tests/test_messagingboard.py:53`).

- [ ] **Step 6: Run it to confirm it fails**

Run: `uv run pytest tests/test_messagingboard.py::test_board_has_an_events_panel_filtered_to_the_stack_objects -v`
Expected: FAIL (no such panel).

- [ ] **Step 7: Add the per-object events panel**

In `src/mqlab/messagingboard.py`, add a `logs` panel (mirror the existing logs-panel builder) with expr:

```python
    events_sel = (
        '{unit="mq-events"} | json '
        '| queueName=`' + request_queue + '` or channelName=~`' + channel_glob + '`'
    )
```

Use the stack's already-parameterized request-queue and channel names (the same single-source values the board already uses for its metric panels — do **not** hardcode a QM or queue name; spec §7 naming alignment). Title it distinctly (e.g. `▤ Events (this stack)`).

- [ ] **Step 8: Run the full dashboard suite**

Run: `uv run pytest tests/test_clusterboard.py tests/test_messagingboard.py -v`
Expected: PASS.

- [ ] **Step 9: Validate**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS (full suite incl. 100% branch coverage gates).

- [ ] **Step 10: Lab verification — render + eyeball**

Render the boards and confirm in Grafana: the cluster log panels no longer show `mq-events` lines; the events feed shows the live stream; the messaging board's events panel shows only events for that stack's objects.

- [ ] **Step 11: Commit**

```bash
vrg-commit --type feat --scope obs --message "surface MQ events on the dashboards + stop log panels leaking the event stream (#31)" \
  --body "Dedicated per-stack events panel ({unit=mq-events} filtered by object); exclude unit=mq-events from the cluster log selectors that wildcard-matched it (.*mq.* / mq-.*). Task 4 of epic .github#31."
```

---

## Self-Review

**Spec coverage** (spec §10 acceptance criteria → task):
- Events enabled declaratively, survive cold rebuild → Task 1.
- `amqsevt` as MQ SERVICE, travels with QM, destructive drain, JSON to journald → Task 2.
- Event JSON reaches Loki, queryable by fields → Task 3.
- Dedicated events feed distinct from logs → Task 4 (Steps 5–7).
- Log panels exclude `mq-events` (no bleed) → Task 4 (Steps 1–4). *(Confirmed real: `clusterboard.py` `.*mq.*` and `mq-.*` match `mq-events`.)*
- Per-object (queue/channel) event filter → Task 4 (Steps 5–7).
- `CMDEV(NODISPLAY)` surfaces human commands without poll noise → Task 1 (Step 2) + verify at Task 4 Step 10.
- Thesis: no bespoke PCF code → Tasks 1–2 (MQSC + wrapper only).
- `vrg-validate` passes → every task's validate step.

**Placeholder scan:** no TBD/TODO; every code step shows the code. Two deliberate build-time confirmations (IBM Docs path/tokens at Task 1 Step 1, Task 2 Step 1) are explicit verification steps, not placeholders — the spec §4/§12 mandates them.

**Type/name consistency:** `unit="mq-events"` (label), `mq-events` (syslog tag), `MQ.EVENT.MONITOR` (service), `mq-event-monitor` (role), `mq_event_*` (role vars), `/opt/mq-event-monitor/run.sh` (wrapper) — used consistently across Tasks 2–4. The Alloy relabel (Task 3) and the label consumed by Task 4 match.

**Note on `uv run pytest`:** the dashboard tests run via `uv run pytest` for the local red/green loop (validation memory); the accept gate is still `vrg-container-run -- vrg-validate`.
