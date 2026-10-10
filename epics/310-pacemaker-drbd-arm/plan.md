# pacemaker-drbd arm — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `pcmk-drbd-ubuntu` stack (mechanism `pacemaker-drbd`) that rebuilds RDQM's shared-nothing HA/DR architecture from upstream Corosync/Pacemaker/DRBD 9 on Ubuntu, using RDQM's captured live configuration as the blueprint, then compare it with `rdqm-rhel` on an identical drill set.

**Architecture:** All implementation lands in **`logical-minds-foundry/mq-resiliency-lab-for-linux`** (this plan lives in `.github`). A read-only capture of the live RDQM arm becomes the committed blueprint (`docs/reference/rdqm-blueprint/`) and seeds a deviation ledger. A new baked box carries LINBIT's DRBD 9 (`drbd-dkms`). Two new roles (`drbd9`, `pdrbd-qm`) form a per-QM DRBD volume under Pacemaker at each site; `mqlab dr` is generalised to dispatch on each stack's own DR verbs so both arms run the DR drills with one command. The SAN arm is not touched; its fate is decided in Task 10.

**Tech Stack:** Ansible (roles/plays, ansible-core only — no galaxy collections); DRBD 9 (`ppa:linbit/linbit-drbd9-stack`, `drbd-dkms`, `drbd-utils`); Pacemaker/Corosync/pcs (Ubuntu 24.04 archive); IBM MQ (the `lab/mq-version` pin, Ubuntu debs via `mq-install`); `mqlab` (Python 3 + Typer, pytest at 100% branch coverage); libvirt/Vagrant lab.

**Spec:** `epics/310-pacemaker-drbd-arm/spec.md` (this directory). Executors read both.

## Global Constraints

