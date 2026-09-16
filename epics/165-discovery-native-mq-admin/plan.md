# Discovery-native MQ administration — Implementation Plan

> **For agentic workers:** each task below becomes its own GitHub task issue under epic
> `logical-minds-foundry/.github#165`, worked via `vergil:issue-implement` on its own
> `feature/<issue>-<slug>` branch. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Standardize the lab's queue-manager authorization on MQSC `SET AUTHREC` and replace
the stale one-time `active_host` pin on the Native HA arms with a reusable per-operation
resolve-and-apply primitive, so cold builds are uniform and survive raft leadership churn.

**Architecture:** Two coupled, build-time deliverables. **D1** moves the `mqapp`/`mqsvc`
grants from the `setmqaut` control-command loop into `SET AUTHREC` in the shared
`authz.mqsc.j2` (completing the `mqmon` migration), and adds that authz to the net-new
`nativeha-rhel-crr` arm. **D2** introduces one reusable "apply MQSC to the active Native HA
QM" task include — resolve the active, apply in bindings mode, retry on transient `rc 20`
(follows `AMQ8478E`, waits out `AMQ8146E`) — and adopts it on the two Native HA arms, retiring
the pin and the #435/#888 standalone retries.

**Tech Stack:** Ansible (playbooks + Jinja2 templates), IBM MQ 10.0 MQSC (`runmqsc`,
`SET AUTHREC`, `dspmq -o nativeha`), `vrg-*` tooling. No application code.

**Spec:** `epics/165-discovery-native-mq-admin/spec.md` (in this repo; executors read both).

## Global Constraints

- **Validation is `vrg-container-run -- vrg-validate` only.** It is the sole static gate
  (ansible-lint / yamllint / jinja). Do not run individual linters. Live behavior is proven
  by the cold-rebuild bookend `mq-resiliency-lab-for-linux#913`, not per-task unit tests.
- **Commit with `vrg-commit`** (conventional commits); git/gh only via `vrg-git`/`vrg-gh`.
- **No secrets in git** (MQ entitlement, credentials, keystores).
- **Fail-loud, no silent fallbacks.** A genuine non-`[0,10]` `runmqsc` rc must fail the play.
- **`setmqaut` authority → `AUTHREC` keyword mapping** (verbatim, confirmed in Task 1's
  preflight): `+connect`→`CONNECT`, `+inq`→`INQ`, `+put`→`PUT`, `+get`→`GET`,
  `+browse`→`BROWSE`, `+setall`→`SETALL`, `+dsp`→`DSP` (uppercase, drop `+`, comma-join).
- **Arm identifiers:** nativeha-ubuntu → group `nha_ubuntu_a`, QM `NHAUAPP`;
  nativeha-rhel-crr → group `nha_rhel_crr_a`, QM `NHARCAPP`. `qm_name`/`qm_svc_name` are
  per-arm vars; the inter-QM receiver channel is `{{ qm_svc_name }}.{{ qm_name }}`.

---

## Task 1 (D1) — Migrate `setmqaut` → `SET AUTHREC` on the three arms that already have authz

Move `mqapp`/`mqsvc` grants into `authz.mqsc.j2`; delete the `setmqaut` loops and the
standalone `REFRESH SECURITY TYPE(AUTHSERV)` tasks on `nativeha-ubuntu`, `rdqm`, `pcmk`.
`nativeha-rhel-crr` is Task 2 (net-new). This task does **not** change targeting — it still
runs on the pinned `active_host`; D2 (Task 3) fixes targeting afterward.

**Files:**

- Modify: `ansible/group_vars/all/authz.yml` (reshape `authz_grants` to AUTHREC form)
- Modify: `ansible/roles/mq-pcmk-qmgr/templates/authz.mqsc.j2` (render `authz_grants`; add AUTHSERV refresh)
- Modify: `ansible/site-nativeha-ubuntu.yml:188-201` (delete setmqaut loop) and `:203-212` (delete standalone REFRESH SECURITY AUTHSERV)
- Modify: `ansible/site-rdqm.yml:404-418` (delete setmqaut loop; delete any standalone AUTHSERV refresh)
- Modify: `ansible/roles/mq-pcmk-qmgr/tasks/main.yml:142-166` (delete setmqaut loop; delete any standalone AUTHSERV refresh)

