# File-sink variant of the resilient MQ event collector — implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a first-class, tested **file** sink to the `mq-event-monitor` role, selected at Ansible render time so the delivered script is single-purpose, and publish a report documenting the exact tested script.

**Architecture:** One role, one template. `run.sh.j2` gains a render-time Jinja branch around only the three lines that differ between the `syslog` (default) and `file` sinks; the resilience core stays single-source. The `SERVICE` definition and host-prep become sink-aware. The existing live-lab harness is parameterized by sink. Deployment onto the Native HA RHEL arm and the two-corner-case validation are operational tasks; the report carries their evidence.

**Tech Stack:** Ansible (Jinja2 templating, `trim_blocks=True`/`lstrip_blocks=False`), bash, IBM MQ 9.4 `amqsevt` + MQSC `SERVICE`, `util-linux`/`procps-ng`, `python3` (stdlib, for JSONL parse in the harness).

## Global Constraints

- **Syslog stays the lab default.** `mq_event_sink: syslog` is the default; the journald→Alloy pipeline is untouched. (spec §2)
- **Byte-identical syslog render guard.** The syslog-rendered `run.sh` MUST be byte-for-byte identical before and after the refactor — an acceptance criterion on the impl. (spec §3.1)
- **Single-purpose delivered artifact.** The rendered `file` script contains no `logger`, no syslog tag, no runtime sink branch. Selection is render-time only. (spec §3.1)
- **Shared resilience core.** Only three lines differ between sinks: `say()`'s destination, the `amqsevt` stdout sink, and the file-only post-run `cat "${ERRLOG}" >&2` flush. The per-run temp `ERRLOG` capture + `grep … | head -1` root-cause logic is shared. (spec §3.1–3.2)
- **Stop semantics unchanged and sink-independent.** `setsid` group-leader + negative-PID group-kill STOPCMD are identical across sinks. (spec §3.2)
- **Validation on the Native HA RHEL arm**, four-assert SUCCESS bar: (1) JSONL well-formed, (2) destructive drain, (3) clean stop/no orphan, (4) 2042 crash recovery. (spec §4)
- **Non-goals (documented as consumer responsibility, not implemented):** rotation (copy-truncate), checkpointed forwarding, lossless/transactional delivery, and the **HA-failover host-local stranding window** — the last stated explicitly as a required report caveat. (spec §2, §5)
- **Circular logging everywhere** → no `LOGGEREV`; the 11-class enable is unchanged. (role `tasks/service.yml`)
- **Report** dated `2026-07-29`; the two `2026-07-28` reports' "one-line change" footnote is replaced with a pointer to it. (spec §5)
- **Commit** with `vrg-commit`; **validate** with `vrg-container-run -- vrg-validate` (the only validation command). Work from the assigned worktree.

---

### Task 1: Role variables + render-time template branch (with byte-identical syslog guard)

**Files:**

- Create: `tools/render-event-run.sh` (renders the wrapper for a given sink, exactly as Ansible would — used for the diff guard, for inspecting the file variant, and for quoting the exact script in the report)
- Modify: `ansible/roles/mq-event-monitor/defaults/main.yml`
- Modify: `ansible/roles/mq-event-monitor/templates/run.sh.j2`

**Interfaces:**

- Produces: role vars `mq_event_sink` (`syslog`|`file`, default `syslog`), `mq_event_file_dir`, `mq_event_data_file`, `mq_event_error_file`. The rendered `file` `run.sh` reads `DATA_FILE`/`ERROR_FILE` (from those vars) and `ERRLOG` (unchanged temp).
- Consumes: existing vars `mq_event_amqsevt_bin`, `mq_event_syslog_tag`, `mq_event_sleep_secs`, `mq_event_open_retry_secs`, `mq_event_logger_max_size`, and `qmgr_name` (supplied at include time).

- [ ] **Step 1: Add the render helper**

Create `tools/render-event-run.sh` (mode 0755):