- **The RDQM blueprint is authoritative.** Wherever the captured RDQM configuration shows how something is done, the open-source arm does the same with upstream parts; wherever it cannot, the deviation is a ledger row. The hypothesis values in Tasks 5–6 are replaced by blueprint values in Task 2's reconciliation step.
- **No change to `pcmk-ubuntu` / `pacemaker-san`.** `mq-pcmk-qmgr`, `drbd-san`, `iscsi-*`, `bake-pcmk-ubuntu.yml`, the `pcmk` box and the SAN networks are not modified (template symlinks in Task 5 leave `mq-pcmk-qmgr`'s files resolving exactly as before).
- **DRBD 9 from LINBIT's PPA (`drbd-dkms`)**, never the in-kernel 8.4 module. The module is built at bake time; a node never compiles a kernel module at boot.
- **No Booth; DR is operator-driven** (mirrors `rdqmdr`).
- **Scope:** core + comparison, **plus** the distributed app↔SVC workload with its TLS and CHLAUTH authz (needed by `mqlab qm e2e`). Observability, exporters, and cockpit boards are out of scope except where the stack registry provides them unchanged.
- **Functional comparison only** — no timing claims between arms.
- **Fail loud.** No `|| true`, no `failed_when: false` on a step that does work, no swallowed errors; probes use `changed_when: false` and an explicit `failed_when`.
- **DRBD lessons pre-applied** (`docs/reference/drbd-operations.md`): `create-md` with `</dev/null` + `timeout`; live state via `drbdadm status` (never `dump-md` on an in-use device); attach asserted via `drbdadm dstate`.
- **Secrets never captured or committed:** no keystores (`*.kdb`, `*.p12`, `*.sth`, `*.jks`), no corosync `authkey`, no DRBD `shared-secret` values, no passwords.
- **Derive, never hardcode:** QM names come from the stack's `short` (`PDRBD` → `PDRBDAPP`); IPs live only in `lab/topology.yaml` and the provision playbook's lab-constant block (the existing `site-pcmk.yml` / `site-rdqm.yml` precedent).
- **Validation gate is exactly one command:** `vrg-container-run -- vrg-validate`. Dev-loop test command: `uv run pytest …` (build-tool use only; never embedded in shipped scripts).
- **Coverage floor: 100% branch** for `src/mqlab`.
- **Cold-rebuild acceptance gate:** bring-up/provisioning changes are accepted only after a full cold rebuild proves them one-pass (Task 7).
- **The human operates the lab.** Agents write the code and the procedures; the human runs live captures, bakes, bootstraps and drills, and pastes results.
- **Each task = one GitHub issue** in the lab repo (born linked under `logical-minds-foundry/.github#310`), on `feature/<issue>-<slug>` off `develop`, committed with `vrg-commit`, PR into `develop`.

## Review Focus

1. **`--rpo0-drill` against a stack whose DR verb does not implement the drill** — must refuse with exit 2, never run the cutover and report success without the RPO-0 check. *(Test: Task 6a, `test_dr_rpo0_drill_refused_when_verb_does_not_declare_it`.)*
2. **A booted node whose kernel differs from the one the DKMS module was built for** — the per-run `drbd9` half must fail loud and name the fix (rebuild the `pdrbd` box), not let `modprobe` silently fall back to the in-kernel 8.4 module. *(Test: Task 3, `test_drbd9_configure_asserts_drbd9_for_running_kernel`.)*
3. **Re-running provision on an already-formed stack** — `create-md`, the initial promote/`mkfs`, and `crtmqm` must be skipped, never re-run over live data. *(Test: Task 5, `test_pdrbd_qm_destructive_steps_are_guarded`; live: Task 7 runs provision twice.)*
4. **DR cutover when the target site's disk is not UpToDate** (replication broken or lagging) — the playbook must refuse unless the operator passes an explicit force, and must say why. *(Test: Task 6, `test_dr_switch_asserts_disk_state_before_promote`; live: drill 6.)*
5. **Capture run when RDQM is not healthy or no RDQM QM exists** — the capture must fail before writing anything, never leave a partial blueprint on disk that looks complete. *(Test: Task 1, `test_capture_preflight_precedes_every_write`.)*

## Task dependency graph

```text
D (#311 docs) ──┬─▶ T1 capture playbook ─▶ T2 run capture + blueprint + ledger ─┐
                ├─▶ T3 DRBD 9 box ────────────────────────────────────────────┤
                └─▶ T6a mqlab dr dispatch ────────────────────────┐            │
                                                                  │            ▼
                                              T4 topology ◀── T2  │     T5 site-A HA ◀── T2,T3,T4
                                                                  ▼            │
                                                       T6 DR (site B + switch) ◀┘ (+T6a)
                                                                  │
                                                       T7 validation: cold rebuild
                                                                  │
                                                       T8 validation: drill set (both arms)
                                                                  │
                                                       T9 comparison report + ledger + parity
                                                                  │
                                                       T10 pacemaker-san decision
                                                                  │
                                              R1 doc review (#1419) ─▶ R2 retrospective (#312)
```

---

### Task 1: Read-only RDQM blueprint capture playbook

**Files:**
- Create: `ansible/capture-rdqm-blueprint.yml`
- Create: `tests/test_capture_rdqm_blueprint.py`

**Interfaces:**
- Consumes: the rendered lab inventory (`build/work/inventory.ini`, groups `rdqm_a`, `rdqm_b`); the RDQM app QM name (`RDQMAPP`, passed as `-e qm_name=…`).
- Produces: a capture tree at `{{ blueprint_out }}/<host>/<item>.txt` for every host in `rdqm_a:rdqm_b`, where `blueprint_out` is passed by the operator (`-e blueprint_out=<abs path>`). Item names (stable; Task 2 commits them): `drbd-version`, `drbd-conf`, `drbd-dump`, `drbd-status`, `packages`, `pcmk-config`, `pcmk-status`, `ocf-agents`, `corosync-conf`, `quorum`, `vgs`, `lvs`, `mounts`, `lsblk`, `rdqm-ini`, `rdqmstatus-qm`, `rdqmstatus-group`, `units`, `mqs-ini`, `dspmqinf`.

- [ ] **Step 1: Write the failing structure tests**

```python
"""Structure tests for ansible/capture-rdqm-blueprint.yml (epic .github#310 T1).

The capture is READ-ONLY and secret-free by construction; these tests pin that
contract statically because the live run needs the RDQM arm up.
"""

import pathlib
import re

import yaml

PLAYBOOK = pathlib.Path("ansible/capture-rdqm-blueprint.yml")
_ALLOWED_REMOTE = {"ansible.builtin.command", "ansible.builtin.shell", "ansible.builtin.assert"}
_SECRET_PATTERNS = re.compile(r"\.kdb|\.p12|\.sth|\.jks|authkey|/ssl/|keystore", re.IGNORECASE)


def _plays() -> list[dict]:
    return yaml.safe_load(PLAYBOOK.read_text())


def _capture_play() -> dict:
    [play] = [p for p in _plays() if p.get("hosts") == "rdqm_a:rdqm_b"]
    return play


def _module(task: dict) -> str:
    return next(k for k in task if k.startswith("ansible.builtin."))


def test_capture_targets_both_rdqm_sites():
    assert _capture_play()["hosts"] == "rdqm_a:rdqm_b"


def test_capture_remote_tasks_are_read_only():
    for task in _capture_play()["tasks"]:
        mod = _module(task)
        if task.get("delegate_to") == "localhost":
            continue  # writing the capture files on the controller
        assert mod in _ALLOWED_REMOTE, f"{task['name']}: {mod} is not a read-only module"
        if mod != "ansible.builtin.assert":
            assert task.get("changed_when") is False, f"{task['name']}: must be changed_when: false"


def test_capture_never_reads_secret_material():
    for task in _capture_play()["tasks"]:
        body = str(task.get("ansible.builtin.shell") or task.get("ansible.builtin.command") or "")
        assert not _SECRET_PATTERNS.search(body), f"{task['name']} touches secret material"


def test_capture_redacts_drbd_shared_secret():
    dump = next(t for t in _capture_play()["tasks"] if t.get("register") == "cap_drbd_dump")
    assert "shared-secret" in dump["ansible.builtin.shell"]
    assert "REDACTED" in dump["ansible.builtin.shell"]


def test_capture_preflight_precedes_every_write():
    tasks = _capture_play()["tasks"]
    first_write = next(i for i, t in enumerate(tasks) if t.get("delegate_to") == "localhost")
    preflight = [i for i, t in enumerate(tasks) if "preflight" in t["name"]]
    assert preflight and max(preflight) < first_write
```

- [ ] **Step 2: Run to verify they fail**

Run: `uv run pytest tests/test_capture_rdqm_blueprint.py -v`
Expected: FAIL — `FileNotFoundError: ansible/capture-rdqm-blueprint.yml`.

- [ ] **Step 3: Write the playbook**

```yaml
---
# Read-only capture of what RDQM GENERATES on a live HA/DR group (epic .github#310 T1).
# The capture is the blueprint the pacemaker-drbd arm is built to. It changes NOTHING on
# the nodes, reads NO secret material (keystores, corosync authkey), and redacts DRBD
# shared-secret values at capture time. Run by the operator with the rdqm-rhel stack up
# and its app QM running:
#   cd ansible && ansible-playbook capture-rdqm-blueprint.yml \
#     -e qm_name=RDQMAPP -e blueprint_out="$(mqlab build path state)/rdqm-blueprint/$(date -u +%Y%m%dT%H%M%SZ)"
- name: Capture the RDQM blueprint (read-only)
  hosts: rdqm_a:rdqm_b
  become: true
  gather_facts: false
  vars:
    qm_lower: "{{ qm_name | lower }}"
  tasks:
    - name: preflight - blueprint_out and qm_name are set
      ansible.builtin.assert:
        that:
          - blueprint_out is defined and blueprint_out | length > 0
          - qm_name is defined and qm_name | length > 0
        fail_msg: "pass -e blueprint_out=<abs dir> -e qm_name=<RDQM QM>"

    - name: preflight - RDQM reports the QM on this node
      ansible.builtin.command: /opt/mqm/bin/rdqmstatus -m {{ qm_name }}
      register: cap_rdqmstatus_qm
      changed_when: false

    - name: preflight - a cluster CLI is present (pcs or crm)
      ansible.builtin.shell: |
        set -e
        if command -v pcs >/dev/null; then echo pcs
        elif command -v crm >/dev/null; then echo crm
        else echo "neither pcs nor crm on PATH" >&2; exit 1; fi
      register: cap_cli
      changed_when: false

    - name: drbd version
      ansible.builtin.shell: drbdadm --version; cat /sys/module/drbd/version
      register: cap_drbd_version
      changed_when: false

    - name: drbd global config and resource files (listing + global/common)
      ansible.builtin.shell: |
        set -e
        cat /etc/drbd.conf
        ls -l /etc/drbd.d/
        cat /etc/drbd.d/global_common.conf
      register: cap_drbd_conf
      changed_when: false

    - name: drbd effective config (shared-secret redacted at capture)
      ansible.builtin.shell: |
        set -o pipefail
        drbdadm dump all | sed -E 's/(shared-secret[[:space:]]+)"[^"]*"/\1"REDACTED"/'
      args:
        executable: /bin/bash
      register: cap_drbd_dump
      changed_when: false

    - name: drbd live status
      ansible.builtin.command: drbdadm status all
      register: cap_drbd_status
      changed_when: false

    - name: installed HA/DR packages
      ansible.builtin.shell: |
        set -o pipefail
        rpm -qa | grep -Ei 'drbd|pacemaker|corosync|pcs|crmsh|resource-agents|rdqm|MQSeries' | sort
      args:
        executable: /bin/bash
      register: cap_packages
      changed_when: false

    - name: pacemaker configuration (text, not CIB XML)
      ansible.builtin.shell: "{{ 'pcs config' if cap_cli.stdout == 'pcs' else 'crm configure show' }}"
      register: cap_pcmk_config
      changed_when: false

    - name: pacemaker status
      ansible.builtin.command: crm_mon -1 -r -A
      register: cap_pcmk_status
      changed_when: false

    - name: OCF resource agents available (providers + agents)
      ansible.builtin.shell: find /usr/lib/ocf/resource.d -maxdepth 2 | sort
      register: cap_ocf_agents
      changed_when: false

    - name: corosync config (authkey is a separate file and is never read)
      ansible.builtin.command: cat /etc/corosync/corosync.conf
      register: cap_corosync_conf
      changed_when: false

    - name: quorum state
      ansible.builtin.command: corosync-quorumtool -s
      register: cap_quorum
      changed_when: false

    - name: volume groups
      ansible.builtin.command: vgs -o vg_name,pv_count,lv_count,vg_size,vg_free
      register: cap_vgs
      changed_when: false

    - name: logical volumes
      ansible.builtin.command: lvs -a -o lv_name,vg_name,lv_size,lv_attr,origin,devices
      register: cap_lvs
      changed_when: false

    - name: mounts under /var/mqm/vols (empty on a non-owner is valid)
      ansible.builtin.shell: findmnt -R -o TARGET,SOURCE,FSTYPE,OPTIONS /var/mqm/vols || [ $? -eq 1 ]
      register: cap_mounts
      changed_when: false

    - name: block devices + filesystems
      ansible.builtin.command: lsblk -f
      register: cap_lsblk
      changed_when: false

    - name: rdqm.ini
      ansible.builtin.command: cat /var/mqm/rdqm.ini
      register: cap_rdqm_ini
      changed_when: false

    - name: rdqmstatus (group view)
      ansible.builtin.command: /opt/mqm/bin/rdqmstatus
      register: cap_rdqmstatus_group
      changed_when: false

    - name: installed systemd units (HA/DR/MQ)
      ansible.builtin.shell: |
        set -o pipefail
        systemctl list-unit-files --no-legend | grep -Ei 'rdqm|drbd|pacemaker|corosync|pcsd|mq' | sort
      args:
        executable: /bin/bash
      register: cap_units
      changed_when: false

    - name: mqs.ini
      ansible.builtin.command: cat /var/mqm/mqs.ini
      register: cap_mqs_ini
      changed_when: false

    - name: dspmqinf (QM data/log paths)
      ansible.builtin.command: su mqm -c '/opt/mqm/bin/dspmqinf -o command {{ qm_name }}'
      register: cap_dspmqinf
      changed_when: false

    - name: write the capture on the controller
      ansible.builtin.copy:
        dest: "{{ blueprint_out }}/{{ inventory_hostname }}/{{ item.key }}.txt"
        content: "{{ item.value.stdout }}\n"
        mode: "0644"
      delegate_to: localhost
      become: false
      loop:
        - { key: drbd-version, value: "{{ cap_drbd_version }}" }
        - { key: drbd-conf, value: "{{ cap_drbd_conf }}" }
        - { key: drbd-dump, value: "{{ cap_drbd_dump }}" }
        - { key: drbd-status, value: "{{ cap_drbd_status }}" }
        - { key: packages, value: "{{ cap_packages }}" }
        - { key: pcmk-config, value: "{{ cap_pcmk_config }}" }
        - { key: pcmk-status, value: "{{ cap_pcmk_status }}" }
        - { key: ocf-agents, value: "{{ cap_ocf_agents }}" }
        - { key: corosync-conf, value: "{{ cap_corosync_conf }}" }
        - { key: quorum, value: "{{ cap_quorum }}" }
        - { key: vgs, value: "{{ cap_vgs }}" }
        - { key: lvs, value: "{{ cap_lvs }}" }
        - { key: mounts, value: "{{ cap_mounts }}" }
        - { key: lsblk, value: "{{ cap_lsblk }}" }
        - { key: rdqm-ini, value: "{{ cap_rdqm_ini }}" }
        - { key: rdqmstatus-qm, value: "{{ cap_rdqmstatus_qm }}" }
        - { key: rdqmstatus-group, value: "{{ cap_rdqmstatus_group }}" }
        - { key: units, value: "{{ cap_units }}" }
        - { key: mqs-ini, value: "{{ cap_mqs_ini }}" }
        - { key: dspmqinf, value: "{{ cap_dspmqinf }}" }
      loop_control:
        label: "{{ item.key }}"
```

Note: `delegate_to: localhost` writes create parent directories via `copy` only if they exist; add, immediately before the write task:

```yaml
    - name: create the per-host capture directory on the controller
      ansible.builtin.file:
        path: "{{ blueprint_out }}/{{ inventory_hostname }}"
        state: directory
        mode: "0755"
      delegate_to: localhost
      become: false
```

(It is a controller-side write, so it sits after every preflight — the ordering test covers it.)

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_capture_rdqm_blueprint.py -v`
Expected: PASS (5 tests).

- [ ] **Step 5: Validate and commit**

Run: `vrg-container-run -- vrg-validate` — expected: green (ansible-lint included).

```bash
vrg-git add ansible/capture-rdqm-blueprint.yml tests/test_capture_rdqm_blueprint.py
vrg-commit --type feat --scope rdqm --message "read-only RDQM blueprint capture playbook (#<T1>)"
```

---

### Task 2: Run the capture; commit the blueprint; seed the deviation ledger; reconcile Tasks 5–6

**Files:**
- Create: `docs/reference/rdqm-blueprint/README.md`
- Create: `docs/reference/rdqm-blueprint/<host>/<item>.txt` (copied from the capture, all 6 hosts)
- Create: `docs/reference/rdqm-vs-vanilla-deviations.md`
- Modify (`.github`): this plan's Tasks 5–6 hypothesis values, via a reconciliation comment on each filed issue (see Step 5)

**Interfaces:**
- Consumes: Task 1's playbook and item names.
- Produces: the committed blueprint tree (Tasks 5, 6 and 9 cite it by path) and the ledger file with stable row IDs (`ST-n` storage, `RP-n` replication, `CL-n` cluster, `DR-n` disaster recovery, `TL-n` lifecycle tooling, `SC-n` replication security, `SP-n` support/installation) that Task 9 finalises.

- [ ] **Step 1: Operator runs the capture (human step)**

With `rdqm-rhel` bootstrapped (HA + DR) and `RDQMAPP` running:

```bash
cd ansible
OUT="$(mqlab build path state)/rdqm-blueprint/$(date -u +%Y%m%dT%H%M%SZ)"
ansible-playbook capture-rdqm-blueprint.yml -e qm_name=RDQMAPP -e blueprint_out="$OUT"
echo "$OUT"
```

Expected: `PLAY RECAP` with `failed=0` for all six `rdqm-*` hosts; `$OUT/<host>/` holds 20 `.txt` files each. The operator pastes `$OUT` into the task issue.

- [ ] **Step 2: Copy into the repo and verify no secrets**

```bash
mkdir -p docs/reference/rdqm-blueprint
cp -R "$OUT"/rdqm-* docs/reference/rdqm-blueprint/
grep -rEn 'shared-secret' docs/reference/rdqm-blueprint | grep -v REDACTED && echo "SECRET LEAK" && exit 1
grep -rEln '\.kdb|\.p12|\.sth|authkey' docs/reference/rdqm-blueprint && echo "CHECK REFERENCES" || true
```

Expected: no `SECRET LEAK`. Any keystore *path* reference (e.g. in `pcs config`) is acceptable — content never is; note it in the README.

- [ ] **Step 3: Write `docs/reference/rdqm-blueprint/README.md`**

Sections (each fact cites its file, e.g. `rdqm-a1/drbd-dump.txt`):
1. **Provenance** — capture date, MQ level (`lab/mq-version`), RHEL point release, DRBD/Pacemaker/Corosync package versions from `packages.txt`, and the `drbd-version.txt` result. State explicitly whether the DRBD module is LINBIT DRBD 9 (resolves spec D2's judgment).
2. **Storage** — VG name, LV(s) per QM (main + snapshot), sizes, filesystem type and mount options, mount path.
3. **Replication** — DRBD resource name(s), per-connection protocol (HA peers vs DR peers), addresses/ports, `quorum`/`on-no-quorum`, `after-sb-*`, `fencing` policy, net/disk options; whether HA and DR are **one resource with per-connection protocols or separate resources**, and how DRBD quorum is scoped so one site can keep quorum.
4. **Cluster** — cluster properties (`stonith-enabled`, `no-quorum-policy`, stickiness), resource list with agent (`ocf:<provider>:<agent>`), clone/promotable meta, constraints (colocation/order/location), and which agents are IBM-proprietary (provider not `heartbeat`/`linbit`/`pacemaker`).
5. **DR wiring** — how site B is configured while secondary (resources present/stopped? DRBD role?), as shown by site-B captures.
6. **RDQM lifecycle surface** — `rdqm.ini` content, units installed.

- [ ] **Step 4: Seed `docs/reference/rdqm-vs-vanilla-deviations.md`**

```markdown
# RDQM vs. pacemaker-drbd — deviation ledger

Source of truth for "what does IBM add". One row per RDQM element; every row cites
the blueprint file it derives from (`docs/reference/rdqm-blueprint/…`). Status:
**exact** (same upstream component, same config), **partial** (equivalent behavior,
different mechanism or values), **none** (no open-source equivalent built).

| ID | RDQM element | Blueprint source | pacemaker-drbd equivalent | Status | Consequence |
|---|---|---|---|---|---|
| ST-1 | `drbdpool` VG on a local disk | rdqm-a1/vgs.txt | `drbdpool` VG on the extra disk (`drbd9` role) | exact | — |
```

Add one row per element found in Step 3 (storage, replication, cluster, DR, tooling, replication TLS, support/installation). For rows whose open-source side is not built yet, set Status to `pending` — Task 9 replaces every `pending`.

- [ ] **Step 5: Reconcile Tasks 5 and 6 with the blueprint**

For each hypothesis variable in Task 5's `defaults/main.yml` and Task 6's DR template (listed in those tasks under **Blueprint inputs**), record the blueprint value and its source file in a comment on the Task 5 and Task 6 issues, headed `Blueprint reconciliation`.

**DR snapshot LV (explicit decision).** IBM documents a second LV per DR queue manager "to support the reverting to snapshot operation" (cached `recovery-requirements-rdqm-dr-solution/content.txt`). Record from `lvs.txt` whether it exists and its relationship to the QM LV, and add a `DR-n` ledger row: RDQM element = the revert-to-snapshot LV; open-source equivalent = not built; Status = **none**; Consequence = the recovery site cannot be reverted to its pre-resync state after a failed or partial resync. This is **not built** by default. If the blueprint shows that ordinary HA or DR operation (not just the revert operation) depends on it, treat that as a structural contradiction and stop for the human as below. Where the blueprint contradicts the hypothesis on a **structural** point — fencing is used; HA and DR are separate DRBD resources; RDQM uses an IBM agent for a role this plan assigns to `ocf:linbit:drbd` — say so in the comment and stop for the human: the affected task's steps are revised before it starts.

- [ ] **Step 6: Validate and commit**

Run: `vrg-container-run -- vrg-validate` — expected: green (markdown lint covers the new docs).

```bash
vrg-git add docs/reference/rdqm-blueprint docs/reference/rdqm-vs-vanilla-deviations.md
vrg-commit --type docs --scope rdqm --message "RDQM blueprint capture and deviation ledger seed (#<T2>)"
```

---

### Task 3: DRBD 9 baked box (`pdrbd`)

**Files:**
- Create: `ansible/roles/drbd9/defaults/main.yml`
- Create: `ansible/roles/drbd9/tasks/main.yml` (configure half)
- Create: `ansible/roles/drbd9/tasks/install.yml` (install half, baked)
- Create: `ansible/bake-pdrbd-ubuntu.yml`
- Modify: `lab/versions.yaml` (`roles:` add `pdrbd`)
- Modify: `lab/boxes/build-fatbox.sh` (MQ-media `case "$BAKE"` arm at the `obs | mq-ubuntu | nativeha-ubuntu | pcmk-ubuntu` line)
- Test: `tests/test_drbd9_role.py`

**Interfaces:**
- Consumes: the `bake` host inventory (`inventory/bake-host.ini`), `mq-install`, the bake-time roles `bake-pcmk-ubuntu.yml` uses.
- Produces: box role `pdrbd` (catalog `roles.pdrbd.bake.ubuntu = pdrbd-ubuntu`); role `drbd9` with vars `drbd9_vg` (default `drbdpool`) and `drbd9_disk` (default `/dev/vdb`); on a booted node, `drbd9` configure guarantees the loaded `drbd` module is 9.x and the VG exists.

- [ ] **Step 1: Write the failing tests**

```python
"""drbd9 role + pdrbd box wiring (epic .github#310 T3)."""

import pathlib

import yaml

from mqlab.versions import load_catalog

ROLE = pathlib.Path("ansible/roles/drbd9/tasks")


def _tasks(name: str) -> list[dict]:
    return yaml.safe_load((ROLE / name).read_text())


def test_pdrbd_box_role_is_catalogued():
    cat = load_catalog()
    assert cat.roles["pdrbd"]["bake"] == {"ubuntu": "pdrbd-ubuntu"}


def test_bake_playbook_bakes_drbd9_install_half_and_mq():
    plays = yaml.safe_load(pathlib.Path("ansible/bake-pdrbd-ubuntu.yml").read_text())
    includes = [
        t["ansible.builtin.include_role"]
        for p in plays
        for t in p.get("tasks", [])
        if "ansible.builtin.include_role" in t
    ]
    assert {"name": "drbd9", "tasks_from": "install"} in includes
    assert any(i["name"] == "mq-install" for i in includes)


def test_drbd9_install_uses_linbit_ppa_and_dkms():
    text = (ROLE / "install.yml").read_text()
    assert "ppa:linbit/linbit-drbd9-stack" in text
    assert "drbd-dkms" in text


def test_drbd9_install_proves_module_version_at_bake():
    names = [t["name"] for t in _tasks("install.yml")]
    assert any("assert the built module is DRBD 9" in n for n in names)


def test_drbd9_configure_asserts_drbd9_for_running_kernel():
    tasks = _tasks("main.yml")
    probe = next(t for t in tasks if t.get("register") == "drbd9_modver")
    assert "uname -r" in probe["ansible.builtin.shell"]
    check = next(t for t in tasks if t["name"].startswith("assert DRBD 9 for the running kernel"))
    assert "rebuild the pdrbd box" in check["ansible.builtin.assert"]["fail_msg"]


def test_build_fatbox_stages_mq_media_for_pdrbd():
    text = pathlib.Path("lab/boxes/build-fatbox.sh").read_text()
    assert "obs | mq-ubuntu | nativeha-ubuntu | pcmk-ubuntu | pdrbd-ubuntu)" in text
```

- [ ] **Step 2: Run to verify they fail**

Run: `uv run pytest tests/test_drbd9_role.py -v`
Expected: FAIL (missing files / `KeyError: 'pdrbd'`).

- [ ] **Step 3: Write `ansible/roles/drbd9/defaults/main.yml`**

```yaml
---
# drbd9 (epic .github#310): LINBIT DRBD 9 for the pacemaker-drbd arm. RDQM ships DRBD 9
# (kmod-drbd-9); the Ubuntu/mainline in-kernel module is 8.4.11 (2-node, no quorum),
# so the in-kernel module is never used here.
drbd9_vg: drbdpool
drbd9_disk: /dev/vdb
```

- [ ] **Step 4: Write `ansible/roles/drbd9/tasks/install.yml` (baked half)**

```yaml
---
# Install half (baked into the pdrbd box by bake-pdrbd-ubuntu.yml). Builds the DRBD 9
# module with DKMS against the box's kernel and PROVES it loads as 9.x, so a node never
# compiles at boot. No per-instance state (the VG needs the instance's fresh disk).
- name: LINBIT DRBD 9 PPA
  ansible.builtin.apt_repository:
    repo: ppa:linbit/linbit-drbd9-stack
    state: present
    update_cache: true
  become: true

- name: DRBD 9 userland + DKMS module + headers for the running kernel
  ansible.builtin.apt:
    name:
      - drbd-dkms
      - drbd-utils
      - dkms
      - "linux-headers-{{ ansible_kernel }}"
      - lvm2
    state: present
    lock_timeout: "{{ apt_lock_timeout }}"
  become: true

- name: load the module (fail loud if DKMS did not produce one)
  ansible.builtin.command: modprobe drbd
  changed_when: true
  become: true

- name: read the loaded module version
  ansible.builtin.command: cat /sys/module/drbd/version
  register: drbd9_bake_ver
  changed_when: false

- name: assert the built module is DRBD 9 (not the in-kernel 8.4)
  ansible.builtin.assert:
    that: drbd9_bake_ver.stdout is match('^9\\.')
    fail_msg: "loaded drbd module is {{ drbd9_bake_ver.stdout }}, expected 9.x from drbd-dkms"

- name: unload so the image is captured clean
  ansible.builtin.command: rmmod drbd
  changed_when: true
  become: true
```

- [ ] **Step 5: Write `ansible/roles/drbd9/tasks/main.yml` (per-run configure half)**

```yaml
---
# Configure half (per run). Install half is a no-op on a baked box (package_facts guard).
- name: read installed packages (skip-if-baked guard)
  ansible.builtin.package_facts:
    manager: apt

- name: DRBD 9 install half (skipped when baked into the pdrbd box)
  ansible.builtin.include_tasks: install.yml
  when: "'drbd-dkms' not in ansible_facts.packages"

- name: DKMS-built drbd module version for the RUNNING kernel
  ansible.builtin.shell: |
    set -o pipefail
    modinfo -k "$(uname -r)" -F version drbd
  args:
    executable: /bin/bash
  register: drbd9_modver
  changed_when: false

- name: assert DRBD 9 for the running kernel (kernel drift guard)
  ansible.builtin.assert:
    that: drbd9_modver.stdout is match('^9\\.')
    fail_msg: >-
      drbd module for kernel {{ ansible_kernel }} is '{{ drbd9_modver.stdout }}', not 9.x —
      the node's kernel differs from the one the box was baked on; rebuild the pdrbd box
      (mqlab box build pdrbd) instead of letting the in-kernel 8.4 module load

- name: load drbd (idempotent; lsmod guard keeps reporting honest)
  ansible.builtin.shell: |
    set -e
    if lsmod | grep -q '^drbd '; then echo already-loaded; else modprobe drbd; echo loaded; fi
  register: drbd9_modprobe
  changed_when: "'loaded' == drbd9_modprobe.stdout"
  become: true

- name: drbdpool-style VG on this instance's extra disk (vgs probe, not a path guard)
  ansible.builtin.shell: |
    set -e
    if ! vgs {{ drbd9_vg }} >/dev/null 2>&1; then
      pvcreate {{ drbd9_disk }}
      vgcreate {{ drbd9_vg }} {{ drbd9_disk }}
      echo VG-CREATED
    fi
  register: drbd9_vg_out
  changed_when: "'VG-CREATED' in drbd9_vg_out.stdout"
  become: true
```

- [ ] **Step 6: Write `ansible/bake-pdrbd-ubuntu.yml`**

Copy `ansible/bake-pcmk-ubuntu.yml` verbatim (all plays: apt-autoupdate-off, the bake play, motd-off, cloud-init-trim/snapd-off, bake-dirs-guard), then in the bake play: rename it `Bake pdrbd-ubuntu (MQ + DRBD 9 install adapters only; no QMs, no cluster, no secrets)`, rewrite the header comment for this box (cluster nodes of `pcmk-drbd-ubuntu`; DRBD 9 module built and proven at bake), and add — directly after the `mq-install` include —

```yaml
    # DRBD 9 (LINBIT drbd-dkms) built against THIS image's kernel and proven to load as
    # 9.x; the per-run drbd9 configure half re-checks it against the booted kernel.
    - name: DRBD 9 module + userland (install half only)
      ansible.builtin.include_role:
        name: drbd9
        tasks_from: install
```

- [ ] **Step 7: Catalogue the box role and stage MQ media for it**

`lab/versions.yaml`, under `roles:` after `pcmk`:

```yaml
  # pacemaker-drbd cluster nodes (epic .github#310): MQ + LINBIT DRBD 9 (drbd-dkms).
  # Not MQ-bearing for hashing, matching pcmk (lab/boxes/_manifest-hash.sh).
  pdrbd: { bake: { ubuntu: pdrbd-ubuntu }, components: [mq-resiliency-observability] }
```

`lab/boxes/build-fatbox.sh`: change `obs | mq-ubuntu | nativeha-ubuntu | pcmk-ubuntu)` to `obs | mq-ubuntu | nativeha-ubuntu | pcmk-ubuntu | pdrbd-ubuntu)` and add `pdrbd-ubuntu` to the comment above it.

- [ ] **Step 8: Run tests; validate**

Run: `uv run pytest tests/test_drbd9_role.py -v` — expected: PASS (6).
Run: `vrg-container-run -- vrg-validate` — expected: green.

- [ ] **Step 9: Operator bakes the box (human step; acceptance for this task)**

```bash
mqlab box build pdrbd
```

Expected: the bake completes; the log shows `assert the built module is DRBD 9` → `ok`, with `/sys/module/drbd/version` starting `9.`. A failure here (DKMS build error, PPA lacks a build for this Ubuntu/arch) stops the epic for re-planning (spec §10). Paste the version line into the issue.

- [ ] **Step 10: Commit**

```bash
vrg-git add ansible/roles/drbd9 ansible/bake-pdrbd-ubuntu.yml lab/versions.yaml lab/boxes/build-fatbox.sh tests/test_drbd9_role.py
vrg-commit --type feat --scope bake --message "pdrbd box: LINBIT DRBD 9 (drbd-dkms) baked and proven to load (#<T3>)"
```

---

### Task 4: Topology — the `pcmk-drbd-ubuntu` stack

**Files:**
- Modify: `lab/topology.yaml` (nodes, groups, stack)
- Modify: `lab/versions.yaml` (`stacks:` entry)
- Modify: `src/mqlab/stacks.py` (`_MECH_LABEL`)
- Modify: `src/mqlab/parity.py` (`MATRIX` row)
- Modify: `src/mqlab/watcherboard.py` (`_owner_resource`: Pacemaker arms own the QM as `mq_qm`)
- Test: `tests/test_topology_pdrbd.py` (create), `tests/test_stacks.py` (dashboard folder), `tests/test_parity.py`, `tests/test_watcherboard.py`

**Interfaces:**
- Consumes: box role `pdrbd` (Task 3).
- Produces: stack `pcmk-drbd-ubuntu` — `mechanism: pacemaker-drbd`, `short: PDRBD` (QM `PDRBDAPP`), `cluster_group: pdrbd_a`, `groups: [pdrbd_a, pdrbd_b]`, `dr_groups: [pdrbd_b]`, `provision: ansible/site-pdrbd.yml`, `qm: {vip: 10.10.1.170, vip_b: 10.10.2.170}`, verbs below. Groups `pdrbd_a = [pdrbd-a1, pdrbd-a2, pdrbd-a3]`, `pdrbd_b = [pdrbd-b1, pdrbd-b2, pdrbd-b3]`. Hosts' hb IPs `172.16.1.71-73` / `172.16.2.81-83` (Tasks 5–6 use them). DNS names derive automatically: `pdrbd-vip-a.client.com`, `pdrbd-vip-b.client.com`.

The verbs reference playbooks created in Tasks 5–6 (`site-pdrbd.yml`, `site-pdrbd-qm-down.yml`, `site-pdrbd-dr-switch.yml`); no test asserts playbook existence, and the stack is not bootstrappable until Task 5 lands — the issue says so.

- [ ] **Step 1: Write the failing tests**

`tests/test_topology_pdrbd.py`:

```python
"""Topology coverage for the pcmk-drbd-ubuntu stack (epic .github#310).

RDQM-faithful open-source arm: shared-nothing (per-node extra disk, no SAN), one floating
IP per site like RDQM, operator-driven DR. Coexists with the other arms (disjoint octets).
"""