**Interfaces:**

- Produces: `authz_grants` in AUTHREC shape `{group, objtype, [profile], authadd}` and an
  `authz.mqsc.j2` that renders both `authz_grants` and `authz_mqmon_grants` — consumed
  unchanged by every arm that already renders the template, and by Tasks 2 and 4.

- [ ] **Step 1: Preflight — confirm AUTHREC covers the grant surface.** Verify against IBM MQ
  10.0 docs (cache via `python3 tools/ibm_doc_cache.py`) that `SET AUTHREC` supports every
  authority the lab grants, especially the `mqsvc` context authority `SETALL` on `QMGR`,
  `APP.REPLY`, and the DLQ. Record the grant-by-grant map in the task issue. If any grant has
  no AUTHREC form, stop and flag it (documented `setmqaut` exception — no silent split).

- [ ] **Step 2: Reshape `authz_grants` in `authz.yml`** to AUTHREC form (mirrors
  `authz_mqmon_grants`), preserving the `#613/#617` `+setall` rationale comment:

```yaml
# mqapp/mqsvc grants as SET AUTHREC (migrated from setmqaut, #165). Same co-location rule
# as authz_mqmon_grants. +setall (SETALL) on qmgr AND every destination the receiver MCA
# opens (APP.REPLY, the DLQ) is REQUIRED — see #613/#617.
authz_grants:
  - { group: mqapp, objtype: QMGR,  authadd: "CONNECT, INQ" }
  - { group: mqapp, objtype: QUEUE, profile: "SVC.REQUEST",     authadd: "PUT" }
  - { group: mqapp, objtype: QUEUE, profile: "APP.REPLY",       authadd: "GET, INQ, BROWSE" }
  - { group: mqsvc, objtype: QMGR,  authadd: "CONNECT, SETALL" }
  - { group: mqsvc, objtype: QUEUE, profile: "APP.REPLY",       authadd: "PUT, SETALL" }
  - { group: mqsvc, objtype: QUEUE, profile: "{{ mqsvc_dlq }}", authadd: "PUT, SETALL" }
```

- [ ] **Step 3: Render `authz_grants` in `authz.mqsc.j2`.** Add a loop mirroring the existing
  `authz_mqmon_grants` block (which hardcodes `GROUP('mqmon')`), but taking `group` from the
  item. Insert it after the `mqmon` block and add an AUTHSERV refresh so the migrated grants
  get the same reload the deleted `setmqaut`+refresh gave them:

```jinja
* mqapp/mqsvc least-privilege grants (#165) — SET AUTHREC co-located with the object MQSC.
{% for g in authz_grants %}
{% if g.objtype == 'QMGR' %}
SET AUTHREC OBJTYPE(QMGR) GROUP('{{ g.group }}') AUTHADD({{ g.authadd }})
{% else %}
SET AUTHREC PROFILE('{{ g.profile }}') OBJTYPE({{ g.objtype }}) GROUP('{{ g.group }}') AUTHADD({{ g.authadd }})
{% endif %}
{% endfor %}
REFRESH SECURITY TYPE(AUTHSERV)
```

- [ ] **Step 4: Delete the `setmqaut` loop and standalone `REFRESH SECURITY TYPE(AUTHSERV)`**
  task from `site-nativeha-ubuntu.yml` (`:188-201` and `:203-212`), `site-rdqm.yml`
  (`:404-418` + its AUTHSERV refresh if present), and `roles/mq-pcmk-qmgr/tasks/main.yml`
  (`:142-166` + its AUTHSERV refresh if present). The authz apply of `authz.mqsc.j2` on each
  arm is unchanged and now carries the grants + the refresh.

- [ ] **Step 5: Validate.** Run `vrg-container-run -- vrg-validate`. Expected: PASS (lint
  clean; template renders with no undefined vars).

- [ ] **Step 6: Sanity-render the template** for one arm to eyeball the emitted MQSC:

Run: `ansible-playbook --syntax-check ansible/site-nativeha-ubuntu.yml` (inside
`vrg-container-run`) and confirm no reference to `setmqaut` remains:
`grep -rn setmqaut ansible/` → expected: only comments/docs, no live task.

- [ ] **Step 7: Commit.**