```bash
#!/usr/bin/env bash
# Render the mq-event-monitor run.sh wrapper for a given sink, exactly as Ansible
# renders it (same Jinja engine + trim_blocks). Used by the byte-identical syslog
# guard, to inspect the file variant, and to quote the exact script in the report.
# Usage: render-event-run.sh <syslog|file> [QM]
set -euo pipefail
sink="${1:?usage: render-event-run.sh <syslog|file> [QM]}"
qm="${2:-EVTCAP}"
root="$(git rev-parse --show-toplevel)"
role="${root}/ansible/roles/mq-event-monitor"
out="$(mktemp)"; trap 'rm -f "${out}"' EXIT
cd "${root}/ansible"
uv run ansible localhost -m template \
  -a "src=${role}/templates/run.sh.j2 dest=${out} mode=0755" \
  -e "@${role}/defaults/main.yml" \
  -e "qmgr_name=${qm}" -e "mq_event_sink=${sink}" >/dev/null
cat "${out}"
```

- [ ] **Step 2: Capture the golden syslog render (BEFORE editing the template)**

Run: `tools/render-event-run.sh syslog > /tmp/run.syslog.golden`
Expected: the current syslog `run.sh`, unchanged (this is the baseline the guard defends).

- [ ] **Step 3: Add the sink + file-path vars to defaults**

In `ansible/roles/mq-event-monitor/defaults/main.yml`, after the existing collector vars, add:

```yaml
# Sink selector — the choice lives here (management layer), resolved at TEMPLATE-RENDER
# time so the delivered run.sh is single-purpose (no runtime sink branch, no logger in
# the file variant). 'syslog' stays the lab default (feeds journald -> Alloy). '.github#152'
mq_event_sink: syslog
# File-sink outputs (only meaningful when mq_event_sink == 'file'). Host-local, mqm-owned.
# .json = the JSONL event stream (the sink); .error = wrapper lifecycle + amqsevt
# diagnostics (the SERVICE STDOUT/STDERR redirect). Per-QM so a failover node can host it.
mq_event_file_dir: /var/mqm/event-monitor
mq_event_data_file: "{{ mq_event_file_dir }}/{{ qmgr_name }}.events.json"
mq_event_error_file: "{{ mq_event_file_dir }}/{{ qmgr_name }}.error"
```

- [ ] **Step 4: Add the render-time branch to `run.sh.j2` (config block)**

Replace the single line `TAG="{{ mq_event_syslog_tag }}"` with the branch below. **Keep the `{%…%}` tags at column 0** (with `lstrip_blocks=False`, an indented tag would leak leading spaces into the render and break byte-identity; `trim_blocks=True` strips the newline after each tag, so the syslog branch reproduces the original line exactly):

```jinja
{% if mq_event_sink == 'file' %}
DATA_FILE="{{ mq_event_data_file }}"         # JSONL event stream — THIS is the sink (append)
ERROR_FILE="{{ mq_event_error_file }}"       # wrapper lifecycle + amqsevt diagnostics (SERVICE STDOUT/STDERR)
{% else %}
TAG="{{ mq_event_syslog_tag }}"
{% endif %}
```

- [ ] **Step 5: Branch the `say()` definition**

Replace the single `say() { logger … }` line with:

```jinja
{% if mq_event_sink == 'file' %}
say() { echo "run.sh[$$]: $*" >&2; }          # wrapper lifecycle -> stderr -> SERVICE STDERR (.error)
{% else %}
say() { logger -t "$TAG" "run.sh[$$]: $*"; }  # wrapper lifecycle -> journald via its own logger
{% endif %}
```

- [ ] **Step 6: Branch the `amqsevt` stdout sink and add the file-only diagnostics flush**

Replace the two-line `stdbuf … 2>"${ERRLOG}" > >(exec logger …)` invocation with:

