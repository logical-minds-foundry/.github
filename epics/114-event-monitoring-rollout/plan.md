# Event Monitoring as a De-Facto Standard on Every Queue Manager — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make MQ instrumentation event monitoring (`#31`) a de-facto standard on every queue manager across every arm, by factoring the enable (`#514`) + collector (`#515`) into one shared role and including it from every QM-creation path.

**Architecture:** Split the existing `mq-event-monitor` role into two halves — a **host-prep half** (assert deps + install the `run.sh` wrapper) that runs on *every* node a QM can fail over to, and an **MQSC half** (enable event classes + define/start the `MQ.EVENT.MONITOR` SERVICE) that runs *once on the active instance*. This mirrors how `mq-diag-logging` is already consumed (`tasks_from: system` on all nodes vs `tasks_from: qmini` per-QM on the active node). Each arm already resolves its active node, so the MQSC half hangs off that existing seam. `mq-qmgr` (the standalone SVCQM) includes both halves on its single host and drops its now-duplicated inline `#514` block.

**Tech Stack:** Ansible (roles + `include_role`/`tasks_from`), IBM MQ 9.4 MQSC (`runmqsc`, `amqsevt` SERVICE), systemd journald + `logger -t mq-events`, the existing alloy→Loki→Grafana pipeline. Validation: `vrg-container-run -- vrg-validate`.

## Global Constraints