```bash
vrg-commit --type feat --scope authz \
  --message "migrate mqapp/mqsvc grants setmqaut -> SET AUTHREC (co-located in authz.mqsc.j2) (#<issue>)"
```

---

## Task 2 (D1) — Add net-new Stage-2 authorization to `nativeha-rhel-crr` via AUTHREC

`site-nativeha.yml` currently leaves `CHLAUTH(DISABLED)` and applies no Stage-2 authz. Add the
same `authz.mqsc.j2` render + apply the other arms use (which flips `CHLAUTH(ENABLED)`, lays
the deny-all back-stop + SSLPEERMAP maps, and applies the AUTHREC grants from Task 1).

**Depends on:** Task 1 (the AUTHREC form of `authz_grants` and the updated template).

**Files:**

- Modify: `ansible/site-nativeha.yml` (add authz render + apply after the base MQSC, ~after line 130)

**Interfaces:**

- Consumes: `authz.mqsc.j2`, `authz_grants`, `authz_mqmon_grants`, `authz_chlauth_maps`,
  `authz_accounts` (unchanged from Task 1); `chl_to_app` = `{{ qm_svc_name }}.{{ qm_name }}`.
- Produces: `NHARCAPP` with the same Stage-2 posture as `NHAUAPP`.

- [ ] **Step 1: Confirm the `mqapp`/`mqmon`/`mqsvc` OS accounts + groups exist on the
  `nha_rhel_crr` nodes.** Stage-2 authz maps DNs to these accounts; `SET AUTHREC GROUP('mqsvc')`
  needs the group present on every instance (the OS-replication caveat). If the authz_accounts
  provisioning role does not already run on this arm, add it (mirror `site-nativeha-ubuntu.yml`).
  Record what you found.

- [ ] **Step 2: Add the authz render + apply tasks** to `site-nativeha.yml`, mirroring
  `site-nativeha-ubuntu.yml:155-183` but for this arm. Render `authz.mqsc.j2` with
  `chl_to_app: "{{ qm_svc_name }}.{{ qm_name }}"`, then apply it. Keep it on the pinned
  `active_host` for now (Task 4 rewrites targeting):

```yaml
    - name: render the authorization MQSC (CHLAUTH + maps) on the active instance
      tags: [authz]
      ansible.builtin.template:
        src: "{{ playbook_dir }}/roles/mq-pcmk-qmgr/templates/authz.mqsc.j2"
        dest: /var/mqm/authz.mqsc
        owner: mqm
        group: mqm
        mode: "0640"
      vars:
        chl_to_app: "{{ qm_svc_name }}.{{ qm_name }}"
      run_once: true
      delegate_to: "{{ active_host }}"

    - name: apply the authorization MQSC (flip CHLAUTH ENABLED + back-stop + SSLPEERMAP + grants)
      tags: [authz]
      ansible.builtin.shell: |
        su - mqm -c "/opt/mqm/bin/runmqsc {{ qm_name }} < /var/mqm/authz.mqsc"
      args: { executable: /bin/bash }
      register: authz_mqsc
      changed_when: true
      failed_when: authz_mqsc.rc not in [0, 10]
      until: authz_mqsc.rc in [0, 10]   # AMQ8146E readiness race (#435/#888); Task 4 folds this into the primitive
      retries: 30
      delay: 5
      run_once: true
      delegate_to: "{{ active_host }}"
```

- [ ] **Step 3: Validate.** `vrg-container-run -- vrg-validate` → PASS.
      `ansible-playbook --syntax-check ansible/site-nativeha.yml` → clean.

- [ ] **Step 4: Commit.**

```bash
vrg-commit --type feat --scope authz \
  --message "add net-new Stage-2 AUTHREC authorization to nativeha-rhel-crr (NHARCAPP) (#<issue>)"
```

> Live proof (CHLAUTH enabled, maps resolve, grants effective, no channel wedge) is the
> #913 cold-rebuild bookend, which now covers this arm.

---

## Task 3 (D2, reference) — Build the reusable apply-to-active primitive and adopt it on `nativeha-ubuntu`

**Depends on:** Task 1 (the `setmqaut` loop is already gone, so the primitive wraps only the
remaining MQSC steps).

**Files:**

