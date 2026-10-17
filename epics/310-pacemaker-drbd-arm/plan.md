# pacemaker-drbd arm — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `pcmk-drbd-ubuntu` stack (mechanism `pacemaker-drbd`) that rebuilds RDQM's shared-nothing HA/DR architecture from upstream Corosync/Pacemaker/DRBD 9 on Ubuntu, using RDQM's captured live configuration as the blueprint, then compare it with `rdqm-rhel` on an identical drill set.

**Architecture:** All implementation lands in **`logical-minds-foundry/mq-resiliency-lab-for-linux`** (this plan lives in `.github`). A read-only capture of the live RDQM arm becomes the committed blueprint (`docs/reference/rdqm-blueprint/`) and seeds a deviation ledger. A new baked box carries LINBIT's DRBD 9 (`drbd-dkms`). Two new roles (`drbd9`, `pdrbd-qm`) form, per QM and like RDQM, a site-local HA DRBD resource under each site's own Pacemaker cluster and a DR resource stacked on top of it that replicates to the other site; `mqlab dr` is generalised to dispatch on each stack's own DR verbs so both arms run the DR drills with one command. The SAN arm is not touched; its fate is decided in Task 10.

**Tech Stack:** Ansible (roles/plays, ansible-core only — no galaxy collections); DRBD 9 (`ppa:linbit/linbit-drbd9-stack`, `drbd-dkms`, `drbd-utils`); Pacemaker/Corosync/pcs (Ubuntu 24.04 archive); IBM MQ (the `lab/mq-version` pin, Ubuntu debs via `mq-install`); `mqlab` (Python 3 + Typer, pytest at 100% branch coverage); libvirt/Vagrant lab.

**Spec:** `epics/310-pacemaker-drbd-arm/spec.md` (this directory). Executors read both.

## Global Constraints