- **Validation is one command only:** `vrg-container-run -- vrg-validate`. Do not run individual linters/formatters. (repo CLAUDE.md)
- **Git/GitHub via wrappers:** `vrg-git`, `vrg-gh`, commit with `vrg-commit --type <t> --scope <s> --message <m>`. Raw `git`/`gh` are denied. No direct commits to `develop`/`main`; work on `feature/<issue>-<slug>`.
- **Implementation repo:** `logical-minds-foundry/mq-resiliency-lab-for-linux`. All paths below are relative to it.
- **The collector is unchanged in mechanism** — `amqsevt -o json` in bindings mode as a `CONTROL(QMGR) SERVTYPE(SERVER)` SERVICE draining `SYSTEM.ADMIN.*.EVENT` to `logger -t mq-events`. This epic changes only *which QMs run it*.
- **Wrapper on every failover-capable node; MQSC once on the active node.** The `run.sh` wrapper (`STARTCMD`) must exist on every node the QM can be active on, or the SERVICE fails to start after failover. The MQSC (enable + define) is a QM-object operation — run it once on the active instance; it replicates (raft / DRBD / shared-LUN) and travels.
- **Event classes verbatim (`#514`):** `AUTHOREV(ENABLED) CHADEV(ENABLED) CHLEV(ENABLED) CONFIGEV(ENABLED) INHIBTEV(ENABLED) LOCALEV(ENABLED) LOGGEREV(ENABLED) PERFMEV(ENABLED) REMOTEEV(ENABLED) SSLEV(ENABLED) STRSTPEV(ENABLED) CMDEV(NODISPLAY)`. `CMDEV(NODISPLAY)` is load-bearing (avoids the exporter's DISPLAY/Inquire-PCF poll-flood); `BRIDGEEV` deliberately absent (z/OS-only).
- **Live verification is per distinct QM-creation mechanism**, not all four stacks: one `mq-nativeha` arm live (the twin confirmed by config-inclusion), `mq-pcmk-qmgr` live, the rdqm path live. `SVCQM` is already live.
- **Idempotency:** every added task must be safe to re-run (a re-provision must not fail or thrash). Follow the existing `changed_when`/`failed_when: rc not in [0, 10]` conventions.

---

### Task 1: Split `mq-event-monitor`; move `#514` in; refactor `mq-qmgr` to include it

Establish the consolidated role the other tasks include. After this task the role has two entry points (`main` = host-prep, `service` = MQSC), and `mq-qmgr` carries no private event logic. **`SVCQM` behaviour is unchanged** — the acceptance gate.

**Files:**
- Modify: `ansible/roles/mq-event-monitor/tasks/main.yml` (reduce to host-prep)
- Create: `ansible/roles/mq-event-monitor/tasks/service.yml` (the MQSC half)
- Modify: `ansible/roles/mq-event-monitor/defaults/main.yml` (add `mq_event_perf_queues`)
- Modify: `ansible/roles/mq-qmgr/tasks/main.yml` (delete inline `#514`; include both halves)
- Modify: `ansible/roles/mq-qmgr/defaults/main.yml` (remove the moved default)

**Interfaces:**
- Produces: `mq-event-monitor` with two entry points. `include_role: {name: mq-event-monitor}` (default `main`) = **host-prep**: asserts `amqsevt` + the journald drop-in present, installs `{{ mq_event_run_dir }}/run.sh`. `include_role: {name: mq-event-monitor, tasks_from: service}` = **MQSC**: enables the event classes on `{{ qmgr_name }}`, sets per-queue perf on `{{ mq_event_perf_queues }}`, defines + starts `{{ mq_event_service_name }}`. Both require `qmgr_name` in scope; `service` also reads `mq_event_perf_queues` (default `[]`).
- Consumes: nothing new — `mq-diag-logging` (the drop-in) must already have run on the node.

- [ ] **Step 1: Move the `mq_event_perf_queues` default into the role**

Add to `ansible/roles/mq-event-monitor/defaults/main.yml` (it currently ends after `mq_event_service_name`):

```yaml
# Application queues that should emit performance events (queue depth high / full).
# Empty by default: PERFMEV is enabled at the QMGR, but per-queue QDPMAXEV/QDPHIEV
# thresholds only make sense on real app queues, so each arm sets its own list at the
# include call site (e.g. ["PCMK.SVC.REQUEST"]). Empty = safe no-op.
mq_event_perf_queues: []
```

Delete the same block (the `mq_event_perf_queues: []` definition and its comment) from `ansible/roles/mq-qmgr/defaults/main.yml`.

- [ ] **Step 2: Create the MQSC half `service.yml`**

Create `ansible/roles/mq-event-monitor/tasks/service.yml` by moving the enable + per-queue-perf tasks out of `mq-qmgr` and adding the service define/start (moved from the current `main.yml`). Full file:

```yaml
---
# MQ event monitoring — QM-level MQSC (#514 enable + #515 SERVICE define). Runs ONCE on
# the active instance; the objects replicate (raft / DRBD / shared LUN) and travel with
# the QM. Host-prep (main.yml: wrapper + asserts) must have run on every failover-capable
# node first. Requires: qmgr_name; optional mq_event_perf_queues.
- name: enable instrumentation events (#514)
  ansible.builtin.shell: |
    set -o pipefail
    printf "ALTER QMGR AUTHOREV(ENABLED) CHADEV(ENABLED) CHLEV(ENABLED) CONFIGEV(ENABLED) INHIBTEV(ENABLED) LOCALEV(ENABLED) LOGGEREV(ENABLED) PERFMEV(ENABLED) REMOTEEV(ENABLED) SSLEV(ENABLED) STRSTPEV(ENABLED) CMDEV(NODISPLAY)\n" \
      | /opt/mqm/bin/runmqsc {{ qmgr_name }}
  args:
    executable: /bin/bash
  become: true
  become_user: mqm
  register: evgate
  changed_when: "'AMQ8005' in evgate.stdout"
  failed_when: evgate.rc not in [0, 10]

- name: enable per-queue performance events on the app queue(s) (#514)
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

# Define the service and start it now. CONTROL(QMGR) auto-starts it on future QM starts.
# STOPCMD kills MQ_SERVER_PID (the started process = amqsevt, since run.sh execs it) so
# QM shutdown stops the collector cleanly.
- name: define + start the MQ event-monitor service (idempotent)
  ansible.builtin.shell: |
    set -o pipefail
    printf "DEFINE SERVICE(%s) REPLACE CONTROL(QMGR) SERVTYPE(SERVER) STARTCMD('%s/run.sh') STARTARG('%s') STOPCMD('/bin/kill') STOPARG('MQ_SERVER_PID') DESCR('Drain SYSTEM.ADMIN.*.EVENT to JSON on journald (mq-events)')\nSTART SERVICE(%s)\n" \
      "{{ mq_event_service_name }}" "{{ mq_event_run_dir }}" "{{ qmgr_name }}" "{{ mq_event_service_name }}" \
      | /opt/mqm/bin/runmqsc {{ qmgr_name }}
  args:
    executable: /bin/bash
  become: true
  become_user: mqm
  register: svc
  changed_when: svc.rc == 0
  failed_when: svc.rc not in [0, 10]
```

- [ ] **Step 3: Reduce `main.yml` to host-prep only**

Rewrite `ansible/roles/mq-event-monitor/tasks/main.yml` to keep the asserts + wrapper install and **drop** the trailing `define + start` shell task (now in `service.yml`). The file becomes:

```yaml
---
# MQ event monitoring — host prep (#515). Runs on EVERY node the QM can fail over to:
# assert the collector's deps and install the run.sh wrapper. The QM-level MQSC (enable +
# SERVICE define) is service.yml, run once on the active instance. Requires: qmgr_name.
- name: assert the amqsevt sample binary is present
  ansible.builtin.stat:
    path: "{{ mq_event_amqsevt_bin }}"
  register: amqsevt_bin

- name: fail loudly if amqsevt is missing
  ansible.builtin.fail:
    msg: >-
      {{ mq_event_amqsevt_bin }} not found — the MQ samples (amqsevt) must be installed on
      the QM host for the event collector to run.
  when: not amqsevt_bin.stat.exists

- name: assert the journald rate-limit drop-in is present (no silent event loss)
  ansible.builtin.stat:
    path: /etc/systemd/journald.conf.d/10-mq.conf
  register: rl_dropin

- name: fail loudly if the rate-limit drop-in is missing
  ansible.builtin.fail:
    msg: >-
      /etc/systemd/journald.conf.d/10-mq.conf is absent — mq-diag-logging must run before
      mq-event-monitor, else 'everything on' event bursts are silently rate-dropped by journald.
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
```

- [ ] **Step 4: Refactor `mq-qmgr` to include both halves; delete its inline `#514`**

In `ansible/roles/mq-qmgr/tasks/main.yml`, replace the whole block from `# Enable ALL Multiplatforms instrumentation-event classes (#514…` through the final `include_role: name: mq-event-monitor` (the enable task, the per-queue-perf task, their comments, and the existing collector include — current lines 78–119) with:

```yaml
# Event monitoring (#514 enable + #515 collector), the lab-wide standard (epic .github#114).
# Host-prep (wrapper + dep asserts) then the QM-level MQSC. SVCQM is a single standalone
# host, so both halves run here together.
- name: event monitoring — host prep (wrapper + asserts)
  ansible.builtin.include_role:
    name: mq-event-monitor

- name: event monitoring — enable classes + define/start collector
  ansible.builtin.include_role:
    name: mq-event-monitor
    tasks_from: service
```

(`qmgr_name` and any `mq_event_perf_queues` set at the `mq-qmgr` call site remain in scope for the includes.)

- [ ] **Step 5: Static validation**

```bash
vrg-container-run -- vrg-validate
```

Expected: PASS (ansible-lint + yamllint clean; no undefined-var or role-structure errors).

- [ ] **Step 6: Live verify SVCQM is unchanged (svc-sim is up)**

Re-provision the SVC layer and confirm `SVCQM` still ends up fully instrumented. From the repo root:

```bash
cd ansible && uv run --project .. ansible-playbook site-distributed-shared.yml --limit svc
```

Then assert state on `svc-sim`:

```bash
uv run --project .. ansible svc -b -m shell -a "su - mqm -c 'echo \"DISPLAY QMGR AUTHOREV CMDEV\" | runmqsc SVCQM; echo \"DISPLAY SVSTATUS(MQ.EVENT.MONITOR)\" | runmqsc SVCQM'"
```

Expected: `AUTHOREV(ENABLED)`, `CMDEV(NODISPLAY)`, `SERVICE(MQ.EVENT.MONITOR) STATUS(RUNNING)` — identical to pre-refactor. If events were previously toggled off on SVCQM (observed drift), they are now re-enabled by this run; that is the correct end state.

- [ ] **Step 7: Commit + report ready**

```bash
vrg-git add ansible/roles/mq-event-monitor ansible/roles/mq-qmgr
vrg-commit --type refactor --scope events \
  --message "split mq-event-monitor into host-prep + service halves; mq-qmgr includes it (#114)" \
  --body "Consolidate #514 enable + #515 collector into the shared mq-event-monitor role (main=host-prep wrapper+asserts, service=MQSC enable+define). mq-qmgr drops its inline #514 block and includes both halves. Single source of truth for the arms to consume. SVCQM unchanged. Refs #114."
vrg-pr-workflow report-ready --issue <T1_ISSUE> --title "refactor(events): consolidate event monitoring into the mq-event-monitor role (#114)" --summary "Split mq-event-monitor into host-prep + MQSC halves and refactor mq-qmgr to include it, removing the duplicated inline #514 block; SVCQM behaviour unchanged." --notes "No functional change to SVCQM; establishes the role the arm tasks (T2–T4) include. Live-verified on svc-sim."
```

---

### Task 2: Wire event monitoring into `mq-nativeha` (nativeha-rhel + nativeha-ubuntu)

**Blocked-by T1.** `mq-nativeha` is OS-agnostic, so one host-prep include covers both arms; the MQSC half hangs off each arm's existing active-instance MQSC block.

**Files:**
- Modify: `ansible/roles/mq-nativeha/tasks/main.yml` (host-prep include, per node)
- Modify: `ansible/site-nativeha.yml` (MQSC include at the active-instance block)
- Modify: `ansible/site-nativeha-ubuntu.yml` (same, the Debian twin)

**Interfaces:**
- Consumes: `mq-event-monitor` (`main` + `service`) from T1; each arm's active-instance resolution (`site-nativeha.yml` "find the active Native HA instance" + the `run_once` MQSC block).
- Produces: `NHARAPP` and `NHAUAPP` fully instrumented; wrapper present on all six `nha-rhel-*` / `nha-ubuntu-*` nodes.

- [ ] **Step 1: Install the wrapper on every Native HA node (host-prep)**

In `ansible/roles/mq-nativeha/tasks/main.yml`, after the existing `seed JSON diagnostic logging … (#282)` include (which guarantees the drop-in dep), add — this role runs on every node in the group, so the wrapper lands everywhere the QM can activate:

```yaml
- name: event monitoring — host prep (wrapper on every failover-capable node) (#114)
  ansible.builtin.include_role:
    name: mq-event-monitor
  vars:
    qmgr_name: "{{ qm_name }}"
```

- [ ] **Step 2: Define + enable on the active instance (once, raft-replicated)**

In `ansible/site-nativeha.yml`, inside the block that resolves the active instance and runs `apply our-side + app MQSC on the active instance (once; raft-replicated)` (the `run_once: true` MQSC play), add a task immediately after that MQSC task:

```yaml
    - name: event monitoring — enable classes + define/start collector (active, once) (#114)
      ansible.builtin.include_role:
        name: mq-event-monitor
        tasks_from: service
      vars:
        qmgr_name: "{{ qm_name }}"
      run_once: true
```

- [ ] **Step 3: Mirror on the Ubuntu twin**

Apply the identical Step-2 change to `ansible/site-nativeha-ubuntu.yml` at its corresponding active-instance MQSC block (same task, same `qmgr_name: "{{ qm_name }}"`, same `run_once: true`).

- [ ] **Step 4: Static validation**

```bash
vrg-container-run -- vrg-validate
```

Expected: PASS.

- [ ] **Step 5: Live verify on the running nativeha-rhel stack**

The nativeha-rhel stack is up (`NHARAPP`, active node floats across `nha_rhel_a`). Re-run the nativeha provision, then assert:

```bash
cd ansible
uv run --project .. ansible-playbook site-nativeha.yml
# find the active instance, then check it:
uv run --project .. ansible nha_rhel_a -b -m shell -a "su - mqm -c 'dspmq -o nativeha -x -m NHARAPP' 2>/dev/null | grep -q \"ROLE(Active)\" && { echo \"DISPLAY QMGR AUTHOREV CMDEV\" | runmqsc NHARAPP; echo \"DISPLAY SVSTATUS(MQ.EVENT.MONITOR)\" | runmqsc NHARAPP; }"
uv run --project .. ansible nha_rhel_a -b -m shell -a "journalctl -t mq-events -n 3 --no-pager -o cat"
```

Expected: on the active node, `AUTHOREV(ENABLED)`, `CMDEV(NODISPLAY)`, `SERVICE(MQ.EVENT.MONITOR) STATUS(RUNNING)`; `journalctl -t mq-events` shows JSON. (nativeha-ubuntu is confirmed by config-inclusion — same role/seam.)

Then prove the **full path to Loki/Grafana** end-to-end (spec acceptance — done once here, since nativeha is the live arm). Force an authority event and confirm it lands in the `mq-events` Loki stream on obs:

```bash
# force a Not-Authorized (MQRC 2035) event as an unprivileged OS user on the active node
uv run --project .. ansible nha_rhel_a -b -m shell -a "id -u nobody >/dev/null && runuser -u nobody -- /opt/mqm/samp/bin/amqsputc NOSUCHQ NHARAPP 2>&1 | grep -i 2035 || true"
sleep 5
# confirm it reached Loki (alloy -> Loki on obs); resolve the obs Loki endpoint from the inventory
OBS=$(awk '/^\[obs\]/{f=1;next} f&&/ansible_host/{print $2;exit}' ../build/work/inventory.ini | sed 's/ansible_host=//')
uv run --project .. ansible mon-probe -m shell -a "curl -sG 'http://${OBS:-obs}:3100/loki/api/v1/query_range' --data-urlencode 'query={unit=\"mq-events\"}' --data-urlencode 'limit=5' | grep -o 'authorityEvent\|AuthorityInfo\|2035\|eventType' | head" 2>&1 | tail -8
```

Expected: the forced event's JSON is present in the Loki `{unit="mq-events"}` stream — proving QM → collector → journald → alloy → Loki → Grafana end to end. (If the obs Loki port/endpoint differs, resolve it from the running obs config; the stream label is `unit="mq-events"` per `#516`.)

- [ ] **Step 6: Commit + report ready**

```bash
vrg-git add ansible/roles/mq-nativeha ansible/site-nativeha.yml ansible/site-nativeha-ubuntu.yml
vrg-commit --type feat --scope events \
  --message "event monitoring on the Native HA arms (rhel + ubuntu) (#114)" \
  --body "Wrapper on every nha node via mq-nativeha; enable + collector define once on the active instance (raft-replicated) in site-nativeha.yml and its ubuntu twin. Refs #114."
vrg-pr-workflow report-ready --issue <T2_ISSUE> --title "feat(events): event monitoring on the Native HA arms (#114)" --summary "Include mq-event-monitor host-prep on every Native HA node and the MQSC half once on the active instance, for both nativeha-rhel and nativeha-ubuntu." --notes "Live-verified on nativeha-rhel (NHARAPP); nativeha-ubuntu covered by the shared OS-agnostic role."
```

---

### Task 3: Wire event monitoring into `mq-pcmk-qmgr` (pcmk-ubuntu)

**Blocked-by T1.** The Pacemaker arm runs the QM on a shared LUN owned by one node; the role runs on all `pcmk_a` nodes. Wrapper on all nodes; MQSC in the existing `run_once` first-time creation block (on the LUN).

**Files:**
- Modify: `ansible/roles/mq-pcmk-qmgr/tasks/main.yml`

**Interfaces:**
- Consumes: `mq-event-monitor` (T1); the role's existing `run_once` first-time `apply MQSC config` block (where the QM is started on the LUN owner and MQSC applied).
- Produces: `PCMKAPP` instrumented; wrapper on all `pcmk_a` nodes.

- [ ] **Step 1: Install the wrapper on every pcmk node (host-prep)**

In `ansible/roles/mq-pcmk-qmgr/tasks/main.yml`, after the existing `seed JSON diagnostic logging (mqs.ini template + journald)` include near the top (the drop-in dep), add (this role runs on all `pcmk_a` nodes):

```yaml
- name: event monitoring — host prep (wrapper on every pcmk node) (#114)
  ansible.builtin.include_role:
    name: mq-event-monitor
  vars:
    qmgr_name: "{{ qm_name }}"
```

- [ ] **Step 2: Define + enable on the LUN owner (inside the first-time creation block)**

In the `when: hostvars[groups['pcmk_a'][0]].mq_fs_owned.rc != 0` block, after the `apply MQSC config` task (which `strmqm`s the QM and applies listener/channels, `run_once: true`) and before the block's clean stop of the QM, add:

```yaml
    - name: event monitoring — enable classes + define/start collector (owner, once) (#114)
      ansible.builtin.include_role:
        name: mq-event-monitor
        tasks_from: service
      vars:
        qmgr_name: "{{ qm_name }}"
      run_once: true
```

The SERVICE object lands in the QM on the shared LUN and follows it on failover; the wrapper (Step 1) is present on whichever node Pacemaker starts it on.

- [ ] **Step 3: Static validation**

```bash
vrg-container-run -- vrg-validate
```

Expected: PASS.

- [ ] **Step 4: Live verify on a brought-up pcmk-ubuntu stack**

pcmk-ubuntu is not currently running — bring it up, then verify:

```bash
uv run mqlab bootstrap pcmk-ubuntu
cd ansible
# PCMKAPP runs on the pacemaker resource owner; find it and check:
uv run --project .. ansible pcmk_a -b -m shell -a "su - mqm -c 'dspmq -m PCMKAPP' 2>/dev/null | grep -q Running && { echo \"DISPLAY QMGR AUTHOREV CMDEV\" | runmqsc PCMKAPP; echo \"DISPLAY SVSTATUS(MQ.EVENT.MONITOR)\" | runmqsc PCMKAPP; journalctl -t mq-events -n 3 --no-pager -o cat; }"
```

Expected: `AUTHOREV(ENABLED)`, `CMDEV(NODISPLAY)`, `SERVICE(MQ.EVENT.MONITOR) STATUS(RUNNING)`, JSON in `mq-events`. Optionally verify failover: move the resource and confirm the SERVICE comes up on the new owner (wrapper present there).

- [ ] **Step 5: Commit + report ready**

```bash
vrg-git add ansible/roles/mq-pcmk-qmgr
vrg-commit --type feat --scope events \
  --message "event monitoring on the pcmk-ubuntu arm (#114)" \
  --body "Wrapper on every pcmk_a node; enable + collector define once on the LUN owner inside the first-time creation block. SERVICE follows the QM across Pacemaker failover. Refs #114."
vrg-pr-workflow report-ready --issue <T3_ISSUE> --title "feat(events): event monitoring on the pcmk-ubuntu arm (#114)" --summary "Include mq-event-monitor host-prep on every pcmk node and the MQSC half once on the LUN owner, so PCMKAPP is instrumented and the collector follows failover." --notes "Live-verified on a brought-up pcmk-ubuntu stack, including a failover check."
```

---

### Task 4: Wire event monitoring into the rdqm path (rdqm-rhel) — the disproportionate-effort task

**Blocked-by T1. Sequenced last, deliberately.** Unlike T2/T3 the rdqm QM-creation seam is **script-based** (`rdqm-qm-create.sh` + `site-rdqm.yml` includes) and was recently rewritten to IBM's coordinated one-`crtmqm`-per-site model (`#561`, `#582`, `#559`, `#591`, `#593`). Do **not** assume it mirrors the role-based arms — validate every step against the current `site-rdqm.yml`.

**Files:**
- Modify: `ansible/site-rdqm.yml` (host-prep on all `rdqm_a`; MQSC gated `when: rdqm_is_active`)

**Interfaces:**
- Consumes: `mq-event-monitor` (T1); the `rdqm-active-node` role (sets `rdqm_is_active`), whose `when: rdqm_is_active` gate already carries QM-level includes (e.g. `mq-diag-logging tasks_from: qmini`).
- Produces: `RDQMAPP` instrumented; wrapper on all `rdqm_a` nodes.

- [ ] **Step 1: Confirm the current active-node seam**

Read `ansible/site-rdqm.yml` and confirm: (a) the play(s) with `hosts: rdqm_a`, (b) the `resolve the active RDQM node + its DataPath` include (`rdqm-active-node`), and (c) the `when: rdqm_is_active` QM-level includes (the `mq-diag-logging tasks_from: qmini` task is the template to mirror). The event MQSC include goes in the same play, after the QM and its base MQSC exist, gated `when: rdqm_is_active`.

```bash
grep -nE "hosts: rdqm_a|rdqm-active-node|rdqm_is_active|tasks_from: qmini|qm_app" ansible/site-rdqm.yml
```

- [ ] **Step 2: Install the wrapper on every rdqm node (host-prep)**

In `ansible/site-rdqm.yml`, in a play targeting `hosts: rdqm_a` that runs after MQ is installed (the same play family that resolves the active node), add a host-prep include that runs on **all** nodes (no `rdqm_is_active` gate — every node can become primary):

```yaml
    - name: event monitoring — host prep (wrapper on every rdqm node) (#114)
      ansible.builtin.include_role:
        name: mq-event-monitor
      vars:
        qmgr_name: "{{ qm_app }}"
```

- [ ] **Step 3: Define + enable on the active RDQM node (once)**

In the play that has already run `resolve the active RDQM node + its DataPath` (so `rdqm_is_active` is set), after the QM's base + inter-QM MQSC exists, add — mirroring the `mq-diag-logging tasks_from: qmini` gating:

```yaml
    - name: event monitoring — enable classes + define/start collector (active RDQM node) (#114)
      ansible.builtin.include_role:
        name: mq-event-monitor
        tasks_from: service
      vars:
        qmgr_name: "{{ qm_app }}"
      when: rdqm_is_active
```

The SERVICE object lands in `RDQMAPP` on the DRBD-replicated DataPath and follows the QM across HA/DR moves; the wrapper (Step 2) is on every node.

- [ ] **Step 4: Static validation**

```bash
vrg-container-run -- vrg-validate
```

Expected: PASS.

- [ ] **Step 5: Live verify on a brought-up rdqm-rhel stack**

rdqm-rhel is not currently running — bring it up, then verify on the active primary:

```bash
uv run mqlab bootstrap rdqm-rhel
cd ansible
uv run --project .. ansible rdqm_a -b -m shell -a "/opt/mqm/bin/rdqmstatus -m RDQMAPP 2>/dev/null | grep -qi 'Running' && { su - mqm -c 'echo \"DISPLAY QMGR AUTHOREV CMDEV\" | runmqsc RDQMAPP; echo \"DISPLAY SVSTATUS(MQ.EVENT.MONITOR)\" | runmqsc RDQMAPP'; journalctl -t mq-events -n 3 --no-pager -o cat; }"
```

Expected: `AUTHOREV(ENABLED)`, `CMDEV(NODISPLAY)`, `SERVICE(MQ.EVENT.MONITOR) STATUS(RUNNING)`, JSON in `mq-events`. Verify failover: `rdqmadm`/move the QM to another node and confirm the SERVICE starts there (wrapper present).

- [ ] **Step 6: Commit + report ready**

```bash
vrg-git add ansible/site-rdqm.yml
vrg-commit --type feat --scope events \
  --message "event monitoring on the rdqm-rhel arm (#114)" \
  --body "Wrapper on every rdqm_a node; enable + collector define once on the active RDQM node (when: rdqm_is_active), landing the SERVICE on the DRBD-replicated DataPath so it travels on HA/DR moves. Refs #114."
vrg-pr-workflow report-ready --issue <T4_ISSUE> --title "feat(events): event monitoring on the rdqm-rhel arm (#114)" --summary "Add the mq-event-monitor host-prep on every rdqm node and the MQSC half on the active RDQM node, completing event-monitoring coverage across every arm." --notes "Live-verified on a brought-up rdqm-rhel stack incl. failover. Completes the #114 rollout; unblocks #110."
```

---

## Self-Review

**Spec coverage:**
- Consolidated `mq-event-monitor` role (spec 3.1) → T1 Steps 1–3. ✅
- De-dup `mq-qmgr`, SVCQM unchanged (spec 3.2 / acceptance) → T1 Steps 4, 6. ✅
- Wire into every seam (spec 3.2 / scope table) → T2 (nativeha ×2), T3 (pcmk), T4 (rdqm). ✅
- Wrapper-everywhere / MQSC-once-on-active (spec doctrine + §5) → each task's host-prep vs `service`/`run_once`/`when: rdqm_is_active` split. ✅
- Live proof per distinct mechanism, not all four stacks (acceptance) → T1 §6 (SVC), T2 §5 (nativeha), T3 §4 (pcmk), T4 §5 (rdqm). ✅
- T4 is the disproportionate-effort task, sequenced last (spec §7) → Task 4 header + Step 1 confirm-against-current. ✅
- `CMDEV(NODISPLAY)` + verbatim class list (spec constraints) → T1 Step 2 (moved verbatim). ✅
- pcmk-rhel excluded; no guardrail (spec §9) → not in any task. ✅

**Placeholder scan:** every code step shows the real file content or the exact task to add; the only literal placeholders are `<T1_ISSUE>`…`<T4_ISSUE>` in `report-ready` (the implementation task numbers, filed after this plan lands — resolved at execution time). No TBD/TODO/"handle errors". ✅

**Consistency:** `mq-event-monitor` entry points (`main` = host-prep, `service` = MQSC) are named identically across T1–T4; `qmgr_name` is passed at every include (the role's var; arms use `qm_name`/`qm_app` locally); `mq_event_service_name`/`mq_event_run_dir` unchanged from role defaults. ✅

**Open questions carried from the spec (resolve at build):** per-arm `mq_event_perf_queues` (default `[]` — set only if an arm has a curated app queue); rdqm include placement (T4 Step 1 confirms against the current `site-rdqm.yml`); failover-survival evidence (T3/T4 Step 5 optional failover check).