import ipaddress
import pathlib

import yaml

from mqlab.stacks import lab_stacks, stack_dr_hosts, stack_members, stack_members_effective
from mqlab.versions import load_catalog

_A = ("pdrbd-a1", "pdrbd-a2", "pdrbd-a3")
_B = ("pdrbd-b1", "pdrbd-b2", "pdrbd-b3")


def _topology() -> dict:
    return yaml.safe_load(pathlib.Path("lab/topology.yaml").read_text())


def test_pdrbd_stack_shape():
    s = lab_stacks()["pcmk-drbd-ubuntu"]
    assert (s.mechanism, s.os_family, s.short) == ("pacemaker-drbd", "ubuntu", "PDRBD")
    assert s.qm.qm_app == "PDRBDAPP"
    assert s.cluster_group == "pdrbd_a"
    assert s.groups == ["pdrbd_a", "pdrbd_b"]
    assert s.dr_groups == ["pdrbd_b"]
    assert s.provision == "ansible/site-pdrbd.yml"


def test_pdrbd_verbs():
    v = lab_stacks()["pcmk-drbd-ubuntu"].verbs
    assert v["qm-create"] == {"playbook": "site-pdrbd.yml"}
    assert v["qm-destroy"] == {"playbook": "site-pdrbd-qm-down.yml"}
    assert v["qm-up"] == {"pcs": "resource enable mq_group"}
    assert v["qm-down"] == {"pcs": "resource disable mq_group"}
    assert "drbdadm status" in v["qm-status"]["cmd"]
    assert v["dr-cutover"] == {"playbook": "site-pdrbd-dr-switch.yml", "rpo0": True}
    assert v["dr-failback"] == {"playbook": "site-pdrbd-dr-switch.yml", "rpo0": True}
    assert "runmqras" in v["diagnostics"]["cmd"]


def test_pdrbd_like_rdqm_has_one_floating_ip_per_site_and_no_ext_vip():
    qm = _topology()["stacks"]["pcmk-drbd-ubuntu"]["qm"]
    assert set(qm) == {"vip", "vip_b"}


def test_pdrbd_nodes_are_shared_nothing_on_the_baked_box():
    nodes = _topology()["nodes"]
    for h in (*_A, *_B):
        assert nodes[h]["box"] == "pdrbd"
        assert nodes[h]["extra_disk"] == 10
        assert not any(n.startswith("net-san") for n in nodes[h]["nics"])


def test_pdrbd_dr_hosts_and_no_dr_members():
    assert stack_dr_hosts("pcmk-drbd-ubuntu") == list(_B)
    full = stack_members("pcmk-drbd-ubuntu")
    assert full == [*_A, *_B]
    assert stack_members_effective("pcmk-drbd-ubuntu", no_dr=True) == list(_A)


def test_pdrbd_addresses_collide_with_no_other_node():
    seen: dict[str, str] = {}
    for host, spec in _topology()["nodes"].items():
        for ip in (spec.get("nics") or {}).values():
            ipaddress.ip_address(ip)
            assert ip not in seen, f"{ip} on {host} and {seen.get(ip)}"
            seen[ip] = host


def test_pdrbd_os_catalog_entry():
    cat = load_catalog()
    assert cat.stacks["pcmk-drbd-ubuntu"]["supported"] == ["ubuntu:24"]
```

(`Catalog.stacks` is the raw per-stack table from `lab/versions.yaml`; if the loader normalises `supported` into `OsRef` objects, compare against `[OsRef("ubuntu", 24)]` instead — check `load_catalog` before writing the assertion.)

Append to `tests/test_stacks.py` (next to `test_dashboard_folder_derives_from_mechanism_and_os`):

```python
def test_dashboard_folder_for_pacemaker_drbd():
    assert dashboard_folder_for("pacemaker-drbd", "ubuntu") == "PCMK-DRBD (Ubuntu)"
```

Append to `tests/test_watcherboard.py` (the stack tiles must not silently show "No data" for the new arm — its Pacemaker resource is `mq_qm`, like `pcmk-ubuntu`):

```python
def test_owner_resource_is_mq_qm_for_both_pacemaker_mechanisms():
    assert watcherboard._owner_resource("pacemaker-san", "PCMK") == "mq_qm"
    assert watcherboard._owner_resource("pacemaker-drbd", "PDRBD") == "mq_qm"
    assert watcherboard._owner_resource("rdqm", "RDQM") == "RDQMAPP"
```

(Match the module import style already used at the top of `tests/test_watcherboard.py`.)

Append to `tests/test_parity.py`:

```python
def test_pdrbd_row_declares_every_verb_not_yet():
    row = parity.MATRIX["pcmk-drbd-ubuntu"]
    assert set(row) == set(parity.VERBS)
    assert set(row.values()) == {parity.Support.NOT_YET}
```

- [ ] **Step 2: Run to verify they fail**

Run: `uv run pytest tests/test_topology_pdrbd.py tests/test_stacks.py tests/test_parity.py tests/test_watcherboard.py -v`
Expected: FAIL (`KeyError: 'pcmk-drbd-ubuntu'`, `ValueError: no dashboard-folder label for mechanism 'pacemaker-drbd'`).

- [ ] **Step 3: Add the nodes, groups and stack to `lab/topology.yaml`**

Nodes (after the Native HA Ubuntu block):

```yaml
  # --- pacemaker-drbd arm (epic .github#310): the RDQM-faithful open-source arm. Six
  # Ubuntu cluster nodes, SHARED-NOTHING like RDQM: each has a local extra disk for the
  # drbdpool VG (one LV per QM, DRBD 9 replicated); no SAN, no net-san-*. Host octets
  # .7x (site A) / .8x (site B) — clear of every other arm so it can coexist. They boot
  # the baked pdrbd box (MQ + LINBIT DRBD 9); host-arch resolution stays at the box layer.
  pdrbd-a1:
    box: pdrbd
    cpus: 2
    memory: 2048
    extra_disk: 10
    nics: { net-mgmt: 10.50.0.71, net-data-a: 10.10.1.71, net-hb-a: 172.16.1.71, net-wan: 10.99.0.71, net-ext: 10.60.0.71 }
  pdrbd-a2:
    box: pdrbd
    cpus: 2
    memory: 2048
    extra_disk: 10
    nics: { net-mgmt: 10.50.0.72, net-data-a: 10.10.1.72, net-hb-a: 172.16.1.72, net-wan: 10.99.0.72, net-ext: 10.60.0.72 }
  pdrbd-a3:
    box: pdrbd
    cpus: 2
    memory: 2048
    extra_disk: 10
    nics: { net-mgmt: 10.50.0.73, net-data-a: 10.10.1.73, net-hb-a: 172.16.1.73, net-wan: 10.99.0.73, net-ext: 10.60.0.73 }
  pdrbd-b1:
    box: pdrbd
    cpus: 2
    memory: 2048
    extra_disk: 10
    nics: { net-mgmt: 10.50.0.81, net-data-b: 10.10.2.81, net-hb-b: 172.16.2.81, net-wan: 10.99.0.81 }
  pdrbd-b2:
    box: pdrbd
    cpus: 2
    memory: 2048
    extra_disk: 10
    nics: { net-mgmt: 10.50.0.82, net-data-b: 10.10.2.82, net-hb-b: 172.16.2.82, net-wan: 10.99.0.82 }
  pdrbd-b3:
    box: pdrbd
    cpus: 2
    memory: 2048
    extra_disk: 10
    nics: { net-mgmt: 10.50.0.83, net-data-b: 10.10.2.83, net-hb-b: 172.16.2.83, net-wan: 10.99.0.83 }
```

Groups (with the other atomic groups):

```yaml
  pdrbd_a: [pdrbd-a1, pdrbd-a2, pdrbd-a3]
  pdrbd_b: [pdrbd-b1, pdrbd-b2, pdrbd-b3]
```

Stack (after `nativeha-ubuntu`):

```yaml
  # pcmk-drbd-ubuntu (epic .github#310): RDQM rebuilt from upstream Corosync/Pacemaker/
  # DRBD 9. Built to RDQM's captured config (docs/reference/rdqm-blueprint/); deviations
  # in docs/reference/rdqm-vs-vanilla-deviations.md. Like RDQM: one floating IP per site
  # (no vip_ext), operator-driven DR (no Booth).
  pcmk-drbd-ubuntu:
    mechanism: pacemaker-drbd
    os_family: ubuntu
    short: PDRBD
    cluster_group: pdrbd_a
    groups: [pdrbd_a, pdrbd_b]
    dr_groups: [pdrbd_b]     # DR site — skipped by `bootstrap --no-dr` (#188)
    provision: ansible/site-pdrbd.yml
    secrets: [pcmk_hacluster_password, mqweb_admin_password]
    qm: { vip: 10.10.1.170, vip_b: 10.10.2.170 }
    alloc: { exporter_app_port: 9165, app_unit: app-pdrbd, svc_port: 1414 }
    verbs:
      qm-create:   { playbook: site-pdrbd.yml }
      qm-destroy:  { playbook: site-pdrbd-qm-down.yml }
      qm-up:       { pcs: "resource enable mq_group" }
      qm-down:     { pcs: "resource disable mq_group" }
      qm-status:   { cmd: "pcs status resources && drbdadm status all" }
      dr-cutover:  { playbook: site-pdrbd-dr-switch.yml, rpo0: true }
      dr-failback: { playbook: site-pdrbd-dr-switch.yml, rpo0: true }
      diagnostics: { cmd: "su - mqm -c '/opt/mqm/bin/runmqras -qmlist {qm}'" }