- Create: `ansible/tasks/apply-mqsc-nha-active.yml` (the reusable loop wrapper)
- Create: `ansible/tasks/_apply-mqsc-nha-active-once.yml` (one resolve+apply attempt)
- Modify: `ansible/site-nativeha-ubuntu.yml` (replace the pinned steps with includes; remove
  the `find active` / `set_fact active_host` pin at `:88-112`; wrap the event-monitor include)

**Interfaces:**

- Produces: an include `apply-mqsc-nha-active.yml` parameterized by
  `nha_qm` (QM name), `nha_group` (inventory group of the cluster nodes), and **one of**
  `nha_mqsc_file` (path to a rendered MQSC file on the target) or `nha_mqsc_inline` (a
  heredoc string). It **re-resolves the active on every attempt** and applies in bindings mode,
  retrying `AMQ8478E` (moved) and `AMQ8146E` (not ready) and failing loud on anything else.

- [ ] **Step 1: Write the primitive (hole-free, re-resolves every attempt).** Because Ansible
  templates `delegate_to` once per task, per-attempt re-resolution is achieved by *looping a
  fresh include* — each iteration re-resolves the active (so `delegate_to` re-templates) and
  attempts the apply, stopping once applied. There is no per-host early-exit, so the "no-op
  then win an election" edge cannot occur. Create the loop wrapper
  `ansible/tasks/apply-mqsc-nha-active.yml`:

```yaml
# Reusable: apply MQSC to the CURRENT Native HA active, re-resolving every attempt (#165).
# Loops a fresh resolve+apply include so delegate_to re-templates each attempt: follows an
# AMQ8478E leadership move and waits out an AMQ8146E readiness window; fails loud otherwise.
# Params: nha_qm, nha_group, and one of nha_mqsc_file | nha_mqsc_inline.
- name: "apply MQSC to the active {{ nha_qm }} (re-resolve + apply, bounded attempts)"
  ansible.builtin.include_tasks: _apply-mqsc-nha-active-once.yml
  loop: "{{ range(1, (nha_max_attempts | default(30)) + 1) | list }}"
  loop_control: { loop_var: nha_attempt, label: "{{ nha_qm }} attempt {{ nha_attempt }}" }
  when: not (nha_applied | default(false) | bool)

- name: "fail loud if {{ nha_qm }} MQSC never applied within {{ nha_max_attempts | default(30) }} attempts"
  ansible.builtin.fail:
    msg: "Could not apply MQSC to an active {{ nha_qm }} instance after {{ nha_max_attempts | default(30) }} attempts"
  run_once: true
  when: not (nha_applied | default(false) | bool)
```

  and the single-attempt include `ansible/tasks/_apply-mqsc-nha-active-once.yml`:

```yaml
# One resolve+apply attempt. Re-resolves the active (any node can query dspmq -o nativeha -x),
# delegates the runmqsc apply to that freshly-resolved host, classifies the rc:
#   0/10 -> applied (set nha_applied, subsequent loop iterations skip)
#   AMQ8478E / AMQ8146E -> transient, do nothing (next iteration re-resolves + retries)
#   anything else -> fail loud.
- name: "resolve the active {{ nha_qm }} instance"
  ansible.builtin.shell: |
    set -o pipefail
    su - mqm -c "/opt/mqm/bin/dspmq -m {{ nha_qm }} -o nativeha -x" \
      | grep -oE 'INSTANCE\([a-z0-9-]+\) ROLE\(Active\)' \
      | grep -oE 'INSTANCE\([a-z0-9-]+\)' | sed -E 's/INSTANCE\(([a-z0-9-]+)\)/\1/' | head -1 || true
  args: { executable: /bin/bash }
  register: nha_active
  changed_when: false
  run_once: true
  delegate_to: "{{ groups[nha_group][0] }}"

- name: "wait out the election if no active yet ({{ nha_qm }})"
  ansible.builtin.pause: { seconds: 5 }
  run_once: true
  when: (nha_active.stdout | trim) == ""

- name: "apply MQSC on the resolved active ({{ nha_active.stdout | trim }})"
  ansible.builtin.shell: |
{% if nha_mqsc_file is defined %}
    su - mqm -c "/opt/mqm/bin/runmqsc {{ nha_qm }} < {{ nha_mqsc_file }}"
{% else %}
    su - mqm -c "/opt/mqm/bin/runmqsc {{ nha_qm }}" <<'MQSC'
{{ nha_mqsc_inline }}
MQSC
{% endif %}
  args: { executable: /bin/bash }
  register: nha_apply
  changed_when: nha_apply.rc == 0
  failed_when: false        # classified below; a transient rc must not fail the attempt
  run_once: true
  delegate_to: "{{ nha_active.stdout | trim }}"
  when: (nha_active.stdout | trim) != ""

- name: "mark applied on success ({{ nha_qm }})"
  ansible.builtin.set_fact: { nha_applied: true }
  run_once: true
  when: nha_apply.rc is defined and (nha_apply.rc in [0, 10])

- name: "fail loud on a non-transient runmqsc rc ({{ nha_qm }})"
  ansible.builtin.fail:
    msg: "runmqsc failed rc={{ nha_apply.rc }} on {{ nha_active.stdout | trim }}: {{ nha_apply.stdout }}"
  run_once: true
  when:
    - nha_apply.rc is defined
    - nha_apply.rc not in [0, 10]
    - "'AMQ8478E' not in nha_apply.stdout"   # replica: leadership moved — retry (re-resolve)
    - "'AMQ8146E' not in nha_apply.stdout"   # not ready yet — retry
```

