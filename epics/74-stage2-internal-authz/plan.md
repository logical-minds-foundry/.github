# Stage 2 — Internal MQ Authorization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the `pcmk-ubuntu` app queue manager from *authenticated-but-unauthorized* into *enforced least-privilege*: map each authenticated TLS certificate DN to a non-privileged service account and lock the queues to a minimal `setmqaut` surface — then prove, on the running lab, that an unauthorized action is refused (even after failover).

**Architecture:** Three OS service accounts (`mqapp`/`mqmon`/`mqsvc`) provisioned on every QM node; `CHLAUTH(ENABLED)` with a deny-all back-stop and per-channel `SSLPEERMAP` rules that assign those accounts by cert DN; minimal `setmqaut` grants to each account's private group; all authored **variable-driven** so the model fans out to the other arms as a per-arm delta. Validation is induced-and-asserted on the live lab (the repo has no behavioral lint gate — the running arm + a cold rebuild are the gate).

**Tech Stack:** Ansible (roles + `runmqsc`/`setmqaut` via `su mqm`), IBM MQ 9.4 CHLAUTH/OAM, pymqi (induced-denial probe), Pacemaker (failover).

## Global Constraints

- **Never hardcode QM or inter-QM channel names.** Use the topology variables `qm_app`, `qm_svc`, `chl_to_app`, `chl_to_svc` (#350/#351). Concrete values on this arm: `qm_app=PCMKAPP`, `qm_svc=SVCQM`, `chl_to_app=SVCQM.PCMKAPP`. The SVRCONN names `APP.SVRCONN`/`MON.SVRCONN` and local queues `APP.REPLY`/`SVC.REQUEST` **are** literals in the role.
- **Scope: `pcmk-ubuntu` only.** `qm_svc`/`SVCQM` (the shared black-box counterparty, #446) is out of scope — it keeps `CHLAUTH(DISABLED)`/`MCAUSER('mqm')`. `nativeha`/`rdqm` fan-out and the research/mqweb tasks are separate epic tasks (Part B).
- **CONNAUTH stays disabled** (`CONNAUTH(' ')`). The certificate is the credential.
- **Service-account names ≤12 chars, no login shell, no password**, each in its own private primary group.
- **Idempotent + cold-rebuild-safe.** Every change must survive a one-pass cold VM rebuild (the acceptance gate) and re-runs. MQSC uses `REPLACE`/`ACTION(REPLACE)`; `setmqaut` is declarative.
- **Grants attach to the group** (`setmqaut -g <group>`), not the principal.
- **Single source of truth:** all accounts, DN maps, and grant sets live in `ansible/group_vars/all/authz.yml` (mirroring `tls.yml`).

---

## File Structure

- `ansible/group_vars/all/authz.yml` — **new.** Accounts, groups, DN→account maps, grant sets, DLQ setting. The single source.
- `ansible/roles/mq-authz-accounts/tasks/main.yml` — **new.** Idempotent OS group+user provisioning from `authz.yml`. Applied to every QM node.
- `ansible/roles/mq-authz-accounts/defaults/main.yml` — **new.** Role defaults.
- `ansible/roles/mq-pcmk-qmgr/templates/authz.mqsc.j2` — **new.** Variable-driven CHLAUTH-enable + deny-all back-stop + `SSLPEERMAP` rules + the `setmqaut` grant block driver. Arm-agnostic content (fan-out reuses it).
- `ansible/roles/mq-pcmk-qmgr/tasks/main.yml` — **modify.** Strip `MCAUSER('mqm')`→blank on the two SVRCONNs; render+apply `authz.mqsc.j2`; apply `setmqaut` grants.
- `ansible/roles/mq-pcmk-qmgr/templates/inter-qm.mqsc.j2` — **modify.** Add `PUTAUT(DEF)` to the `chl_to_app` RCVR.
- `ansible/site-pcmk.yml` — **modify.** Add a play applying `mq-authz-accounts` across `pcmk_a` + `pcmk_b` (all 6 nodes).
- `clients/authz_probe.py` — **new.** pymqi induced-denial probe (connect as an identity, attempt an operation, assert `2035`).
- `ansible/site-pcmk-authz-validate.yml` — **new.** Induced-denial + positive + post-failover validation playbook.

---

## Part A — `pcmk-ubuntu` implementation

### Task 1: `authz.yml` — the single source of truth

**Files:**
- Create: `ansible/group_vars/all/authz.yml`

**Interfaces:**
- Produces: the vars `authz_accounts` (list of `{name, group}`), `authz_chlauth_maps` (list of `{channel, sslpeer, mcauser}`), `authz_grants` (list of `{group, object_type, object, authorities}`), `mqsvc_dlq`. Consumed by Tasks 2–4.

- [ ] **Step 1: Write the var file**

```yaml
---
# Stage 2 internal authorization (#74) — single source for accounts, DN→account
# maps, and OAM grant sets. Mirrors tls.yml. Arm-agnostic: the DNs are identical
# across arms (partial-DN PKI); only per-arm channel names differ (chl_to_app).
# NOTE: the SSLPEERMAP DNs are FINER than the Stage-1 channel SSLPEER (org-only):
# Stage 2 reads the OU to tell mqapp (OU=apps) from mqmon (OU=ops).

authz_accounts:
  - { name: mqapp, group: mqapp }
  - { name: mqmon, group: mqmon }
  - { name: mqsvc, group: mqsvc }

# DN→account maps. channel is a literal for SVRCONNs; the inter-QM receiver is the
# chl_to_app VARIABLE (resolved by the template).
authz_chlauth_maps:
  - { channel: "APP.SVRCONN", sslpeer: "O=app-org,OU=apps", mcauser: mqapp }
  - { channel: "MON.SVRCONN", sslpeer: "O=app-org,OU=ops",  mcauser: mqmon }
  - { channel: "{{ chl_to_app }}", sslpeer: "{{ tls_peer_svc }}", mcauser: mqsvc }

# Minimal grant sets (starting hypothesis; hardened by the induced tests).
authz_grants:
  # client app: put requests, get replies
  - { group: mqapp, object_type: qmgr,  object: "",            authorities: "+connect +inq" }
  - { group: mqapp, object_type: queue, object: "SVC.REQUEST", authorities: "+put" }
  - { group: mqapp, object_type: queue, object: "APP.REPLY",   authorities: "+get +inq +browse" }
  # monitoring: read-only — connect + inquire + subscribe metric topics; NO put/get.
  # NOTE (to narrow — do NOT ship the root grant): SYSTEM.BASE.TOPIC is the topic-tree
  # ROOT and over-grants (subscribe to everything), against the least-privilege thesis.
  # It is a PLACEHOLDER: the exact topic object scoping the $SYS/MQ resource-monitoring
  # tree is pinned against the LIVE exporter in Task 5, then substituted here.
  - { group: mqmon, object_type: qmgr,  object: "",                 authorities: "+connect +inq" }
  - { group: mqmon, object_type: topic, object: "SYSTEM.BASE.TOPIC", authorities: "+sub" }
  - { group: mqmon, object_type: queue, object: "APP.REPLY",        authorities: "+dsp +inq" }
  - { group: mqmon, object_type: queue, object: "SVC.REQUEST",      authorities: "+dsp +inq" }
  # service app (counterparty receiver MCA): put reply, put DLQ; nothing else
  - { group: mqsvc, object_type: qmgr,  object: "",                authorities: "+connect" }
  - { group: mqsvc, object_type: queue, object: "APP.REPLY",       authorities: "+put" }
  - { group: mqsvc, object_type: queue, object: "{{ mqsvc_dlq }}", authorities: "+put" }

# DLQ the receiver MCA falls back to. INTERIM (pending the per-channel-vs-QM-level
# DLQ research, Part B): grant +put on the QM-wide DLQ + set it as DEADQ (authz.mqsc.j2)
# so an undeliverable reply can't stall the RCVR channel — a resiliency lab must not
# ship a known channel-stall. The research owns the FINAL shape (a dedicated
# per-counterparty DLQ vs this shared one); the N5 induced test (validation-hardening
# follow-up) confirms the fallback works and FEEDS the research — it does not
# rubber-stamp this interim.
mqsvc_dlq: "SYSTEM.DEAD.LETTER.QUEUE"
```

- [ ] **Step 2: Sanity-check YAML parses**

Run: `cd ansible && python3 -c "import yaml,sys; yaml.safe_load(open('group_vars/all/authz.yml'))" && echo OK`
Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add ansible/group_vars/all/authz.yml
git commit -m "feat(authz): single-source vars for Stage 2 accounts, DN maps, grants (#74)"
```

---

### Task 2: `mq-authz-accounts` role — provision the OS principals on every node

**Files:**
- Create: `ansible/roles/mq-authz-accounts/tasks/main.yml`
- Create: `ansible/roles/mq-authz-accounts/defaults/main.yml`
- Modify: `ansible/site-pcmk.yml` (add an all-nodes play)

**Interfaces:**
- Consumes: `authz_accounts` (Task 1).
- Produces: OS groups + users `mqapp`/`mqmon`/`mqsvc` on every host the play targets.

- [ ] **Step 1: Write the role defaults**

```yaml
---
# mq-authz-accounts: create the OAM service principals. Overridden by authz.yml.
authz_accounts: []
```

- [ ] **Step 2: Write the role tasks**

```yaml
---
# Idempotent OAM principals (#74). Group-based OAM authorizes by PRIMARY GROUP,
# so each account is a system user in its own private group, no shell/password.
# Must exist on EVERY node the QM can land on (HA + DR) or authorization breaks
# after failover — the substrate-consistency invariant (spec §6).
- name: authz groups
  ansible.builtin.group:
    name: "{{ item.group }}"
    system: true
  loop: "{{ authz_accounts }}"
  loop_control: { label: "{{ item.group }}" }

- name: authz service accounts (no shell, no login)
  ansible.builtin.user:
    name: "{{ item.name }}"
    group: "{{ item.group }}"
    system: true
    shell: /usr/sbin/nologin
    create_home: false
    password: "!"
    password_lock: true
  loop: "{{ authz_accounts }}"
  loop_control: { label: "{{ item.name }}" }
```

- [ ] **Step 3: Wire the role into `site-pcmk.yml` across `pcmk_a` + `pcmk_b`**

Add this play near the top of `ansible/site-pcmk.yml` (before the QM-create play, so the accounts exist before any `setmqaut`). Insert after the `import_playbook` lines:

```yaml
- name: authz service accounts on every app-QM node (HA + DR)
  hosts: pcmk_a:pcmk_b
  become: true
  roles: [mq-authz-accounts]
```

- [ ] **Step 4: Verify idempotency + presence (live, after a run)**

Run (on each pcmk node): `id mqapp && id mqmon && id mqsvc`
Expected: each resolves with its own primary group (e.g. `uid=… (mqapp) gid=… (mqapp) groups=…(mqapp)`).
Run the play twice; expected: second run reports `ok` (no `changed`) for the account tasks.

- [ ] **Step 5: Commit**

```bash
git add ansible/roles/mq-authz-accounts ansible/site-pcmk.yml
git commit -m "feat(authz): provision mqapp/mqmon/mqsvc on every pcmk node (#74)"
```

---

### Task 3: CHLAUTH-enable + deny-all back-stop + `SSLPEERMAP` maps

**Files:**
- Create: `ansible/roles/mq-pcmk-qmgr/templates/authz.mqsc.j2`
- Modify: `ansible/roles/mq-pcmk-qmgr/tasks/main.yml` (strip MCAUSER; render+apply the snippet)
- Modify: `ansible/roles/mq-pcmk-qmgr/templates/inter-qm.mqsc.j2` (PUTAUT on the RCVR)

**Interfaces:**
- Consumes: `authz_chlauth_maps`, `mqsvc_dlq` (Task 1); `qm_name`, `chl_to_app`, `tls_peer_svc` (topology).
- Produces: `CHLAUTH(ENABLED)` with the deny-all back-stop + three maps live on the QM; the RCVR runs `PUTAUT(DEF)`.

- [ ] **Step 1: Write the authz MQSC snippet (variable-driven, arm-agnostic)**

Create `ansible/roles/mq-pcmk-qmgr/templates/authz.mqsc.j2`:

```jinja
* Stage 2 authorization (#74) for {{ qm_name }} — arm-agnostic; fan-out reuses this.
* Enable CHLAUTH, deny-all back-stop FIRST, then per-channel SSLPEERMAP maps.
* The only path to an identity is a map; anything unmatched hits the back-stop.
ALTER QMGR CHLAUTH(ENABLED)
* Wire the receiver fallback DLQ so an undeliverable message has somewhere to go.
ALTER QMGR DEADQ({{ mqsvc_dlq }})
SET CHLAUTH('*') TYPE(ADDRESSMAP) ADDRESS('*') USERSRC(NOACCESS) ACTION(REPLACE)
{% for m in authz_chlauth_maps %}
SET CHLAUTH('{{ m.channel }}') TYPE(SSLPEERMAP) SSLPEER('{{ m.sslpeer }}') USERSRC(MAP) MCAUSER('{{ m.mcauser }}') ACTION(REPLACE)
{% endfor %}
REFRESH SECURITY TYPE(CONNAUTH)
```

- [ ] **Step 2: Strip the fixed `MCAUSER('mqm')` from the two SVRCONNs**

In `ansible/roles/mq-pcmk-qmgr/tasks/main.yml`, the "apply MQSC config" block (lines ~73–74), remove `MCAUSER('"'"'mqm'"'"') ` from both `DEFINE CHANNEL(APP.SVRCONN…)` and `DEFINE CHANNEL(MON.SVRCONN…)` so the identity comes only from the CHLAUTH map. Leave `ALTER QMGR CHLAUTH(DISABLED) CONNAUTH(' ')` as-is here — Task 3 Step 4 flips CHLAUTH via the snippet *after* the channels exist (blank MCAUSER is only safe once the back-stop is in place).

Result (both lines, MCAUSER removed):

```text
DEFINE CHANNEL(APP.SVRCONN) CHLTYPE(SVRCONN) TRPTYPE(TCP) SSLCIPH({{ tls_cipher }}) SSLCAUTH(REQUIRED) SSLPEER('"'"'{{ tls_peer_client }}'"'"') HBINT(15) KAINT(15) REPLACE\n
DEFINE CHANNEL(MON.SVRCONN) CHLTYPE(SVRCONN) TRPTYPE(TCP) SSLCIPH({{ tls_cipher }}) SSLCAUTH(REQUIRED) SSLPEER('"'"'{{ tls_peer_client }}'"'"') REPLACE\n
```

- [ ] **Step 3: Add `PUTAUT(DEF)` to the receiver channel**

In `ansible/roles/mq-pcmk-qmgr/templates/inter-qm.mqsc.j2`, line 9, add `PUTAUT(DEF)` to the RCVR so put-authority keys off the mapped `mqsvc`, not the message-context user:

```jinja
DEFINE CHANNEL({{ chl_to_app }}) CHLTYPE(RCVR) TRPTYPE(TCP) SSLCIPH({{ tls_cipher }}) SSLCAUTH(REQUIRED) SSLPEER('{{ tls_peer_svc }}') PUTAUT(DEF) REPLACE
```

- [ ] **Step 4: Render + apply the authz snippet after the inter-QM MQSC**

In `ansible/roles/mq-pcmk-qmgr/tasks/main.yml`, immediately after the "apply the inter-QM MQSC" task (line ~109), add (inside the same first-time `block`, so it persists on the LUN one-pass):

```yaml
    # Stage 2 authorization (#74): applied AFTER the channels + inter-QM objects
    # exist, so the maps/back-stop reference real channels. Atomic runmqsc batch
    # before endmqm, so no legitimate channel bounces through the deny-all window.
    - name: render the authorization MQSC (CHLAUTH + maps)
      ansible.builtin.template:
        src: authz.mqsc.j2
        dest: /var/mqm/authz.mqsc
        owner: mqm
        group: mqm
        mode: "0640"
      run_once: true

    - name: apply the authorization MQSC
      ansible.builtin.shell: |
        su mqm -c '/opt/mqm/bin/strmqm {{ qm_name }}' || true
        su mqm -c '/opt/mqm/bin/runmqsc {{ qm_name }} < /var/mqm/authz.mqsc'
        su mqm -c '/opt/mqm/bin/endmqm -w {{ qm_name }}'
      run_once: true
      changed_when: true
```

- [ ] **Step 5: Verify the rules are live (after a run)**

Run: `su mqm -c "echo 'DIS CHLAUTH(*)' | runmqsc {{ qm_app }}"`
Expected: the deny-all `ADDRESSMAP … USERSRC(NOACCESS)` and three `SSLPEERMAP` rules mapping to `mqapp`/`mqmon`/`mqsvc`; `DIS QMGR CHLAUTH` shows `ENABLED`.

- [ ] **Step 6: Commit**

```bash
git add ansible/roles/mq-pcmk-qmgr/templates/authz.mqsc.j2 ansible/roles/mq-pcmk-qmgr/tasks/main.yml ansible/roles/mq-pcmk-qmgr/templates/inter-qm.mqsc.j2
git commit -m "feat(authz): CHLAUTH-enable + deny-all back-stop + SSLPEERMAP maps on pcmk (#74)"
```

---

### Task 4: `setmqaut` grants — the minimal authority surface

**Files:**
- Modify: `ansible/roles/mq-pcmk-qmgr/tasks/main.yml`

**Interfaces:**
- Consumes: `authz_grants` (Task 1); `qm_name`. Requires Task 2 (accounts) + Task 3 (channels/maps).
- Produces: OAM authority records on the QM (on the LUN, so they follow it).

- [ ] **Step 1: Add the grant-application task**

In `ansible/roles/mq-pcmk-qmgr/tasks/main.yml`, immediately after the "apply the authorization MQSC" task, add (same first-time block). This builds one `setmqaut` invocation per grant; `-t qmgr` grants omit `-n`:

```yaml
    - name: apply setmqaut grants (least-privilege OAM)
      ansible.builtin.command:
        cmd: >
          su mqm -c "/opt/mqm/bin/setmqaut -m {{ qm_name }}
          {% if item.object_type != 'qmgr' %}-n {{ item.object }} {% endif %}-t {{ item.object_type }}
          -g {{ item.group }} {{ item.authorities }}"
      loop: "{{ authz_grants }}"
      loop_control: { label: "{{ item.group }} -> {{ item.object_type }} {{ item.object }}" }
      run_once: true
      register: setmqaut_res
      changed_when: true
      failed_when: setmqaut_res.rc != 0
```

- [ ] **Step 2: Refresh the authorization service so grants bite immediately**

Add directly after (grants apply on next connect, but refresh makes verification deterministic):

```yaml
    - name: refresh the authorization service
      ansible.builtin.shell: |
        su mqm -c 'strmqm {{ qm_name }}' || true
        printf 'REFRESH SECURITY TYPE(AUTHSERV)\n' | su mqm -c 'runmqsc {{ qm_name }}'
        su mqm -c 'endmqm -w {{ qm_name }}'
      run_once: true
      changed_when: true
```

- [ ] **Step 3: Verify grants (after a run)**

Run: `su mqm -c "dmpmqaut -m {{ qm_app }} -g mqmon"`
Expected: `mqmon` shows `connect,inq` on qmgr, `sub` on the topic, `dsp,inq` on the queues — and **no** `put`/`get`. `dmpmqaut -g mqsvc` shows `put` on `APP.REPLY` + the DLQ only.

- [ ] **Step 4: Commit**

```bash
git add ansible/roles/mq-pcmk-qmgr/tasks/main.yml
git commit -m "feat(authz): minimal setmqaut grants per service account on pcmk (#74)"
```

---

### Task 5: Induced-denial probe + validation playbook

**Files:**
- Create: `clients/authz_probe.py`
- Create: `ansible/site-pcmk-authz-validate.yml`

**Interfaces:**
- Consumes: the running, authorized `pcmk-ubuntu` arm (Tasks 1–4 applied). Uses the `mq_prometheus` (ops, `OU=ops`) keystore already placed by `mq-exporter`/`mq-client` for the `mqmon` identity.
- Produces: pass/fail acceptance for the epic's `validation` task.

- [ ] **Step 1: Write the induced-denial probe**

Create `clients/authz_probe.py` — connect over a SVRCONN with a given cert and attempt an operation, asserting the expected MQ reason code. (Follows the existing `clients/app_requester.py` pymqi + TLS idiom: `MQCD` with `SSLCipherSpec`, `MQSCO` with `KeyRepository`.)

```python
#!/usr/bin/env python3
"""Induced-denial probe (#74): connect as an identity and assert the authorization
outcome. Exit 0 only if the observed reason code matches --expect."""
import argparse, sys
import pymqi

def main():
    p = argparse.ArgumentParser()
    p.add_argument("--qmgr", required=True)
    p.add_argument("--conn", required=True)          # host(port)
    p.add_argument("--channel", required=True)       # e.g. MON.SVRCONN
    p.add_argument("--cipher", required=True)         # tls_cipher
    p.add_argument("--keyrepo", required=True)        # stem, no .kdb
    p.add_argument("--queue", required=True)          # target queue
    p.add_argument("--op", choices=["put", "get"], required=True)
    p.add_argument("--expect", type=int, required=True)  # e.g. 2035
    a = p.parse_args()

    cd = pymqi.CD()
    cd.ChannelName = a.channel.encode()
    cd.ConnectionName = a.conn.encode()
    cd.ChannelType = pymqi.CMQC.MQCHT_CLNTCONN
    cd.TransportType = pymqi.CMQC.MQXPT_TCP
    cd.SSLCipherSpec = a.cipher.encode()
    sco = pymqi.SCO()
    sco.KeyRepository = a.keyrepo.encode()

    observed = 0
    qmgr = None
    try:
        qmgr = pymqi.QueueManager(None)
        qmgr.connect_with_options(a.qmgr, cd=cd, sco=sco)
        q = pymqi.Queue(qmgr, a.queue)
        if a.op == "put":
            q.put(b"authz-probe")
        else:
            q.get()
        q.close()
        observed = 0  # MQRC_NONE — the op succeeded
    except pymqi.MQMIError as e:
        observed = e.reason
    finally:
        if qmgr:
            try: qmgr.disconnect()
            except pymqi.MQMIError: pass

    ok = observed == a.expect
    print(f"channel={a.channel} op={a.op} queue={a.queue} observed={observed} expect={a.expect} -> {'PASS' if ok else 'FAIL'}")
    sys.exit(0 if ok else 1)

if __name__ == "__main__":
    main()
```

- [ ] **Step 2: Write the validation playbook**

Create `ansible/site-pcmk-authz-validate.yml`. It runs on the app host (where the pymqi venv + keystores live). Assumes `authz_probe.py` is synced alongside the other `clients/` scripts.

```yaml
---
# Stage 2 authorization validation (#74): induced-denial + positive + post-failover.
# Interim scripted checks (first consumers of the #38 live-lab validation pattern).
- name: Stage 2 authorization — negative + positive
  hosts: app
  become: true
  vars:
    app_qm: "{{ qm_app }}"
    app_conn: "pcmk-vip-a.client.com(1414)"
    mon_keyrepo: /var/mqm/ssl/mq_prometheus   # stem placed by mq-exporter/mq-client
    app_keyrepo: /var/mqm/ssl/app-client      # stem placed by mq-client
  tasks:
    # NEGATIVE N1: mqmon (OU=ops via MON.SVRCONN) must be denied a put.
    - name: mqmon MQPUT to APP.REPLY is denied (2035)
      ansible.builtin.command: >
        python3 clients/authz_probe.py --qmgr {{ app_qm }} --conn "{{ app_conn }}"
        --channel MON.SVRCONN --cipher {{ tls_cipher }} --keyrepo {{ mon_keyrepo }}
        --queue APP.REPLY --op put --expect 2035
      args: { chdir: /opt/mqlab }   # wherever clients/ is synced on the app host
      changed_when: false

    # NEGATIVE N3: mqapp (OU=apps via APP.SVRCONN) may PUT SVC.REQUEST but is denied
    # a GET from it — least privilege within the app's own queues.
    - name: mqapp MQGET from SVC.REQUEST is denied (2035)
      ansible.builtin.command: >
        python3 clients/authz_probe.py --qmgr {{ app_qm }} --conn "{{ app_conn }}"
        --channel APP.SVRCONN --cipher {{ tls_cipher }} --keyrepo {{ app_keyrepo }}
        --queue SVC.REQUEST --op get --expect 2035
      args: { chdir: /opt/mqlab }
      changed_when: false

    # POSITIVE: the app trade flow still completes end-to-end.
    - name: app trade flow still works (app_requester round-trip)
      ansible.builtin.command: python3 clients/app_requester.py --once
      args: { chdir: /opt/mqlab }
      changed_when: false
      register: tradeflow
      failed_when: tradeflow.rc != 0

# POST-FAILOVER: move the QM to another node, re-run one negative + one positive.
- name: force a Pacemaker failover
  hosts: pcmk_a
  become: true
  run_once: true
  tasks:
    - name: move mq_group to another node
      ansible.builtin.command: pcs resource move mq_group
      changed_when: true
    - name: wait for the QM to settle on the new node
      ansible.builtin.command: pcs resource status mq_qm
      register: r
      until: "'Started' in r.stdout"
      retries: 30
      delay: 5
      changed_when: false
    - name: clear the move constraint
      ansible.builtin.command: pcs resource clear mq_group
      changed_when: true

- name: re-run authorization checks after failover
  hosts: app
  become: true
  vars:
    app_qm: "{{ qm_app }}"
    app_conn: "pcmk-vip-b.client.com(1414)"       # DR/other-node VIP
    mon_keyrepo: /var/mqm/ssl/mq_prometheus
  tasks:
    - name: mqmon still denied a put after failover (2035)
      ansible.builtin.command: >
        python3 clients/authz_probe.py --qmgr {{ app_qm }} --conn "{{ app_conn }}"
        --channel MON.SVRCONN --cipher {{ tls_cipher }} --keyrepo {{ mon_keyrepo }}
        --queue APP.REPLY --op put --expect 2035
      args: { chdir: /opt/mqlab }
      changed_when: false
    - name: app trade flow still works after failover
      ansible.builtin.command: python3 clients/app_requester.py --once
      args: { chdir: /opt/mqlab }
      changed_when: false
```

> **Grounding note for the implementer:** confirm three things against the live arm before finalizing — (1) the exact `mq_prometheus` and `app-client` keystore stems/paths placed by `mq-exporter`/`mq-client`; (2) that `app_requester.py` supports a one-shot `--once` (add it if not); (3) the app-host sync path for `clients/` (`chdir`). These are known-unknowns, not guesses — verify, don't assume.

> **Scoped this slice: N1 + N3 (+ positives + post-failover).** Spec §10's other
> negatives are a **validation-hardening follow-up** (Part B), for good reason:
> **N4** (assert-ownership) can't be induced cleanly client-side — spoofing
> `MQMD.UserIdentifier` needs `MQPMO_SET_IDENTITY_CONTEXT` + `+setid`, so N4 is a
> receiver-side demo entangled with the `+setall`/`PUTAUT(CTX)` research and belongs
> with it. **N5** (undeliverable/DLQ) is coupled to the per-channel-DLQ research.
> **N2** (unmapped cert → back-stop) needs a throwaway unmapped cert. All three land
> once their coupled research does.

- [ ] **Step 3: Run the validation against the live arm**

Run: `vrg-container-run -- ansible-playbook ansible/site-pcmk-authz-validate.yml` (or the lab's playbook-run wrapper on the box).
Expected: the negative probe prints `... observed=2035 expect=2035 -> PASS`; the positive trade flow returns 0; both repeat green after failover.

- [ ] **Step 4: Commit**

```bash
git add clients/authz_probe.py ansible/site-pcmk-authz-validate.yml
git commit -m "test(authz): induced-denial + positive + post-failover validation for pcmk (#74)"
```

---

### Task 6: Cold-rebuild acceptance

**Files:** none (acceptance gate).

- [ ] **Step 1: Cold rebuild** the `pcmk-ubuntu` arm from a fresh VM (per the cold-rebuild acceptance gate) and confirm it comes up **one-pass** with authorization enforced — the accounts exist on every node, the CHLAUTH maps + grants are live, and `site-pcmk-authz-validate.yml` passes without manual intervention.
- [ ] **Step 2:** Record the cold-rebuild result on the epic's `validation` task as `Outcome: SUCCESS`.

---

## Part B — Epic task register (filed as issues via `paad:alignment`)

These are the remaining epic tasks. Part A is the first implementable slice; these follow. Dependencies noted for `--blocked-by`.

| Task | Kind | Depends on | Notes |
|---|---|---|---|
| Research: `+setall` / `PUTAUT(CTX)` | research (task) | — | Verify against IBM docs whether `PUTAUT(CTX)` forces `+setid`/`+setall` and whether our `PUTAUT(DEF)` avoids it (spec §8). May confirm or adjust Task 3/4. |
| Research: per-channel vs QM-level DLQ | research (task) | — | Settle whether a dedicated per-counterparty DLQ is possible; then confirm/replace the interim `mqsvc_dlq` = `SYSTEM.DEAD.LETTER.QUEUE` grant (spec §7.1). |
| Induced-denial validation (pcmk) | **validation** | Tasks 1–5 merged | The `site-pcmk-authz-validate.yml` run (N1 + N3 + positives + post-failover) + cold rebuild → `Outcome: SUCCESS`. |
| Validation hardening: N2 + N4 + N5 | **validation** | `+setall` research (N4), per-channel-DLQ research (N5) | The spec §10 negatives deferred from the first slice: **N2** unmapped cert → back-stop (`AMQ9777`); **N4** assert-ownership (receiver-side, needs `+setid`/context — pairs with the `+setall` research); **N5** undeliverable/DLQ path (pairs with the DLQ research). |
| Fan-out to `nativeha` | task (impl) | pcmk validation | Apply `authz.yml` + `mq-authz-accounts` + the `authz.mqsc.j2` snippet to the Native HA arm; bind maps to its channel names (`NHARAPP.NHARSVC`). |
| Fan-out to `rdqm` | task (impl) | pcmk validation | Same delta for the RDQM arm (`RDQMAPP`); accounts on all RDQM nodes. |
| Confirm mqweb REST authorization posture | **validation** | — | `pymqrest`→mqweb is governed by mqweb roles, not `MCAUSER`; verify it is not wide-open admin (outside the OAM model). |

**Closing bookends (already created):** documentation `.github#75` (this spec+plan PR), closing brainstorm `.github#76` (LDAP authz + DLQ framework), documentation review `mq-resiliency-lab-for-linux#611` (site docs).

---

## Self-Review

**Spec coverage:** §3–§6 → Tasks 1–4; §5.1a cutover safety → Task 3 ordering (channels first, CHLAUTH flip after, atomic batch); §7/§7.1 grants + DLQ fork → Task 1/4 + `mqsvc_dlq` (interim, explicitly research-owned) + Part B DLQ research; §8 `+setall`/DLQ research → Part B; §9 follow-ups → Part B + bookends; §10 validation → Task 5 (N1 + N3 + positives + post-failover now; N2/N4/N5 in the Part B validation-hardening task, each coupled to its research); §6 substrate invariant + cold rebuild → Task 2 (all-node) + Task 6. Covered, with the §10 negatives split first-slice-vs-hardening per the alignment review.

**Alignment decisions folded in (2026-07-13):** (1) Task 5 expanded to N1+N3; N2/N4/N5 deferred to a Part B validation-hardening task coupled to the `+setall` and DLQ research. (2) The `mqsvc` DLQ grant is an explicit *interim* (QM-wide DLQ, outage-safe), with the per-channel-DLQ research owning the final shape. (3) The `mqmon` `+sub` on `SYSTEM.BASE.TOPIC` is flagged a to-be-narrowed placeholder (pinned against the live exporter in Task 5), not shipped as a root grant.

**Placeholders:** none — every step has concrete file paths, MQSC, YAML, `setmqaut`, and probe code. Three explicitly-flagged known-unknowns (exporter keystore path, `app_requester --once`, app-host sync path) are marked "verify on the live arm," not silently assumed.

**Type/name consistency:** `authz_accounts`/`authz_chlauth_maps`/`authz_grants`/`mqsvc_dlq` are defined in Task 1 and consumed with the same names in Tasks 2–4. Channel/queue/DN literals match the spec and the grounded role code.
