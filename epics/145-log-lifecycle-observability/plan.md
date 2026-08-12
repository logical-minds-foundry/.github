# Native HA log-lifecycle observability — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make MQ log behaviour log-type-aware in the lab — enable `LOGGEREV` on the linear-equivalent Native HA arms via declare-and-verify — and make the replicated/linear log lifecycle observable and teachable.

**Architecture:** Two observability channels feed one cockpit panel group. Channel A (logger events) rides the existing `mq-event-monitor` `amqsevt` service (already `CONTROL(QMGR)`, already on the Native HA arms). Channel B is a new stdlib, non-MQI collector (`src/mqlab/loglifecycle.py`) modelled verbatim on `src/mqlab/nativehastate.py`: it runs on all three Native HA instances via a systemd timer, shells out to `du`/`df`/filesystem + `dspmq -o nativeha` for per-instance role, and renders a node_exporter textfile. The per-QM cockpit board (`src/mqlab/qmboard.py`) gains a time-series-led, instance-aware log-health band. A live-lab induce-and-assert playbook proves it end to end.

**Tech Stack:** Ansible roles; Python 3.12 stdlib (collector/board — no PyMQI, no compiled deps); node_exporter textfile collectors; Grafana/Loki cockpit; `runmqsc`/`dspmq`; pytest.

## Global Constraints