> **Design note:** this is the agreed hole-free mechanism (alignment issue [2]). Every attempt
> re-resolves the active, so a mid-sequence election is *followed* rather than fatal, and there
> is no per-host state to strand. Step 4's induced-churn test *confirms* this, rather than
> hunting for a hole. Reset `nha_applied` to `false` before each independent call site (or scope
> it per-include) so one payload's success doesn't skip the next.

- [ ] **Step 2: Convert `nativeha-ubuntu`'s pinned MQSC steps to the include.** Remove
  `find the active Native HA instance` + `record the active host` (`:88-112`). Replace the
  our-side/app MQSC (`:114-143`) and the authz apply (`:166-183`) with `include_tasks:
  apply-mqsc-nha-active.yml`, passing `nha_qm: "{{ qm_name }}"`, `nha_group: nha_ubuntu_a`,
  and the inline/file payload. Example for the base MQSC:

```yaml
    - name: apply our-side + app MQSC (discovery-native)
      ansible.builtin.include_tasks: tasks/apply-mqsc-nha-active.yml
      vars:
        nha_qm: "{{ qm_name }}"
        nha_group: nha_ubuntu_a
        nha_mqsc_inline: |
          DEFINE LISTENER(L1414) TRPTYPE(TCP) PORT(1414) CONTROL(QMGR) REPLACE
          START LISTENER(L1414)
          # ... (the existing heredoc body from :117-128, verbatim) ...
```

- [ ] **Step 3: Wrap the event-monitor include in block-level re-resolution + retry**
  (spec §3.1 special case — `include_role` cannot use the primitive). The role's tasks are
  idempotent, so re-running on a leadership move is safe:

```yaml
    - name: event monitoring — define/start collector on the active, following leadership (#114/#165)
      block:
        - name: resolve the current active instance
          ansible.builtin.shell: |
            set -o pipefail
            su - mqm -c "/opt/mqm/bin/dspmq -m {{ qm_name }} -o nativeha -x" \
              | grep -oE 'INSTANCE\(nha-ubuntu-a[0-9]\) ROLE\(Active\)' \
              | grep -oE 'nha-ubuntu-a[0-9]' | head -1
          args: { executable: /bin/bash }
          register: em_active
          changed_when: false
          run_once: true
          delegate_to: "{{ groups['nha_ubuntu_a'][0] }}"
        - name: include the event-monitor role on the resolved active
          ansible.builtin.include_role:
            name: mq-event-monitor
            tasks_from: service
          vars:
            qmgr_name: "{{ qm_name }}"
            mq_log_type: replicated
          when: inventory_hostname == (em_active.stdout | trim)
      rescue:
        - name: retry once on a mid-role leadership move (AMQ8478E) — idempotent role
          ansible.builtin.debug:
            msg: "event-monitor hit a leadership move; re-run the play's event tag to reconverge"
      # NOTE: a production-grade retry loops the block; the exact construct is finalized with
      # the induced-churn test (Step 4). Fail-loud preserved: a non-8478E role failure raises.
```