- **The RDQM blueprint is authoritative.** Wherever the captured RDQM configuration shows how something is done, the open-source arm does the same with upstream parts; wherever it cannot, the deviation is a ledger row. Task 2's reconciliation (lab #1427/#1428) replaced the pre-capture hypothesis in Tasks 5–6 with blueprint values; `.github#362` rewrote those tasks for RDQM's two-resource design (see "Evidence for Tasks 5, 6 and 8").
- **No change to `pcmk-ubuntu` / `pacemaker-san`.** `mq-pcmk-qmgr`, `drbd-san`, `iscsi-*`, `bake-pcmk-ubuntu.yml`, the `pcmk` box and the SAN networks are not modified (template symlinks in Task 5 leave `mq-pcmk-qmgr`'s files resolving exactly as before).
- **DRBD 9 from LINBIT's PPA (`drbd-dkms`)**, never the in-kernel 8.4 module. The module is built at bake time; a node never compiles a kernel module at boot.
- **No Booth; DR is operator-driven** (mirrors `rdqmdr`).
- **Scope:** core + comparison, **plus** the distributed app↔SVC workload with its TLS and CHLAUTH authz (needed by `mqlab qm e2e`). Observability, exporters, and cockpit boards are out of scope except where the stack registry provides them unchanged.
- **Functional comparison only** — no timing claims between arms.
- **Fail loud.** No `|| true`, no `failed_when: false` on a step that does work, no swallowed errors; probes use `changed_when: false` and an explicit `failed_when`.
- **DRBD lessons pre-applied** (`docs/reference/drbd-operations.md`): `create-md` with `</dev/null` + `timeout`; live state via `drbdadm status` (never `dump-md` on an in-use device); attach asserted via `drbdadm dstate`. The stacked DR resource is not listed by `drbdadm status all` (blueprint README §2.1), so its probes use `drbdsetup status`, which never blocks either.
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
3. **Re-running provision on an already-formed stack** — `create-md` (HA and stacked DR), the initial promotes/`mkfs`, and `crtmqm` must be skipped, never re-run over live data; and a formed QM's DR shape must never change under it. *(Test: Task 5, `test_pdrbd_qm_destructive_steps_are_guarded`, extended in Task 6 with the stacked steps, plus Task 6 `test_dr_shape_is_fixed_at_creation`; live: Task 7 runs provision twice.)*
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
                                                       T6b DRBD kernel TLS (lab #1609)
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

### Evidence for Tasks 5, 6 and 8 (reconciled by `.github#362`)

Task 2 (lab #1425) captured the blueprint, and it contradicted this plan on a structural point: **RDQM builds HA and DR as two separate DRBD resources**, not one six-peer resource. The owner decided on 2026-10-17 (`.github#362`) to copy that design with upstream parts. Tasks 5, 6 and 8 below are rewritten to match. Every value carries its source:

- **Blueprint files** are cited as `rdqm-a1/drbd-dump.txt` and similar. They are relative to the lab repo's `docs/reference/rdqm-blueprint/` (the 2026-10-16 capture: MQ 10.0.1.0, RHEL 10.2, LINBIT DRBD 9.3.3, drbd-utils 9.33.1). Its `README.md` § numbers are cited as `BP README §n`. Ledger rows are cited by ID (`RP-1`, `CL-5`, …) from `docs/reference/rdqm-vs-vanilla-deviations.md`.
- **DRBD 9 behaviour** is cited from upstream, fetched 2026-10-17:
  - **[UG]** *DRBD 9.0 User's Guide*, <https://linbit.com/drbd-user-guide/drbd-guide-9_0-en/>. Sections used: "Creating a stacked 3-node setup" / "Device stacking considerations", "Internal meta data", "Configuring quorum", "Configuring multiple paths", "Automatic promotion of resources".
  - **[conf]** `drbd.conf-9.0(5)`, <https://manpages.ubuntu.com/manpages/noble/man5/drbd.conf-9.0.5.html>.
  - **[opts]** drbd-utils `documentation/v9/drbdsetup-options.xml`, the option text that `drbd.conf-9.0(5)` and `drbdsetup(8)` include: <https://github.com/LINBIT/drbd-utils/blob/master/documentation/v9/drbdsetup-options.xml>.
  - **[ocf]** The `ocf:linbit:drbd` agent source: <https://github.com/LINBIT/drbd-utils/blob/master/scripts/drbd.ocf>.
- **Pacemaker:** **[PE]** *Pacemaker Explained* 2.1, "Collective resources": <https://clusterlabs.org/projects/pacemaker/doc/2.1/Pacemaker_Explained/html/collective.html>.
- **IBM:** `rdqmdr`, MQ 10.0.x: <https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=reference-rdqmdr-manage-dr-rdqm-instances>. *Creating DR/HA RDQMs*, MQ 10.0.x: <https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=availability-creating-drha-rdqms>. Both are cached under `refs/ibm-docs/ibm-mq/10.0.x/` in the lab's cache bucket.

Claims are labelled **Data** (what a source shows) or **Judgment** (reasoning on top of it). Every judgment is a review point for the owner.

**Owner decisions applied (`.github#362`, final):**

1. HA and DR are separate resources. The HA resource is site-local, and the DR resource is stacked on top of it. The filesystem mounts the stacked device.
2. The snapshot LV (DR-3) is not built.
3. The blueprint values are adopted.
4. Replication TLS is lab #1609, which runs after Task 6. Neither template sets `tls yes` until that task lands.
5. The QM agent stays `systemd:mq-<QM>`. `ocf:ibm:rdqm` and `ocf:ibm:rdqmx` are ledgered as deviations.

#### The shape decision: mount device and `--no-dr` (settled)

**Data.**

- **An HA/DR QM:** RDQM mounts the **stacked DR device**. The device stack is `vdb → drbdpool-rdqmapp_00 → drbd100 → drbd7001`, and the filesystem is on `/dev/drbd7001` (`rdqm-a1/lsblk.txt`, `rdqm-a1/mounts.txt`). Pacemaker mounts `device="/dev/drbd/by-res/rdqmapp.dr/0"` (`rdqm-a1/pcmk-config.txt`).
- **RDQM fixes the DR role when it creates the QM.** It formats the stacked device as part of `crtmqm -sx -rr p|s`. Lab evidence: the AMQ3817E failure `mkfs.ext4 /dev/drbd/by-res/rdqmapp.dr/0` (lab `docs/reference/lab-gotchas.md`, "RDQM DR cold-create"). IBM documents the DR role as a `crtmqm` create parameter (*Creating DR/HA RDQMs*, 10.0.x: "-rr p Indicates that the initial DR role is primary").
- **An HA-only QM has no `.dr` device at all.** The lab's `--no-dr` path creates HA-only with `crtmqm -sx -reh`, which has "no DR link (`-reh`, no `.dr` device)" (lab `lab/scripts/rdqm-qm-create.sh`; `ansible/site-rdqm.yml`).
- **Stacked metadata shrinks the device.** "The stacked device must use internal meta data", so its usable size is slightly smaller ([UG] "Device stacking considerations"). Internal metadata is "an area at the *end* of the device", and on a device that already holds a filesystem, "the newly created metadata would overwrite data at the end of the lower-level device" unless you first "Shrink your existing file system" ([UG] "Internal meta data").

**Decision.** The shape is fixed when the QM is created, from `dr_enabled`, as RDQM does with `crtmqm -rr`:

| `dr_enabled` | Resources | `mkfs` and mount device (one variable, `pdrbd_fs_device`) |
|---|---|---|
| `false` (`bootstrap --no-dr`) | HA `<qm>` only | `/dev/drbd/by-res/<qm>/0` (minor 100) |
| `true` (default) | HA `<qm>`, plus DR `<qm>.dr` stacked on top, at **both** sites from the start | `/dev/drbd/by-res/<qm>.dr/0` (minor 7001) |

Task 5 builds the HA-only shape, which is all its `--no-dr` acceptance exercises. Task 6 adds the DR/HA shape. Changing the shape of a formed QM is **refused**: the role fails loud and tells the operator to tear down and re-create. Task 6 Step 7 already re-creates from a teardown.

**Rejected: "Task 5 mounts `drbd100`, Task 6 moves the mount onto the stacked device".** It is not feasible on live data:

- `drbdadm --stacked create-md` writes the stacked resource's internal metadata into the last sectors of `drbd100`. The ext4 filesystem from Task 5 spans the whole of `drbd100`, so its tail would be overwritten ([UG] "Internal meta data").
- Doing it safely means taking the QM down and running an offline `resize2fs` shrink before `create-md`. That moves the filesystem under the QM in exactly the way this plan's fail-loud and no-re-format rules forbid.
- RDQM has no such conversion either: the DR role is a `crtmqm` parameter.

**Also rejected: "always build both resources, with the DR peer absent under `--no-dr`".** It does not match RDQM, whose `--no-dr` QM has no `.dr` resource. It would also leave a permanently unconnected `quorum majority` / `on-no-quorum io-error` member under the live filesystem, which is a needless I/O-error exposure. **(Judgment.)**

**Judgment, flagged for review:** the blueprint captured only the HA/DR shape. The HA-only Pacemaker shape (the filesystem colocated with the HA Promoted role, mounted from the HA device) is derived by removing the DR layer from RDQM's chain. It is **not** observed.

---

### Task 5: Site-A HA formation (`pdrbd-qm`, HA-only shape) + the distributed workload

**Files:**

- Create: `ansible/roles/pdrbd-qm/defaults/main.yml`
- Create: `ansible/roles/pdrbd-qm/templates/qm.res.j2` (the HA resource)
- Create: `ansible/roles/pdrbd-qm/templates/global_common.conf.j2` (RDQM's global/common section)
- Create: `ansible/roles/pdrbd-qm/tasks/main.yml`
- Create: `ansible/roles/pdrbd-qm/handlers/main.yml`
- Create: `ansible/templates/mq/inter-qm.mqsc.j2`, `ansible/templates/mq/authz.mqsc.j2` (moved from `ansible/roles/mq-pcmk-qmgr/templates/`)
- Create: symlinks `ansible/roles/mq-pcmk-qmgr/templates/inter-qm.mqsc.j2 -> ../../../templates/mq/inter-qm.mqsc.j2` and same for `authz.mqsc.j2`
- Create: `ansible/_pdrbd-cluster-ha.yml`, `ansible/site-pdrbd.yml`, `ansible/site-pdrbd-qm-down.yml`
- Test: `tests/test_pdrbd_qm_role.py`

**Interfaces:**

- **Consumes:**
  - Task 3 `drbd9` (VG `drbdpool`) and Task 4's stack, groups and hb IPs.
  - `pcmk-cluster` (vars `pcmk_cluster_name`, `pcmk_site_nodes`), `mq-install`, `mq-authz-accounts`, `mq-diag-logging`, `mq-event-monitor`, `pki-distribute` and `site-distributed-shared.yml`.
  - The provision-phase extra-vars `qm_app`, `qm_svc`, `chl_to_svc`, `chl_to_app` and `svc_req_queue`.
- **Produces:**
  - Per QM, the **HA resource** `{{ qm_name | lower }}`: minor 100, port 7000, site-local, backed by LV `drbdpool/{{ qm_name | lower }}_00` and mounted at `/var/mqm/vols/{{ qm_name | lower }}` from `pdrbd_fs_device`.
  - Pacemaker resources: promotable clone `drbd_qm-clone` (primitive `drbd_qm`) and group `mq_group` = `mq_fs` → `mq_vip` → `mq_qm`. The names are kept because Task 4's merged verbs (`pcs resource enable mq_group`) and `watcherboard._owner_resource` (`mq_qm`) use them.
  - Role vars `pdrbd_site_group` (`pdrbd_a`/`pdrbd_b`), `pdrbd_site_peers` (this site's 3 nodes, `{name, addr_hb, addr_wan, node_id}`) and `pdrbd_dr` (false here). Task 6 consumes all three.

**Blueprint values (settled; these replace the pre-capture hypothesis):**

| Item | Value | Source |
|---|---|---|
| VG / LV | `drbdpool` / `<qm>_00`, `3072M` | `rdqm-a1/vgs.txt`, `rdqm-a1/lvs.txt`, `rdqm-a1/pcmk-config.txt` (`rdqmapp-size=3072`) |
| Filesystem / options / mount | `ext4` / defaults (`rw,relatime`; `seclabel` is SELinux) / `/var/mqm/vols/<qm>` | `rdqm-a1/mounts.txt` |
| HA resource members | this site's 3 nodes only, `connection-mesh`, node-ids 0–2 at **each** site | `rdqm-a1/drbd-dump.txt`, `rdqm-b1/drbd-dump.txt` |
| HA network / port / minor | `net-hb-<site>` / `7000` / `100` | `rdqm-a1/drbd-dump.txt`, `rdqm-a1/rdqm-ini.txt`, `rdqm-a1/pcmk-config.txt` (`rdqmapp-drbd-port`, `rdqmapp-drbd-minor`) |
| HA protocol | C: not set, so the DRBD default applies. [opts] `protocol C`: "This is the default protocol." | `rdqm-a1/drbd-conf.txt` (`rdqmapp.res` has no `protocol`) |
| HA `options` | `quorum 2; on-no-quorum io-error; auto-promote no;` | `rdqm-a1/drbd-conf.txt` |
| HA `net` / `disk` | `verify-alg sha1; rr-conflict retry-connect;` / `disable-write-same yes;` | `rdqm-a1/drbd-conf.txt` |
| Not set | `after-sb-*`, `fencing` (default `dont-care`, [conf]), handlers (IBM `rdqmhandler`, TL-3), `tls` (#1609) | `rdqm-a1/drbd-dump.txt`, RP-7, SC-1 |
| Global / common | `usage-count no; udev-always-use-vnr;`. Options: `twopc-timeout 100`. Net: `ping-timeout 40; socket-check-timeout 5; max-buffers 40k;`. Disk: `c-fill-target 1M; c-max-rate 500M; c-min-rate 100M; resync-without-replication no;` | `rdqm-a1/drbd-conf.txt` (`global_common.conf`), RP-6, TL-5 |
| Cluster properties | `stonith-enabled=false`; `no-quorum-policy` **not set** (default `stop`); `resource-stickiness=1` | `rdqm-a1/pcmk-config.txt`, CL-2, CL-3 |
| HA clone | `ocf:linbit:drbd drbd_resource=<qm> adjust_master_score="0 10 10000 10000"`, promotable, `notify=true`, no `clone-max` (so 3) | `rdqm-a1/pcmk-config.txt`, CL-4 |
| DRBD op timeouts | monitor 20 s Unpromoted / 10 s Promoted (each timeout 20); start 240; stop 100; promote 90; demote 90 | `rdqm-a1/pcmk-config.txt` |
| Filesystem | `ocf:heartbeat:Filesystem` with `force_unmount=safe fast_stop=no`; ops start 60 / stop 60; **no monitor op** | `rdqm-a1/pcmk-config.txt`, CL-8 |
| Floating IP | `ocf:heartbeat:IPaddr2 cidr_netmask=24`, monitor 30 s | `rdqm-a1/pcmk-config.txt`, CL-9 |
| QM resource | **deviation:** `systemd:mq-<QM>` instead of `ocf:ibm:rdqm` (start 43200 s, stop 300 s, monitor 30 s / 60 s). `ocf:ibm:rdqmx` is not reproduced | `rdqm-a1/pcmk-config.txt`, `rdqm-a1/ocf-agents.txt`, CL-6, CL-7 |
| Chain | RDQM uses fs → rdqmx → ip → QM, each `inf` colocated with and ordered after the previous one, plus `location <qm> 50: <preferred node>` | `rdqm-a1/pcmk-config.txt`, CL-10 |

**Judgments in this table:**

- **The group.** `mq_group` (`mq_fs` → `mq_vip` → `mq_qm`) is the same chain, because a group "is logically equivalent to" INFINITY colocations plus start orders between successive members ([PE], "Groups"). The location preference goes on `mq_group`, since [PE] says "it is usually preferable to reference the group itself". RDQM puts it on the QM, which pulls the chain with it.
- **QM start timeout.** `mq_qm` copies RDQM's monitor (30 s / 60 s) and stop (300 s), but uses start `300` instead of `43200`. Twelve hours is the IBM agent's log-replay allowance, and on a `systemd` unit it would hide a hung `strmqm` (the fail-loud rule). The ledger records it under CL-6.
- **Floating IP `nic`.** Not set. RDQM pins `nic=enp6s0`, and IPaddr2 can infer the NIC from the address's subnet. The ledger records it under CL-9 as `partial`.

- [ ] **Step 1: Write the failing structure tests**

```python
"""pdrbd-qm role structure (epic .github#310 T5): destructive steps are guarded,
no masked failures, the values are the RDQM blueprint's, and the SAN role is untouched."""

import hashlib
import pathlib

import yaml

ROLE = pathlib.Path("ansible/roles/pdrbd-qm")
TASKS = ROLE / "tasks/main.yml"
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


# --- blueprint values (lab docs/reference/rdqm-blueprint/, reconciled on #1427) ---


def test_ha_res_is_rdqm_shaped():
    """rdqm-a1/drbd-conf.txt rdqmapp.res: site-local mesh, quorum 2, io-error, no auto-promote."""
    text = (ROLE / "templates/qm.res.j2").read_text()
    for want in (
        "quorum 2;", "on-no-quorum io-error;", "auto-promote no;", "connection-mesh",
        "verify-alg sha1;", "rr-conflict retry-connect;", "disable-write-same yes;",
        "meta-disk internal;", "pdrbd_site_peers",
    ):
        assert want in text, want
    # Not set by RDQM: split-brain auto-recovery, DRBD fencing, the old six-peer WAN wiring.
    for banned in ("after-sb-", "fencing", "suspend-io", "addr_wan", "protocol A"):
        assert banned not in text, banned


def test_global_common_is_rdqms():
    """rdqm-a1/drbd-conf.txt global_common.conf (RDQM replaces the stock file)."""
    text = (ROLE / "templates/global_common.conf.j2").read_text()
    for want in (
        "usage-count no;", "udev-always-use-vnr;", "twopc-timeout 100;", "ping-timeout 40;",
        "socket-check-timeout 5;", "max-buffers 40k;", "c-fill-target 1M;", "c-max-rate 500M;",
        "c-min-rate 100M;", "resync-without-replication no;",
    ):
        assert want in text, want


def test_defaults_carry_blueprint_values():
    d = yaml.safe_load((ROLE / "defaults/main.yml").read_text())
    assert (d["pdrbd_ha_port"], d["pdrbd_ha_minor"]) == (7000, 100)
    assert d["pdrbd_lv"] == "{{ pdrbd_res }}_00"
    assert d["pdrbd_lv_size"] == "3072M"
    assert d["pdrbd_stickiness"] == 1
    assert d["pdrbd_dr"] is False


def test_pacemaker_values_match_blueprint():
    text = TASKS.read_text()
    for want in (
        'adjust_master_score="0 10 10000 10000"', "force_unmount=safe", "fast_stop=no",
        "resource-stickiness={{ pdrbd_stickiness }}", "stonith-enabled=false",
    ):
        assert want in text, want
    assert "no-quorum-policy" not in text  # RDQM leaves the default (stop): CL-2


def test_mkfs_targets_the_mounted_device():
    """One variable names the device mkfs formats AND Pacemaker mounts (stacked metadata
    makes the DR device smaller than the HA device, so they must never diverge)."""
    mkfs = _by_name("initial promote + mkfs")["ansible.builtin.shell"]
    assert "{{ pdrbd_fs_device }}" in mkfs
    assert "device={{ pdrbd_fs_device }}" in TASKS.read_text()
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

- [ ] **Step 4: `ansible/roles/pdrbd-qm/defaults/main.yml` (blueprint values)**

```yaml
---
# pdrbd-qm (epic .github#310): one RDQM-shaped replicated volume per QM. Every value is the
# RDQM blueprint's (lab docs/reference/rdqm-blueprint/; reconciled on #1427, adopted by the
# owner 2026-10-17 under .github#362); each line names its source file.
pdrbd_vg: drbdpool                    # rdqm-a1/vgs.txt
pdrbd_res: "{{ qm_name | lower }}"    # HA resource = lower-case QM (rdqm-a1/drbd-dump.txt)
pdrbd_lv: "{{ pdrbd_res }}_00"        # rdqm-a1/lvs.txt (rdqmapp_00)
pdrbd_lv_size: 3072M                  # rdqm-a1/pcmk-config.txt rdqmapp-size=3072; lvs.txt 3.00g
pdrbd_fstype: ext4                    # rdqm-a1/mounts.txt
pdrbd_mount_opts: defaults            # rdqm-a1/mounts.txt (rw,relatime; seclabel is SELinux)
pdrbd_mount_root: /var/mqm/vols       # rdqm-a1/mounts.txt
pdrbd_mount: "{{ pdrbd_mount_root }}/{{ pdrbd_res }}"
pdrbd_ha_port: 7000                   # rdqm-a1/pcmk-config.txt rdqmapp-drbd-port
pdrbd_ha_minor: 100                   # rdqm-a1/pcmk-config.txt rdqmapp-drbd-minor
pdrbd_stickiness: 1                   # rdqm-a1/pcmk-config.txt rsc_defaults
pdrbd_preferred_score: 50             # rdqm-a1/pcmk-config.txt rdqmapp_preferred_location
# Shape (see "The shape decision"): HA-only here. Task 6 adds the DR/HA shape, chosen from
# dr_enabled when the QM is FIRST created and never changed on a formed QM.
pdrbd_dr: false
# The ONE device mkfs formats and Pacemaker mounts. HA-only: the HA device (by-res needs
# udev-always-use-vnr, rendered in global_common.conf as RDQM does).
pdrbd_fs_device: "/dev/drbd/by-res/{{ pdrbd_res }}/0"
# Set by the play from the inventory + topology hb IPs:
#   pdrbd_site_group: pdrbd_a | pdrbd_b
#   pdrbd_site_peers: [{ name, addr_hb, addr_wan, node_id }]   # this site's 3 nodes only
```

- [ ] **Step 5: The DRBD templates**

`ansible/roles/pdrbd-qm/templates/global_common.conf.j2` — RDQM replaces the stock file (it keeps it as `global_common.conf.rdqmsave`; `rdqm-a1/drbd-conf.txt`, TL-5):

```jinja
# {{ ansible_managed }} — RDQM's global/common section, value for value
# (lab docs/reference/rdqm-blueprint/rdqm-a1/drbd-conf.txt, global_common.conf).
# Units per drbdsetup-options.xml: twopc-timeout and ping-timeout are tenths of a second.
global {
    usage-count no;
    udev-always-use-vnr;
}

common {
    options {
        twopc-timeout 100;
    }
    disk {
        c-fill-target 1M;
        c-max-rate 500M;
        c-min-rate 100M;
        resync-without-replication no;
    }
    net {
        ping-timeout 40;
        socket-check-timeout 5;
        max-buffers 40k;
    }
}
```

`ansible/roles/pdrbd-qm/templates/qm.res.j2` — the **HA resource**, laid out as RDQM's `rdqmapp.res` (`rdqm-a1/drbd-conf.txt`):

```jinja
# {{ ansible_managed }} — RDQM-shaped per-QM HA resource (epic .github#310).
# SITE-LOCAL: only this site's three nodes, a connection-mesh over net-hb-<site>. The other
# site is never a member, so a WAN cut cannot cost this site HA quorum (RP-1/RP-2).
# Values: lab docs/reference/rdqm-blueprint/rdqm-a1/drbd-conf.txt (rdqmapp.res).
# Deliberately NOT set, as in RDQM: protocol (default C), the split-brain policies (RP-7),
# the fence policy (default dont-care), handlers (IBM rdqmhandler; TL-3), tls (lab #1609).
resource {{ pdrbd_res }} {
    options {
        quorum 2;
        on-no-quorum io-error;
        auto-promote no;
    }
    net {
        verify-alg sha1;
        rr-conflict retry-connect;
    }
    disk {
        disable-write-same yes;
    }
{% for p in pdrbd_site_peers %}
    on {{ p.name }} {
        node-id {{ p.node_id }};
        address ipv4 {{ p.addr_hb }}:{{ pdrbd_ha_port }};
        volume 0 {
            device minor {{ pdrbd_ha_minor }};
            disk /dev/{{ pdrbd_vg }}/{{ pdrbd_lv }};
            meta-disk internal;
        }
    }
{% endfor %}
    connection-mesh {
        hosts {{ pdrbd_site_peers | map(attribute='name') | join(' ') }};
    }
}
```

**Known hazard — settled.** The pre-capture design had one six-peer resource with `quorum majority`. A WAN cut would have left each site with 3 of 6 peers, and `on-no-quorum` would have frozen the live site. RDQM avoids this structurally. Its HA resource holds only the three nodes of one site, with a fixed `quorum 2` (`rdqm-a1/drbd-dump.txt`, `rdqm-b1/drbd-dump.txt`). Each site is its own three-node corosync cluster (expected votes 3, quorum 2: `rdqm-a1/quorum.txt`, `rdqm-b1/quorum.txt`). The other site reaches the QM's data only through the separate stacked DR resource (Task 6). A WAN cut therefore cannot touch HA quorum: **Data**, BP README §2.2.

One question remains open, and it is a DR-resource question, not an HA one: whether the live primary keeps **DR** quorum on a WAN cut (`quorum majority` over 2 members). Task 8 Drill 3b settles it on both arms.

- [ ] **Step 6: `ansible/roles/pdrbd-qm/tasks/main.yml`**

```yaml
---
# pdrbd-qm (epic .github#310): the per-QM RDQM-shaped volume + Pacemaker resources, on the
# local site group (pdrbd_site_group). First-time formation is guarded by `pdrbd_formed`
# (Pacemaker already owns drbd_qm at THIS site) so a re-run never re-creates md,
# re-formats, or re-creates the QM over live data. "Once per site" is
# `inventory_hostname == pdrbd_creator`, never run_once (Task 6 runs both sites in one play).

- name: event monitoring + JSON diagnostic logging host prep (arm-neutral roles)
  ansible.builtin.include_role:
    name: "{{ item }}"
  loop: [mq-diag-logging, mq-event-monitor]
  vars:
    qmgr_name: "{{ qm_name }}"

- name: this site's creation node
  ansible.builtin.set_fact:
    pdrbd_creator: "{{ groups[pdrbd_site_group][0] }}"

- name: probe — does this site's Pacemaker already own the QM's HA DRBD resource?
  ansible.builtin.command: pcs resource status drbd_qm
  register: pdrbd_owned
  changed_when: false
  failed_when: pdrbd_owned.rc not in [0, 1]
  become: true

- name: record formation state
  ansible.builtin.set_fact:
    pdrbd_formed: "{{ pdrbd_owned.rc == 0 }}"

- name: ocf:linbit:drbd on this node accepts the blueprint parameters
  # RDQM's agent is drbd-pacemaker 9.33.1 (rdqm-a1/packages.txt); ours comes from the LINBIT
  # PPA's drbd-utils. Prove the two parameters exist before any resource uses them.
  ansible.builtin.shell: |
    set -o pipefail
    crm_resource --show-metadata ocf:linbit:drbd | grep -c -E 'name="(adjust_master_score|wfc_timeout)"'
  args:
    executable: /bin/bash
  register: pdrbd_agent_params
  changed_when: false
  failed_when: pdrbd_agent_params.rc != 0 or (pdrbd_agent_params.stdout | int) != 2
  become: true

- name: per-QM logical volume <qm>_00 in the drbdpool VG (lvs probe)
  ansible.builtin.shell: |
    set -e
    if ! lvs {{ pdrbd_vg }}/{{ pdrbd_lv }} >/dev/null 2>&1; then
      lvcreate -y -L {{ pdrbd_lv_size }} -n {{ pdrbd_lv }} {{ pdrbd_vg }}
      echo LV-CREATED
    fi
  register: pdrbd_lv_out
  changed_when: "'LV-CREATED' in pdrbd_lv_out.stdout"
  become: true

- name: keep the stock global_common.conf once (RDQM keeps .rdqmsave; TL-5)
  ansible.builtin.command: cp -n /etc/drbd.d/global_common.conf /etc/drbd.d/global_common.conf.pdrbdsave
  args:
    creates: /etc/drbd.d/global_common.conf.pdrbdsave
  become: true

- name: DRBD global/common section (RDQM's values)
  ansible.builtin.template:
    src: global_common.conf.j2
    dest: /etc/drbd.d/global_common.conf
    mode: "0644"
  become: true

- name: HA resource definition (site-local, RDQM's rdqmapp.res)
  ansible.builtin.template:
    src: qm.res.j2
    dest: "/etc/drbd.d/{{ pdrbd_res }}.res"
    mode: "0644"
  become: true

- name: the rendered DRBD config parses (an option this drbd-utils rejects fails here, loud)
  ansible.builtin.command: drbdadm dump all
  changed_when: false
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

- name: first-time formation on this site's creation node
  when:
    - not pdrbd_formed
    - inventory_hostname == pdrbd_creator
  become: true
  block:
    - name: wait for the site peers to connect
      # `drbdadm status` omits the connection field while a peer is Connected
      # (rdqm-a1/drbd-status.txt shows only `rdqm-a2 role:Secondary`), so the pre-capture
      # grep could never match; `drbdsetup status --verbose` prints it (confirm live, Step 11).
      ansible.builtin.shell: |
        set -o pipefail
        for i in $(seq 1 60); do
          n=$(drbdsetup status --verbose {{ pdrbd_res }} | grep -c 'connection:Connected' || [ $? -eq 1 ])
          [ "$n" -ge 2 ] && { echo connected; exit 0; }
          sleep 2
        done
        drbdsetup status --verbose {{ pdrbd_res }}; exit 1
      args:
        executable: /bin/bash
      changed_when: false

    # auto-promote is `no` (blueprint), so every promotion is explicit.
    - name: initial promote + mkfs (fresh volume, skip the full initial sync)
      ansible.builtin.shell: |
        set -e
        drbdadm new-current-uuid --clear-bitmap {{ pdrbd_res }}/0
        drbdadm primary {{ pdrbd_res }}
        mkfs.{{ pdrbd_fstype }} -q {{ pdrbd_fs_device }}
      changed_when: true

    # Shell, not ansible.posix.mount: the lab installs no galaxy collections (#156).
    - name: mount the volume on the creation node
      ansible.builtin.shell: |
        set -e
        mkdir -p {{ pdrbd_mount }}
        mountpoint -q {{ pdrbd_mount }} || mount -t {{ pdrbd_fstype }} -o {{ pdrbd_mount_opts }} {{ pdrbd_fs_device }} {{ pdrbd_mount }}
        mkdir -p {{ pdrbd_mount }}/qmgr {{ pdrbd_mount }}/log
        chown -R mqm:mqm {{ pdrbd_mount }}
      changed_when: true

    - name: create the queue manager on the replicated volume (RDQM path layout)
      ansible.builtin.command: >
        su mqm -c '/opt/mqm/bin/crtmqm -md {{ pdrbd_mount }}/qmgr -ld {{ pdrbd_mount }}/log
        -p 1414 {{ qm_name }}'
      changed_when: true
```

The `--clear-bitmap` skip of the initial sync is the `drbd-operations.md` "real fix" (blank disks, all peers up). The QM path layout matches RDQM's `DataPath=/var/mqm/vols/rdqmapp/qmgr/rdqmapp` (`rdqm-a1/mqs-ini.txt`, ST-5).

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
  # Every RDQM node carries the same addmqinf definition, site B included (rdqm-*/dspmqinf.txt).
  ansible.builtin.command:
    cmd: "su mqm -c '{{ hostvars[groups['pdrbd_a'][0]].pdrbd_addmqinf.stdout }}'"
  when:
    - not pdrbd_formed
    - inventory_hostname != groups['pdrbd_a'][0]
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
  # Deviation CL-6: RDQM runs the QM under the IBM agent ocf:ibm:rdqm (owner decision 5).
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

- name: cluster properties (rdqm-a1/pcmk-config.txt; the quorum policy stays at its default, CL-2)
  ansible.builtin.shell: |
    set -e
    pcs property set stonith-enabled=false
    pcs resource defaults update resource-stickiness={{ pdrbd_stickiness }}
  when: inventory_hostname == pdrbd_creator
  changed_when: true
  become: true

- name: Pacemaker resources (first-time only; built in a CIB file and pushed atomically)
  # Values: rdqm-a1/pcmk-config.txt. Chain: HA Promoted -> mq_group (fs -> vip -> qm).
  # HA-only shape (Task 5). Judgment: derived by removing RDQM's DR layer; not captured.
  when:
    - not pdrbd_formed
    - inventory_hostname == pdrbd_creator
  become: true
  ansible.builtin.shell: |
    set -e
    cib=$(mktemp); trap 'rm -f "$cib"' EXIT
    pcs cluster cib "$cib"
    pcs -f "$cib" resource create drbd_qm ocf:linbit:drbd drbd_resource={{ pdrbd_res }} \
      adjust_master_score="0 10 10000 10000" \
      op monitor interval=20s timeout=20s role=Unpromoted \
      op monitor interval=10s timeout=20s role=Promoted \
      op start timeout=240s op stop timeout=100s op promote timeout=90s op demote timeout=90s \
      promotable notify=true
    pcs -f "$cib" resource create mq_fs ocf:heartbeat:Filesystem device={{ pdrbd_fs_device }} \
      directory={{ pdrbd_mount }} fstype={{ pdrbd_fstype }} options={{ pdrbd_mount_opts }} \
      force_unmount=safe fast_stop=no \
      op start timeout=60s op stop timeout=60s --no-default-ops --group mq_group
    pcs -f "$cib" resource create mq_vip ocf:heartbeat:IPaddr2 ip={{ qm_vip }} cidr_netmask=24 \
      op monitor interval=30s --group mq_group --after mq_fs
    pcs -f "$cib" resource create mq_qm systemd:mq-{{ qm_name }} \
      op monitor interval=30s timeout=60s op start timeout=300s op stop timeout=300s \
      --group mq_group --after mq_vip
    pcs -f "$cib" constraint colocation add mq_group with Promoted drbd_qm-clone INFINITY
    pcs -f "$cib" constraint order promote drbd_qm-clone then start mq_group
    pcs -f "$cib" constraint location mq_group prefers {{ pdrbd_creator }}={{ pdrbd_preferred_score }}
    pcs cluster cib-push "$cib" --config
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
  when: inventory_hostname == pdrbd_creator
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

**Notes on `pcs`:**

- **Syntax.** `pcs` 0.11 on Ubuntu 24.04 takes `--group` / `--after` and `--no-default-ops`. Confirm both, and how the clone meta (`notify=true`) lands, with `pcs resource create --help` and `pcs resource config` on the first live run. `--no-default-ops` keeps `pcs` from adding a Filesystem monitor that RDQM does not have.
- **Role names.** RDQM's `crm` output uses the legacy `Master`/`Slave` role names (BP README §0). `pcs` 0.11 spells them `Promoted`/`Unpromoted`; the two pairs are the same roles.

- [ ] **Step 7: `ansible/_pdrbd-cluster-ha.yml` (site A building block)**

```yaml
---
# Site A of the pcmk-drbd-ubuntu stack (epic .github#310): DRBD 9 + VG, MQ, the site's
# 3-node Pacemaker cluster. Like RDQM, every site is its OWN cluster; none spans the sites
# (rdqm-*/corosync-conf.txt, rdqm-*/quorum.txt; CL-1).
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

RDQM uses no fencing at either the Pacemaker or the DRBD level (`stonith-enabled=false`, no fencing devices, no DRBD `fencing`/`fence-peer`: `rdqm-a1/pcmk-config.txt`, `rdqm-a1/drbd-dump.txt`). So no `pcmk-stonith` play is added.

- [ ] **Step 8: `ansible/site-pdrbd.yml`**

```yaml
---
# One provision for pcmk-drbd-ubuntu (epic .github#310), composed like site-pcmk.yml:
# 1) site A HA, 2) authz accounts, 3) the per-QM volume + resources, 4) the distributed
# workload (qm e2e needs it). Task 6 adds site B and the DR/HA shape.
- name: preflight — the DR/HA shape is not built yet
  hosts: localhost
  gather_facts: false
  tasks:
    - name: refuse a DR provision until Task 6 (lab #1428) lands
      ansible.builtin.assert:
        that: not (dr_enabled | default(true) | bool)
        fail_msg: >-
          pcmk-drbd-ubuntu builds only the HA-only shape until Task 6 (#1428) lands; run
          `mqlab bootstrap pcmk-drbd-ubuntu --no-dr`

- import_playbook: _pdrbd-cluster-ha.yml

- name: authz service accounts on every app-QM node
  hosts: pdrbd_a
  become: true
  roles: [mq-authz-accounts]

- name: per-QM replicated volume + resources (site A)
  hosts: pdrbd_a
  become: true
  vars:
    qm_name: "{{ qm_app }}"
    qm_vip: 10.10.1.170
    svc_conn: "svc-sim-ext.service.com"
    pdrbd_site_group: pdrbd_a
    # Site-local HA peers, node-ids 0-2 (RDQM numbers each site 0-2: rdqm-a1/drbd-dump.txt,
    # rdqm-b1/drbd-dump.txt). addr_wan is unused by the HA template; Task 6's DR template uses it.
    pdrbd_site_peers:
      - { name: pdrbd-a1, addr_hb: 172.16.1.71, addr_wan: 10.99.0.71, node_id: 0 }
      - { name: pdrbd-a2, addr_hb: 172.16.1.72, addr_wan: 10.99.0.72, node_id: 1 }
      - { name: pdrbd-a3, addr_hb: 172.16.1.73, addr_wan: 10.99.0.73, node_id: 2 }
  roles: [pdrbd-qm]

- import_playbook: site-distributed-shared.yml
  vars:
    our_qm: "{{ qm_app }}"
    # Like RDQM: no partner-facing VIP; the SVC QM's sender to us carries a CONNAME
    # list of the three site-A net-ext node addresses (site-rdqm.yml precedent).
    our_conn: "pdrbd-a1-ext.client.com(1414),pdrbd-a2-ext.client.com(1414),pdrbd-a3-ext.client.com(1414)"
    app_conn: "pdrbd-vip-a.client.com(1414)"
```

The preflight play is temporary: Task 6 deletes it. Read `ansible/site-rdqm.yml:429-445` and copy any further vars its `site-distributed-shared.yml` import passes (e.g. `app_tls`, `svc_partner_*`), substituting the `pdrbd-*` names. Confirm the node `-ext` DNS names exist (`mqlab dns` / `tests/test_dns.py` pattern: `<host>-ext.client.com` for hosts with `net-ext`).

- [ ] **Step 9: `ansible/site-pdrbd-qm-down.yml` (`qm-destroy`)**

```yaml
---
# qm-destroy for pcmk-drbd-ubuntu: remove the Pacemaker resources, take the DRBD resource
# down, wipe its metadata, remove the <qm>_00 LV and the QM definition on every node.
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
    pdrbd_lv: "{{ qm_name | lower }}_00"
  tasks:
    - name: drbd down + wipe-md + lvremove + resource file
      ansible.builtin.shell: |
        set -e
        if drbdadm status {{ pdrbd_res }} >/dev/null 2>&1; then drbdadm down {{ pdrbd_res }}; fi
        if [ -e /etc/drbd.d/{{ pdrbd_res }}.res ]; then
          timeout 60 drbdadm wipe-md --force {{ pdrbd_res }} </dev/null
          rm -f /etc/drbd.d/{{ pdrbd_res }}.res
        fi
        if lvs drbdpool/{{ pdrbd_lv }} >/dev/null 2>&1; then lvremove -y drbdpool/{{ pdrbd_lv }}; fi
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

Run: `uv run pytest tests/test_pdrbd_qm_role.py -v` — expected: PASS (11).
Run: `vrg-container-run -- vrg-validate` — expected: green.

- [ ] **Step 11: Operator brings up site A HA-only (human step)**

```bash
mqlab bootstrap pcmk-drbd-ubuntu --no-dr
mqlab qm status pcmk-drbd-ubuntu
mqlab qm e2e pcmk-drbd-ubuntu
```

**Expected:**

- `qm status` shows `drbd_qm-clone` Promoted on one node and Unpromoted on two, with `mq_group` Started on the promoted node.
- `drbdadm status` shows the resource UpToDate, with two Connected peers.
- `qm e2e` round-trips every request.
- `findmnt /var/mqm/vols/pdrbdapp` on the promoted node shows source `/dev/drbd100`.
- `lvs` shows `drbdpool/pdrbdapp_00`.

Run `mqlab bootstrap pcmk-drbd-ubuntu --no-dr` a **second** time. Expected: `changed` only on probes and adjust, with no create-md, mkfs or crtmqm (Review Focus 3). Also run `pcs config` once and record the actual `crm_mon -1 -r` "Promoted:" line format for Task 6's parse. Paste the outputs into the issue.

- [ ] **Step 12: Ledger update + commit**

Update `docs/reference/rdqm-vs-vanilla-deviations.md`. Set Status for every ST/RP/CL row that is now built, from `pending` to `exact`/`partial`/`none` with the consequence:

- ST-2: now `exact` (`<qm>_00`).
- RP-2, RP-4, RP-6 and RP-7: as built.
- CL-2, CL-3, CL-4 and CL-8: as built.
- CL-6: `partial`. Name the start-timeout and agent deviation.
- CL-7: `none` (`rdqmx`).
- CL-9: `partial`. No `nic`.
- CL-10: the HA-only chain. The DR half follows in Task 6.
- ST-4 and RP-1 stay `pending` until Task 6.

```bash
vrg-git add ansible/roles/pdrbd-qm ansible/templates/mq ansible/roles/mq-pcmk-qmgr/templates \
  ansible/_pdrbd-cluster-ha.yml ansible/site-pdrbd.yml ansible/site-pdrbd-qm-down.yml \
  tests/test_pdrbd_qm_role.py docs/reference/rdqm-vs-vanilla-deviations.md
vrg-commit --type feat --scope pdrbd --message "site-A HA: RDQM-shaped per-QM DRBD 9 volume under Pacemaker + distributed workload (#1427)"
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

### Task 6: DR — site B as its own HA cluster; the stacked DR resource; operator-driven DR demote/promote

**Files:**

- Create: `ansible/_pdrbd-dr-replication.yml` (site B building block)
- Create: `ansible/roles/pdrbd-qm/templates/qm.dr.res.j2` (the stacked DR resource)
- Create: `ansible/site-pdrbd-dr-switch.yml`
- Create: `ansible/tasks/rpo0-drill-seed.yml`, `ansible/tasks/rpo0-drill-verify.yml`
- Create: `ansible/vars/pdrbd-peers.yml`
- Modify: `ansible/site-pdrbd.yml` (delete Task 5's preflight play; site B; one `pdrbd-qm` play over both sites)
- Modify: `ansible/roles/pdrbd-qm/defaults/main.yml`, `ansible/roles/pdrbd-qm/tasks/main.yml` (the DR/HA shape)
- Modify: `ansible/site-pdrbd-qm-down.yml` (DR resource, both sites)
- Modify: `lab/topology.yaml` (`pcmk-drbd-ubuntu` `qm-status` verb)
- Test: `tests/test_pdrbd_dr_switch.py`, `tests/test_pdrbd_qm_role.py`, `tests/test_topology_pdrbd.py`

**Interfaces:**

- **Consumes:** Task 5's role `pdrbd-qm` and its vars (`pdrbd_site_group`, `pdrbd_site_peers`, `pdrbd_dr`, `pdrbd_fs_device`, the resource `drbd_qm` and the group `mq_group`), and Task 6a's extra-vars `dr_direction`, `qm_name` and `rpo0_drill`.
- **Produces:**
  - Per QM, the **DR resource** `{{ qm_name | lower }}.dr`. It is stacked on the HA resource, uses minor 7001 and port 7001, and has 2 members: the local site's HA primary, and the other site as one logical peer.
  - Pacemaker: a promotable clone `drbd_dr_qm-clone` (primitive `drbd_dr_qm`) at each site. Its `target-role` is the DR role marker: `Promoted` at the DR primary site and `Unpromoted` at the secondary. A CIB property `pdrbd: <qm>-dr-role` (`1`/`2`) mirrors RDQM's `rdqm: <qm>-dr-role`.
  - `site-pdrbd-dr-switch.yml`, honoring `dr_direction` (`a2b`|`b2a`), `rpo0_drill` (bool) and `dr_force` (bool, default false; forced DR only).
  - The shared RPO-0 task files: inputs `rpo0_node` and `qm_name`; the seed sets fact `rpo0_token`.

**Blueprint values (settled):**

| Item | Value | Source |
|---|---|---|
| Site B | its own 3-node cluster (`corosync_votequorum`, expected 3, quorum 2) and its own HA resource `<qm>` (node-ids 0–2, `net-hb-b`, port 7000, minor 100) | `rdqm-b1/quorum.txt`, `rdqm-b1/corosync-conf.txt`, `rdqm-b1/drbd-dump.txt` |
| DR resource | `<qm>.dr`, `stacked-on-top-of <qm>`, with no `disk` line: its backing device is the HA device `/dev/drbd100` | `rdqm-a1/drbd-conf.txt` (`rdqmapp.dr.res`); BP README §2.1 (supplementary probe `drbdsetup show`) |
| DR members | 2. `stacked-on-top-of` gets node-id 0 at site A and 1 at site B. The remote is **one logical node** (node-id 1 at A, 0 at B) with `volume 0 { disk _remote; }` | `rdqm-a1/drbd-dump.txt`, `rdqm-b1/drbd-dump.txt` |
| DR paths | 3 `path`s in one `connection`, from **this node's** WAN address to each remote-site WAN address. The file is rendered **per node**: on a2 the local `host` is `rdqm-a2 10.99.0.32` | `rdqm-a1/drbd-conf.txt`, `rdqm-a2/drbd-dump.txt`, `rdqm-a3/drbd-dump.txt` |
| DR port / minor / net | `7001` / `7001` / `net-wan` | `rdqm-a1/pcmk-config.txt` (`rdqmapp-dr-port`, `rdqmapp-dr-local-ip`, `rdqmapp-dr-remote-ip`), `rdqm-a1/drbd-dump.txt` |
| DR `net` | `protocol A; verify-alg sha1; allow-remote-read no;` (`tls yes` waits for #1609) | `rdqm-a1/drbd-conf.txt` |
| DR `options` | `quorum majority; on-no-quorum io-error; auto-promote no;` | `rdqm-a1/drbd-conf.txt` |
| DR clone | `ocf:linbit:drbd drbd_resource=<qm>.dr wfc_timeout=false`, promotable, `notify=true`, **`clone-max=1`**, with the same op timeouts as HA | `rdqm-a1/pcmk-config.txt`, CL-5 |
| DR role marker | DR clone `target-role=Promoted` at site A and `Unpromoted` at site B; `rdqmapp-dr-role` is `1` at A and `2` at B | `rdqm-a1/pcmk-config.txt`, `rdqm-b1/pcmk-config.txt`, DR-1, DR-2 |
| Chain | HA promote → DR start (DR colocated with HA Master); DR start and DR promote → fs (fs colocated with DR Master); fs → [rdqmx] → ip → QM, all `inf` | `rdqm-a1/pcmk-config.txt` |
| Site B while secondary | HA clone **Promoted** on b1 (b2 and b3 Unpromoted); DR clone running **Unpromoted** on b1; fs, IP and QM **Stopped** | `rdqm-b1/pcmk-status.txt`, `rdqm-b1/drbd-status.txt`, DR-1 |
| Mount | `/dev/drbd/by-res/<qm>.dr/0` (`drbd7001`) | `rdqm-a1/pcmk-config.txt`, `rdqm-a1/lsblk.txt`, ST-4 |
| Status tooling | `drbdadm status all` does **not** list the stacked resource; `drbdsetup status --verbose` does | BP README §2.1 |
| Snapshot LV | not built (owner decision 2); no `before-resync-*` handlers | DR-3 |

**DRBD 9 stacking facts this task relies on (Data, upstream):**

- `stacked-on-top-of` is "Used instead of an on section for configuring a stacked resource" ([conf]).
- "The stacked device is the active one". The lower device "must be in the Primary role" for the stacked device to run. The stacked device's metadata is internal, which makes it slightly smaller ([UG] "Device stacking considerations").
- Stacked operations take `drbdadm --stacked` (`-S`). The documented sequence is: bring up and promote the lower resource, then `create-md --stacked`, `up --stacked` and `primary --stacked` ([UG] "Creating a stacked 3-node setup").
- `ocf:linbit:drbd` detects a stacked resource itself. It falls back to `drbdadm --stacked sh-dev` and then runs every later call as `drbdadm -S` ([ocf] `_drbd_validate_all`).
- The agent's `stop` runs `drbdadm down` ([ocf] `drbd_stop`), so Pacemaker owns the stacked resource's up/down once the clone exists.
- `wfc_timeout`: "Unless set to the empty string or any non-digits, wait (at most) this many seconds for the connection(s)". So `false` means no wait ([ocf]). Its default is `5`. `adjust_master_score` defaults to `"0 10 1000 10000"`; RDQM raises the third value to `10000` ([ocf]; `rdqm-a1/pcmk-config.txt`).
- Stacking "is deprecated in DRBD version 9.x" ([UG], [conf]). **Judgment:** RDQM 10.0.1 ships it on DRBD 9.3.3, and this arm copies RDQM. The deprecation is recorded in the ledger (RP-1 consequence), not designed around.

**DR quorum (open; Task 8 Drill 3b settles it):**

- **Data.** `quorum majority` "requires … a majority partition of DRBD nodes". "If all missing nodes are marked as outdated, a partition always has quorum" ([opts], [UG] "Configuring quorum"). When a peer is lost ungracefully, DRBD "has to be assumed that it forms a partition with all the missing outdated nodes" ([opts]). On RDQM, each DR member reported `quorum( no -> yes )` alone at attach (BP README §2.2, supplementary probe).
- **Judgment.** None of these sources says whether the live DR primary of a **2-member** resource keeps quorum when the WAN drops ungracefully against an UpToDate peer. If it does not, `on-no-quorum io-error` errors the live QM's filesystem on a WAN outage.
  - RDQM may rely on its `disconnected` handler (`/opt/mqm/bin/rdqmhandler`, TL-3), which this arm does not reproduce.
  - The plan copies the blueprint values (owner decision 3) and **tests** the behaviour (Step 7's early check, then Task 8 Drill 3b). It does not invent a mitigation.

- [ ] **Step 1: Write the failing structure tests**

`tests/test_pdrbd_dr_switch.py`:

```python
"""site-pdrbd-dr-switch.yml structure (epic .github#310 T6): RDQM-style DR demote/promote
of the DR clone only; the HA layer at both sites is never touched."""

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


def _task(fragment: str) -> dict:
    return _all_tasks()[_index(fragment)][1]


def test_dr_switch_validates_direction():
    assert _index("assert dr_direction") == 0


def test_dr_switch_asserts_disk_state_before_promote():
    assert _index("assert the target's disk is UpToDate") < _index("promote at the target site")


def test_dr_switch_force_is_explicit():
    gate = _task("assert the target's disk is UpToDate")
    assert "dr_force" in str(gate["ansible.builtin.assert"]["that"])


def test_dr_switch_rpo0_seed_before_demote_and_verify_after_promote():
    seed, demote = _index("RPO-0 seed"), _index("stop and demote at the source site")
    promote, verify = _index("promote at the target site"), _index("RPO-0 verify")
    assert seed < demote < promote < verify


def test_dr_switch_never_masks_a_failure():
    text = PB.read_text()
    assert "|| true" not in text and "failed_when: false" not in text


def test_dr_switch_flips_only_the_dr_clone():
    """RDQM's switch leaves each site's HA clone running (rdqm-b1/pcmk-status.txt: HA
    Promoted on b1 under the DR secondary); only the DR clone's target-role moves."""
    demote = _task("stop and demote at the source site")["ansible.builtin.shell"]
    promote = _task("promote at the target site")["ansible.builtin.shell"]
    assert "drbd_dr_qm-clone target-role=Unpromoted" in demote
    assert "drbd_dr_qm-clone target-role=Promoted" in promote
    assert "drbd_qm-clone" not in PB.read_text().replace("drbd_dr_qm-clone", "")


def test_dr_switch_demotes_source_before_promoting_target():
    """rdqmdr -p "fails if the primary instance of the queue manager is still running and
    the DR replication link is still functioning" (IBM, rdqmdr, 10.0.x)."""
    assert _index("assert the source DR resource is Secondary") < _index("promote at the target site")
```

Append to `tests/test_pdrbd_qm_role.py`:

```python
def test_dr_res_is_rdqm_shaped():
    """rdqm-a1/drbd-conf.txt rdqmapp.dr.res: stacked, 2 members, 3 paths, protocol A."""
    text = (ROLE / "templates/qm.dr.res.j2").read_text()
    for want in (
        "stacked-on-top-of {{ pdrbd_res }}", "quorum majority;", "on-no-quorum io-error;",
        "auto-promote no;", "protocol A;", "verify-alg sha1;", "allow-remote-read no;",
        "disk _remote;", "path {", "pdrbd_remote_peers", "inventory_hostname",
    ):
        assert want in text, want
    for banned in ("after-sb-", "fencing", "meta-disk", "connection-mesh"):
        assert banned not in text, banned


def test_dr_clone_matches_blueprint():
    text = TASKS.read_text()
    for want in (
        "drbd_resource={{ pdrbd_dr_res }} wfc_timeout=false",
        "clone-max=1",
        "constraint colocation add drbd_dr_qm-clone with Promoted drbd_qm-clone INFINITY",
        "constraint order promote drbd_qm-clone then start drbd_dr_qm-clone",
        "constraint colocation add mq_group with Promoted drbd_dr_qm-clone INFINITY",
        "constraint order promote drbd_dr_qm-clone then start mq_group",
    ):
        assert want in text, want


def test_dr_shape_is_fixed_at_creation():
    """The DR role is a creation-time property (RDQM: crtmqm -rr); a formed QM never
    changes shape under live data — the stacked metadata would overwrite the fs tail."""
    guard = _by_name("the DR shape of a formed QM never changes")
    assert "pdrbd_dr_formed" in str(guard["ansible.builtin.assert"]["that"])
    assert "qm-destroy" in guard["ansible.builtin.assert"]["fail_msg"]


def test_dr_secondary_never_runs_first_time_formation():
    """Site B forms its OWN HA resource (RDQM: HA Promoted on b1) but never mkfs/crtmqm."""
    for frag in ("initial promote + mkfs", "create the queue manager"):
        task, inherited = next((t, w) for t, w in _tasks() if frag in t["name"])
        assert "groups['pdrbd_a'][0]" in inherited + str(task.get("when", "")), frag
```

**Updated tests, whose premises changed:**

- `test_pdrbd_qm_destructive_steps_are_guarded` (**Review Focus 3**, name kept). Its fragment tuple gains the DR steps:

  ```python
      for frag in ("create-md", "stacked create-md", "HA initial promote",
                   "initial promote + mkfs", "create the queue manager"):
  ```

  The content changed because the stacked resource adds a second metadata create and a second initial promote. Both are destructive on re-run, so both must sit behind `pdrbd_formed`.
- `test_defaults_carry_blueprint_values`. Replace `assert d["pdrbd_dr"] is False` with:

  ```python
      assert "dr_enabled" in d["pdrbd_dr"]
      assert (d["pdrbd_dr_port"], d["pdrbd_dr_minor"]) == (7001, 7001)
  ```

- `test_dr_secondary_never_runs_first_time_formation` (name kept from the pre-capture plan). The old premise was that site B skips formation entirely. Under the two-resource design, site B forms its own HA resource and only skips the QM creation. The test now checks that.
- `tests/test_topology_pdrbd.py::test_pdrbd_verbs`. Change `assert "drbdadm status" in v["qm-status"]["cmd"]` to `assert "drbdsetup status --verbose" in v["qm-status"]["cmd"]`, because `drbdadm status all` omits the stacked resource (BP README §2.1).

- [ ] **Step 2: Run to verify they fail**

Run: `uv run pytest tests/test_pdrbd_dr_switch.py tests/test_pdrbd_qm_role.py tests/test_topology_pdrbd.py -v` — expected: FAIL (switch playbook and DR template missing; role lacks the DR shape; verb unchanged).

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

- [ ] **Step 4: Site B as its own cluster, the peer list, the DR template, and the DR/HA shape in `pdrbd-qm`**

**(a) `ansible/_pdrbd-dr-replication.yml`.** This is site B's building block: the same blocks as site A, in its **own** cluster. No cluster spans the sites (`rdqm-b1/corosync-conf.txt`, CL-1).

```yaml
---
# Site B of pcmk-drbd-ubuntu (epic .github#310): an independent 3-node site with its own
# Pacemaker cluster and its own HA resource, like RDQM's site B (rdqm-b1/quorum.txt,
# rdqm-b1/drbd-dump.txt). It becomes the DR secondary through the stacked <qm>.dr resource.
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

**(b) `ansible/vars/pdrbd-peers.yml`.** This holds both sites' nodes. **Each site's HA list is filtered from it, and nothing ever widens a resource across sites.** The pre-capture approach of widening `pdrbd_peers` and running `drbdadm adjust` is deleted.

```yaml
---
# pcmk-drbd-ubuntu DRBD nodes (epic .github#310). addr_hb: the site-local HA resource
# (net-hb-<site>, node-ids 0-2 at EACH site, as rdqm-a1/rdqm-b1 drbd-dump.txt). addr_wan:
# the stacked DR resource's paths (net-wan). Must match lab/topology.yaml pdrbd-* nics.
pdrbd_peers_all:
  - { name: pdrbd-a1, addr_hb: 172.16.1.71, addr_wan: 10.99.0.71, node_id: 0, site: a }
  - { name: pdrbd-a2, addr_hb: 172.16.1.72, addr_wan: 10.99.0.72, node_id: 1, site: a }
  - { name: pdrbd-a3, addr_hb: 172.16.1.73, addr_wan: 10.99.0.73, node_id: 2, site: a }
  - { name: pdrbd-b1, addr_hb: 172.16.2.81, addr_wan: 10.99.0.81, node_id: 0, site: b }
  - { name: pdrbd-b2, addr_hb: 172.16.2.82, addr_wan: 10.99.0.82, node_id: 1, site: b }
  - { name: pdrbd-b3, addr_hb: 172.16.2.83, addr_wan: 10.99.0.83, node_id: 2, site: b }
```

Add the topology guard to `tests/test_topology_pdrbd.py`:

```python
def test_pdrbd_peer_vars_match_topology():
    peers = yaml.safe_load(pathlib.Path("ansible/vars/pdrbd-peers.yml").read_text())["pdrbd_peers_all"]
    nodes = _topology()["nodes"]
    for p in peers:
        nics = nodes[p["name"]]["nics"]
        hb = nics.get("net-hb-a") or nics.get("net-hb-b")
        assert (p["addr_hb"], p["addr_wan"]) == (hb, nics["net-wan"]), p["name"]
    for site in ("a", "b"):  # site-local HA node-ids 0-2, as RDQM numbers each site
        assert sorted(p["node_id"] for p in peers if p["site"] == site) == [0, 1, 2]
```

**(c) `ansible/roles/pdrbd-qm/templates/qm.dr.res.j2`.** This is the stacked DR resource, laid out as RDQM's `rdqmapp.dr.res` (`rdqm-a1/drbd-conf.txt`). It is rendered on **every** node of the site, each naming itself. The DR resource runs on whichever node is the site's HA primary (`# on rdqm-a1 rdqm-a2 rdqm-a3` in `rdqm-a1/drbd-dump.txt`).

```jinja
# {{ ansible_managed }} — RDQM-shaped per-QM DR resource, STACKED on the HA resource
# (epic .github#310). 2 members: this site's HA primary (stacked-on-top-of) and the other
# site as ONE logical peer reached over three net-wan paths, protocol A.
# Values: lab docs/reference/rdqm-blueprint/rdqm-a1/drbd-conf.txt (rdqmapp.dr.res).
# Rendered per node: the local `host` is this node (rdqm-a2/drbd-dump.txt names rdqm-a2).
# NOT set, as in RDQM: disk (the HA device is the backing disk), the split-brain policies,
# the fence policy, handlers (IBM rdqmhandler incl. the snapshot LV path; TL-3/DR-3), tls (#1609).
resource {{ pdrbd_dr_res }} {
    options {
        quorum majority;
        on-no-quorum io-error;
        auto-promote no;
    }
    volume 0 {
        device minor {{ pdrbd_dr_minor }};
    }
    stacked-on-top-of {{ pdrbd_res }} {
        node-id {{ pdrbd_dr_local_id }};
    }
    on {{ pdrbd_dr_peer_name }} {
        node-id {{ 1 - (pdrbd_dr_local_id | int) }};
        volume 0 {
            disk _remote;
        }
    }
    connection {
        net {
            protocol A;
            verify-alg sha1;
            allow-remote-read no;
        }
{% for r in pdrbd_remote_peers %}
        path {
            host {{ inventory_hostname }} address ipv4 {{ pdrbd_my_wan }}:{{ pdrbd_dr_port }};
            host {{ pdrbd_dr_peer_name }} address ipv4 {{ r.addr_wan }}:{{ pdrbd_dr_port }};
        }
{% endfor %}
    }
}
```

**Paths and the logical peer:**

- **Data.** "DRBD allows configuring multiple paths per connection". "The TCP transport uses only one path at a time", and it moves to the next path when the connection drops ([UG] "Configuring multiple paths"). That is how one logical peer follows whichever remote node is that site's HA primary. RDQM reports `DR current remote IP address: 10.99.0.41` (`rdqm-a1/rdqmstatus-qm.txt`).
- **Judgment.** RDQM names the logical peer `encrypted.remote`, which is also its TLS certificate SAN (BP README §2.1). This arm names it `dr-remote` until lab #1609 decides the TLS naming. The ledger records it under RP-5 as a naming difference.
- **Judgment.** `disk _remote` is copied verbatim. It is a placeholder for a disk that only the other site has, and the local node never opens it.

**(d) `pdrbd-qm` defaults: add the DR/HA shape.** Replace Task 5's `pdrbd_dr: false` and `pdrbd_fs_device` lines with:

```yaml
# Shape, fixed when the QM is FIRST created (RDQM: crtmqm -rr; see "The shape decision").
pdrbd_dr: "{{ dr_enabled | default(true) | bool }}"
pdrbd_dr_res: "{{ pdrbd_res }}.dr"    # rdqm-a1/drbd-dump.txt (rdqmapp.dr)
pdrbd_dr_port: 7001                   # rdqm-a1/pcmk-config.txt rdqmapp-dr-port
pdrbd_dr_minor: 7001                  # rdqm-a1/drbd-dump.txt device minor 7001
pdrbd_dr_peer_name: dr-remote         # RDQM: encrypted.remote (its TLS SAN); judgment, RP-5
# The ONE device mkfs formats and Pacemaker mounts: the STACKED device when DR is built
# (rdqm-a1/pcmk-config.txt device=/dev/drbd/by-res/rdqmapp.dr/0), else the HA device.
pdrbd_fs_device: "/dev/drbd/by-res/{{ pdrbd_dr_res if pdrbd_dr | bool else pdrbd_res }}/0"
```

The play derives the per-host values from `pdrbd_peers_all` and `pdrbd_site` (see (e)):

- `pdrbd_site_peers`: this site's three nodes.
- `pdrbd_remote_peers`: the other site's three nodes.
- `pdrbd_my_wan`: this node's `addr_wan`.
- `pdrbd_dr_local_id`: `0` at site A, `1` at site B (`rdqm-a1/drbd-dump.txt`, `rdqm-b1/drbd-dump.txt`).

**(e) `ansible/site-pdrbd.yml`.**

- Delete Task 5's preflight play.
- Import `_pdrbd-dr-replication.yml` after `_pdrbd-cluster-ha.yml`, under `when: dr_enabled | default(true) | bool`.
- Run `mq-authz-accounts` on `pdrbd_a{{ ':pdrbd_b' if (dr_enabled | default(true) | bool) else '' }}`.
- Replace Task 5's site-A-only `pdrbd-qm` play with **one** play over both sites. The formation steps then interleave across the two sites in task order (Ansible's linear strategy). The stacked resource needs that: site B's DR member must be up before site A clears the DR bitmap.

```yaml
- name: per-QM replicated volume + resources (site A, and site B as DR secondary)
  hosts: "pdrbd_a{{ ':pdrbd_b' if (dr_enabled | default(true) | bool) else '' }}"
  become: true
  vars_files: [vars/pdrbd-peers.yml]
  vars:
    qm_name: "{{ qm_app }}"
    svc_conn: "svc-sim-ext.service.com"
    pdrbd_site: "{{ 'a' if inventory_hostname in groups['pdrbd_a'] else 'b' }}"
    pdrbd_site_group: "pdrbd_{{ pdrbd_site }}"
    qm_vip: "{{ '10.10.1.170' if pdrbd_site == 'a' else '10.10.2.170' }}"
    pdrbd_site_peers: "{{ pdrbd_peers_all | selectattr('site', 'equalto', pdrbd_site) | list }}"
    pdrbd_remote_peers: "{{ pdrbd_peers_all | rejectattr('site', 'equalto', pdrbd_site) | list }}"
    pdrbd_my_wan: "{{ (pdrbd_peers_all | selectattr('name', 'equalto', inventory_hostname) | first).addr_wan }}"
    pdrbd_dr_local_id: "{{ 0 if pdrbd_site == 'a' else 1 }}"
  roles: [pdrbd-qm]
```

**(f) `pdrbd-qm` tasks: the DR/HA shape.** Change Task 5's `tasks/main.yml` as follows.

**(f1) After `record formation state`**, add the DR probe and the shape guard:

```yaml
- name: probe — does this site's Pacemaker already own the QM's DR DRBD resource?
  ansible.builtin.command: pcs resource status drbd_dr_qm
  register: pdrbd_dr_owned
  changed_when: false
  failed_when: pdrbd_dr_owned.rc not in [0, 1]
  become: true

- name: the DR shape of a formed QM never changes (RDQM fixes it at crtmqm -rr)
  # Moving a live filesystem from the HA device onto a stacked device is not done: the
  # stacked resource's internal metadata goes at the END of the HA device ([UG] "Internal
  # meta data") and would overwrite the filesystem's tail.
  ansible.builtin.assert:
    that: not pdrbd_formed or ((pdrbd_dr_formed | bool) == (pdrbd_dr | bool))
    fail_msg: >-
      {{ qm_name }} was formed {{ 'with' if pdrbd_dr_formed | bool else 'without' }} DR, and
      this run asks for {{ 'DR' if pdrbd_dr | bool else 'no DR' }}. The DR shape is fixed at
      creation; destroy and re-create the QM (qm-destroy, then bootstrap) to change it.
  vars:
    pdrbd_dr_formed: "{{ pdrbd_dr_owned.rc == 0 }}"
```

**(f2) After the HA `.res` template**, render the DR template on every node when `pdrbd_dr`, before `drbdadm dump all`. A `.dr.res` that this drbd-utils rejects then fails at the parse check, loud.

```yaml
- name: DR resource definition (stacked; rendered per node, RDQM's rdqmapp.dr.res)
  ansible.builtin.template:
    src: qm.dr.res.j2
    dest: "/etc/drbd.d/{{ pdrbd_dr_res }}.res"
    mode: "0644"
  when: pdrbd_dr | bool
  become: true
```

**(f3) Replace Task 5's `first-time formation on this site's creation node` block** with three blocks. The task names keep the test fragments. "Once per site" is `inventory_hostname == pdrbd_creator`. "Site A only" is `inventory_hostname == groups['pdrbd_a'][0]`.

```yaml
- name: HA formation on each site's creation node
  when:
    - not pdrbd_formed
    - inventory_hostname == pdrbd_creator
  become: true
  block:
    - name: wait for the site peers to connect
      # `drbdadm status` omits the connection field while Connected (rdqm-a1/drbd-status.txt
      # shows only `rdqm-a2 role:Secondary`), so count it from `drbdsetup status --verbose`.
      ansible.builtin.shell: |
        set -o pipefail
        for i in $(seq 1 60); do
          n=$(drbdsetup status --verbose {{ pdrbd_res }} | grep -c 'connection:Connected' || [ $? -eq 1 ])
          [ "$n" -ge 2 ] && { echo connected; exit 0; }
          sleep 2
        done
        drbdsetup status --verbose {{ pdrbd_res }}; exit 1
      args:
        executable: /bin/bash
      changed_when: false

    # Blank disks, all site peers up: declare UpToDate without a full sync
    # (docs/reference/drbd-operations.md "the real fix"). auto-promote is no, so promote.
    - name: HA initial promote (fresh volume, skip the full initial sync)
      ansible.builtin.shell: |
        set -e
        drbdadm new-current-uuid --clear-bitmap {{ pdrbd_res }}/0
        drbdadm primary {{ pdrbd_res }}
      changed_when: true

- name: DR formation on each site's creation node (stacked on the now-Primary HA device)
  when:
    - pdrbd_dr | bool
    - not pdrbd_formed
    - inventory_hostname == pdrbd_creator
  become: true
  block:
    # drbd-operations.md lessons, applied to the stacked resource: probe live state first
    # (`drbdsetup status` never blocks; `drbdadm status all` does not list stacked
    # resources, BP README §2.1), bound every metadata op, close stdin. The stacked
    # metadata is written INTO the HA device and so replicates to the site peers.
    - name: stacked create-md (DR metadata inside the HA device)
      ansible.builtin.shell: |
        set -e
        if drbdsetup status {{ pdrbd_dr_res }} >/dev/null 2>&1; then
          echo resource-up
        elif timeout 30 drbdadm -S dump-md {{ pdrbd_dr_res }} </dev/null >/dev/null 2>&1; then
          echo md-exists
        else
          timeout 60 drbdadm -S create-md --force {{ pdrbd_dr_res }} </dev/null
          echo created
        fi
      register: pdrbd_dr_md
      changed_when: "'created' in pdrbd_dr_md.stdout"

    - name: stacked up
      ansible.builtin.shell: |
        set -e
        if drbdsetup status {{ pdrbd_dr_res }} >/dev/null 2>&1; then echo already-up
        else drbdadm -S up {{ pdrbd_dr_res }}; echo up; fi
      register: pdrbd_dr_up
      changed_when: "'up' == pdrbd_dr_up.stdout"

    - name: assert the stacked DR disk attached
      ansible.builtin.command: drbdsetup status {{ pdrbd_dr_res }}
      register: pdrbd_dr_state
      changed_when: false
      failed_when: >-
        pdrbd_dr_state.rc != 0
        or 'disk:Diskless' in pdrbd_dr_state.stdout
        or pdrbd_dr_state.stdout | length == 0

- name: QM creation (site A creation node only — site B receives it over the DR resource)
  when:
    - not pdrbd_formed
    - inventory_hostname == groups['pdrbd_a'][0]
  become: true
  block:
    - name: wait for the DR peer to connect (both sites' DR members are up)
      ansible.builtin.shell: |
        set -o pipefail
        for i in $(seq 1 90); do
          drbdsetup status --verbose {{ pdrbd_dr_res }} | grep -q 'connection:Connected' && { echo connected; exit 0; }
          sleep 2
        done
        drbdsetup status --verbose {{ pdrbd_dr_res }}; exit 1
      args:
        executable: /bin/bash
      changed_when: false
      when: pdrbd_dr | bool

    # Both DR members are blank and connected, so clear the DR bitmap too: the DR initial
    # sync is skipped, and mkfs + QM writes replicate as ordinary protocol-A traffic.
    # mkfs targets pdrbd_fs_device: the STACKED device when DR is built, because the
    # stacked metadata makes it smaller than the HA device ([UG] stacking considerations).
    - name: initial promote + mkfs (on the device that is finally mounted)
      ansible.builtin.shell: |
        set -e
        {% if pdrbd_dr | bool %}
        drbdadm -S new-current-uuid --clear-bitmap {{ pdrbd_dr_res }}/0
        drbdadm -S primary {{ pdrbd_dr_res }}
        {% endif %}
        mkfs.{{ pdrbd_fstype }} -q {{ pdrbd_fs_device }}
      changed_when: true
```

**Then:**

- Keep Task 5's remaining creation-node tasks (mount, `crtmqm`, keystore, MQSC, event monitor, `endmqm`, capture `addmqinf`) **inside this site-A block**, unchanged. They use `pdrbd_fs_device`.
- Replace Task 5's `release the volume and demote` with a release on **each** site's creation node, run after the site-A block:

```yaml
- name: release the volumes (Pacemaker owns promotion and the stacked up/down from here)
  ansible.builtin.shell: |
    set -e
    if mountpoint -q {{ pdrbd_mount }}; then umount {{ pdrbd_mount }}; fi
    {% if pdrbd_dr | bool %}
    drbdadm -S secondary {{ pdrbd_dr_res }}
    drbdadm -S down {{ pdrbd_dr_res }}
    {% endif %}
    drbdadm secondary {{ pdrbd_res }}
  when:
    - not pdrbd_formed
    - inventory_hostname == pdrbd_creator
  changed_when: true
  become: true
```

**Notes on formation:**

- **The release order is forced.** The stacked resource is taken down before the HA resource is demoted, because the lower device "must be in the Primary role" while the stacked one runs ([UG]). The HA resource stays up as Secondary, which matches Task 5.
- **Judgment.** Taking the stacked resource down keeps its metadata, including the cleared-bitmap UUIDs, so the DR clone's first start reconnects without a full sync. Step 5b's watched wait proves this live and does not assume it.
- **`drbdadm -S`.** It follows [UG]. [ocf] notes that drbd-utils ≥ 9.5.0 also resolves an explicitly named stacked resource without `--stacked`, so `-S` is belt and braces. If this drbd-utils rejects `-S` on a subcommand, drop it for that subcommand and record which one.

**(f4) The `addmqinf` task** (Task 5) already takes the definition from `groups['pdrbd_a'][0]` and runs on every other node, so all six nodes carry it, as RDQM's do (`rdqm-b1/dspmqinf.txt`).

**(f5) Replace Task 5's `Pacemaker resources` task.** It runs on each site's creation node, first time only, and builds a CIB file that is pushed atomically. It takes the DR layer when `pdrbd_dr`:

```yaml
- name: Pacemaker resources (first-time only; built in a CIB file and pushed atomically)
  # Values: rdqm-a1/pcmk-config.txt (site A), rdqm-b1/pcmk-config.txt (site B).
  when:
    - not pdrbd_formed
    - inventory_hostname == pdrbd_creator
  become: true
  ansible.builtin.shell: |
    set -e
    cib=$(mktemp); trap 'rm -f "$cib"' EXIT
    pcs cluster cib "$cib"
    pcs -f "$cib" resource create drbd_qm ocf:linbit:drbd drbd_resource={{ pdrbd_res }} \
      adjust_master_score="0 10 10000 10000" \
      op monitor interval=20s timeout=20s role=Unpromoted \
      op monitor interval=10s timeout=20s role=Promoted \
      op start timeout=240s op stop timeout=100s op promote timeout=90s op demote timeout=90s \
      promotable notify=true
    {% if pdrbd_dr | bool %}
    pcs -f "$cib" resource create drbd_dr_qm ocf:linbit:drbd drbd_resource={{ pdrbd_dr_res }} wfc_timeout=false \
      op monitor interval=20s timeout=20s role=Unpromoted \
      op monitor interval=10s timeout=20s role=Promoted \
      op start timeout=240s op stop timeout=100s op promote timeout=90s op demote timeout=90s \
      promotable notify=true clone-max=1
    # The DR role marker (rdqm-a1 Promoted / rdqm-b1 Unpromoted on ms_drbd_dr_rdqmapp).
    pcs -f "$cib" resource meta drbd_dr_qm-clone target-role={{ 'Promoted' if pdrbd_site == 'a' else 'Unpromoted' }}
    {% endif %}
    pcs -f "$cib" resource create mq_fs ocf:heartbeat:Filesystem device={{ pdrbd_fs_device }} \
      directory={{ pdrbd_mount }} fstype={{ pdrbd_fstype }} options={{ pdrbd_mount_opts }} \
      force_unmount=safe fast_stop=no \
      op start timeout=60s op stop timeout=60s --no-default-ops --group mq_group
    pcs -f "$cib" resource create mq_vip ocf:heartbeat:IPaddr2 ip={{ qm_vip }} cidr_netmask=24 \
      op monitor interval=30s --group mq_group --after mq_fs
    pcs -f "$cib" resource create mq_qm systemd:mq-{{ qm_name }} \
      op monitor interval=30s timeout=60s op start timeout=300s op stop timeout=300s \
      --group mq_group --after mq_vip
    {% if pdrbd_dr | bool %}
    pcs -f "$cib" constraint colocation add drbd_dr_qm-clone with Promoted drbd_qm-clone INFINITY
    pcs -f "$cib" constraint order promote drbd_qm-clone then start drbd_dr_qm-clone
    pcs -f "$cib" constraint colocation add mq_group with Promoted drbd_dr_qm-clone INFINITY
    pcs -f "$cib" constraint order promote drbd_dr_qm-clone then start mq_group
    pcs -f "$cib" constraint order start drbd_dr_qm-clone then start mq_group
    {% else %}
    pcs -f "$cib" constraint colocation add mq_group with Promoted drbd_qm-clone INFINITY
    pcs -f "$cib" constraint order promote drbd_qm-clone then start mq_group
    {% endif %}
    pcs -f "$cib" constraint location mq_group prefers {{ pdrbd_creator }}={{ pdrbd_preferred_score }}
    pcs cluster cib-push "$cib" --config
  changed_when: true

- name: DR role marker in the CIB (mirrors RDQM's rdqm:<qm>-dr-role; 1 primary, 2 secondary)
  ansible.builtin.command: >-
    crm_attribute --type crm_config --set-name pdrbd --name {{ pdrbd_res }}-dr-role
    --update {{ 1 if pdrbd_site == 'a' else 2 }}
  when:
    - pdrbd_dr | bool
    - not pdrbd_formed
    - inventory_hostname == pdrbd_creator
  changed_when: true
  become: true
```

The two DR orders both copy RDQM: `ms_drbd_dr_rdqmapp:promote` and `ms_drbd_dr_rdqmapp:start`, each → `p_fs_rdqmapp:start` (`rdqm-a1/pcmk-config.txt`). Confirm `crm_attribute --set-name` with `crm_attribute --help` on the box. **Judgment:** `1`/`2` copies RDQM's values; their meaning (primary/secondary) is inferred from `rdqmstatus` (`rdqm-a1/rdqmstatus-qm.txt` "DR role: Primary", `rdqm-b1/rdqmstatus-qm.txt` "DR role: Secondary").

**(f6) Replace Task 5's `wait for the QM to run under Pacemaker`** with a per-site end-state check:

- **Site A** (`groups['pdrbd_a'][0]`): the existing wait for `mq_qm` Started.
- **Site B** (`groups['pdrbd_b'][0]`, when `pdrbd_dr`): assert the blueprint's DR-secondary steady state (`rdqm-b1/pcmk-status.txt`). That means `drbd_qm-clone` Promoted on one node, `drbd_dr_qm-clone` running Unpromoted on that same node, and `mq_fs`/`mq_vip`/`mq_qm` Stopped. Poll `crm_mon -1 -r` up to 300 s and fail with its output on timeout.

- [ ] **Step 5: `ansible/site-pdrbd-dr-switch.yml` (RDQM-style DR demote/promote)**

**What `rdqmdr` changes (Judgment, inferred; BP README §4).** The capture shows only the steady state.

- **Data.** The two sites differ in exactly two places: the DR clone's `target-role` (`Promoted` at A, `Unpromoted` at B) and the `rdqm:` property `rdqmapp-dr-role` (`1`/`2`). Each site's HA clone stays Promoted regardless (`rdqm-a1/pcmk-config.txt`, `rdqm-b1/pcmk-config.txt`, `rdqm-b1/pcmk-status.txt`).
- **Data (IBM).** `rdqmdr -s` makes a primary into a secondary. `-p` makes a secondary into a primary, and "fails if the primary instance of the queue manager is still running and the DR replication link is still functioning" (IBM `rdqmdr`, 10.0.x).
- **Inference.** `rdqmdr -s` sets the DR clone to `Unpromoted`. The constraint chain then stops QM → ip → fs, and the DR resource demotes. `rdqmdr -p` sets it to `Promoted`, and the chain starts fs → ip → QM. Each also flips `<qm>-dr-role`. The playbook does exactly that, and nothing else: **the HA clone at both sites is never disabled.**
- **Task 8.** Drill 5 compares the result against a live `rdqmdr` switch on RDQM. Record `crm configure show` on both RDQM sites before and after.

```yaml
---
# Operator-driven cross-site switch for pcmk-drbd-ubuntu (epic .github#310): RDQM-style DR
# demote (`rdqmdr -s`) at the source, then DR promote (`rdqmdr -p`) at the target. Only the
# DR clone's target-role and the dr-role marker move; each site's HA layer stays up (as
# rdqm-b1/pcmk-status.txt shows under a DR secondary). Run via
# `mqlab dr cutover|failback pcmk-drbd-ubuntu [--rpo0-drill]` (dr_direction, qm_name,
# rpo0_drill). dr_force=true (manual only) promotes the target after the source site is
# LOST — the forced-DR drill; its RPO>0 is the expected outcome.
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
    pdrbd_dr_res: "{{ qm_name | lower }}.dr"
    src: "{{ hostvars['localhost'].dr_src }}"
    dst: "{{ hostvars['localhost'].dr_dst }}"
    dr_force_b: "{{ dr_force | default(false) | bool }}"
    # The node running a given role (ROLE, substituted per task) of the DR clone, from
    # crm_mon. A top-level resource line resets the match, so another clone's role line
    # is never read by mistake.
    dr_clone_node_awk: >-
      awk -v want=ROLE '/^  \* /{f=index($0,"drbd_dr_qm-clone")>0}
      f && $0 ~ "^    \\* "want{gsub(/[\[\]]/,""); print $NF; exit}'
  tasks:
    - name: find the source site's DR-promoted node (skipped on a forced cutover)
      ansible.builtin.shell: |
        set -o pipefail
        crm_mon -1 -r | {{ dr_clone_node_awk | replace('ROLE', 'Promoted') }}
      args:
        executable: /bin/bash
      delegate_to: "{{ groups[src][0] }}"
      register: src_promoted
      changed_when: false
      failed_when: src_promoted.rc != 0 or src_promoted.stdout | length == 0
      become: true
      when: not dr_force_b

    - name: RPO-0 seed on the live QM
      ansible.builtin.include_tasks: tasks/rpo0-drill-seed.yml
      vars:
        rpo0_node: "{{ src_promoted.stdout }}"
      when: rpo0_drill | default(false) | bool

    - name: stop and demote at the source site (DR clone to Unpromoted; HA stays up)
      ansible.builtin.shell: |
        set -e
        pcs resource meta drbd_dr_qm-clone target-role=Unpromoted --wait=300
        crm_attribute --type crm_config --set-name pdrbd --name {{ pdrbd_res }}-dr-role --update 2
      delegate_to: "{{ groups[src][0] }}"
      changed_when: true
      become: true
      when: not dr_force_b

    - name: assert the source DR resource is Secondary (rdqmdr -p refuses a running primary)
      ansible.builtin.command: drbdsetup status {{ pdrbd_dr_res }}
      delegate_to: "{{ src_promoted.stdout }}"
      register: src_dr
      changed_when: false
      failed_when: src_dr.rc != 0 or 'role:Secondary' not in (src_dr.stdout.splitlines() | first | default(''))
      become: true
      when: not dr_force_b

    - name: find the target site's DR member (its HA primary, DR clone Unpromoted)
      ansible.builtin.shell: |
        set -o pipefail
        crm_mon -1 -r | {{ dr_clone_node_awk | replace('ROLE', 'Unpromoted') }}
      args:
        executable: /bin/bash
      delegate_to: "{{ groups[dst][0] }}"
      register: dst_member
      changed_when: false
      failed_when: dst_member.rc != 0 or dst_member.stdout | length == 0
      become: true

    - name: target disk state (stacked resource; `drbdadm status all` does not list it)
      ansible.builtin.shell: |
        set -o pipefail
        drbdsetup status {{ pdrbd_dr_res }} | sed -n 's/^ *disk:\([A-Za-z]*\).*/\1/p' | head -n 1
      args:
        executable: /bin/bash
      delegate_to: "{{ dst_member.stdout }}"
      register: dst_dstate
      changed_when: false
      failed_when: dst_dstate.rc != 0 or dst_dstate.stdout | length == 0
      become: true

    - name: assert the target's disk is UpToDate (or Outdated under explicit dr_force)
      ansible.builtin.assert:
        that: >-
          dst_dstate.stdout == 'UpToDate'
          or (dr_force_b and dst_dstate.stdout == 'Outdated')
        fail_msg: >-
          target DR disk is {{ dst_dstate.stdout }} — refusing to promote. Wait for resync,
          or pass -e dr_force=true for a forced DR that accepts the RPO>0 loss.

    - name: promote at the target site (DR clone to Promoted; the chain starts fs, vip, QM)
      ansible.builtin.shell: |
        set -e
        pcs resource meta drbd_dr_qm-clone target-role=Promoted --wait=600
        crm_attribute --type crm_config --set-name pdrbd --name {{ pdrbd_res }}-dr-role --update 1
      delegate_to: "{{ groups[dst][0] }}"
      changed_when: true
      become: true

    - name: assert the target DR resource is Primary and the QM Started
      # A refused promote fails here with the live state, e.g. `quorum:no` on a forced
      # cutover — a recorded finding (Task 8 drill 6), never a silent no-op.
      ansible.builtin.shell: |
        set -o pipefail
        st=$(drbdsetup status --verbose {{ pdrbd_dr_res }})
        grep -q 'role:Primary' <<<"$(head -n 1 <<<"$st")" || { echo "$st" >&2; exit 1; }
        crm_mon -1 -r | grep -Eq 'mq_qm.*Started' || { crm_mon -1 -r >&2; exit 1; }
      args:
        executable: /bin/bash
      delegate_to: "{{ dst_member.stdout }}"
      changed_when: false
      become: true

    - name: RPO-0 verify at the new live QM
      ansible.builtin.include_tasks: tasks/rpo0-drill-verify.yml
      vars:
        rpo0_node: "{{ dst_member.stdout }}"
      when: rpo0_drill | default(false) | bool
```

**Notes on the switch:**

- **Forced DR (`dr_force`).** The source site is lost, so every source step is skipped. Site B's DR clone is still running Unpromoted on its HA primary (DR-1), and promotion goes through Pacemaker exactly as in a planned switch. The only difference is the relaxed disk-state gate. Whether DRBD lets a 2-member DR resource promote after an ungraceful loss of its peer is the open DR-quorum question above: Drill 6 records it, and the playbook does not work around it.
- **After a forced DR.** The resources leave `after-sb-*` unset (RP-7). When site A returns, its stacked DR resource reconnects to a peer that diverged. What DRBD does then is observed in Drill 6, not assumed. Reconnecting the old primary is an operator step (`drbdadm -S connect --discard-my-data <qm>.dr` on the stale side), and Drill 6 records it with its output.
- **The `crm_mon` parse.** It assumes the Pacemaker 3.0.1 list format captured in `rdqm-b1/pcmk-status.txt`: top-level resources are indented two spaces and start with an asterisk, and clone roles are indented four spaces (`Promoted:` / `Unpromoted (target-role):`). Task 5 Step 11 recorded Ubuntu's `crm_mon` format. Adjust the awk to it before relying on it. The tests assert the ordering, and the live run asserts the parse.

- [ ] **Step 5b: Watched wait for DR sync (spec §7)**

Formation clears both bitmaps, so a fresh build needs no full DR sync. The watched wait still runs:

- It proves that, rather than assuming it.
- It catches a DR resource that came up Inconsistent.
- It covers any provision where a full sync does start, for example after a forced DR.

A stalled sync must fail, not hang. Add this test to `tests/test_pdrbd_dr_switch.py`:

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
    assert "drbdsetup status" in body  # drbdadm status all omits the stacked resource
    assert "dr_enabled" in str(plays[wait]["tasks"][0].get("when", ""))
```

Then append to `ansible/site-pdrbd.yml`, after the `pdrbd-qm` play (Step 4(e)). That play's name contains "site B" and "volume", so the test finds it.

```yaml
# Watch the DR resource reach UpToDate/UpToDate: log progress, fail if the sync percentage
# does not advance within pdrbd_sync_stall_secs, and fail outright past pdrbd_sync_max_secs
# — a stall is a failure, not a wait (spec §7). Runs on every site-A node; only the node
# holding the stacked resource (the site-A HA primary) has it and does the work.
- name: wait for DR sync (the stacked DR resource UpToDate on both members), watched
  hosts: pdrbd_a
  become: true
  vars:
    pdrbd_dr_res: "{{ qm_app | lower }}.dr"
    pdrbd_sync_stall_secs: 180
    pdrbd_sync_max_secs: 3600
  tasks:
    - name: poll drbdsetup status until the DR peer is UpToDate
      ansible.builtin.shell: |
        set -o pipefail
        if ! drbdsetup status {{ pdrbd_dr_res }} >/dev/null 2>&1; then echo not-here; exit 0; fi
        start=$(date +%s); last_pct=""; last_move=$start
        while :; do
          st=$(drbdsetup status --verbose {{ pdrbd_dr_res }})
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
      register: pdrbd_dr_sync
      changed_when: false
      when: dr_enabled | default(true) | bool  # plays take no `when:`; gate the task

    - name: exactly one site-A node held the DR resource and reported in-sync
      ansible.builtin.assert:
        that: >-
          groups['pdrbd_a'] | map('extract', hostvars, ['pdrbd_dr_sync', 'stdout'])
          | select('match', '^in-sync') | list | length == 1
        fail_msg: "no site-A node (or more than one) holds {{ pdrbd_dr_res }} — check crm_mon"
      run_once: true
      when: dr_enabled | default(true) | bool
```

**Resync tuning.** The pre-capture plan proposed the `drbd-operations.md` buffers (`max-buffers 80k`, `sndbuf-size 2M`, `rcvbuf-size 2M`, `c-plan-ahead 0`, `resync-rate 500M`) "unless the blueprint shows RDQM's own values". It does: `max-buffers 40k`, `c-fill-target 1M`, `c-max-rate 500M` and `c-min-rate 100M`, in the common section (`rdqm-a1/drbd-conf.txt`, RP-6). Task 5's `global_common.conf.j2` applies them to both resources. Their `c-min-rate 100M` also removes `drbd-operations.md`'s root cause #2 (a `c-min-rate` of 0 lets application I/O starve the resync).

Before relying on the `grep` patterns, run `drbdsetup status --verbose <qm>.dr` during a live sync once (for example after Drill 6's reconnect). Confirm the state words (`replication:SyncSource`, `done:`) for this DRBD 9 version.

- [ ] **Step 5c: `site-pdrbd-qm-down.yml` and the `qm-status` verb**

**`site-pdrbd-qm-down.yml`.** It now covers both sites (`hosts: "pdrbd_a{{ ':pdrbd_b' if (dr_enabled | default(true) | bool) else '' }}"`), and each step runs once **per site** (`inventory_hostname == groups['pdrbd_' + site][0]`), never `run_once`.

- **Resource delete order:** `mq_qm mq_vip mq_fs drbd_dr_qm drbd_qm`.
- **Per node:** if `drbdsetup status <qm>.dr` succeeds, run `drbdadm -S down <qm>.dr`. Remove `/etc/drbd.d/<qm>.dr.res`. Then run the HA teardown (down, wipe-md, rm, lvremove). The stacked metadata lives inside the HA device, so it goes with the LV.
- **Per site:** `crm_attribute --type crm_config --set-name pdrbd --name <qm>-dr-role --delete` when present.

**`lab/topology.yaml` `pcmk-drbd-ubuntu` verbs.** Set `qm-status: { cmd: "pcs status resources && drbdsetup status --verbose" }`. `drbdadm status all` omits the stacked resource (BP README §2.1). This matches the updated `test_pdrbd_verbs`.

- [ ] **Step 6: Run tests; validate**

Run: `uv run pytest tests/test_pdrbd_dr_switch.py tests/test_pdrbd_qm_role.py tests/test_topology_pdrbd.py -v` — expected: PASS.
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

The teardown matters: the shape is fixed when the QM is created, so the DR/HA shape is built fresh rather than converted from Task 5's HA-only volume.

**Expected, matching the blueprint state by state:**

- **The DR-sync wait** ends `in-sync after <n>s`.
- **Site A** (`rdqm-a1/drbd-status.txt`, `rdqm-a1/pcmk-status.txt`):
  - `drbd_qm-clone` is Promoted on a1, and `drbd_dr_qm-clone` is Promoted on a1.
  - `drbdsetup status --verbose pdrbdapp.dr` shows Primary, `disk:UpToDate` and `quorum:yes`, Connected to `dr-remote`.
  - `findmnt /var/mqm/vols/pdrbdapp` shows `/dev/drbd7001`.
  - `lsblk` shows `drbdpool-pdrbdapp_00 → drbd100 → drbd7001`.
- **Site B** (`rdqm-b1/pcmk-status.txt`, `rdqm-b1/drbd-status.txt`):
  - `drbd_qm-clone` is Promoted on b1, Unpromoted on b2 and b3.
  - `drbd_dr_qm-clone` is Unpromoted on b1.
  - `mq_group` is Stopped.
  - `pdrbdapp.dr` is Secondary and UpToDate.
- **The switches.** Both succeed with the RPO-0 token found. After each switch the two sites have exchanged exactly the DR clone role and the `dr-role` marker, and each site's HA clone stayed Promoted throughout.
- **`qm e2e`** round-trips after failback.

Paste the outputs into the issue, including `pcs config` from both sites.

**Early DR-quorum check (Judgment: early warning, not acceptance).** With the stack in its post-failback steady state, run Task 8 Drill 3b once on this arm. If the live primary's QM takes I/O errors when the WAN is cut, stop and report to the owner before Task 7. That would be a finding about copying `quorum majority` / `on-no-quorum io-error` onto a 2-member resource without IBM's `disconnected` handler.

- [ ] **Step 8: Ledger + parity + commit**

**Ledger.** Update the DR, replication and cluster rows of `docs/reference/rdqm-vs-vanilla-deviations.md`:

- **RP-1:** `exact` for the two-resource shape (record the stacking deprecation in [UG] as the consequence).
- **ST-4:** `exact` (stacked device mounted).
- **RP-3:** settled by Drill 3b in Task 8.
- **RP-5:** `partial` (logical peer named `dr-remote`, not `encrypted.remote`).
- **CL-5:** `exact`.
- **CL-10:** completed with the DR half.
- **DR-1:** `exact`.
- **DR-2:** `partial`. The DR clone `target-role` and marker flips are inferred; the mechanism is a playbook, not `rdqmdr`.
- **TL-2:** `rdqmdr` → `site-pdrbd-dr-switch.yml`.

**Parity.** Set `pcmk-drbd-ubuntu` `MATRIX` entries in `src/mqlab/parity.py` for `dr-bootstrap`, `cutover` and `failback` to `SUPPORTED`, and update `tests/test_parity.py` to match. The remaining verbs stay `NOT_YET` until Task 9.

```bash
vrg-git add ansible/_pdrbd-dr-replication.yml ansible/site-pdrbd-dr-switch.yml ansible/tasks/rpo0-drill-*.yml \
  ansible/vars/pdrbd-peers.yml ansible/site-pdrbd.yml ansible/site-pdrbd-qm-down.yml ansible/roles/pdrbd-qm \
  lab/topology.yaml tests/test_topology_pdrbd.py tests/test_pdrbd_dr_switch.py tests/test_pdrbd_qm_role.py \
  src/mqlab/parity.py tests/test_parity.py docs/reference/rdqm-vs-vanilla-deviations.md
vrg-commit --type feat --scope pdrbd --message "DR: site B cluster, stacked DR resource, RDQM-style DR demote/promote with RPO-0 drill (#1428)"
```

---

### Task 6b: DRBD kernel TLS on both resources (lab #1609)

Filed by the owner on 2026-10-17 as lab `logical-minds-foundry/mq-resiliency-lab-for-linux#1609`, blocked by Task 6 (#1428). Task 7's cold rebuild is blocked by it, so the one-pass acceptance covers the arm with TLS on. The fix list is in the issue body; this plan does not restate it. Steps follow this plan's shape:

1. Failing structure tests.
2. Bake, role and template changes:
   - `ktls-utils` / `tlshd` in the `pdrbd` box.
   - Per-node certificates from the lab CA.
   - `net { tls yes; }` in **both** `qm.res.j2` and `qm.dr.res.j2`. In the DR template it goes in the `connection { net { … } }` block, where RDQM has it (`rdqm-a1/drbd-conf.txt`).
3. `vrg-container-run -- vrg-validate`.
4. Live proof on a cold-built stack that every HA and DR connection is encrypted. RDQM shows it as `tls:yes` per peer in `drbdadm status` (`rdqm-a1/drbd-status.txt`). A connection without TLS fails loud.
5. The ledger: SC-1.

The logical DR peer name (`dr-remote`, RP-5) is that task's to revisit if the certificate SAN needs it, as RDQM's `encrypted.remote` does.

---

### Task 7 (validation): Cold rebuild brings `pcmk-drbd-ubuntu` up in one pass

Filed with `vrg-issue-create --kind validation`. It is blocked by Task 6b (lab #1609), which is itself blocked by Task 6, and by Task 3's box. It is not PR-workable; run it with `issue-validate`.

- **Precondition self-check:** Tasks 3–6 and 6b are merged to `develop`, and `mqlab box status` lists `pdrbd` as current (not stale) for the host arch.
- **Procedure:**
  1. `mqlab teardown pcmk-drbd-ubuntu` (if anything is up), then a full cold rebuild per `docs/site` (destroy the guests and rebuild from the cached boxes).
  2. `mqlab bootstrap pcmk-drbd-ubuntu` — one invocation, no manual intervention.
  3. `mqlab qm status pcmk-drbd-ubuntu`; `mqlab qm e2e pcmk-drbd-ubuntu`.
  4. `mqlab bootstrap pcmk-drbd-ubuntu` again (idempotency, Review Focus 3).
  5. `mqlab dr cutover pcmk-drbd-ubuntu --rpo0-drill`; `mqlab dr failback pcmk-drbd-ubuntu --rpo0-drill`.
- **Acceptance:**
  - Step 2 succeeds in one pass, **and its transcript shows the watched DR-sync wait ending `in-sync after <n>s`**. Record `n`.
  - Step 3 shows the blueprint's steady state on both sites (Task 6 Step 7 list), all DRBD peers UpToDate on both resources, TLS on every connection (#1609), and e2e green.
  - Step 4 reports no create-md, stacked create-md, mkfs or crtmqm.
  - Step 5 succeeds both ways with the token found.
- **Results template:** `Outcome: SUCCESS|FAILURE`, the bootstrap transcript path, the `qm status` output, the e2e summary and the two dr outputs.

### Task 8 (validation): Drill set on `rdqm-rhel` and `pcmk-drbd-ubuntu`

Filed with `--kind validation`, blocked by Task 7. Arms run **one at a time**: bring one up, drill, record, tear down, then the other.

For each arm, record per drill:

- the commands run;
- the observed end state (`mqlab qm status`);
- whether the QM ran on more than one node at any point. The split-brain check is that `drbdsetup status --verbose` on all nodes of both sites shows at most one Primary per resource: the HA resource per site, and the DR resource across the sites. `drbdadm status all` omits the stacked DR resource (BP README §2.1);
- `mqlab qm e2e` before and after.

**Resource names.** Both arms now have two DRBD resources per QM:

- RDQM: `rdqmapp` (HA) and `rdqmapp.dr` (DR).
- pcmk-drbd: `pdrbdapp` and `pdrbdapp.dr`.

The node that holds the DR resource is the site's HA primary.

| # | Drill | Procedure (both arms) | Expected (both arms; differences are findings, not failures) |
|---|---|---|---|
| 1 | Controlled move | `mqlab qm e2e <stack>`; `mqlab qm down <stack>`; `mqlab qm up <stack>`; `mqlab qm e2e <stack>` | QM stops cleanly, restarts; e2e green before and after |
| 2 | Hard kill of the active node | find the active node (`mqlab qm status`); `virsh destroy <its libvirt domain>` (resolve the domain name the way `lab/scripts/nativeha-fault-suite.sh` does); wait; `mqlab qm status`; `mqlab qm e2e`; restart the domain | QM restarts on a surviving node of the same site, and the DR resource follows it to the new HA primary; no data loss (e2e green) |
| 3 | HA network loss | `lab/scripts/net-down.sh net-hb-a`; observe 2 min; `mqlab qm status`; `lab/scripts/net-up.sh net-hb-a`; observe recovery | No node runs the QM without quorum; after restore one Primary, resync completes, e2e green |
| 3b | **DR quorum on a WAN cut** (both arms; RP-3) | Start from a steady state with the DR pair UpToDate/UpToDate (`drbdsetup status --verbose <qm>.dr` on the site-A DR primary). Start a background `mqlab qm e2e <stack>` loop. Then run `lab/scripts/net-down.sh net-wan` and observe for 2 min. Each minute, record `drbdsetup status --verbose <qm>.dr` on the site-A DR primary (`quorum:`, `role:`, `disk:`), the QM state (`mqlab qm status`) and the e2e loop's result. Then `lab/scripts/net-up.sh net-wan`, wait for the DR resource to reconnect and resync to UpToDate, and run e2e. | **The question:** does the live DR primary keep DR quorum (`quorum:yes`), and does the QM keep serving e2e through the cut, with `quorum majority` / `on-no-quorum io-error` over 2 members? Record the answer per arm. If RDQM keeps serving and pcmk-drbd takes I/O errors, the difference is attributable to IBM's `disconnected` handler (TL-3) **(judgment, to be argued in Task 9)**. It becomes a ledger finding and an owner decision, not a silent config change. After restore: one DR Primary, UpToDate both sides, e2e green |
| 4 | Loss of 2 of 3 HA nodes | `virsh destroy` two site-A nodes including the active one; `mqlab qm status` on the survivor | The survivor does **not** run the QM (no quorum: HA `quorum 2`); restoring one node restores service |
| 5 | DR cutover + failback | Record `crm configure show` (RDQM) / `pcs config` (pcmk-drbd) on both sites before. Run `mqlab dr cutover <stack> --rpo0-drill`; `mqlab qm e2e <stack>`; record both sites' config again. Then `mqlab dr failback <stack> --rpo0-drill`; `mqlab qm e2e <stack>` | Both succeed, with the token found both ways (RPO 0). **RDQM:** the before/after diff shows what `rdqmdr -s`/`-p` actually change. It checks the inference behind Task 6's switch (only the DR clone's `target-role` and `<qm>-dr-role` flip). Any other change is a finding for the ledger (DR-2) |
| 6 | Forced DR with degraded link | `lab/scripts/drbd-degrade.sh break <node> <res>` with the arm's **explicit** node and the **DR** resource (RDQM: the site-A HA primary and `rdqmapp.dr`; pcmk-drbd: the site-A DR primary and `pdrbdapp.dr`), never the script's SAN defaults. Put a few messages. Then stop site A (`virsh destroy` its three nodes) and force-promote at site B (RDQM: `rdqmdr -p -m <QM>` per IBM's forced procedure; pcmk-drbd: `ansible-playbook site-pdrbd-dr-switch.yml -e dr_direction=a2b -e qm_name=PDRBDAPP -e dr_force=true`). Count messages lost. Then restart site A and record what each arm's DR resource does on reconnect, with the exact recovery commands run | The forced promote succeeds only with the explicit force. The loss equals the post-break tail. Without force, the pcmk-drbd playbook refuses (Review Focus 4). If the pcmk-drbd promote is refused for lack of DR quorum (`quorum:no` in the playbook's failure output), that is the RP-3 finding; record it, do not work around it. On reconnect, `after-sb-*` is unset on both arms (RP-7): record the split-brain handling and the operator's recovery |
| 7 | **`ocf:ibm:rdqmx` probe** (RDQM arm only; CL-7) | Read-only. `file /usr/lib/ocf/resource.d/ibm/rdqmx`. If it is a script, read it. `crm_resource --show-metadata ocf:ibm:rdqmx` (the metadata was empty per BP README §3.1). During drills 1 and 5, capture the Pacemaker log lines for `p_rdqmx_rdqmapp` start/stop (`journalctl -u pacemaker` filtered on `rdqmx`) and any files or processes they touch | A written description of what `rdqmx` does, with its evidence (Data), or an explicit "could not determine" (also Data). It feeds CL-7 in Task 9; the open-source arm has no counterpart either way (owner decision 5) |

Notes:

- **Drill 6 pre-check.** `drbd-degrade.sh` runs plain `drbdadm disconnect <res>` with the resource named explicitly. [ocf] notes that drbd-utils ≥ 9.5.0 resolves an explicitly named stacked resource without `--stacked`. If `break` reports the stacked resource as unknown on either arm, fixing the script is a separate code task, filed before the drill runs. A drill that could not be executed is a FAILURE.
- **Drill 3b and the network.** `net-wan` is the DR network on both arms (RP-5). The arms run one at a time, so the cut affects only the arm under test.

**Results template:** one table per arm with the eight rows (1, 2, 3, 3b, 4, 5, 6, 7; drill 7 on RDQM only), each with Outcome, observed end state, split-brain check, e2e before/after and notes. Add `Outcome: SUCCESS` when every drill was executed and recorded on both arms. A behavioural difference between arms is a recorded finding, not a FAILURE; a drill that could not be executed is a FAILURE.

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