- **Collectors are stdlib-only / non-MQI** — shell out to CLIs (`du`, `df`, `ls`, `dspmq`); no PyMQI, no compiled deps. The `.py` file must deploy verbatim to the nodes *and* be importable by unit tests (the `nativehastate.py` invariant).
- **Fail loud, must-apply MQSC** — every `ALTER` that must apply asserts `rc 0` AND `AMQ8005I`; anything else is a real failure (the `mq-event-monitor` post-#721 discipline).
- **Declare-and-verify** — the live queue manager's actual log type is the authority; the declaration is checked against it and fails loud on drift, never trusted over it.
- **Dashboards lead with time-series** — trend panels first, not point-in-time tiles; `rate()` for counters (check `# TYPE`), gauges raw; verify metric names/types on a live exporter before wiring.
- **QM name is a variable, never hardcoded** — one source (e.g. `QM_NATIVE`), per the cockpit de-hardcoding direction.
- **Instance-aware** — Channel B collects on all three instances, tagged by instance + role (active/replica); the panel leads with the active but exposes per-replica divergence.
- **Validation** — `vrg-container-run -- vrg-validate` is the ONLY validation command; 100% branch coverage; `uv run pytest` for the dev loop only.
- **Scope guard** — only the Native HA arms are configured; circular arms (RDQM/PCMK/single) get only the verified negative. No manual archiving; alerting deferred.

---

### Task 1: Spike — verify the replicated-log lifecycle on a live Native HA arm (S1–S4)

De-risks every downstream task. Investigation task; deliverable is an engineering note, not code. **Its findings are applied forward** and can adjust Tasks 2–4.

**Files:**

- Create: `docs/reports/YYYY-MM-DD-nativeha-log-lifecycle-spike.md`

**Produces (consumed by Tasks 2–4):**

- **S1** — whether `ALTER QMGR LOGGEREV(ENABLED)` returns `AMQ8005I` on a replicated-log QM (Task 2 depends on this).
- **S2** — the exact logger events that fire under automatic log management and the extent watermarks they carry (Task 3/4 event element).
- **S3** — which log-health signals are pollable without MQI (filesystem extent naming/counts, `du`/`df` targets) vs. only via the event stream (Task 3 collector scope; Task 4 media-image element).
- **S4** — whether `LOGGEREV(DISABLED)` is accepted on a circular QM, deciding Task 2's circular branch (explicit `DISABLED` vs. verify-and-omit).

- [ ] **Step 1: Bring up (or reuse) a live Native HA arm and locate the active instance**

```bash
# from the lab host, against the running nha-rhel arm
mqlab lab status                 # confirm the nha arm is up
ssh nha-rhel-a1 "su - mqm -c '/opt/mqm/bin/dspmq -m $QM_NATIVE -o nativeha -x'"
# note which INSTANCE reports ROLE(Active)
```

- [ ] **Step 2: S1 — attempt to enable LOGGEREV on the active instance; capture verbatim output**

```bash
ssh <active> "su - mqm -c \"printf 'ALTER QMGR LOGGEREV(ENABLED)\n' | /opt/mqm/bin/runmqsc $QM_NATIVE\""
# record: rc, AMQ8005I (accepted) vs AMQ8518E/other (rejected)
ssh <active> "su - mqm -c \"printf 'DISPLAY QMGR LOGGEREV\n' | /opt/mqm/bin/runmqsc $QM_NATIVE\""
```

- [ ] **Step 3: S2 — provoke a logger event and inspect it**

```bash
# put/get load to roll an extent, then read the event queue with the sample browser
ssh <active> "su - mqm -c '/opt/mqm/samp/bin/amqsevt -m $QM_NATIVE -q SYSTEM.ADMIN.LOGGER.EVENT -o json' | head -40"
# record the event's fields: ArchiveLog/CurrentLog/MediaRecoveryLog/RestartRecoveryLog extent numbers,
# and whether a media-image event carries a timestamp
```

- [ ] **Step 4: S3 — enumerate non-MQI log-health signals on each instance**

```bash
for n in a1 a2 a3; do
  echo "== nha-rhel-$n =="
  ssh nha-rhel-$n "su - mqm -c 'ls -1 /var/mqm/qmgrs/$QM_NATIVE/active | head; df -P /var/mqm/qmgrs/$QM_NATIVE; du -sm /var/mqm/qmgrs/$QM_NATIVE/active'"
done
# record: log extent file naming/counting scheme, log dir path, disk fields — the collector's raw inputs.
# Note active-vs-replica differences.
```

- [ ] **Step 5: S4 — test the circular negative on a circular arm**

```bash
# against a circular QM (e.g. an rdqm or single arm)
ssh <circular-qm-host> "su - mqm -c \"printf 'ALTER QMGR LOGGEREV(DISABLED)\n' | /opt/mqm/bin/runmqsc $QM\""
# record: accepted (AMQ8005I) or rejected (AMQ8518E). Decides Task 2's circular branch.
```

- [ ] **Step 6: Write the spike note (S1–S4 answered) and commit**

Record each finding with the verbatim evidence. State explicitly, for each downstream task, what the finding pins down.

```bash
git add docs/reports/*-nativeha-log-lifecycle-spike.md
git commit -m "docs(report): Native HA log-lifecycle spike — S1-S4 findings (#145)"
```

---

### Task 2: Log-type-aware `LOGGEREV` (declare-and-verify)

**Files:**

- Modify: `ansible/roles/mq-event-monitor/tasks/service.yml:13-30` (the #721 note + the `ALTER QMGR` step)
- Modify: `ansible/roles/mq-event-monitor/defaults/main.yml` (default `mq_log_type: circular`)
- Modify: `ansible/site-nativeha.yml` and `ansible/site-nativeha-ubuntu.yml` (declare `mq_log_type: replicated` where `mq-event-monitor` is included, ~L102-105)

**Consumes:** S1 (enable accepted on replicated), S4 (circular branch form).
**Produces:** a `LOGGEREV`-correct event-monitor role; every arm asserts its choice.

- [ ] **Step 1: Add the declared default (safe default = circular)**

In `defaults/main.yml`:

```yaml
# Log type this arm's QM runs. Native HA arms override to 'replicated' (a linear-
# equivalent log, per IBM); everything else is circular. Drives LOGGEREV, which is
# only valid on a linear/replicated QM (#721 / #145).
mq_log_type: circular
```

- [ ] **Step 2: Declare `replicated` on the Native HA plays**

In `site-nativeha.yml` / `site-nativeha-ubuntu.yml`, at the `include_role: mq-event-monitor` block:

```yaml
      vars:
        qmgr_name: "{{ qm_name }}"
        mq_log_type: replicated
```

- [ ] **Step 3: Verify declared-vs-actual log type before acting (fail loud)**

Prepend a verify step in `service.yml` (run once on the active instance):

```yaml
- name: read the live queue manager's actual log type (#145)
  ansible.builtin.shell: |
    set -o pipefail
    printf "DISPLAY QMGR LOGTYPE\n" | /opt/mqm/bin/runmqsc {{ qmgr_name }}
  args: {executable: /bin/bash}
  become: true
  become_user: mqm
  register: qm_logtype
  changed_when: false

- name: assert the declared log type matches reality (#145)
  ansible.builtin.assert:
    that: >-
      (mq_log_type == 'replicated' and 'LOGTYPE(REPLICATED)' in qm_logtype.stdout)
      or (mq_log_type == 'circular' and 'LOGTYPE(CIRCULAR)' in qm_logtype.stdout)
    fail_msg: >-
      Declared mq_log_type={{ mq_log_type }} but the live QM reports
      {{ qm_logtype.stdout | regex_search('LOGTYPE\\([A-Z]+\\)') }} — drift; refusing to proceed.
```

> Confirm the exact `DISPLAY QMGR` attribute name for log type against S-findings; if `LOGTYPE` is not displayable, fall back to reading `qm.ini` `LogType` via a `become` `slurp`.

- [ ] **Step 4: Condition the `LOGGEREV` clause on log type**

Replace the fixed `ALTER QMGR ...` with a type-aware clause. Enabled arm (replicated):

```yaml
- name: enable instrumentation events, incl. LOGGEREV on replicated/linear arms (#514/#145)
  ansible.builtin.shell: |
    set -o pipefail
    printf "ALTER QMGR AUTHOREV(ENABLED) CHADEV(ENABLED) CHLEV(ENABLED) CONFIGEV(ENABLED) INHIBTEV(ENABLED) LOCALEV(ENABLED) PERFMEV(ENABLED) REMOTEEV(ENABLED) SSLEV(ENABLED) STRSTPEV(ENABLED){{ ' LOGGEREV(ENABLED)' if mq_log_type == 'replicated' else '' }} CMDEV(NODISPLAY)\n" \
      | /opt/mqm/bin/runmqsc {{ qmgr_name }}
  args: {executable: /bin/bash}
  become: true
  become_user: mqm
  register: evgate
  changed_when: "'AMQ8005' in evgate.stdout"
  failed_when: evgate.rc != 0 or 'AMQ8005' not in evgate.stdout
```

> If S4 shows `LOGGEREV(DISABLED)` is *accepted* on circular QMs, add it to the else-branch as an explicit verified negative; if rejected, the omission above **is** the verified negative (paired with the Step-3 assertion). Update the #721 comment block to state the resolved behaviour and remove "revisit if linear is ever adopted".

- [ ] **Step 5: Validate**

```bash
vrg-container-run -- vrg-validate      # ansible-lint + the whole gate
```

Expected: PASS (no `failed_when` regressions; lint clean).

- [ ] **Step 6: Commit**

```bash
git add ansible/roles/mq-event-monitor ansible/site-nativeha.yml ansible/site-nativeha-ubuntu.yml
git commit -m "feat(events): log-type-aware LOGGEREV via declare-and-verify (#145)"
```

*(Live acceptance — `AMQ8005I` on the nha arms, the drift assertion firing on a forced mismatch — is proven by the cold-rebuild + live-lab validation gates, not by pytest.)*

---

### Task 3: Log-health collector (`loglifecycle.py`) + deploy role

A **separate module** for clean separation (log-health vs. cluster-state, each testable alone, extraction-friendly per #79), but **deployed and scheduled by the existing `nativeha-state` role/timer** — no parallel role, no second timer — and it **reuses** `nativehastate.py`'s `dspmq -o nativeha` role-detection rather than re-implementing it (alignment decision, issue 1 → option A).

**Files:**

- Create: `src/mqlab/loglifecycle.py` (imports the role-detection helper from `mqlab.nativehastate`; no duplicate `dspmq` parsing)
- Create: `tests/test_loglifecycle.py`
- Modify: `ansible/roles/nativeha-state/tasks/main.yml` (also install the `lab-loglifecycle-state` script and have the existing timer's service invoke it — a small wrapper runs both collectors, each writing its own `.prom` textfile)
- Modify: `ansible/roles/nativeha-state/templates/lab-nativeha-state.service.j2` (run the log-health collector alongside the cluster-state one on the same fire)
- No new role, no new timer, no `observability.yml` change (the role already runs on the nha groups)

**Consumes:** S3 (pollable signals: log dir path, extent file scheme, disk fields); `nativehastate.py`'s role-detection helper (active/replica per instance).
**Produces (consumed by Task 4):** node_exporter metrics —
`mqlab_log_disk_used_bytes{qm,instance,role}`,
`mqlab_log_disk_total_bytes{qm,instance,role}`,
`mqlab_log_extents_active{qm,instance,role}`,
`mqlab_log_extents_inactive{qm,instance,role}`,
`mqlab_log_sample_stale{qm,instance,role}` (1 on timeout). Exact metric names are the contract Task 4 wires to.

- [ ] **Step 1: Write failing tests for the pure parse functions**

```python
# tests/test_loglifecycle.py
from mqlab.loglifecycle import parse_disk, count_extents, render_prom, instance_role

def test_parse_disk_from_df_posix():
    out = "Filesystem 1024-blocks Used Available Capacity Mounted\n/dev/vda2 51200000 20480000 30720000 40% /var/mqm\n"
    d = parse_disk(out)
    assert d["used_bytes"] == 20480000 * 1024
    assert d["total_bytes"] == 51200000 * 1024

def test_count_extents_splits_active_inactive():
    # S3 fixture: replace with the real extent naming scheme the spike records
    listing = ["S0000000.LOG", "S0000001.LOG", "S0000002.LOG"]
    c = count_extents(listing, active_marker="active")
    assert c["active"] >= 1

def test_instance_role_from_dspmq_nativeha():
    out = "QMNAME(QM1) INSTANCE(nha-rhel-a1) ROLE(Active)\n"
    assert instance_role(out, "nha-rhel-a1") == "active"

def test_render_prom_is_node_exporter_textfile():
    rows = {"qm": "QM1", "instance": "nha-rhel-a1", "role": "active",
            "used_bytes": 100, "total_bytes": 200, "active": 2, "inactive": 1, "stale": 0}
    text = render_prom(rows)
    assert 'mqlab_log_disk_used_bytes{qm="QM1",instance="nha-rhel-a1",role="active"} 100' in text
```

- [ ] **Step 2: Run tests, verify they fail**

```bash
uv run pytest tests/test_loglifecycle.py -v
```

Expected: FAIL (module not found).

- [ ] **Step 3: Implement `loglifecycle.py` (stdlib-only), following the `nativehastate.py` skeleton**

Structure: module docstring stating the deploy-verbatim + importable invariant; `parse_disk(df_out)`, `count_extents(listing, ...)` pure functions; **role comes from the shared helper** — `import` `nativehastate`'s `dspmq -o nativeha` parser and derive active/replica from it rather than re-parsing `dspmq` here (the `instance_role` in the Step-1 test is a thin wrapper over that helper, not a second parser); `probe()` running each source with a bounded `subprocess.run(..., timeout=...)` → STALE on timeout; `render_prom(rows)` → node_exporter textfile; `main()` argparse writing to its own `.prom` in the textfile dir. Log dir path / extent scheme from S3. No PyMQI, no third-party imports.

- [ ] **Step 4: Run tests to green + branch coverage**

```bash
uv run pytest tests/test_loglifecycle.py -v
# add cases until every branch (timeout->stale, missing-field, replica-vs-active) is covered
```

Expected: PASS, 100% branch coverage on the module.

- [ ] **Step 5: Extend the existing `nativeha-state` role to also run the log-health collector**

No new role, no new timer. In `ansible/roles/nativeha-state/tasks/main.yml`, deploy `loglifecycle.py` to the nodes as `/usr/local/bin/lab-loglifecycle-state` alongside the existing `lab-nativeha-state`. In `lab-nativeha-state.service.j2`, run both collectors on the same fire (a small wrapper, or a second `ExecStart=`), each writing its **own** `.prom` textfile. This lands on **every** nha instance (not just active) because the role already targets all three. Confirm the existing timer cadence (~5s) suits log-health sampling.

- [ ] **Step 6: Validate + commit**

```bash
vrg-container-run -- vrg-validate
git add src/mqlab/loglifecycle.py tests/test_loglifecycle.py ansible/roles/nativeha-state
git commit -m "feat(obs): non-MQI log-health collector via the nativeha-state role (#145)"
```

---

### Task 4: Cockpit log-health panel group (extend `qmboard.py`)

**Files:**

- Modify: `src/mqlab/qmboard.py` (add a log-health band builder; wire into `render_qm_board`, ~L342)
- Modify: `tests/test_qmboard.py`

**Consumes:** Task 3 metric names (`mqlab_log_*`); S2 (event fields for the media-image element).
**Produces:** the log-health band in the per-QM board JSON for the Native HA arms.

- [ ] **Step 1: Write failing tests for the band's PromQL + panel shape**

```python
# tests/test_qmboard.py (add)
def test_log_health_band_uses_instance_series():
    from mqlab.qmboard import _log_health_band
    panels = _log_health_band("ds", "QM1", y=800)
    joined = str(panels)
    assert "mqlab_log_disk_used_bytes" in joined
    assert "mqlab_log_extents_active" in joined
    assert 'by (instance' in joined or 'instance=' in joined   # instance-aware
    assert panels[0]["type"] in ("timeseries",)                 # time-series-led, not stat
```

- [ ] **Step 2: Run tests, verify fail**

```bash
uv run pytest tests/test_qmboard.py -k log_health -v
```

Expected: FAIL (`_log_health_band` undefined).

- [ ] **Step 3: Implement `_log_health_band`, following the existing `_qm_band` / `_queue_block` builders**

Time-series-led panels: log-disk % (used/total, `by (instance)`), active/inactive extent counts over time, fill trend; a media-image-recency panel **only if S2/S3 found a source** (else omit, degrading gracefully); reuse `_events_panel(loki_uid, ...)` filtered to logger events for the event log. QM name stays the `qm` parameter — no literals. Wire the band into `render_qm_board` behind the arm being Native HA.

- [ ] **Step 4: Run tests to green + coverage**

```bash
uv run pytest tests/test_qmboard.py -v
```

Expected: PASS, branch coverage restored to 100%.

- [ ] **Step 5: Verify metric names/types on a live exporter, then regenerate boards**

```bash
# against a live nha exporter, confirm names + # TYPE before trusting rate()/gauge choices
curl -s <nha-exporter>/metrics | grep -E '^# TYPE mqlab_log_|^mqlab_log_'
mqlab <board-render-cmd>        # regenerate the per-QM dashboards
```

- [ ] **Step 6: Validate + commit**

```bash
vrg-container-run -- vrg-validate
git add src/mqlab/qmboard.py tests/test_qmboard.py
git commit -m "feat(cockpit): time-series log-health band on the per-QM board (#145)"
```

---

### Task 5: Live-lab induce-and-assert validation playbook (under epic #38)

**Files:**

- Create: `ansible/validate-nativeha-log-lifecycle.yml` (or the framework's `validate <system>` convention from #38)

**Consumes:** Tasks 2–4 (LOGGEREV on, collector emitting, panel wired).
**Produces:** a repeatable, human-out-of-the-loop proof.

- [ ] **Step 1: Induce log churn / force an extent roll on the active instance**

A play step that drives enough put/get (or an explicit `runmqsc` action) to roll at least one log extent on `$QM_NATIVE`'s active instance.

- [ ] **Step 2: Assert the logger event fired**

Poll the event stream (journald / Loki, where `mq-event-monitor` lands events) for the logger event within a bounded window; `fail_msg` if absent.

- [ ] **Step 3: Assert the collector reflected it**

Scrape the nha exporter and assert `mqlab_log_extents_active` (or the disk/extent series) moved as expected on the active instance and that all three instances report a fresh (non-STALE) sample; `fail_msg` otherwise.

- [ ] **Step 4: Negative path — prove the declare-and-verify guardrail fails loud on drift (spec §7)**

Run the `mq-event-monitor` verify step against a **deliberately wrong** declaration (e.g. `mq_log_type: circular` forced on the replicated `$QM_NATIVE`) and assert the play **aborts** with the Task-2 drift message — an assertion that never fires is a dead guardrail. Restore the correct declaration afterward.

```yaml
- name: drift guardrail must fail loud (negative path, #145)
  block:
    - ansible.builtin.include_role:
        name: mq-event-monitor
      vars: {qmgr_name: "{{ qm_name }}", mq_log_type: circular}   # intentionally wrong
    - ansible.builtin.fail:
        msg: "drift guardrail did NOT fire — declare-and-verify is broken"
  rescue:
    - ansible.builtin.debug:
        msg: "drift guardrail fired as expected"
```

- [ ] **Step 5: Validate + commit**

```bash
vrg-container-run -- vrg-validate
git add ansible/validate-nativeha-log-lifecycle.yml
git commit -m "test(live-lab): induce-and-assert Native HA log-lifecycle observability (#145, #38)"
```

---

### Task 6: Operator runbook (versioned site docs)

**Files:**

- Create: `docs/site/docs/guides/nativeha-log-lifecycle-guide.md`
- Modify: the site nav/index that lists guides

**Consumes:** all prior tasks + the spike note.
**Produces:** the teaching-oriented operator guide (the doc-review bookend, #806, verifies this landed).

- [ ] **Step 1: Write the guide**

Cover: the three log types and why Native HA is replicated (= linear + automatic log management + automatic media images); what the automation does (extent create/delete/reuse, AMQ7490; media images); how to read the log-health band (disk %, extent counts, media-image recency, the logger-event log) and what healthy vs. divergent looks like per instance; and the one real constraint — **automatic log management without archiving → no backup queue manager** (and why Native HA makes that moot). Link the spike note for the verified specifics.

- [ ] **Step 2: Validate + commit**

```bash
vrg-container-run -- vrg-validate     # includes the docs/site build/lint
git add docs/site
git commit -m "docs(site): operator runbook for the Native HA log lifecycle (#145)"
```

---

## Operational gates (seeded at epic level — Task-9 of epic-create)

- **Cold-rebuild validation** — the Task-2 role change + the Task-3 collector come up **one-pass** on a cold-rebuilt Native HA arm (LOGGEREV accepted, collector emitting on all three instances, timer healthy). The lab's standing cold-rebuild acceptance gate.
- **Live-lab validation run** — execute Task 5's playbook on the running lab; `Outcome: SUCCESS`.

## Self-Review

- **Spec coverage:** §4.1 spike → Task 1; §4.2 declare-and-verify LOGGEREV → Task 2, with its "forced drift fails loud" acceptance (§7) exercised by Task 5 Step 4; §4.3 Channel A → Task 2 (existing `amqsevt` gains the class) + Channel B collector → Task 3 (separate module, deployed by the existing `nativeha-state` role/timer, reusing its `dspmq` role-detection) + panel → Task 4, incl. instance scoping (Task 3 all-three + role tag; Task 4 instance-aware series) and spike-gated media-image recency (Task 4 Step 3); §4.4 validation → Task 5; §4.5 runbook → Task 6; §7 cold-rebuild + live-lab → operational gates. No uncovered requirement.
- **Placeholders:** the two deliberately spike-gated unknowns (exact `DISPLAY QMGR` log-type attribute; extent file naming scheme) are explicitly routed to Task 1 findings with a named fallback, not left vague.
- **Type consistency:** the `mqlab_log_*{qm,instance,role}` metric names are defined in Task 3's Produces and consumed verbatim in Task 4's tests; `_log_health_band` / `parse_disk` / `count_extents` / `instance_role` / `render_prom` names are consistent across their tasks.