```

Before committing, confirm the chosen values are free: `grep -nE '10\.10\.[12]\.170|9165|app-pdrbd' lab/topology.yaml` must show only the new lines.

- [ ] **Step 4: `lab/versions.yaml` `stacks:` entry**

```yaml
  # Ubuntu 24.04 only until LINBIT's drbd9 PPA is proven on 26.04 (epic .github#310).
  pcmk-drbd-ubuntu: { supported: [ubuntu:24], default: ubuntu:24 }
```

- [ ] **Step 5: `src/mqlab/stacks.py` and `src/mqlab/parity.py`**

```python
_MECH_LABEL = {
    "pacemaker-san": "PCMK",
    "pacemaker-drbd": "PCMK-DRBD",
    "rdqm": "RDQM",
    "native-ha": "Native HA",
}
```

```python
    # pcmk-drbd-ubuntu (epic .github#310) starts NOT_YET; Task 9 sets each verb from
    # the recorded drill evidence.
    "pcmk-drbd-ubuntu": dict.fromkeys(VERBS, Support.NOT_YET),
```

`src/mqlab/watcherboard.py`:

```python
_PACEMAKER_MECHANISMS = ("pacemaker-san", "pacemaker-drbd")


def _owner_resource(mechanism: str, short: str) -> str:
    """The `cluster_resource_owner` resource label for a stack. Pacemaker arms own the QM
    via the pacemaker resource id (mq_qm); the other mechanisms own by the short-derived
    QM name (#351)."""
    if mechanism in _PACEMAKER_MECHANISMS:
        return "mq_qm"
    return f"{short}APP"
```

- [ ] **Step 6: Run tests to verify they pass; validate**

Run: `uv run pytest tests/test_topology_pdrbd.py tests/test_stacks.py tests/test_parity.py tests/test_watcherboard.py tests/test_dns.py tests/test_inventory.py -v` — expected: PASS (existing DNS/inventory tests must still pass with the new hosts and VIPs).
Run: `vrg-container-run -- vrg-validate` — expected: green, 100% branch coverage.

- [ ] **Step 7: Commit**

```bash
vrg-git add lab/topology.yaml lab/versions.yaml src/mqlab/stacks.py src/mqlab/parity.py src/mqlab/watcherboard.py tests/test_watcherboard.py tests/test_topology_pdrbd.py tests/test_stacks.py tests/test_parity.py
vrg-commit --type feat --scope topology --message "pcmk-drbd-ubuntu stack: nodes, groups, verbs, label, parity row (#<T4>)"
```

---

### Task 5: Site-A HA formation (`pdrbd-qm`) + the distributed workload

**Files:**
- Create: `ansible/roles/pdrbd-qm/defaults/main.yml`
- Create: `ansible/roles/pdrbd-qm/templates/qm.res.j2`
- Create: `ansible/roles/pdrbd-qm/tasks/main.yml`
- Create: `ansible/roles/pdrbd-qm/handlers/main.yml`
- Create: `ansible/templates/mq/inter-qm.mqsc.j2`, `ansible/templates/mq/authz.mqsc.j2` (moved from `ansible/roles/mq-pcmk-qmgr/templates/`)
- Create: symlinks `ansible/roles/mq-pcmk-qmgr/templates/inter-qm.mqsc.j2 -> ../../../templates/mq/inter-qm.mqsc.j2` and same for `authz.mqsc.j2`
- Create: `ansible/_pdrbd-cluster-ha.yml`, `ansible/site-pdrbd.yml`, `ansible/site-pdrbd-qm-down.yml`
- Test: `tests/test_pdrbd_qm_role.py`

**Interfaces:**
- Consumes: Task 3 `drbd9` (VG `drbdpool`), Task 4 stack/groups/hb IPs, `pcmk-cluster` (vars `pcmk_cluster_name`, `pcmk_site_nodes`), `mq-install`, `mq-authz-accounts`, `mq-diag-logging`, `mq-event-monitor`, `pki-distribute`, `site-distributed-shared.yml`; provision-phase extra-vars `qm_app`, `qm_svc`, `chl_to_svc`, `chl_to_app`, `svc_req_queue`.
- Produces: per-QM resource `{{ qm_name | lower }}` (DRBD device `/dev/drbd{{ pdrbd_minor }}`, LV `drbdpool/{{ qm_name | lower }}`, mount `/var/mqm/vols/{{ qm_name | lower }}`); Pacemaker resources `drbd_qm` (promotable clone `drbd_qm-clone`) and group `mq_group` = `mq_fs` → `mq_vip` → `mq_qm`; role vars `pdrbd_site_group` (`pdrbd_a`/`pdrbd_b`) and `pdrbd_peers` (list of `{name, addr_hb, addr_wan, node_id, site}`) consumed again by Task 6.

**Blueprint inputs (Task 2 reconciles each):** `pdrbd_lv_size`, `pdrbd_fstype`, `pdrbd_mount_opts`, `pdrbd_port`, `pdrbd_minor`, the `.res` `options`/`net`/`disk` blocks (quorum, `on-no-quorum`, `after-sb-*`, `fencing`), cluster properties (`stonith-enabled`, `no-quorum-policy`), the DRBD agent (`ocf:linbit:drbd` hypothesis), constraint set, and the replication network (hb hypothesis).

- [ ] **Step 1: Write the failing structure tests**

```python
"""pdrbd-qm role structure (epic .github#310 T5): destructive steps are guarded,
no masked failures, and the SAN role is untouched."""

import hashlib
import pathlib

import yaml

TASKS = pathlib.Path("ansible/roles/pdrbd-qm/tasks/main.yml")
# sha256 of ansible/roles/mq-pcmk-qmgr/tasks/main.yml on develop at 84d9bda (2026-10-10):
# the SAN arm's role must not change in this epic (spec D1/D7).
_MQ_PCMK_QMGR_TASKS_SHA256 = "bbb113886fc261828f299262138891224c828036a982ec97c64b106e4967e59f"


def _tasks() -> list[tuple[dict, str]]:
    """Every task with the `when` it inherits from an enclosing block."""
    out: list[tuple[dict, str]] = []
    for t in yaml.safe_load(TASKS.read_text()):
        if "block" in t:
            out.extend((inner, str(t.get("when", ""))) for inner in t["block"])
        else:
            out.append((t, ""))
    return out


def _by_name(fragment: str) -> dict:
    return next(t for t, _ in _tasks() if fragment in t["name"])


def test_pdrbd_qm_destructive_steps_are_guarded():
    for frag in ("create-md", "initial promote + mkfs", "create the queue manager"):
        task, inherited = next((t, w) for t, w in _tasks() if frag in t["name"])
        guard = inherited + str(task.get("when", ""))
        assert "pdrbd_formed" in guard, f"{frag} is not guarded by pdrbd_formed"


def test_pdrbd_qm_create_md_cannot_hang():
    body = _by_name("create-md")["ansible.builtin.shell"]
    assert "</dev/null" in body and "timeout" in body


def test_pdrbd_qm_asserts_attach():
    task = _by_name("assert the drbd disk attached")
    assert "Diskless" in task["failed_when"]


def test_pdrbd_qm_never_masks_a_failure():
    text = TASKS.read_text()
    assert "|| true" not in text
    assert "failed_when: false" not in text


def test_mq_pcmk_qmgr_templates_still_resolve():
    for name in ("inter-qm.mqsc.j2", "authz.mqsc.j2"):
        p = pathlib.Path("ansible/roles/mq-pcmk-qmgr/templates") / name
        assert p.is_symlink() and p.resolve() == (pathlib.Path("ansible/templates/mq") / name).resolve()


def test_mq_pcmk_qmgr_tasks_unchanged():
    data = pathlib.Path("ansible/roles/mq-pcmk-qmgr/tasks/main.yml").read_bytes()
    assert hashlib.sha256(data).hexdigest() == _MQ_PCMK_QMGR_TASKS_SHA256
```

If `develop` has moved and that file legitimately changed before this task starts (not in this epic), update the constant from `sha256sum ansible/roles/mq-pcmk-qmgr/tasks/main.yml` on the branch point and note the commit in the comment.

- [ ] **Step 2: Run to verify they fail**

Run: `uv run pytest tests/test_pdrbd_qm_role.py -v`
Expected: FAIL (`FileNotFoundError` on the role).

- [ ] **Step 3: Move the shared MQSC templates (old path keeps working)**

```bash
mkdir -p ansible/templates/mq
vrg-git mv ansible/roles/mq-pcmk-qmgr/templates/inter-qm.mqsc.j2 ansible/templates/mq/inter-qm.mqsc.j2
vrg-git mv ansible/roles/mq-pcmk-qmgr/templates/authz.mqsc.j2 ansible/templates/mq/authz.mqsc.j2
ln -s ../../../templates/mq/inter-qm.mqsc.j2 ansible/roles/mq-pcmk-qmgr/templates/inter-qm.mqsc.j2
ln -s ../../../templates/mq/authz.mqsc.j2 ansible/roles/mq-pcmk-qmgr/templates/authz.mqsc.j2
```

Confirm `lab/boxes/_manifest-hash.sh` follows symlinks for the role closure it digests (`grep -n 'find\|readlink\|-L' lab/boxes/_manifest-hash.sh`); the `pcmk` box does not bake these templates, so its digest must not change — run `lab/boxes/_manifest-hash.sh` for the pcmk box before and after and confirm the same digest.

- [ ] **Step 4: `ansible/roles/pdrbd-qm/defaults/main.yml` (hypothesis values; Task 2 reconciles)**

```yaml
---
# pdrbd-qm (epic .github#310): one RDQM-style replicated volume per QM. Every value below
# is the PRE-CAPTURE HYPOTHESIS; the Blueprint reconciliation comment on this task's
# issue (Task 2) is authoritative and its values replace these before this task starts.
pdrbd_vg: drbdpool
pdrbd_lv_size: 3G              # RDQM crtmqm -fs default (blueprint: lvs.txt)
pdrbd_fstype: ext4             # blueprint: mounts.txt / lsblk.txt
pdrbd_mount_opts: defaults     # blueprint: mounts.txt
pdrbd_mount_root: /var/mqm/vols
pdrbd_port: 7001               # RDQM: "next free port number above 7000" (IBM Docs)
pdrbd_minor: 100               # blueprint: drbd-dump.txt (device minor)
pdrbd_res: "{{ qm_name | lower }}"
pdrbd_mount: "{{ pdrbd_mount_root }}/{{ pdrbd_res }}"
pdrbd_stonith_enabled: false   # blueprint: pcmk-config.txt (stonith-enabled)
pdrbd_no_quorum_policy: stop   # blueprint: pcmk-config.txt
# Site membership: set by the play (Tasks 5/6) from the inventory + topology hb IPs.
#   pdrbd_peers: [{ name, addr_hb, addr_wan, node_id, site: a|b }]
#   pdrbd_site_group: pdrbd_a | pdrbd_b
```

- [ ] **Step 5: `ansible/roles/pdrbd-qm/templates/qm.res.j2`**

```jinja
# {{ ansible_managed }} — RDQM-faithful per-QM DRBD 9 resource (epic .github#310).
# options/net blocks are copied from the RDQM blueprint (docs/reference/rdqm-blueprint/
# <host>/drbd-dump.txt). Connections: same-site pairs replicate synchronously (protocol C)
# over that site's hb network; cross-site pairs (DR, only when DR is enabled) replicate
# asynchronously (protocol A) over net-wan — the hb networks are per-site, not routed.
resource {{ pdrbd_res }} {
  options {
    quorum majority;
    on-no-quorum suspend-io;
  }
  net {
    after-sb-0pri discard-zero-changes;
    after-sb-1pri discard-secondary;
    after-sb-2pri disconnect;
  }
  device minor {{ pdrbd_minor }};
  disk /dev/{{ pdrbd_vg }}/{{ pdrbd_res }};
  meta-disk internal;
{% for p in pdrbd_peers %}
  on {{ p.name }} {
    node-id {{ p.node_id }};
  }
{% endfor %}
{% for i in range(pdrbd_peers | length) %}{% for j in range(i + 1, pdrbd_peers | length) %}
{% set a = pdrbd_peers[i] %}{% set b = pdrbd_peers[j] %}{% set local = a.site == b.site %}
  connection {
    host {{ a.name }} address {{ a.addr_hb if local else a.addr_wan }}:{{ pdrbd_port }};
    host {{ b.name }} address {{ b.addr_hb if local else b.addr_wan }}:{{ pdrbd_port }};
    net { protocol {{ 'C' if local else 'A' }}; }
  }
{% endfor %}{% endfor %}
}
```

`pdrbd_peers` entries are `{name, addr_hb, addr_wan, node_id, site}`; with DR disabled (`--no-dr`) it holds only site A, so only the three protocol-C connections render.

**Known hazard the blueprint must settle before this task starts:** with one six-peer resource and `quorum majority`, cutting the WAN leaves each site with 3 of 6 peers — no majority — so `on-no-quorum suspend-io` would freeze the live site on a DR-link outage. RDQM does not behave that way, so it either scopes quorum differently (e.g. a fixed `quorum` count, or DR peers excluded from quorum) or uses separate HA and DR resources. Task 2's reconciliation records which, and this template's `options` block (or the resource split) is changed to match **before** implementation. Drill 3 and drill 6 then prove the chosen behaviour on both arms.

- [ ] **Step 6: `ansible/roles/pdrbd-qm/tasks/main.yml`**

```yaml
---
# pdrbd-qm (epic .github#310): the per-QM RDQM-style volume + Pacemaker group, on the
# local site group (pdrbd_site_group). First-time formation is guarded by `pdrbd_formed`
# (Pacemaker already owns drbd_qm) so a re-run never re-creates md, re-formats, or
# re-creates the QM over live data.

- name: event monitoring + JSON diagnostic logging host prep (arm-neutral roles)
  ansible.builtin.include_role:
    name: "{{ item }}"
  loop: [mq-diag-logging, mq-event-monitor]
  vars:
    qmgr_name: "{{ qm_name }}"

- name: probe — does Pacemaker already own this QM's DRBD resource?
  ansible.builtin.command: pcs resource status drbd_qm
  register: pdrbd_owned
  changed_when: false
  failed_when: pdrbd_owned.rc not in [0, 1]
  run_once: true
  become: true

- name: record formation state
  ansible.builtin.set_fact:
    pdrbd_formed: "{{ pdrbd_owned.rc == 0 }}"

- name: per-QM logical volume in the drbdpool VG (lvs probe)
  ansible.builtin.shell: |
    set -e
    if ! lvs {{ pdrbd_vg }}/{{ pdrbd_res }} >/dev/null 2>&1; then
      lvcreate -y -L {{ pdrbd_lv_size }} -n {{ pdrbd_res }} {{ pdrbd_vg }}
      echo LV-CREATED
    fi
  register: pdrbd_lv
  changed_when: "'LV-CREATED' in pdrbd_lv.stdout"
  become: true