```jinja
{% if mq_event_sink == 'file' %}
  stdbuf -oL {{ mq_event_amqsevt_bin }} -m "${QM}" -o json_compact \
    2>"${ERRLOG}" >> "${DATA_FILE}"
{% else %}
  stdbuf -oL {{ mq_event_amqsevt_bin }} -m "${QM}" -o json_compact \
    2>"${ERRLOG}" > >(exec logger --size {{ mq_event_logger_max_size }} -t "${TAG}")
{% endif %}
```

Then, immediately **after** the shared `reason=$(grep … head -1 …)` line, add the file-only flush so each run's raw diagnostics reach `.error` (the temp `ERRLOG` is overwritten each loop, so this flushes only the current run):

```jinja
{% if mq_event_sink == 'file' %}
  cat "${ERRLOG}" >&2   # flush this run's raw amqsevt diagnostics into .error (SERVICE STDERR redirect)
{% endif %}
```

- [ ] **Step 7: Regression guard — syslog render is byte-identical**

Run: `tools/render-event-run.sh syslog | diff - /tmp/run.syslog.golden`
Expected: **empty output, exit 0.** If not empty, the branch perturbed the shared body — fix the template (usually a stray blank line from a mis-placed tag) until the diff is empty. This gate is mandatory.

- [ ] **Step 8: Inspect the file render — single-purpose, correct**

Run: `tools/render-event-run.sh file`
Expected, by inspection: no `logger`, no `TAG`, no `{%`/`if` in the output; `DATA_FILE`/`ERROR_FILE` set; the loop line ends `>> "${DATA_FILE}"`; `say() { echo … >&2; }`; a `cat "${ERRLOG}" >&2` line after `reason=`; the `setsid`/group-kill comments and loop are unchanged.

- [ ] **Step 9: Validate and commit**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.
Then:

```bash
vrg-git add tools/render-event-run.sh ansible/roles/mq-event-monitor/defaults/main.yml ansible/roles/mq-event-monitor/templates/run.sh.j2
vrg-commit --type feat --scope events --message "sink-selectable event-monitor wrapper: render-time file variant (.github#152)" --body "Adds mq_event_sink (syslog default | file). File sink resolved at template-render time so the delivered run.sh is single-purpose. Shared resilience core; syslog render byte-identical (guarded via tools/render-event-run.sh)."
```

---

### Task 2: Sink-aware SERVICE definition

**Files:**

- Modify: `ansible/roles/mq-event-monitor/tasks/service.yml` (the "define + start the MQ event-monitor service" task)

**Interfaces:**

- Consumes: `mq_event_sink`, `mq_event_error_file`, `mq_event_data_file`, `mq_event_service_name`, `mq_event_run_dir`, `qmgr_name`.
- Produces: a `DEFINE SERVICE` that, for `file`, adds `STDOUT`/`STDERR` → `.error` and a file-naming `DESCR`; for `syslog`, is unchanged.

- [ ] **Step 1: Add sink-derived vars to the define task**

On the `- name: define + start the MQ event-monitor service` task, add a `vars:` block:

```yaml
  vars:
    _svc_stdio: "{{ \"STDOUT('\" + mq_event_error_file + \"') STDERR('\" + mq_event_error_file + \"') \" if mq_event_sink == 'file' else '' }}"
    _svc_descr: "{{ 'Drain SYSTEM.ADMIN.*.EVENT as JSONL to ' + mq_event_data_file + ' (.error = diagnostics)' if mq_event_sink == 'file' else 'Drain SYSTEM.ADMIN.*.EVENT to JSON on journald (mq-events)' }}"
```

- [ ] **Step 2: Thread the vars into the `printf` MQSC**

Change the `printf` format + args so the STDIO clause and DESCR come from the vars. The `STARTCMD`/`STARTARG`/`STOPCMD`/`STOPARG` are unchanged (sink-independent group kill):

