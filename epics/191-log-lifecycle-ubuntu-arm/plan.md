# Native HA log-lifecycle observability — Ubuntu arm — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: use `superpowers:subagent-driven-development` or
> `superpowers:executing-plans` to work this plan task-by-task. Steps use `- [ ]` checkboxes.

**Goal:** Prove the #145 Native HA log-lifecycle observability on the Ubuntu arm
(`nativeha-ubuntu` / `NHAU`), and generalize the one RHEL-hard-scoped validation playbook so a
single parameterized playbook validates either arm — RHEL default behavior unchanged.

**Architecture:** The core (LOGGEREV #809, collector #810, band #811, runbook #813) is already
OS-generic and wired for the Ubuntu group; no core change. One code task generalizes the live-lab
validation playbook and folds in two trivial cleanups; two operational (validation) gates prove the
Ubuntu arm end-to-end on a cold-rebuild.

## Global Constraints

- **RHEL non-regression by construction** — the generalized playbook's default path (no extra-vars)
  is byte-for-byte the current RHEL invocation. The host-group default is expressed **inline in the
  play's `hosts:` templating** (`hosts: "{{ nha_group | default('nha_rhel_a') }}"`), never a `vars:`
  block (a play's `hosts:` is bound before `vars:`).
- **QM name verified, never assumed** — the Ubuntu QM (expected `NHAUAPP` from `short=NHAU`) is
  confirmed via `dspmq` at run time.
- **Validation** — `vrg-container-run -- vrg-validate` is the ONLY validation command for the code
  task. The two operational gates run via `issue-validate` and record PASS/FAIL as a comment
  (not PR-workable; close only on `Outcome: SUCCESS`).
- **Scope guard** — no changes to `LOGGEREV`/collector/band code; no refactor of the per-OS
  `site-nativeha*.yml` provisioning plays.

---

### Task 1: Generalize the validation playbook (+ trivial cleanups) — code task

**Files:**
- Modify: `ansible/validate-nativeha-log-lifecycle.yml` (parameterize host-group + QM)
- Modify: `src/mqlab/stacks.py`, `src/mqlab/phases.py` (correct the stale reserved-stack comments)
- Modify: `docs/site/docs/guides/nativeha-log-lifecycle-guide.md` (one-line both-arms note)

**Produces (consumed by Task 3):** a single playbook that runs against either arm via
`-e nha_group=<group> -e qm_app=<QM>`, defaulting to RHEL.

- [ ] **Step 1: Parameterize the host group.** Replace `hosts: nha_rhel_a` with
  `hosts: "{{ nha_group | default('nha_rhel_a') }}"`. Replace every
  `delegate_to: "{{ groups['nha_rhel_a'][0] }}"` (and any `groups['nha_rhel_a']` lookup) with
  `groups[nha_group][0]`, and set `nha_group: "{{ nha_group | default('nha_rhel_a') }}"` in the
  play `vars:` so the `delegate_to`/lookups resolve consistently. Keep `qm: "{{ qm_app |
  default('NHARAPP') }}"`.
- [ ] **Step 2: Correct the stale comments.** In `src/mqlab/stacks.py` and `src/mqlab/phases.py`,
  fix the "`nativeha-ubuntu` is a reserved stack that cannot be bootstrapped" comments — the stack
  is fully wired (`cluster_group`, `groups`, `provision` all set in `lab/topology.yaml`). Pick a
  still-accurate reserved-stack example, or reword to describe the reserved shape without naming
  `nativeha-ubuntu`.
- [ ] **Step 3: Runbook note.** In `docs/site/docs/guides/nativeha-log-lifecycle-guide.md`, add one
  line stating the guide applies to both Native HA arms (RHEL `nativeha-rhel`/`NHARAPP` and Ubuntu
  `nativeha-ubuntu`/`NHAU`).
- [ ] **Step 4: Validate + commit.**
  ```bash
  vrg-container-run -- vrg-validate      # ansible-lint + full gate; must PASS
  ```
  Expected: PASS, RHEL default path unchanged. Commit:
  `feat(live-lab): parameterize log-lifecycle validation playbook for either nha arm`.

*(RHEL non-regression is guaranteed by the default path; a live RHEL re-run of the generalized
playbook is an optional cheap confirmation, not a gate.)*

---

### Task 2: Validation — cold-rebuild the Ubuntu arm (the #814 equivalent)

**Kind:** `validation` (run via `issue-validate`; record PASS/FAIL as a comment).
**Blocked-by:** nothing within the epic — it validates already-merged (#145) core code on Ubuntu;
runnable once the arm is bootstrapped. (Bake `mq-nativeha-ubuntu` as needed.)

- [ ] **Precondition:** `nativeha-ubuntu` cold-rebuilt from `develop`
  (`mqlab teardown nativeha-ubuntu` if up → `mqlab bootstrap nativeha-ubuntu`).
- [ ] **Assert (one-pass provision):** QM up (`dspmq -o nativeha -x`: one Active + two Replica in
  `nha_ubuntu_a`); `DISPLAY QMGR LOGGEREV` → `ENABLED` with `qm.ini LogType=REPLICATED`;
  `mqlab_log_*` present on all three `nha_ubuntu_a` instances (`sample_stale=0`, active/replica
  divergence visible); `lab-nativeha-state.timer` active on all three.
- [ ] Record `Outcome: SUCCESS` (or FAIL + evidence + follow-on fix task).

---

### Task 3: Validation — live-lab induce-and-assert on Ubuntu (the #815 equivalent)

**Kind:** `validation` (run via `issue-validate`).
**Blocked-by:** Task 1 (generalized playbook) **and** Task 2 (arm cold-rebuilt + up).

- [ ] Confirm the Ubuntu QM name via `dspmq`; run the generalized playbook:
  `ANSIBLE_CONFIG=ansible/ansible.cfg ansible-playbook ansible/validate-nativeha-log-lifecycle.yml
  -e nha_group=nha_ubuntu_a -e qm_app=<Ubuntu QM>`.
- [ ] **Assert:** the induce rolls an extent; the logger-event / collector-moved / all-three-fresh
  assertions pass; the negative-path drift guardrail aborts loud and is rescued. `PLAY RECAP`
  `failed=0` (the guardrail abort is the intentional `rescued=1`).
- [ ] Record `Outcome: SUCCESS` (or FAIL + evidence + follow-on fix task).

---

## Operational gates

Tasks 2 and 3 **are** the operational gates (cold-rebuild + live-lab), per the epic's spec §Design.
No separate cold-rebuild gate is needed beyond Task 2.

## Bookends (already seeded)

- Documentation (spec + plan) — .github#192 (this PR).
- Documentation-review sweep — mq-resiliency-lab-for-linux#994 (runs before the retrospective).
- Retrospective (terminal) — .github#193.

## Self-Review

- **Spec coverage:** §Design-1 (generalize) → Task 1; §Design-2 (cold-rebuild) → Task 2;
  §Design-3 (live-lab) → Task 3; the two folded cleanups → Task 1 Steps 2–3. Acceptance criteria
  map 1:1 (generalized-green + RHEL-identical → Task 1; cold-rebuild PASS → Task 2; live-lab PASS →
  Task 3; docs → Task 1 Step 3 + the #994 sweep).
- **Dependency shape:** Task 1 ∥ Task 2; Task 3 blocked-by {1, 2}. No uncovered requirement; no
  invented behaviour (the Ubuntu QM name is verify-at-runtime).