- name: DRBD resource definition (blueprint-derived)
  ansible.builtin.template:
    src: qm.res.j2
    dest: "/etc/drbd.d/{{ pdrbd_res }}.res"
    mode: "0644"
  become: true

- name: create-md (only when the resource is not up and has no metadata)
  # Two wedge modes from docs/reference/drbd-operations.md: an in-use device blocks
  # forever (probe `drbdadm status` first — it never blocks), and a down device can
  # prompt on stdin (</dev/null + timeout bound it). set -e: a real failure is loud.
  ansible.builtin.shell: |
    set -e
    if drbdadm status {{ pdrbd_res }} >/dev/null 2>&1; then
      echo resource-up
    elif timeout 30 drbdadm dump-md {{ pdrbd_res }} </dev/null >/dev/null 2>&1; then
      echo md-exists
    else
      timeout 60 drbdadm create-md --force {{ pdrbd_res }} </dev/null
      echo created
    fi
  register: pdrbd_md
  changed_when: "'created' in pdrbd_md.stdout"
  when: not pdrbd_formed
  become: true

- name: bring the resource up (adjust when already up — never masked)
  ansible.builtin.shell: |
    set -e
    if drbdadm status {{ pdrbd_res }} >/dev/null 2>&1; then
      drbdadm adjust {{ pdrbd_res }}; echo adjusted
    else
      drbdadm up {{ pdrbd_res }}; echo up
    fi
  register: pdrbd_up
  changed_when: "'up' == pdrbd_up.stdout"
  become: true

- name: assert the drbd disk attached
  ansible.builtin.command: drbdadm dstate {{ pdrbd_res }}
  register: pdrbd_dstate
  changed_when: false
  failed_when: >-
    pdrbd_dstate.rc != 0
    or 'Diskless' in pdrbd_dstate.stdout
    or 'Unconfigured' in pdrbd_dstate.stdout
  become: true

- name: first-time formation on the creation node
  when: not pdrbd_formed
  run_once: true
  become: true
  block:
    - name: wait for the site peers to connect
      ansible.builtin.shell: |
        set -o pipefail
        for i in $(seq 1 60); do
          n=$(drbdadm status {{ pdrbd_res }} | grep -c 'connection:Connected' || [ $? -eq 1 ])
          [ "$n" -ge 2 ] && { echo connected; exit 0; }
          sleep 2
        done
        drbdadm status {{ pdrbd_res }}; exit 1
      args:
        executable: /bin/bash
      changed_when: false

    - name: initial promote + mkfs (fresh volume, skip the full initial sync)
      ansible.builtin.shell: |
        set -e
        drbdadm new-current-uuid --clear-bitmap {{ pdrbd_res }}/0
        drbdadm primary {{ pdrbd_res }}
        mkfs.{{ pdrbd_fstype }} -q /dev/drbd{{ pdrbd_minor }}
      changed_when: true

    # Shell, not ansible.posix.mount: the lab installs no galaxy collections (#156).
    - name: mount the volume on the creation node
      ansible.builtin.shell: |
        set -e
        mkdir -p {{ pdrbd_mount }}
        mountpoint -q {{ pdrbd_mount }} || mount -t {{ pdrbd_fstype }} -o {{ pdrbd_mount_opts }} /dev/drbd{{ pdrbd_minor }} {{ pdrbd_mount }}
        mkdir -p {{ pdrbd_mount }}/qmgr {{ pdrbd_mount }}/log
        chown -R mqm:mqm {{ pdrbd_mount }}
      changed_when: true

    - name: create the queue manager on the replicated volume (RDQM path layout)
      ansible.builtin.command: >
        su mqm -c '/opt/mqm/bin/crtmqm -md {{ pdrbd_mount }}/qmgr -ld {{ pdrbd_mount }}/log
        -p 1414 {{ qm_name }}'
      changed_when: true
```

Continue the block (still first-time, creation node) with the arm-neutral configuration, mirroring `mq-pcmk-qmgr` tasks 1–3 but on `{{ pdrbd_mount }}`:

```yaml
    - name: place the QM keystore on the replicated volume (TLS)
      ansible.builtin.include_role:
        name: pki-distribute
      vars:
        pki_qm: "{{ qm_name }}"
        pki_keystore_dir: "{{ pdrbd_mount }}/qmgr/{{ qm_name }}/ssl"

    - name: start the QM for configuration
      ansible.builtin.command: su mqm -c '/opt/mqm/bin/strmqm {{ qm_name }}'
      changed_when: true

    - name: render the inter-QM MQSC (shared template)
      ansible.builtin.template:
        src: "{{ playbook_dir }}/templates/mq/inter-qm.mqsc.j2"
        dest: /var/mqm/inter-qm.mqsc
        mode: "0644"

    - name: apply the inter-QM MQSC
      ansible.builtin.shell: su mqm -c '/opt/mqm/bin/runmqsc {{ qm_name }} < /var/mqm/inter-qm.mqsc'
      changed_when: true

    - name: render the authorization MQSC (shared template)
      ansible.builtin.template:
        src: "{{ playbook_dir }}/templates/mq/authz.mqsc.j2"
        dest: /var/mqm/authz.mqsc
        mode: "0644"

    - name: apply the authorization MQSC
      ansible.builtin.shell: su mqm -c '/opt/mqm/bin/runmqsc {{ qm_name }} < /var/mqm/authz.mqsc'
      changed_when: true

    - name: event monitoring — enable classes + define/start collector
      ansible.builtin.include_role:
        name: mq-event-monitor
        tasks_from: qm
      vars:
        qmgr_name: "{{ qm_name }}"

    - name: clean stop (hand the QM to Pacemaker)
      ansible.builtin.command: su mqm -c '/opt/mqm/bin/endmqm -w {{ qm_name }}'
      changed_when: true

    - name: capture the addmqinf command
      ansible.builtin.shell: |
        set -o pipefail
        su mqm -c '/opt/mqm/bin/dspmqinf -o command {{ qm_name }}' | grep '^addmqinf'
      args:
        executable: /bin/bash
      register: pdrbd_addmqinf
      changed_when: false

    - name: release the volume and demote (Pacemaker owns promotion from here)
      ansible.builtin.shell: |
        set -e
        umount {{ pdrbd_mount }}
        drbdadm secondary {{ pdrbd_res }}
      changed_when: true
```

Before writing the keystore/event-monitor/authz includes, open `ansible/roles/mq-pcmk-qmgr/tasks/main.yml:56-200` and copy the **exact** role names, `tasks_from`, and variable names it uses for `pki-distribute`, `mq-event-monitor` and the setmqaut grant set; the snippet above shows the shape, the source of truth for argument names is that file. Express the setmqaut grants it applies as `SET AUTHREC` lines appended to the rendered authz MQSC (repo convention: MQSC `SET AUTHREC`, not `setmqaut`), with the same principals and authorities.

Then, on every node of the site group (outside the block):

```yaml
- name: teach the QM definition to the other site nodes
  ansible.builtin.command:
    cmd: "su mqm -c '{{ hostvars[groups[pdrbd_site_group][0]].pdrbd_addmqinf.stdout }}'"
  when:
    - not pdrbd_formed
    - inventory_hostname != groups[pdrbd_site_group][0]
  register: pdrbd_addmqinf_out
  failed_when: pdrbd_addmqinf_out.rc != 0 and 'already' not in (pdrbd_addmqinf_out.stderr | default(''))
  changed_when: pdrbd_addmqinf_out.rc == 0
  become: true

- name: mount point on every node
  ansible.builtin.file:
    path: "{{ pdrbd_mount }}"
    state: directory
    owner: mqm
    group: mqm
    mode: "0775"
  become: true

- name: queue-manager systemd unit (Pacemaker is the only starter)
  ansible.builtin.copy:
    dest: "/etc/systemd/system/mq-{{ qm_name }}.service"
    mode: "0644"
    content: |
      [Unit]
      Description=IBM MQ queue manager {{ qm_name }} (pacemaker-drbd managed)
      [Service]
      Type=forking
      User=mqm
      ExecStart=/opt/mqm/bin/strmqm {{ qm_name }}
      ExecStop=/opt/mqm/bin/endmqm -w {{ qm_name }}
      TimeoutStartSec=300
  notify: daemon-reload
  become: true

- name: ensure the unit is disabled (cluster-only start)
  ansible.builtin.systemd:
    name: "mq-{{ qm_name }}.service"
    enabled: false
    daemon_reload: true
  become: true

- name: cluster properties (blueprint-derived)
  ansible.builtin.shell: |
    set -e
    pcs property set stonith-enabled={{ pdrbd_stonith_enabled | lower }}
    pcs property set no-quorum-policy={{ pdrbd_no_quorum_policy }}
    pcs resource defaults update resource-stickiness=1000
  run_once: true
  changed_when: true
  become: true

- name: Pacemaker resources (first-time only)
  when: not pdrbd_formed
  run_once: true
  become: true
  ansible.builtin.shell: |
    set -e
    pcs resource create drbd_qm ocf:linbit:drbd drbd_resource={{ pdrbd_res }} \
      op monitor interval=15s role=Promoted op monitor interval=30s role=Unpromoted \
      promotable promoted-max=1 promoted-node-max=1 clone-max=3 clone-node-max=1 notify=true
    pcs resource create mq_fs ocf:heartbeat:Filesystem device=/dev/drbd{{ pdrbd_minor }} \
      directory={{ pdrbd_mount }} fstype={{ pdrbd_fstype }} options={{ pdrbd_mount_opts }} \
      op monitor interval=20s timeout=40s OCF_CHECK_LEVEL=20 group mq_group
    pcs resource create mq_vip ocf:heartbeat:IPaddr2 ip={{ qm_vip }} cidr_netmask=24 \
      group mq_group after mq_fs
    pcs resource create mq_qm systemd:mq-{{ qm_name }} group mq_group after mq_vip
    pcs constraint colocation add mq_group with Promoted drbd_qm-clone INFINITY
    pcs constraint order promote drbd_qm-clone then start mq_group
  changed_when: true

- name: wait for the QM to run under Pacemaker
  ansible.builtin.shell: |
    set -o pipefail
    for i in $(seq 1 60); do
      pcs status resources | grep -Eq 'mq_qm.*Started' && { echo started; exit 0; }
      sleep 5
    done
    pcs status resources; exit 1
  args:
    executable: /bin/bash
  run_once: true
  changed_when: false
  become: true
```

`handlers/main.yml`:

```yaml
---
- name: daemon-reload
  ansible.builtin.systemd:
    daemon_reload: true
  become: true
```

If `pcs` on Ubuntu 24.04 (0.11.7) rejects the `group`/`after` keyword forms, use `--group`/`--after` (still accepted through 0.12.x, per the comment in `mq-pcmk-qmgr/tasks/main.yml`).

- [ ] **Step 7: `ansible/_pdrbd-cluster-ha.yml` (site A building block)**

```yaml
---
# Site A of the pcmk-drbd-ubuntu stack (epic .github#310): DRBD 9 + VG, MQ, the site's
# 3-node Pacemaker cluster, then the per-QM replicated volume and resource group.
- name: wait for site A to be SSH-ready (cold-boot guard)
  hosts: pdrbd_a
  gather_facts: false
  tasks:
    - name: wait for connection
      ansible.builtin.wait_for_connection:
        timeout: 300

- name: acl for unprivileged become
  hosts: pdrbd_a
  become: true
  tasks:
    - name: acl package
      ansible.builtin.apt:
        name: acl

- name: DRBD 9 + drbdpool VG
  hosts: pdrbd_a
  roles: [drbd9]

- name: MQ product (no-op on the baked box)
  hosts: pdrbd_a
  roles: [mq-install]

- name: site A Pacemaker cluster
  hosts: pdrbd_a
  vars:
    pcmk_cluster_name: mqpdrbd-a
    pcmk_site_nodes:
      - { name: pdrbd-a1, hb: 172.16.1.71 }
      - { name: pdrbd-a2, hb: 172.16.1.72 }
      - { name: pdrbd-a3, hb: 172.16.1.73 }
  roles: [pcmk-cluster]
```

If the blueprint shows fencing is enabled, add a `pcmk-stonith` play here with the same `fence_hypervisor_*` vars `_pcmk-cluster-ha.yml` uses, and set `pdrbd_stonith_enabled: true`.

- [ ] **Step 8: `ansible/site-pdrbd.yml`**

```yaml
---
# One full-HADR provision for pcmk-drbd-ubuntu (epic .github#310), composed like
# site-pcmk.yml: 1) site A HA, 2) site B DR receiver (Task 6), 3) authz accounts,
# 4) the per-QM volume + group, 5) the distributed workload (qm e2e needs it).
- import_playbook: _pdrbd-cluster-ha.yml

- name: authz service accounts on every app-QM node (HA + DR)
  hosts: "pdrbd_a{{ ':pdrbd_b' if (dr_enabled | default(true) | bool) else '' }}"
  become: true
  roles: [mq-authz-accounts]

- name: per-QM replicated volume + resource group (site A)
  hosts: pdrbd_a
  become: true
  vars:
    qm_name: "{{ qm_app }}"
    qm_vip: 10.10.1.170
    svc_conn: "svc-sim-ext.service.com"
    pdrbd_site_group: pdrbd_a
    # Site-A peers only in this task; Task 6 widens this to both sites when DR is enabled.
    pdrbd_peers:
      - { name: pdrbd-a1, addr_hb: 172.16.1.71, addr_wan: 10.99.0.71, node_id: 0, site: a }
      - { name: pdrbd-a2, addr_hb: 172.16.1.72, addr_wan: 10.99.0.72, node_id: 1, site: a }
      - { name: pdrbd-a3, addr_hb: 172.16.1.73, addr_wan: 10.99.0.73, node_id: 2, site: a }
  roles: [pdrbd-qm]

- import_playbook: site-distributed-shared.yml
  vars:
    our_qm: "{{ qm_app }}"
    # Like RDQM: no partner-facing VIP; the SVC QM's sender to us carries a CONNAME
    # list of the three site-A net-ext node addresses (site-rdqm.yml precedent).
    our_conn: "pdrbd-a1-ext.client.com(1414),pdrbd-a2-ext.client.com(1414),pdrbd-a3-ext.client.com(1414)"
    app_conn: "pdrbd-vip-a.client.com(1414)"
```

Read `ansible/site-rdqm.yml:429-445` and copy any further vars its `site-distributed-shared.yml` import passes (e.g. `app_tls`, `svc_partner_*`), substituting the `pdrbd-*` names. Confirm the node `-ext` DNS names exist (`mqlab dns` / `tests/test_dns.py` pattern: `<host>-ext.client.com` for hosts with `net-ext`).

- [ ] **Step 9: `ansible/site-pdrbd-qm-down.yml` (`qm-destroy`)**

```yaml
---
# qm-destroy for pcmk-drbd-ubuntu: remove the group + DRBD clone, take the resource down,
# wipe its metadata, remove the LV and the QM definition on every site-A node.
- name: remove the Pacemaker resources
  hosts: pdrbd_a
  become: true
  tasks:
    - name: delete mq_group members and drbd_qm (absent is fine; any other error is loud)
      ansible.builtin.shell: |
        set -e
        for r in mq_qm mq_vip mq_fs drbd_qm; do
          if pcs resource status "$r" >/dev/null 2>&1; then pcs resource delete "$r" --force; fi
        done
      run_once: true
      changed_when: true