```yaml
    printf "DEFINE SERVICE(%s) REPLACE CONTROL(QMGR) SERVTYPE(SERVER) STARTCMD('/usr/bin/setsid') STARTARG('-w -- %s/run.sh %s') {{ _svc_stdio }}STOPCMD('/bin/kill') STOPARG('-TERM -- -+MQ_SERVER_PID+') DESCR('%s')\nSTART SERVICE(%s)\n" \
      "{{ mq_event_service_name }}" "{{ mq_event_run_dir }}" "{{ qmgr_name }}" "{{ _svc_descr }}" "{{ mq_event_service_name }}" \
      | /opt/mqm/bin/runmqsc {{ qmgr_name }}
```

(Note the `{{ _svc_stdio }}` sits directly before `STOPCMD` and carries its own trailing space; for `syslog` it is empty, so the string collapses to the exact original MQSC.)

- [ ] **Step 3: Confirm the syslog MQSC is unchanged**

Run (from `ansible/`): render both DESCR/STDIO expansions and confirm syslog is empty/original —

```bash
uv run ansible localhost -m debug \
  -a "msg=[{{ _svc_stdio | default('') }}][{{ _svc_descr }}]" \
  -e "@roles/mq-event-monitor/defaults/main.yml" -e qmgr_name=EVTCAP -e mq_event_sink=syslog \
  -e "_svc_stdio={{ \"STDOUT('\" + mq_event_error_file + \"') STDERR('\" + mq_event_error_file + \"') \" if mq_event_sink == 'file' else '' }}" \
  -e "_svc_descr={{ 'Drain SYSTEM.ADMIN.*.EVENT as JSONL to ' + mq_event_data_file + ' (.error = diagnostics)' if mq_event_sink == 'file' else 'Drain SYSTEM.ADMIN.*.EVENT to JSON on journald (mq-events)' }}"
```

Expected (syslog): `[][Drain SYSTEM.ADMIN.*.EVENT to JSON on journald (mq-events)]` — empty STDIO clause and the original DESCR, so the produced MQSC matches the committed string. (Re-run with `mq_event_sink=file` to eyeball the file expansion.)

- [ ] **Step 4: Validate and commit**

Run: `vrg-container-run -- vrg-validate` → PASS.

```bash
vrg-git add ansible/roles/mq-event-monitor/tasks/service.yml
vrg-commit --type feat --scope events --message "sink-aware event-monitor SERVICE: STDOUT/STDERR -> .error for file sink (.github#152)" --body "For mq_event_sink=file the DEFINE SERVICE redirects the wrapper's STDOUT/STDERR to the .error file; syslog path unchanged (empty STDIO clause, original DESCR)."
```

---

### Task 3: Host-prep — create the file-sink directory; re-scope the journald assert

**Files:**

- Modify: `ansible/roles/mq-event-monitor/tasks/main.yml`

**Interfaces:**

- Consumes: `mq_event_sink`, `mq_event_file_dir`.
- Produces: on every failover-capable node, when `sink == file`, an mqm-owned `mq_event_file_dir`; the journald rate-limit assert applies only when `sink == syslog`.

- [ ] **Step 1: Scope the journald rate-limit assert to the syslog sink**

The rate-limit drop-in (`/etc/systemd/journald.conf.d/10-mq.conf`) only matters when journald is the sink. Add `when: mq_event_sink == 'syslog' and not rl_dropin.stat.exists` to the `- name: fail loudly if the rate-limit drop-in is missing` task (leave the `stat` task as-is; it is a harmless probe). The `amqsevt`-present assert stays unconditional — both sinks need it.

- [ ] **Step 2: Create the file-sink directory when file sink is selected**

After the `- name: run dir for the collector wrapper` task, add:

```yaml
- name: file-sink output directory (mqm-owned) — only when the file sink is selected (#152)
  ansible.builtin.file:
    path: "{{ mq_event_file_dir }}"
    state: directory
    owner: mqm
    group: mqm
    mode: "0750"
  become: true
  when: mq_event_sink == 'file'
```

- [ ] **Step 3: Validate and commit**

Run: `vrg-container-run -- vrg-validate` → PASS.