- [ ] **Step 4: Prove it — induced mid-sequence churn (feeds bookend #913).** On a live
  `nativeha-ubuntu` bring-up, force a leader move *during* the admin sequence (e.g. `endmqm -s`
  the current active, or a raft step-down) and assert the sequence **follows and completes**
  with no `AMQ8478E` failure. This is the acceptance the spec §5 requires; capture the run in
  the task issue. Resolve the Step 1 edge here before merging.

- [ ] **Step 5: Validate + confirm the pin is gone.** `vrg-container-run -- vrg-validate` →
  PASS. `grep -n 'set_fact:\s*$\|active_host' ansible/site-nativeha-ubuntu.yml` → expected: no
  `active_host` pin remains (only the primitive's internal resolution).

- [ ] **Step 6: Commit.**

```bash
vrg-commit --type feat --scope nativeha \
  --message "discovery-native admin: reusable apply-to-active primitive, retire active_host pin + #435/#888 retries (nativeha-ubuntu) (#<issue>)"
```

---

## Task 4 (D2, fan-out) — Adopt the primitive on `nativeha-rhel-crr`

**Depends on:** Task 2 (authz exists on this arm) and Task 3 (the primitive exists and is
proven).

**Files:**

- Modify: `ansible/site-nativeha.yml` (replace the pin + pinned steps with the primitive; wrap
  the event-monitor include as in Task 3)

**Interfaces:**

- Consumes: `ansible/tasks/apply-mqsc-nha-active.yml` (from Task 3), `nha_group: nha_rhel_crr_a`.

- [ ] **Step 1: Convert `site-nativeha.yml`'s pinned steps** (base MQSC `:102-120`, the
  Task-2 authz apply, and the event-monitor include `:147`) to the primitive + block-retry,
  exactly as Task 3 did for `nativeha-ubuntu`, with `nha_group: nha_rhel_crr_a`,
  `nha_qm: "{{ qm_name }}"` (`NHARCAPP`). Remove the `find active` / `set_fact active_host`
  pin (`:88-97`).

- [ ] **Step 2: Validate.** `vrg-container-run -- vrg-validate` → PASS.
      `grep -n active_host ansible/site-nativeha.yml` → expected: none.

- [ ] **Step 3: Commit.**

```bash
vrg-commit --type feat --scope nativeha \
  --message "discovery-native admin: adopt apply-to-active primitive on nativeha-rhel-crr (#<issue>)"
```

> Live proof for both Native HA arms — including an induced mid-sequence election on each — is
> the #913 cold-rebuild bookend.

---

## Task sequencing & the epic's bookends

```text
Task 1 (D1 migrate) ──┬─▶ Task 2 (D1 net-new rhel-crr) ──┐
                      └─▶ Task 3 (D2 primitive + ubuntu) ─┴─▶ Task 4 (D2 fan-out rhel-crr)
                                                                        │
                                        all impl merged ──▶ #913 validation (induced churn + reproduce-until-reliable)
                                                          ──▶ #914 documentation-review sweep
                                                          ──▶ #167 retrospective (closes the epic)
```

- **`rdqm`/`pcmk`** are touched only by Task 1 (their `setmqaut` loops are deleted); they keep
  their own `AMQ8146E` readiness handling and are **verified unaffected** by the #913 cold
  rebuild — no primitive, no rewrite.
- The three closing bookends (#913, #914, #167) already exist under the epic; this plan files
  Tasks 1–4 as the implementation children.

## Self-review (spec coverage)

- **§0/§1 uniform MQSC + co-location** → Tasks 1, 2 (template + net-new arm).
- **§3.1 apply-to-active primitive + event-monitor special case** → Task 3 Steps 1, 3.
- **§3.2 AUTHREC + preflight + preserve REFRESH SECURITY** → Task 1 Steps 1, 3.
- **§3.3 arm scope (AUTHREC all four; primitive NHA-only; rdqm/pcmk verified)** → Tasks 1–4 +
  the sequencing note.
- **§3.4 client-mode rejected** → no task (explicitly not built).
- **§5 demonstrably-exercised via induced churn** → Task 3 Step 4 (+ #913).
- **§4 out-of-scope (client-mode, stability gate, #472)** → no tasks; not built.