- name: take the volume down and remove it
  hosts: pdrbd_a
  become: true
  vars:
    pdrbd_res: "{{ qm_name | lower }}"
  tasks:
    - name: drbd down + wipe-md + lvremove + resource file
      ansible.builtin.shell: |
        set -e
        if drbdadm status {{ pdrbd_res }} >/dev/null 2>&1; then drbdadm down {{ pdrbd_res }}; fi
        if [ -e /etc/drbd.d/{{ pdrbd_res }}.res ]; then
          timeout 60 drbdadm wipe-md --force {{ pdrbd_res }} </dev/null
          rm -f /etc/drbd.d/{{ pdrbd_res }}.res
        fi
        if lvs drbdpool/{{ pdrbd_res }} >/dev/null 2>&1; then lvremove -y drbdpool/{{ pdrbd_res }}; fi
      changed_when: true

    - name: remove the QM definition (rmvmqinf; absent is fine)
      ansible.builtin.shell: |
        set -e
        if su mqm -c "/opt/mqm/bin/dspmq -m {{ qm_name }}" >/dev/null 2>&1; then
          su mqm -c "/opt/mqm/bin/rmvmqinf {{ qm_name }}"
        fi
      changed_when: true
```

- [ ] **Step 10: Run tests; validate**

Run: `uv run pytest tests/test_pdrbd_qm_role.py -v` — expected: PASS (6).
Run: `vrg-container-run -- vrg-validate` — expected: green.

- [ ] **Step 11: Operator brings up site A HA-only (human step)**

```bash
mqlab bootstrap pcmk-drbd-ubuntu --no-dr
mqlab qm status pcmk-drbd-ubuntu
mqlab qm e2e pcmk-drbd-ubuntu
```

Expected: `qm status` shows `drbd_qm-clone` Promoted on one node, Unpromoted on two, `mq_group` Started on the promoted node, and `drbdadm status` shows the resource UpToDate with two Connected peers; `qm e2e` round-trips every request. Run `mqlab bootstrap pcmk-drbd-ubuntu --no-dr` a **second** time: expected `changed` only on probes/adjust, no create-md, mkfs or crtmqm (Review Focus 3). Paste both outputs into the issue.

- [ ] **Step 12: Ledger update + commit**

Update `docs/reference/rdqm-vs-vanilla-deviations.md`: set Status for every ST/RP/CL row now built (from `pending` to `exact`/`partial`/`none` with the consequence).

```bash
vrg-git add ansible/roles/pdrbd-qm ansible/templates/mq ansible/roles/mq-pcmk-qmgr/templates \
  ansible/_pdrbd-cluster-ha.yml ansible/site-pdrbd.yml ansible/site-pdrbd-qm-down.yml \
  tests/test_pdrbd_qm_role.py docs/reference/rdqm-vs-vanilla-deviations.md
vrg-commit --type feat --scope pdrbd --message "site-A HA: RDQM-style per-QM DRBD 9 volume under Pacemaker + distributed workload (#<T5>)"
```

---

### Task 6a: `mqlab dr` dispatches on stack DR verbs; `--rpo0-drill` generalised

**Files:**
- Modify: `src/mqlab/cli.py` (`_dr_run` and helpers, around `_RDQM_DR_CUTOVER_SCRIPT`)
- Modify: `lab/topology.yaml` (`rdqm-rhel` verbs: add `dr-cutover` / `dr-failback`)
- Modify: `tests/test_cli_dr.py`
- Modify: `docs/site/docs/operate/index.md` (the `mqlab dr` usage)

**Interfaces:**
- Consumes: `Stack.verbs` (raw dict), `lab_script()`, `repo_root()`, `_render_inventory`, `build_deps`, `run_steps`.
- Produces: verb contract for `dr-cutover` / `dr-failback`: a mapping with exactly one kind key — `script: <lab/scripts name>` (run as `bash <script> <a2b|b2a> <QM>`, `RPO0_DRILL=1` in env when drilling) or `playbook: <ansible/ name>` (run as `ansible-playbook <pb> -e dr_direction=<a2b|b2a> -e qm_name=<QM> -e rpo0_drill=<true|false>` from `ansible/`) — plus optional `rpo0: true` declaring that the implementation performs the RPO-0 seed/verify. Task 6 relies on the playbook form and its three extra-vars.

- [ ] **Step 1: Update the test fixtures and write the failing tests**

In `tests/test_cli_dr.py`, extend `_RDQM_TOPO`'s `verbs:` with:

```python
    "      dr-cutover: { script: rdqm-dr-cutover.sh, rpo0: true }\n"
    "      dr-failback: { script: rdqm-dr-cutover.sh, rpo0: true }\n"
```

Add fixtures and tests:

```python
_PDRBD_TOPO = (
    "nodes:\n  pdrbd-a1: {nics: {net-mgmt: 10.50.0.71}}\n"
    "groups:\n  pdrbd_a: [pdrbd-a1]\n"
    "stacks:\n  pcmk-drbd-ubuntu:\n    mechanism: pacemaker-drbd\n"
    "    os_family: ubuntu\n    short: PDRBD\n"
    "    cluster_group: pdrbd_a\n    groups: [pdrbd_a]\n"
    "    qm: { vip: 10.10.1.170, vip_b: 10.10.2.170 }\n"
    "    verbs:\n"
    "      dr-cutover: { playbook: site-pdrbd-dr-switch.yml, rpo0: true }\n"
    "      dr-failback: { playbook: site-pdrbd-dr-switch.yml, rpo0: true }\n"
    "svc: { short: SVC, conn: 10.60.0.50, exporter_port: 9158 }\n"
)

# A stack whose DR verb exists but does NOT implement the RPO-0 drill.
_NORPO_TOPO = _PDRBD_TOPO.replace(", rpo0: true", "")


def test_dr_cutover_runs_playbook_verb_with_direction(monkeypatch, tmp_path):
    _seed(monkeypatch, tmp_path, _PDRBD_TOPO)
    runner = RecordingRunner(results=[ScriptedResult([])])
    monkeypatch.setattr(cli, "build_deps", lambda verb, ts: _deps(runner))
    result = CliRunner().invoke(cli.app, ["dr", "cutover", "pcmk-drbd-ubuntu"])
    assert result.exit_code == 0, result.output
    cmd = runner.recorded[-1]
    assert cmd.argv == [
        "ansible-playbook", "site-pdrbd-dr-switch.yml",
        "-e", "dr_direction=a2b", "-e", "qm_name=PDRBDAPP", "-e", "rpo0_drill=false",
    ]
    assert str(cmd.cwd).endswith("/ansible")


def test_dr_failback_playbook_rpo0_drill(monkeypatch, tmp_path):
    _seed(monkeypatch, tmp_path, _PDRBD_TOPO)
    runner = RecordingRunner(results=[ScriptedResult([])])
    monkeypatch.setattr(cli, "build_deps", lambda verb, ts: _deps(runner))
    result = CliRunner().invoke(cli.app, ["dr", "failback", "pcmk-drbd-ubuntu", "--rpo0-drill"])
    assert result.exit_code == 0, result.output
    assert runner.recorded[-1].argv[-4:] == ["-e", "qm_name=PDRBDAPP", "-e", "rpo0_drill=true"]
    assert "dr_direction=b2a" in runner.recorded[-1].argv


def test_dr_rpo0_drill_refused_when_verb_does_not_declare_it(monkeypatch, tmp_path):
    _seed(monkeypatch, tmp_path, _NORPO_TOPO)
    runner = RecordingRunner(results=[])
    monkeypatch.setattr(cli, "build_deps", lambda verb, ts: _deps(runner))
    result = CliRunner().invoke(cli.app, ["dr", "cutover", "pcmk-drbd-ubuntu", "--rpo0-drill"])
    assert result.exit_code == 2
    assert "does not implement the RPO-0 drill" in result.output
    assert runner.recorded == []  # never cut over without the check the operator asked for


def test_dr_without_drill_runs_on_a_verb_without_rpo0(monkeypatch, tmp_path):
    _seed(monkeypatch, tmp_path, _NORPO_TOPO)
    runner = RecordingRunner(results=[ScriptedResult([])])
    monkeypatch.setattr(cli, "build_deps", lambda verb, ts: _deps(runner))
    result = CliRunner().invoke(cli.app, ["dr", "cutover", "pcmk-drbd-ubuntu"])
    assert result.exit_code == 0, result.output


def test_dr_unknown_verb_kind_exits_2(monkeypatch, tmp_path):
    _seed(monkeypatch, tmp_path, _PDRBD_TOPO.replace("playbook: site-pdrbd", "pcs: site-pdrbd"))
    result = CliRunner().invoke(cli.app, ["dr", "cutover", "pcmk-drbd-ubuntu"])
    assert result.exit_code == 2
    assert "unsupported dr-cutover kind 'pcs'" in result.output
```

Replace `test_dr_non_rdqm_stack_exits_2` with:

```python
def test_dr_stack_without_dr_verb_exits_2(monkeypatch, tmp_path):
    _seed(monkeypatch, tmp_path, _PCMK_TOPO)
    result = CliRunner().invoke(cli.app, ["dr", "cutover", "pcmk-ubuntu"])
    assert result.exit_code == 2
    assert "pcmk-ubuntu does not declare a dr-cutover verb" in result.output
```

Existing rdqm tests (`argv[1].endswith('/lab/scripts/rdqm-dr-cutover.sh')`, `argv[2:] == ['a2b', 'RDQMAPP']`, `env == {'RPO0_DRILL': '1'}` / `None`) stay as they are — the script path must keep producing exactly that.

- [ ] **Step 2: Run to verify the new tests fail**

Run: `uv run pytest tests/test_cli_dr.py -v`
Expected: the five new tests and the replaced test FAIL ("only the rdqm mechanism…"); the rdqm tests still PASS.

- [ ] **Step 3: Implement the dispatch in `src/mqlab/cli.py`**

Replace `_RDQM_DR_CUTOVER_SCRIPT` and `_dr_run` with:

```python
# A stack declares its own DR operation (#867 generalised, epic .github#310): verbs
# dr-cutover / dr-failback carry exactly one kind key — `script` (a lab/scripts/ entry run
# as `bash <script> <direction> <QM>`, RPO0_DRILL=1 in env when drilling) or `playbook`
# (an ansible/ playbook run with dr_direction / qm_name / rpo0_drill extra-vars) — plus an
# optional `rpo0: true` declaring that it performs the RPO-0 seed/verify. --rpo0-drill on a
# verb without it is refused: an operator who asked for the check never gets a silent pass.
_DR_KINDS = ("script", "playbook")


def _dr_impl(stack: Stack, verb: str) -> tuple[str, str, bool]:
    impl = dict(stack.verbs.get(f"dr-{verb}") or {})
    if not impl:
        typer.echo(f"dr {verb}: stack {stack.name} does not declare a dr-{verb} verb", err=True)
        raise typer.Exit(code=2)
    rpo0 = bool(impl.pop("rpo0", False))
    [(kind, value)] = impl.items()
    if kind not in _DR_KINDS:
        typer.echo(f"dr {verb}: unsupported dr-{verb} kind {kind!r} on {stack.name}", err=True)
        raise typer.Exit(code=2)
    return kind, str(value), rpo0


def _dr_command(stack: Stack, kind: str, value: str, direction: str, *, rpo0_drill: bool) -> Command:
    if kind == "script":
        argv = ["bash", str(lab_script(value)), direction, stack.qm.qm_app]
        env = {"RPO0_DRILL": "1"} if rpo0_drill else None
        return Command(argv, cwd=repo_root() / "ansible", env=env)  # noqa: S607
    argv = [
        "ansible-playbook",
        value,
        "-e",
        f"dr_direction={direction}",
        "-e",
        f"qm_name={stack.qm.qm_app}",
        "-e",
        f"rpo0_drill={'true' if rpo0_drill else 'false'}",
    ]
    return Command(argv, cwd=repo_root() / "ansible")  # noqa: S607


def _dr_run(stack_name: str, direction: str, verb: str, *, rpo0_drill: bool) -> None:
    # Resolve + gate the stack, resolve ITS dr verb, render the inventory the verb's
    # ansible reads, then run it.
    stack = _stack_qm_or_exit(stack_name)
    kind, value, rpo0 = _dr_impl(stack, verb)
    if rpo0_drill and not rpo0:
        typer.echo(
            f"dr {verb}: stack {stack.name}'s dr-{verb} does not implement the RPO-0 drill "
            "(declare `rpo0: true` only on a verb that seeds and verifies it)",
            err=True,
        )
        raise typer.Exit(code=2)
    _require_record_if_live(stack.name)  # a live stack with no OS record is refused (#280)
    deps = build_deps(verb, datetime.now(tz=UTC).strftime("%Y%m%dT%H%M%SZ"))
    try:
        _render_inventory(deps)
        step = CommandStep(
            f"{stack.name} dr {verb}",
            _dr_command(stack, kind, value, direction, rpo0_drill=rpo0_drill),
        )
        run_steps(
            [step],
            runner=deps.runner,
            renderer=deps.renderer,
            transcript=deps.transcript,
            step_mode=False,
            pauser=deps.pauser,
        )
    except StepFailedError as exc:
        raise typer.Exit(code=exc.exit_code) from exc
    finally:
        deps.transcript.close()
```

Update the `--rpo0-drill` option help to: `"seed a persistent message before the cut and assert it survives at the peer (RPO-0); only on stacks whose DR verb declares rpo0"`. Keep the section comment above `dr_app` accurate (it now dispatches per stack).

- [ ] **Step 4: Declare RDQM's DR verbs in `lab/topology.yaml`**

Under `rdqm-rhel.verbs`:

```yaml
      dr-cutover:  { script: rdqm-dr-cutover.sh, rpo0: true }
      dr-failback: { script: rdqm-dr-cutover.sh, rpo0: true }
```

The Native HA stacks keep their existing `playbook` DR verbs (no `rpo0`): they become reachable through `mqlab dr` without the drill, as a by-product (spec §5.2); not validated in this epic.

- [ ] **Step 5: Run tests; validate**

Run: `uv run pytest tests/test_cli_dr.py tests/test_topology_nativeha.py -v` — expected: PASS.
Run: `vrg-container-run -- vrg-validate` — expected: green, 100% branch coverage (the `script`/`playbook` branches, both `rpo0_drill` values, the missing-verb, unknown-kind and refused-drill exits are all covered by the tests above).

- [ ] **Step 6: Docs + commit**

In `docs/site/docs/operate/index.md`, where `mqlab dr` is described, state: `mqlab dr cutover|failback <stack> [--rpo0-drill]` runs the stack's own declared DR verb; `--rpo0-drill` is accepted only for stacks whose verb implements it (today `rdqm-rhel` and, after Task 6, `pcmk-drbd-ubuntu`).

```bash
vrg-git add src/mqlab/cli.py lab/topology.yaml tests/test_cli_dr.py docs/site/docs/operate/index.md
vrg-commit --type feat --scope dr --message "mqlab dr dispatches on each stack's declared DR verbs; rpo0 drill gated per verb (#<T6a>)"
```

---

### Task 6: DR — site B joins; operator-driven cutover/failback playbook

**Files:**
- Create: `ansible/_pdrbd-dr-replication.yml`
- Create: `ansible/site-pdrbd-dr-switch.yml`
- Create: `ansible/tasks/rpo0-drill-seed.yml`, `ansible/tasks/rpo0-drill-verify.yml`
- Create: `ansible/vars/pdrbd-peers.yml`
- Modify: `ansible/site-pdrbd.yml` (import site B; both sites load the peer list)
- Modify: `ansible/roles/pdrbd-qm/tasks/main.yml` (`pdrbd_dr_secondary` path)
- Test: `tests/test_pdrbd_dr_switch.py`, `tests/test_pdrbd_qm_role.py`, `tests/test_topology_pdrbd.py`

**Interfaces:**
- Consumes: Task 5 role `pdrbd-qm` and its vars (`pdrbd_peers`, `pdrbd_site_group`, resource `drbd_qm`, group `mq_group`); Task 6a extra-vars `dr_direction`, `qm_name`, `rpo0_drill`.
- Produces: `site-pdrbd-dr-switch.yml` honoring `dr_direction` (`a2b`|`b2a`), `rpo0_drill` (bool), and `dr_force` (bool, default false — forced DR only); the shared RPO-0 task files (inputs `rpo0_node`, `qm_name`; seed sets fact `rpo0_token`).

**Blueprint inputs:** whether site B's Pacemaker cluster holds the resources while secondary (hypothesis: defined, `target-role=Stopped`), the DRBD role/connection state of site B while secondary, and the cross-site connection protocol (hypothesis A).

- [ ] **Step 1: Write the failing structure tests**

```python
"""site-pdrbd-dr-switch.yml structure (epic .github#310 T6)."""