```bash
vrg-git add ansible/roles/mq-event-monitor/tasks/main.yml
vrg-commit --type feat --scope events --message "event-monitor host-prep: file-sink dir + scope journald assert to syslog (.github#152)"
```

---

### Task 4: Parameterize the validation harness by sink

**Files:**

- Modify: `tools/validate-event-monitor-wrapper.sh`
- Modify: `docs/reference/event-monitor-wrapper-validation.md`

**Interfaces:**

- Consumes: the deployed collector (either sink).
- Produces: `validate-event-monitor-wrapper.sh <QM> [SERVICE] [SINK] [DATA_FILE] [ERROR_FILE]` — default `SINK=syslog` preserves the current 2-arg invocation; `SINK=file` requires `DATA_FILE`/`ERROR_FILE` and runs the four-assert bar.

- [ ] **Step 1: Add sink parameters and sink-aware readers**

Extend the arg parsing and add readers. After the existing `SVC=` line:

```bash
SINK="${3:-syslog}"
DATA_FILE="${4:-}"
ERROR_FILE="${5:-}"
if [ "${SINK}" = "file" ] && { [ -z "${DATA_FILE}" ] || [ -z "${ERROR_FILE}" ]; }; then
  printf 'usage (file sink): %s <QM> <SERVICE> file <DATA_FILE> <ERROR_FILE>\n' "$0" >&2; exit 2
fi

# Count JSON events in the sink; print run.sh lifecycle lines from the sink.
events_count() {
  if [ "${SINK}" = "file" ]; then grep -c '"eventSource"' "${DATA_FILE}" 2>/dev/null || echo 0
  else journalctl -t "${TAG:-mq-events}" --no-pager -o cat 2>/dev/null | grep -c '"eventSource"'; fi
}
lifecycle() {
  if [ "${SINK}" = "file" ]; then grep 'run.sh\[' "${ERROR_FILE}" 2>/dev/null | tail -8
  else journalctl -t mq-events --no-pager -o cat 2>/dev/null | grep 'run.sh\[' | tail -8; fi
}
```

Replace the two inline `journalctl … grep -c '"eventSource"'` / `journalctl … grep 'run.sh\['` uses (B1 events line ~68, B3 lifecycle line ~106) with `events_count` / `lifecycle`.

- [ ] **Step 2: Add the file-sink asserts (JSONL well-formed + destructive drain)**

After the B1 checkpoint block, add a file-only assert group (guarded by `[ "${SINK}" = "file" ]`):

```bash
if [ "${SINK}" = "file" ]; then
  # A1 JSONL well-formed: last 20 lines each parse as standalone JSON
  if tail -n 20 "${DATA_FILE}" 2>/dev/null | python3 -c 'import json,sys; [json.loads(l) for l in sys.stdin if l.strip()]' 2>/dev/null; then
    pass "A1 .json is well-formed JSONL (last 20 lines parse)"
  else fail "A1 .json not valid JSONL"; fi
  # A2 destructive drain: force events with a self-contained define-then-delete of a throwaway
  # queue (config create/delete + command events; mutates nothing persistent, cleans up after
  # itself), then confirm .json grew AND the event queues sit drained (CURDEPTH 0).
  before=$(events_count)
  printf 'DEFINE QLOCAL(EVT.DRAIN.PROBE) REPLACE\nDELETE QLOCAL(EVT.DRAIN.PROBE)\n' | mqsc >/dev/null 2>&1
  sleep 4
  after=$(events_count)
  depth=$(printf 'DISPLAY QLOCAL(SYSTEM.ADMIN.QMGR.EVENT) CURDEPTH\n' | mqsc | grep -oE 'CURDEPTH\([0-9]+\)' | grep -oE '[0-9]+' | head -1)
  if [ "${after:-0}" -gt "${before:-0}" ] && [ "${depth:-1}" -eq 0 ]; then
    pass "A2 destructive drain (events ${before}->${after} in .json; SYSTEM.ADMIN.QMGR.EVENT CURDEPTH=0)"
  else fail "A2 drain (before=${before} after=${after} depth=${depth})"; fi
fi
```