import pathlib

import yaml

PB = pathlib.Path("ansible/site-pdrbd-dr-switch.yml")


def _all_tasks() -> list[tuple[str, dict]]:
    out = []
    for play in yaml.safe_load(PB.read_text()):
        for t in play.get("tasks", []):
            for inner in t.get("block", [t]):
                out.append((play["name"], inner))
    return out


def _index(fragment: str) -> int:
    return next(i for i, (_, t) in enumerate(_all_tasks()) if fragment in t["name"])


def test_dr_switch_validates_direction():
    assert _index("assert dr_direction") == 0


def test_dr_switch_asserts_disk_state_before_promote():
    assert _index("assert the target's disk is UpToDate") < _index("promote at the target site")


def test_dr_switch_force_is_explicit():
    gate = next(t for _, t in _all_tasks() if "assert the target's disk is UpToDate" in t["name"])
    assert "dr_force" in str(gate["ansible.builtin.assert"]["that"])


def test_dr_switch_rpo0_seed_before_demote_and_verify_after_promote():
    seed, demote = _index("RPO-0 seed"), _index("stop and demote at the source site")
    promote, verify = _index("promote at the target site"), _index("RPO-0 verify")
    assert seed < demote < promote < verify


def test_dr_switch_never_masks_a_failure():
    text = PB.read_text()
    assert "|| true" not in text and "failed_when: false" not in text
```

- [ ] **Step 2: Run to verify they fail**

Run: `uv run pytest tests/test_pdrbd_dr_switch.py -v` — expected: FAIL (file missing).

- [ ] **Step 3: Shared RPO-0 task files (mirroring `lab/scripts/rdqm-dr-cutover.sh` `rpo0_seed`/verify)**

`ansible/tasks/rpo0-drill-seed.yml`:

```yaml
---
# RPO-0 drill seed (epic .github#310; mirrors rdqm-dr-cutover.sh rpo0_seed): a uniquely
# tagged PERSISTENT message on the live QM before the cut. Inputs: rpo0_node, qm_name.
- name: RPO-0 seed — token
  ansible.builtin.set_fact:
    rpo0_token: "RPO0-{{ lookup('pipe', 'date -u +%Y%m%dT%H%M%SZ') }}-{{ 99999 | random }}"
  run_once: true

- name: RPO-0 seed — define the drill queue and put the token
  ansible.builtin.shell: |
    set -e
    echo 'DEFINE QLOCAL(DR.RPO0.DRILL) DEFPSIST(YES) REPLACE' | su mqm -c '/opt/mqm/bin/runmqsc {{ qm_name }}'
    printf '%s\n' '{{ rpo0_token }}' | su mqm -c '/opt/mqm/samp/bin/amqsput DR.RPO0.DRILL {{ qm_name }}'
  delegate_to: "{{ rpo0_node }}"
  run_once: true
  changed_when: true
  become: true
```

`ansible/tasks/rpo0-drill-verify.yml`:

```yaml
---
# RPO-0 drill verify: the token must be retrievable at the new live site. A miss fails.
- name: RPO-0 verify — get from the drill queue at the new live QM
  ansible.builtin.shell: |
    set -o pipefail
    su mqm -c '/opt/mqm/samp/bin/amqsget DR.RPO0.DRILL {{ qm_name }}'
  args:
    executable: /bin/bash
  delegate_to: "{{ rpo0_node }}"
  run_once: true
  register: rpo0_got
  changed_when: true
  become: true

- name: RPO-0 verify — assert the token survived the cut
  ansible.builtin.assert:
    that: rpo0_token in rpo0_got.stdout
    fail_msg: "RPO-0 violated: token {{ rpo0_token }} not found at {{ rpo0_node }} ({{ qm_name }})"
  run_once: true
```

Check `amqsget`'s exit code when the queue drains (it waits ~15 s then exits 0 after "no more messages"); if it is non-zero on a drained queue in this MQ level, keep `set -o pipefail` but accept that specific rc via `failed_when: rpo0_got.rc not in [0, <observed rc>]` and record the observed value in a comment.

- [ ] **Step 4: `ansible/_pdrbd-dr-replication.yml` (site B receiver)**

```yaml
---
# Site B of pcmk-drbd-ubuntu (epic .github#310): the DR receiver. Same building blocks as
# site A; the per-QM volume is joined as a DRBD peer (no create-md over site A's data —
# create-md here makes EMPTY metadata that syncs from site A), and the resource group is
# defined STOPPED so only an operator cutover starts it (RDQM DR secondary semantics).
- name: wait for site B to be SSH-ready (cold-boot guard)
  hosts: pdrbd_b
  gather_facts: false
  tasks:
    - name: wait for connection
      ansible.builtin.wait_for_connection:
        timeout: 300

- name: acl for unprivileged become
  hosts: pdrbd_b
  become: true
  tasks:
    - name: acl package
      ansible.builtin.apt:
        name: acl

- name: DRBD 9 + drbdpool VG
  hosts: pdrbd_b
  roles: [drbd9]

- name: MQ product (no-op on the baked box)
  hosts: pdrbd_b
  roles: [mq-install]

- name: site B Pacemaker cluster
  hosts: pdrbd_b
  vars:
    pcmk_cluster_name: mqpdrbd-b
    pcmk_site_nodes:
      - { name: pdrbd-b1, hb: 172.16.2.81 }
      - { name: pdrbd-b2, hb: 172.16.2.82 }
      - { name: pdrbd-b3, hb: 172.16.2.83 }
  roles: [pcmk-cluster]
```

Then: (a) create `ansible/vars/pdrbd-peers.yml` holding the full peer list, and load it with `vars_files: [vars/pdrbd-peers.yml]` in **both** the site-A and site-B `pdrbd-qm` plays of `site-pdrbd.yml` (replacing the site-A play's inline `pdrbd_peers` from Task 5) —

```yaml
---
# pcmk-drbd-ubuntu DRBD peers (epic .github#310). hb = same-site sync replication;
# wan = cross-site async DR. Must match lab/topology.yaml pdrbd-* nics.
pdrbd_peers_all:
  - { name: pdrbd-a1, addr_hb: 172.16.1.71, addr_wan: 10.99.0.71, node_id: 0, site: a }
  - { name: pdrbd-a2, addr_hb: 172.16.1.72, addr_wan: 10.99.0.72, node_id: 1, site: a }
  - { name: pdrbd-a3, addr_hb: 172.16.1.73, addr_wan: 10.99.0.73, node_id: 2, site: a }
  - { name: pdrbd-b1, addr_hb: 172.16.2.81, addr_wan: 10.99.0.81, node_id: 3, site: b }
  - { name: pdrbd-b2, addr_hb: 172.16.2.82, addr_wan: 10.99.0.82, node_id: 4, site: b }
  - { name: pdrbd-b3, addr_hb: 172.16.2.83, addr_wan: 10.99.0.83, node_id: 5, site: b }
pdrbd_peers: "{{ pdrbd_peers_all if (dr_enabled | default(true) | bool) else pdrbd_peers_all | selectattr('site', 'equalto', 'a') | list }}"
```

Add to `tests/test_topology_pdrbd.py` a guard that the vars file matches topology:

```python
def test_pdrbd_peer_vars_match_topology():
    peers = yaml.safe_load(pathlib.Path("ansible/vars/pdrbd-peers.yml").read_text())["pdrbd_peers_all"]
    nodes = _topology()["nodes"]
    for p in peers:
        nics = nodes[p["name"]]["nics"]
        hb = nics.get("net-hb-a") or nics.get("net-hb-b")
        assert (p["addr_hb"], p["addr_wan"]) == (hb, nics["net-wan"]), p["name"]
```

Task 5's template already renders cross-site pairs over `addr_wan` with protocol A, so widening `pdrbd_peers` is the only change site A needs; the site-A play re-renders its `.res` and `drbdadm adjust` adds the new connections. Which network RDQM uses for HA vs DR replication comes from the blueprint; if it differs from hb/wan, change the two address keys, not the template shape.

(b) import `_pdrbd-dr-replication.yml` after `_pdrbd-cluster-ha.yml` under `when: dr_enabled | default(true) | bool`; (c) add a site-B play running `pdrbd-qm` with `pdrbd_site_group: pdrbd_b`, `qm_vip: 10.10.2.170`, and a role var `pdrbd_dr_secondary: true`. In `pdrbd-qm`, when `pdrbd_dr_secondary` is true: skip the whole first-time formation block (the data comes from site A); still render the `.res`, `create-md` (empty metadata on site B's own LV), `up`, assert attach, `addmqinf` using site A's captured `pdrbd_addmqinf` (`hostvars[groups['pdrbd_a'][0]].pdrbd_addmqinf.stdout` — the site-A play must run first in the same `ansible-playbook` invocation), and create the Pacemaker resources with `pcs resource create … --disabled` on `drbd_qm` (so site B never promotes on its own). Add the guard test to `tests/test_pdrbd_qm_role.py`:

```python
def test_dr_secondary_never_runs_first_time_formation():
    block = next(t for t in yaml.safe_load(TASKS.read_text()) if t.get("name") == "first-time formation on the creation node")
    assert "pdrbd_dr_secondary" in str(block["when"])
```

and make the block's `when:` read `not pdrbd_formed and not (pdrbd_dr_secondary | default(false))`.

- [ ] **Step 5: `ansible/site-pdrbd-dr-switch.yml`**

```yaml
---
# Operator-driven cross-site switch for pcmk-drbd-ubuntu (epic .github#310) — the
# open-source counterpart of `rdqmdr -s` (source) + `rdqmdr -p` (target). Run via
# `mqlab dr cutover|failback pcmk-drbd-ubuntu [--rpo0-drill]`, which passes dr_direction,
# qm_name and rpo0_drill. dr_force=true (manual only) permits promoting an Outdated
# target after the source site is lost — the forced-DR drill.
- name: resolve direction
  hosts: localhost
  gather_facts: false
  tasks:
    - name: assert dr_direction (and no RPO-0 drill on a forced cutover)
      ansible.builtin.assert:
        that:
          - dr_direction in ['a2b', 'b2a']
          - not ((dr_force | default(false) | bool) and (rpo0_drill | default(false) | bool))
        fail_msg: >-
          dr_direction must be a2b or b2a (got {{ dr_direction | default('unset') }}), and a
          forced cutover cannot run the RPO-0 drill (RPO>0 is its expected outcome)

    - name: source/target groups
      ansible.builtin.set_fact:
        dr_src: "{{ 'pdrbd_a' if dr_direction == 'a2b' else 'pdrbd_b' }}"
        dr_dst: "{{ 'pdrbd_b' if dr_direction == 'a2b' else 'pdrbd_a' }}"

- name: switch
  hosts: localhost
  gather_facts: false
  vars:
    pdrbd_res: "{{ qm_name | lower }}"
    src: "{{ hostvars['localhost'].dr_src }}"
    dst: "{{ hostvars['localhost'].dr_dst }}"
  tasks:
    - name: find the source site's promoted node (skipped on a forced cutover)
      ansible.builtin.shell: |
        set -o pipefail
        crm_mon -1 -r | awk '/drbd_qm-clone/{f=1} f && /Promoted:/{gsub(/[\[\]]/,""); print $NF; exit}'
      args:
        executable: /bin/bash
      delegate_to: "{{ groups[src][0] }}"
      register: src_promoted
      changed_when: false
      failed_when: src_promoted.rc != 0 or src_promoted.stdout | length == 0
      become: true
      when: not (dr_force | default(false))

    - name: RPO-0 seed on the live QM
      ansible.builtin.include_tasks: tasks/rpo0-drill-seed.yml
      vars:
        rpo0_node: "{{ src_promoted.stdout }}"
      when: rpo0_drill | default(false) | bool

    - name: stop and demote at the source site (disable the group, then the DRBD clone)
      ansible.builtin.shell: |
        set -e
        pcs resource disable mq_group --wait=300
        pcs resource disable drbd_qm-clone --wait=300
      delegate_to: "{{ groups[src][0] }}"
      changed_when: true
      become: true
      when: not (dr_force | default(false))

    - name: target disk state
      ansible.builtin.command: drbdadm dstate {{ pdrbd_res }}
      delegate_to: "{{ groups[dst][0] }}"
      register: dst_dstate
      changed_when: false
      become: true

    - name: assert the target's disk is UpToDate (or Outdated under explicit dr_force)
      ansible.builtin.assert:
        that: >-
          dst_dstate.stdout.split('/')[0] == 'UpToDate'
          or ((dr_force | default(false)) and dst_dstate.stdout.split('/')[0] == 'Outdated')
        fail_msg: >-
          target disk is {{ dst_dstate.stdout }} — refusing to promote. Wait for resync,
          or pass -e dr_force=true for a forced DR that accepts the RPO>0 loss.

    - name: promote at the target site (enable the DRBD clone, then the group)
      ansible.builtin.shell: |
        set -e
        pcs resource enable drbd_qm-clone --wait=300
        pcs resource enable mq_group --wait=300
      delegate_to: "{{ groups[dst][0] }}"
      changed_when: true
      become: true

    - name: find the target site's promoted node
      ansible.builtin.shell: |
        set -o pipefail
        crm_mon -1 -r | awk '/drbd_qm-clone/{f=1} f && /Promoted:/{gsub(/[\[\]]/,""); print $NF; exit}'
      args:
        executable: /bin/bash
      delegate_to: "{{ groups[dst][0] }}"
      register: dst_promoted
      changed_when: false
      failed_when: dst_promoted.rc != 0 or dst_promoted.stdout | length == 0
      become: true

    - name: RPO-0 verify at the new live QM
      ansible.builtin.include_tasks: tasks/rpo0-drill-verify.yml
      vars:
        rpo0_node: "{{ dst_promoted.stdout }}"
      when: rpo0_drill | default(false) | bool
```

Run `crm_mon -1 -r` on a live site once (Task 5's bring-up) and adjust the `awk` to the actual "Promoted:" line format of this Pacemaker version before relying on it; the test asserts ordering, the live run asserts the parse.

- [ ] **Step 5b: Watched wait for the DR initial sync (spec §7)**

Site B joins after site A already holds the QM, so its peers start a **full** sync over `net-wan`. Provision must not report success until DR is actually in sync, and a stalled sync must fail, not hang. Add this test to `tests/test_pdrbd_dr_switch.py`:

```python
SITE = pathlib.Path("ansible/site-pdrbd.yml")


def test_provision_waits_for_dr_sync_with_a_stall_window():
    plays = yaml.safe_load(SITE.read_text())
    names = [p.get("name", "") for p in plays]
    site_b = next(i for i, n in enumerate(names) if "site B" in n and "volume" in n)
    wait = next(i for i, n in enumerate(names) if "wait for DR sync" in n)
    assert site_b < wait
    body = str(plays[wait])
    assert "pdrbd_sync_stall_secs" in body and "pdrbd_sync_max_secs" in body
    assert "dr_enabled" in str(plays[wait]["tasks"][0].get("when", ""))
```

Then append to `ansible/site-pdrbd.yml`, after the site-B `pdrbd-qm` play (name that play `per-QM replicated volume (site B, DR secondary)` so the test can find it):

```yaml
# The DR receiver starts a FULL initial sync (site A already holds the QM). Watch it: log
# progress, fail if the sync percentage does not advance within pdrbd_sync_stall_secs, and
# fail outright past pdrbd_sync_max_secs — a stall is a failure, not a wait (spec §7).
- name: wait for DR sync (every peer UpToDate), watched
  hosts: "{{ groups['pdrbd_a'][0] }}"
  become: true
  vars:
    pdrbd_res: "{{ qm_app | lower }}"
    pdrbd_sync_stall_secs: 180
    pdrbd_sync_max_secs: 3600
  tasks:
    - name: poll drbdadm status until all peers are UpToDate
      ansible.builtin.shell: |
        set -o pipefail
        start=$(date +%s); last_pct=""; last_move=$start
        while :; do
          st=$(drbdadm status {{ pdrbd_res }})
          if ! grep -Eq 'peer-disk:(Inconsistent|Outdated|DUnknown)|replication:(SyncSource|SyncTarget|WFBitMap)' <<<"$st"; then
            echo "in-sync after $(( $(date +%s) - start ))s"; exit 0
          fi
          pct=$(grep -Eo 'done:[0-9.]+' <<<"$st" | sort -t: -k2 -n | head -1)
          now=$(date +%s)
          if [ "$pct" != "$last_pct" ]; then
            echo "$(date -u +%H:%M:%S) sync ${pct:-pending}"; last_pct=$pct; last_move=$now
          fi
          if [ $(( now - last_move )) -ge {{ pdrbd_sync_stall_secs }} ]; then
            echo "DR sync STALLED at ${pct:-pending} for {{ pdrbd_sync_stall_secs }}s" >&2; echo "$st" >&2; exit 1
          fi
          if [ $(( now - start )) -ge {{ pdrbd_sync_max_secs }} ]; then
            echo "DR sync exceeded {{ pdrbd_sync_max_secs }}s" >&2; echo "$st" >&2; exit 1
          fi
          sleep 10
        done
      args:
        executable: /bin/bash
      changed_when: false
      when: dr_enabled | default(true) | bool  # plays take no `when:`; gate the task
```

The resync buffers from `docs/reference/drbd-operations.md` (`max-buffers 80k`, `sndbuf-size 2M`, `rcvbuf-size 2M`, `c-plan-ahead 0`, `resync-rate 500M`) belong in the cross-site `connection { net { … } }` / `disk { … }` blocks of `qm.res.j2` unless the blueprint shows RDQM's own values — without them the #67 finding applies (~250 KB/s initial resync). Before relying on the `grep` patterns, run `drbdadm status <res>` during a live sync once and confirm the state words (`replication:SyncSource`, `done:`) for this DRBD 9 version.

- [ ] **Step 6: Run tests; validate**

Run: `uv run pytest tests/test_pdrbd_dr_switch.py tests/test_pdrbd_qm_role.py -v` — expected: PASS.
Run: `vrg-container-run -- vrg-validate` — expected: green.

- [ ] **Step 7: Operator brings up full HA+DR and exercises the switch (human step)**

```bash
mqlab teardown pcmk-drbd-ubuntu
mqlab bootstrap pcmk-drbd-ubuntu
mqlab qm status pcmk-drbd-ubuntu
mqlab dr cutover pcmk-drbd-ubuntu --rpo0-drill
mqlab dr failback pcmk-drbd-ubuntu --rpo0-drill
mqlab qm e2e pcmk-drbd-ubuntu
```

Expected: site B's DRBD peers show UpToDate before the cutover; both switches succeed with the RPO-0 token found; `qm e2e` round-trips after failback. Paste outputs into the issue.

- [ ] **Step 8: Ledger + parity + commit**

Update the DR rows of `docs/reference/rdqm-vs-vanilla-deviations.md`. Set `pcmk-drbd-ubuntu` `MATRIX` entries in `src/mqlab/parity.py` for `dr-bootstrap`, `cutover`, `failback` to `SUPPORTED` and update `tests/test_parity.py` accordingly (the remaining verbs stay `NOT_YET` until Task 9).

```bash
vrg-git add ansible/_pdrbd-dr-replication.yml ansible/site-pdrbd-dr-switch.yml ansible/tasks/rpo0-drill-*.yml ansible/vars/pdrbd-peers.yml tests/test_topology_pdrbd.py \
  ansible/site-pdrbd.yml ansible/roles/pdrbd-qm tests/test_pdrbd_dr_switch.py tests/test_pdrbd_qm_role.py \
  src/mqlab/parity.py tests/test_parity.py docs/reference/rdqm-vs-vanilla-deviations.md
vrg-commit --type feat --scope pdrbd --message "DR: site B receiver + operator-driven cross-site switch with RPO-0 drill (#<T6>)"
```

---

### Task 7 (validation): Cold rebuild brings `pcmk-drbd-ubuntu` up in one pass

Filed with `vrg-issue-create --kind validation`, blocked-by Task 6 (and Task 3's box). Not PR-workable; run with `issue-validate`.

- **Precondition self-check:** Tasks 3–6 merged to `develop`; `mqlab box status` lists `pdrbd` as current (not stale) for the host arch.
- **Procedure:**
  1. `mqlab teardown pcmk-drbd-ubuntu` (if anything is up), then a full cold rebuild per `docs/site` (destroy the guests and rebuild from the cached boxes).
  2. `mqlab bootstrap pcmk-drbd-ubuntu` — one invocation, no manual intervention.
  3. `mqlab qm status pcmk-drbd-ubuntu`; `mqlab qm e2e pcmk-drbd-ubuntu`.
  4. `mqlab bootstrap pcmk-drbd-ubuntu` again (idempotency, Review Focus 3).
  5. `mqlab dr cutover pcmk-drbd-ubuntu --rpo0-drill`; `mqlab dr failback pcmk-drbd-ubuntu --rpo0-drill`.
- **Acceptance:** step 2 succeeds in one pass **and its transcript shows the watched DR-sync wait ending `in-sync after <n>s`** (record `n`); step 3 shows the promoted node, the group Started, all DRBD peers UpToDate, e2e green; step 4 reports no create-md/mkfs/crtmqm; step 5 succeeds both ways with the token found.
- **Results template:** `Outcome: SUCCESS|FAILURE`, bootstrap transcript path, `qm status` output, e2e summary, the two dr outputs.

### Task 8 (validation): Drill set on `rdqm-rhel` and `pcmk-drbd-ubuntu`

Filed with `--kind validation`, blocked-by Task 7. Arms run **one at a time** (bring one up, drill, record, tear down, then the other). For each arm, record per drill: the commands run, the observed end state (`mqlab qm status`), whether the QM ran on more than one node at any point (split-brain check: `drbdadm status` on all site nodes shows at most one Primary), and `mqlab qm e2e` before/after.

| # | Drill | Procedure (both arms) | Expected (both arms; differences are findings, not failures) |
|---|---|---|---|
| 1 | Controlled move | `mqlab qm e2e <stack>`; `mqlab qm down <stack>`; `mqlab qm up <stack>`; `mqlab qm e2e <stack>` | QM stops cleanly, restarts; e2e green before and after |
| 2 | Hard kill of the active node | find the active node (`mqlab qm status`); `virsh destroy <its libvirt domain>` (resolve the domain name the way `lab/scripts/nativeha-fault-suite.sh` does); wait; `mqlab qm status`; `mqlab qm e2e`; restart the domain | QM restarts on a surviving node of the same site; no data loss (e2e green) |
| 3 | HA network loss | `lab/scripts/net-down.sh net-hb-a`; observe 2 min; `mqlab qm status`; `lab/scripts/net-up.sh net-hb-a`; observe recovery | No node runs the QM without quorum; after restore one Primary, resync completes, e2e green |
| 4 | Loss of 2 of 3 HA nodes | `virsh destroy` two site-A nodes including the active one; `mqlab qm status` on the survivor | The survivor does **not** run the QM (no quorum); restoring one node restores service |
| 5 | DR cutover + failback | `mqlab dr cutover <stack> --rpo0-drill`; `mqlab qm e2e <stack>`; `mqlab dr failback <stack> --rpo0-drill`; `mqlab qm e2e <stack>` | Both succeed, token found both ways (RPO 0) |
| 6 | Forced DR with degraded link | `lab/scripts/drbd-degrade.sh break <node> <res>` with the arm's **explicit** node and DRBD resource (RDQM: the site-A HA primary and the RDQM resource name from the blueprint; pcmk-drbd: the promoted site-A node and `pdrbdapp`) — never the script's SAN defaults; put a few messages; then forced promote at site B (RDQM: `rdqmdr -p -m <QM>` per IBM's forced procedure; pcmk-drbd: `ansible-playbook site-pdrbd-dr-switch.yml -e dr_direction=a2b -e qm_name=PDRBDAPP -e dr_force=true` after stopping site A); count messages lost | The forced promote succeeds only with the explicit force; the loss equals the post-break tail; without force the pcmk-drbd playbook refuses (Review Focus 4) |

**Results template:** one table per arm with the six rows (Outcome, observed end state, split-brain check, e2e before/after, notes), plus `Outcome: SUCCESS` when every drill was executed and recorded on both arms (a behavioral difference between arms is a recorded finding, not a FAILURE; a drill that could not be executed is a FAILURE).

---

### Task 9: Comparison report, finalized ledger, accurate parity matrix

**Files:**
- Create: `docs/reports/<YYYY-MM-DD>-rdqm-vs-pacemaker-drbd-comparison.md`
- Modify: `docs/reference/rdqm-vs-vanilla-deviations.md` (no `pending` rows remain)
- Modify: `src/mqlab/parity.py`, `tests/test_parity.py` (both rows from evidence)

**Interfaces:**
- Consumes: the Task 8 results comment (both arms), the ledger, the blueprint.

- [ ] **Step 1: Parity rows from evidence (failing test first)**

In `tests/test_parity.py`, replace the `pcmk-drbd-ubuntu` and add an `rdqm-rhel` assertion that encodes the Task 8 evidence exactly — e.g. if every verb was exercised on both arms:

```python
def test_rdqm_and_pdrbd_rows_reflect_the_drill_evidence():
    for arm in ("rdqm-rhel", "pcmk-drbd-ubuntu"):
        assert set(parity.MATRIX[arm].values()) == {parity.Support.SUPPORTED}, arm
```

If a verb was not exercised (e.g. `add-node`/`evacuate-node` were not drilled), assert `NOT_YET` for exactly those and say so in the report. Run: `uv run pytest tests/test_parity.py -v` → FAIL; then update `MATRIX` (replace `dict.fromkeys(VERBS, Support.NOT_YET)` for `rdqm-rhel` with the evidenced mapping and drop the stale comment "RDQM rows start NOT_YET until P3/P4 land"); re-run → PASS.

- [ ] **Step 2: Finalize the ledger** — every row `exact`/`partial`/`none`, each with its consequence and a citation (blueprint file or Task 8 drill row).

- [ ] **Step 3: Write the report**

Sections: (1) Question and method (blueprint-first build; one-at-a-time drills; functional only — no timing comparison, and why). (2) Data — the drill tables from Task 8 for both arms, verbatim, and the ledger summary by category with counts. (3) Judgment — what IBM adds, argued only from the ledger rows and drill differences, each claim citing its row; separate subsections for "matched with upstream parts", "matched with effort" (what it took), and "not matched". (4) Limits — lab scale, arm64 vs x86, single run per drill. (5) Implications for the `pacemaker-san` decision (input to Task 10). Cite external facts with checkable links (LINBIT on DRBD 8.4/9, IBM Docs cached pages by `source_url`).

- [ ] **Step 4: Validate and commit**

Run: `vrg-container-run -- vrg-validate` — expected: green.

```bash
vrg-git add docs/reports docs/reference/rdqm-vs-vanilla-deviations.md src/mqlab/parity.py tests/test_parity.py
vrg-commit --type docs --scope report --message "RDQM vs pacemaker-drbd comparison report; ledger and parity from evidence (#<T9>)"
```

---

### Task 10: `pacemaker-san` decision

**Files:**
- Create: `docs/reports/<YYYY-MM-DD>-pacemaker-san-decision.md`

- [ ] **Step 1: Draft the options with evidence** — (a) retire `pcmk-ubuntu`/`pacemaker-san` from this lab; (b) keep it as a demoted, non-co-maintained contrast arm; (c) extract the reusable SAN pieces (`iscsi-target`, `iscsi-initiator`, `drbd-san`, the `san` box) for a future database-resiliency lab, then retire here. For each: what it removes or keeps (roles, boxes, networks, topology entries, docs), maintenance and cold-rebuild cost, and what Task 9's report says the SAN arm still demonstrates.
- [ ] **Step 2: Human decides** — present the draft; record the decision, date, and rationale (paraphrased, not quoted).
- [ ] **Step 3: Follow-on** — if (a) or (c): file a follow-on epic brainstorm via `triage-capture` (`idea`) naming the decision doc; if full `pcmk-drbd-ubuntu` parity (observability, cockpit) becomes needed because `pcmk-ubuntu` is retired, name it in the same capture. Record the follow-on refs in the doc.
- [ ] **Step 4: Validate and commit**

```bash
vrg-git add docs/reports/*-pacemaker-san-decision.md
vrg-commit --type docs --scope decision --message "pacemaker-san decision record (#<T10>)"
```

---

## Bookend tasks (already created)

- **D** — `logical-minds-foundry/.github#311` — this spec + plan (closed by the docs PR).
- **R1** — `logical-minds-foundry/mq-resiliency-lab-for-linux#1419` — documentation review sweep (blocked-by Task 10; spawns per-repo doc tasks as needed).
- **R2** — `logical-minds-foundry/.github#312` — retrospective (terminal; `epic-retrospective`).

## Filing (after the docs PR merges)

File Tasks 1, 2, 3, 4, 5, 6a, 6, 9, 10 in `logical-minds-foundry/mq-resiliency-lab-for-linux` with `vrg-issue-create --epic logical-minds-foundry/.github#310 --repo logical-minds-foundry/mq-resiliency-lab-for-linux`, each with `--blocked-by` per the dependency graph; Tasks 7 and 8 with `--kind validation` and `--blocked-by` Task 6 / Task 7 respectively; then add `Blocked-by:` Task 10 to R1 (#1419). Replace each `#<Tn>` placeholder in commit messages with the filed issue number.