- [ ] **Step 3: Show the file-sink 2042 evidence from `.error`**

In B3, after the `lifecycle` call, add (file only) a grep of `.error` for the root-cause reason so the captured evidence includes it:

```bash
if [ "${SINK}" = "file" ]; then
  printf 'root-cause 2042 in .error:\n'; grep -iE 'MQRC_OBJECT_IN_USE|2042' "${ERROR_FILE}" 2>/dev/null | tail -3
fi
```

- [ ] **Step 4: Update the reference doc**

In `docs/reference/event-monitor-wrapper-validation.md`, add a short "File sink" subsection under Step 2: the invocation `validate-event-monitor-wrapper.sh <QM> MQ.EVENT.MONITOR file <DATA_FILE> <ERROR_FILE>` (paths from `mq_event_data_file`/`mq_event_error_file`, default `/var/mqm/event-monitor/<QM>.events.json` and `…/<QM>.error`), the four asserts (A1 JSONL, A2 drain, B2 stop, B3 2042), and where file-sink evidence is captured (`docs/reports/assets/mq-event-monitor-file-sink/`). Note B1 still applies (checkpoint), and that syslog invocation is unchanged (default sink).

- [ ] **Step 5: Validate and commit**

Run: `vrg-container-run -- vrg-validate` → PASS (shellcheck on the harness must stay clean).

```bash
vrg-git add tools/validate-event-monitor-wrapper.sh docs/reference/event-monitor-wrapper-validation.md
vrg-commit --type feat --scope events --message "sink-parameterize the event-monitor validation harness (.github#152)" --body "One harness, both sinks: reads .json/.error for file, journald for syslog. Adds A1 (JSONL well-formed) + A2 (destructive drain) to the four-assert file-sink SUCCESS bar; default sink=syslog preserves the existing invocation."
```

**This completes the single impl PR (Tasks 1–4).** Report it ready with `vrg-pr-workflow report-ready`; the human runs `vrg-submit-pr` into `develop`.

---

### Task 5 (operational — `deployment`): provision the file variant onto the Native HA RHEL arm

Run with `issue-deploy` after the impl PR merges. **Not a code PR.**

- [ ] Find the active node: `dspmq -m <QM> -o nativeha -x` → the `INSTANCE(<node>) ROLE(Active)` line.
- [ ] Provision the Native HA RHEL arm with the file sink as a **run-time extra-var override** (so the committed default stays `syslog` and the arm reverts to syslog on the next normal provision — no permanent change is committed):

```bash
cd ansible
uv run ansible-playbook site-nativeha.yml -e mq_event_sink=file
```

  Host-prep creates `/var/mqm/event-monitor/` (mqm-owned) on every instance; the active instance re-DEFINEs the SERVICE with `STDOUT`/`STDERR` → `<QM>.error` and starts it. `DEFINE … REPLACE` swaps the running collector to the file variant.

- [ ] Confirm deployed: `DISPLAY SVSTATUS(MQ.EVENT.MONITOR)` reads `STATUS(RUNNING)`, `/var/mqm/event-monitor/<QM>.events.json` is being appended, `<QM>.error` carries the `run.sh[…] starting …` line. Record `Outcome: SUCCESS` on the deployment issue.

### Task 6 (operational — `validation`): the two corner cases + file-output check

Run with `issue-validate` after deployment closes. **Not a code PR.** Blocked-by Task 5.

- [ ] On the active node, run the harness in file mode:

```bash
cd ansible
uv run --project .. ansible <active-node> -b -m script \
  -a "../tools/validate-event-monitor-wrapper.sh <QM> MQ.EVENT.MONITOR file /var/mqm/event-monitor/<QM>.events.json /var/mqm/event-monitor/<QM>.error"
```

- [ ] Require **`== summary: ALL SCENARIOS PASS ==`** (exit 0), i.e. all four asserts (A1 JSONL, A2 drain, B2 clean stop, B3 2042 recovery) plus B1 checkpoint pass.
- [ ] Capture full stdout + a 20-line `.json` sample + the `.error` tail into `docs/reports/assets/mq-event-monitor-file-sink/`. Record `Outcome: SUCCESS` (with the evidence path) on the validation issue. On any `FAIL`, leave the issue open — it is a real regression.

### Task 7 (docs PR): the report + footnote fixes

Blocked-by Task 6 (ships real evidence). Closed by a same-repo PR into `develop`.

**Files:**

- Create: `docs/reports/2026-07-29-mq-event-monitor-file-sink-resilient.md`
- Modify: `docs/reports/2026-07-28-mq-event-monitor-resilient-service.md` (the "Choosing the sink" footnote)
- Modify: `docs/reports/2026-07-28-mq-event-monitoring-to-file.md` (its file-sink-vs-syslog note, if it forwards to the footnote)
- Add: evidence under `docs/reports/assets/mq-event-monitor-file-sink/`

- [ ] **Step 1: Write the report.** Structure: purpose/scope; a Quick start that IS the whole setup (samples present → enable classes → drop the script → `DEFINE SERVICE` file variant → verify with `tail -f <…>.json`); **the exact rendered file script** quoted verbatim from `tools/render-event-run.sh file` (no "change this line" caveats); the exact `DEFINE SERVICE` (file); the resilience explanation (setsid group-leader, restart loop, 2042 window) carried from the sibling report; the captured **test evidence** for A1/A2/B2/B3 with validation-provenance framing; and the retained non-goals — rotation, forwarding, lossless — **plus the HA-failover host-local stranding window stated explicitly** (host-local files; unforwarded tail strands on failover; keeping the forwarder current bounds but cannot close it; this is a file-sink-specific window syslog does not have, and a concrete reason syslog is the lab default/recommendation).

- [ ] **Step 2: Retire the "one-line change" footnote.** In `2026-07-28-mq-event-monitor-resilient-service.md`, replace the "Choosing the sink" paragraph that says *"`run.sh` is a one-line change"* with a pointer: the file sink is a **tested, install-as-written** variant — see `2026-07-29-mq-event-monitor-file-sink-resilient.md`. Do the same for any forwarding reference in `2026-07-28-mq-event-monitoring-to-file.md`. Change nothing else in those reports.

- [ ] **Step 3: Validate and hand off.** `vrg-container-run -- vrg-validate` → PASS. Report ready via `vrg-pr-workflow report-ready`; human runs `vrg-submit-pr`.

---

## Self-Review

**Spec coverage:**

- §3.1 selectable sink + render-time selection + byte-identical guard → Task 1 (Steps 3–7).
- §3.2 file script contract (stdout→.json, say→stderr, shared per-run reason grep + `cat >&2` flush) → Task 1 (Steps 4–6).
- §3.3 sink-aware SERVICE (STDOUT/STDERR→.error) → Task 2.
- §3.4 variables → Task 1 (Step 3); host-prep dir + re-scoped assert → Task 3.
- §4 validation (four asserts, Native HA arm, sink-parameterized harness) → Task 4 + Task 6.
- §5 report + footnote fixes + HA-failover caveat → Task 7.
- §6 task ordering impl→deploy→validate→report → Tasks 1–4 (impl), 5 (deploy), 6 (validate), 7 (report).

**Placeholder scan:** none — every code step carries concrete content and exact commands.

**Type/name consistency:** `mq_event_sink`, `mq_event_file_dir`, `mq_event_data_file`, `mq_event_error_file`, `DATA_FILE`, `ERROR_FILE`, `ERRLOG`, `SINK` used consistently across tasks; harness arg order `<QM> [SERVICE] [SINK] [DATA_FILE] [ERROR_FILE]` matches Task 6's invocation.
