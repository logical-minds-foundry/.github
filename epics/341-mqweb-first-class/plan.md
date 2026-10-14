# MQ Web (Console + REST) first-class — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. In this org, each task below is one GitHub issue under epic `.github#341`. Drive it with `vergil:issue-implement`; operational tasks use `vergil:issue-validate`.

**Goal:** Re-enable per-node mqweb on every lab queue manager node and make the MQ Console and the
admin REST API first-class and properly secured: retire anonymous access, use IBM's three roles,
add a queue-manager-enforced operator tier, and give humans a way into the Console that follows
the live queue manager.

**Architecture:** mqweb runs locally on every QM node (in-installation `MQSeriesWeb`, local
bindings), using the existing `ansible/roles/mqweb` role with the `mqweb_enabled` gate removed.
Security lives in the role's `mqwebuser.xml.j2`:

- a basic registry with three human users mapped to IBM's three roles;
- every password in `mqwebuser.xml` encoded once, with one server AES key, following IBM's
  procedure (spec S16);
- an `mqoperator` OS account plus `SET AUTHREC` grants on every QM.

Humans reach the Console through a fixed vergil port forward into the dev VM. On the dev VM,
`mqlab console open <stack>` resolves the live instance (VIP, or a Native HA state probe) and runs
a local TCP forwarder to it.

**Tech Stack:**

- Ansible: roles and playbooks under `ansible/`, Jinja templates, MQSC.
- Python 3: the `mqlab` CLI in `src/mqlab/` (Typer), tests with `pytest`.
- IBM MQ 10.0.1.0 mqweb (WebSphere Liberty), including `setmqweb` and `securityUtility`.
- vergil-vm port forwards (`vergil.toml`).

**Spec:** `epics/341-mqweb-first-class/spec.md` in this directory. Executors read both documents.
Spec section numbers are cited as `spec §N`.

**Issue numbers.** `#<T1>` … `#<T7>` stand for the lab issues filed from this plan (epic-create
step 9). Each implementer substitutes their own issue number in commit messages and comments.

## Global Constraints

- **MQ version.** The lab pins MQ `10.0.1.0` (`lab/mq-version`). Cite IBM `SSYHRD_10.0.0` topics
  only (spec §0).
- **Follow IBM to the letter.** Every departure goes in the spec §8 deviations register, with its
  reason and risk.
- **Human logins are password = username:** `mqadmin`/`mqadmin`, `mqreadonly`/`mqreadonly`,
  `mqoperator`/`mqoperator`.
- **No machine REST identity.** The state probe authenticates as `mqreadonly`.
- **Roles:**
  - `mqadmin` → `MQWebAdmin`;
  - `mqreadonly` → `MQWebAdminRO`;
  - `mqoperator` → `MQWebUser`.
- **`mqoperator` grants, the IBM minimum [S15]:**
  - QMGR `+dsp +connect`;
  - `SYSTEM.ADMIN.TOPIC` `+sub +dsp`;
  - `SYSTEM.REST.REPLY.QUEUE` `+put +get +browse +inq`;
  - `SYSTEM.ADMIN.COMMAND.QUEUE` `+put`;
  - plus display-only object access: queues `+dsp +inq +browse`, and channels, topics and
    namelists `+dsp`.
- **mqweb defaults to keep:** HTTPS only (`httpPort -1`), CSRF on, CORS unset, LTPA at its
  defaults, org-CA `mqweb` certificate over TLS 1.3, `httpHost` `*` (deviation D1).
- **Password encoding follows IBM S16 to the letter:**
  - one AES key per mqweb server, made with `securityUtility generateAESKey` and the real lab
    secret `mqweb_aes_passphrase`;
  - the key is generated once and copied to every QM node, and included from `mqwebuser.xml`;
  - **every** password in the file, including the keystore's, is encoded with
    `securityUtility encode --encoding=aes --aesConfigFile=<key>`.

  The key and the encoded strings are stored once under `mqlab.paths.state("mqweb")`. Never
  hardcode a `build/` path.
- **Shell policy.** Use `vrg-git` / `vrg-gh` only. Commit with `vrg-commit`.
- **Validation.** `vrg-container-run -- vrg-validate` is the only gate.
- **Tests.** Run with `uv run pytest …` during development. Coverage is 100% branch for `src/`.
- **Cold-rebuild acceptance.** Every provisioning change is accepted only after a full VM cold
  rebuild proves it in one pass (T6).

## Review Focus

These are the inputs and failures most likely to hurt someone that no task's happy-path test
exercises. Each one's test is added to the task that owns the code.

1. **Re-provisioning a live lab must not restart mqweb.** A second render of `mqwebuser.xml` must
   be byte-identical. *Test:* Task 2, Step 6 renders twice and compares.
2. **Native HA with no instance reporting `running`**, for example mid-election: `console open`
   must fail loudly and name the stack. It must not forward to a replica or pick the first
   candidate. *Test:* Task 5, `test_resolve_live_raises_when_none_running`.
3. **A candidate that is unreachable or refuses TLS** (a node down during failover) is skipped,
   not fatal, as long as another candidate reports `running`. *Test:* Task 5,
   `test_resolve_live_skips_unreachable_candidate`.
4. **A new QM-bearing play that forgets `mq-authz-accounts`** would silently break `mqoperator` on
   that arm. *Test:* Task 3, `test_every_qm_play_applies_authz_accounts`.
5. **The local forwarder port is already in use** (a previous `console open` is still running):
   replace the old forwarder, which the user re-runs after a failover, rather than fail with a raw
   `EADDRINUSE`. *Test:* Task 5, `test_forwarder_replaces_previous_unit`.

## File map

| File | Change | Task |
|---|---|---|
| `docs/reference/mqweb-spike-findings.md` | Create: the spike's measured answers to spec §3 | 1 |
| `ansible/roles/mqweb/templates/mqwebuser.xml.j2` | Rewrite: 3 users, 3 roles, encoded passwords, no `EVERYONE` | 2 |
| `ansible/roles/mqweb/tasks/encode.yml` | Create: encode once per user, persist under `state("mqweb")` | 2 |
| `ansible/roles/mqweb/tasks/main.yml` | Include `encode.yml` before the registry render | 2 |
| `ansible/roles/mqweb/defaults/main.yml` | Drop `mqweb_console_anonymous`; add `mqweb_users` and the encoding | 2 |
| `ansible/group_vars/all/lab-ui.yml` | Drop `mqweb_console_anonymous` | 2 |
| `ansible/group_vars/all/admin.yml` | Drop `mqweb_admin_user` / `mqweb_admin_password` | 2 |
| `lab/topology.yaml` | Replace `mqweb_admin_password` with `mqweb_aes_passphrase` in every stack's `secrets` | 2 |
| `src/mqlab/cli.py` | Export `MQWEB_STATE_DIR` for provision (2); remove `rest` verb and the `pymqrest` PKI entity, add `console open` (5) | 2, 5 |
| `tests/test_mqweb_registry.py` | Create | 2 |
| `ansible/group_vars/all/authz.yml` | Add `mqoperator` to `authz_accounts`; add `authz_mqoperator_grants` | 3 |
| `ansible/roles/mq-pcmk-qmgr/templates/mqoperator.mqsc.j2` | Create: `SET AUTHREC` for `mqoperator` | 3 |
| `ansible/roles/mq-pcmk-qmgr/tasks/main.yml`, `ansible/site-rdqm.yml`, `ansible/site-nativeha-ubuntu.yml`, `ansible/site-nativeha.yml`, `ansible/roles/mq-qmgr/tasks/main.yml`, `ansible/site-distributed-shared.yml` | Apply `mqoperator.mqsc.j2`; add `mq-authz-accounts` where it is missing | 3 |
| `tests/test_mqweb_operator_authz.py` | Create | 3 |
| the 8 stack/DR plays carrying `role: mqweb`, and `ansible/roles/mq-qmgr/tasks/main.yml` | Remove the `when: mqweb_enabled` gates | 4 |
| `tests/test_mqweb_disabled.py` → `tests/test_mqweb_always_on.py` | Replace | 4 |
| `src/mqlab/rest.py` → `src/mqlab/mqweb_endpoints.py`; `tests/test_rest.py` → `tests/test_mqweb_endpoints.py` | Rename; drop the `rest` verb | 5 |
| `src/mqlab/mqweb_probe.py`, `tests/test_mqweb_probe.py` | Create: state probe and live-instance resolution | 5 |
| `src/mqlab/console.py`, `tests/test_console.py` | Create: forwarder unit, URL and login printout | 5 |
| `vergil.toml` | Add `9443\|127.0.0.1:19443` to each lab profile's `port_forwards` | 5 |
| `pyproject.toml`, `uv.lock` | Remove `pymqrest` | 5 |
| `tests/test_cli_pki.py` | Drop `pymqrest` from `_FIXED_CNS` | 5 |
| `docs/reference/mqweb-endpoint-verification.md` | Rewrite for roles and discovery | 7 |

---

### Task 1: Spike — per-node mqweb cost, replica behaviour, REST version (gate)

**Kind:** implementation that ships a findings document; no product code. **Issue repo:** lab.
**Blocked by:** `.github#342`.

**Files:**

- Create: `docs/reference/mqweb-spike-findings.md`
- Modify: the spec (`.github` repo, `epics/341-mqweb-first-class/spec.md`) — §3 answers and §8,
  in a follow-up `.github` PR under `#342`'s branch or a new docs task.

**Interfaces:**

- Produces: the REST version string the probe uses (`v2` or `v3`), recorded in the findings
  document as `REST_API_VERSION=<v>`, and the measured per-node cold-start and memory figures.

- [ ] **Step 1: Bring up the primary arm with mqweb on.** As the human operator, from a cold
  rebuild on macOS/arm64:

  ```bash
  mqlab bootstrap nativeha-rhel-crr -e mqweb_enabled=true
  ```

  If `mqlab bootstrap` has no `-e` pass-through, set `mqweb_enabled: true` in a scratch branch's
  `ansible/group_vars/all/admin.yml` instead. Record which one you used.
  Expected: the bootstrap finishes; `systemctl is-active mqweb.service` → `active` on all six
  nodes (`nha-rhel-crr-a1..a3`, `-b1..b3`).

- [ ] **Step 2: Measure cold start and memory per node.**

  ```bash
  for h in nha-rhel-crr-a1 nha-rhel-crr-a2 nha-rhel-crr-a3 nha-rhel-crr-b1 nha-rhel-crr-b2 nha-rhel-crr-b3; do
    ssh "$h" "sudo systemctl restart mqweb.service; s=\$(date +%s); \
      until curl -sk -o /dev/null https://127.0.0.1:9443/ibmmq/console/; do sleep 2; done; \
      echo $h ready_s=\$(( \$(date +%s) - s )); \
      ps -o rss= -C java | awk '{s+=\$1} END {print \"rss_kb=\" s}'"
  done
  ```

  Record each node's `ready_s` and `rss_kb` in a table in the findings document. Repeat Steps 1–2
  on the cloud x86 host.

- [ ] **Step 3: Replica and CRR behaviour.** Using the current lab credentials
  (`MQWEB_ADMIN_USER` / `MQWEB_ADMIN_PASSWORD` from `build/state/mqweb.env`; Task 2 retires
  them), on each of the 6 nodes:

  ```bash
  curl -sk -u "$U:$P" https://<node-data-ip>:9443/ibmmq/rest/v2/admin/qmgr/NHARCAPP | python3 -m json.tool
  curl -sk -u "$U:$P" https://<node-data-ip>:9443/ibmmq/rest/v3/admin/qmgr/NHARCAPP | python3 -m json.tool
  ```

  Record the `state` value (`running` / `replica` / other) per node and per API version. Open the
  Console on the active node and on one replica. Screenshot both into `build/temp/`. Describe what
  each shows, including whether the Native HA / replication status view renders on the active
  node.

- [ ] **Step 4: Decide the REST version.** If `v3` answers on 10.0.1, use `v3`. Otherwise use
  `v2`. Write `REST_API_VERSION=v3` (or `v2`) into the findings document.

- [ ] **Step 5: Gate.** Write a "Verdict" section with one of:
  - **PASS:** every node ready within `TimeoutStartSec`, the bootstrap one-pass, and replica
    states as expected.
  - **PAUSE:** anything else. Name the surprise.

  On PAUSE, stop the epic and raise it with the human (spec §3). Do not continue to Task 2.

- [ ] **Step 6: Commit and hand off.**

  ```bash
  vrg-git add docs/reference/mqweb-spike-findings.md
  vrg-commit --type docs --scope reference --message "record the mqweb per-node spike findings (#<T1>)"
  vrg-container-run -- vrg-validate
  vrg-pr-workflow report-ready
  ```

  Then amend spec §3 (answers) and §8 (any new deviation) in `.github`.

---

### Task 2: Registry and roles — 3 users, IBM roles, encode-once passwords, no anonymous

**Issue repo:** lab. **Blocked by:** T1 (PASS).

**Files:**

- Modify: `ansible/roles/mqweb/templates/mqwebuser.xml.j2`
- Create: `ansible/roles/mqweb/tasks/encode.yml`
- Modify: `ansible/roles/mqweb/tasks/main.yml` (include `encode.yml` before the
  `mqweb basic registry` task)
- Modify: `ansible/roles/mqweb/defaults/main.yml`, `ansible/group_vars/all/lab-ui.yml`,
  `ansible/group_vars/all/admin.yml`, `lab/topology.yaml` (`secrets:` lines 348, 365, 381, 405)
- Modify: `src/mqlab/cli.py` (the provision secret block near line 2737)
- Test: `tests/test_mqweb_registry.py`

**Interfaces:**

- Produces:
  - Ansible fact `mqweb_encoded_passwords: dict[str, str]`, keyed by user name plus the key
    `keystore`.
  - Var `mqweb_users: list[{name, role}]`.
  - Env vars `MQWEB_STATE_DIR` (set by `mqlab` for every provision phase) and
    `MQWEB_AES_PASSPHRASE` (from the `mqweb_aes_passphrase` stack secret).
  - In the controller state dir (`<state("mqweb")>`, all mode 0600):
    - `aes-key.xml`, the AES key config;
    - `<user>.enc` and `keystore.enc`, one encoded string each;
    - `keystore.sha256`, the hash of the plaintext keystore password the `.enc` was made from.
  - On every QM node: `{{ mqweb_server_dir }}/aes-key.xml` (owner mqm, 0600).

- [ ] **Step 1: Read IBM's procedure.** It's already confirmed during alignment and cached at
  `build/cache/refs/ibm-docs/ibm-mq/10.0.x/mcras-securing-your-mqweb-server-configuration-passwords-by-using-securityutility-command/content.txt`
  (spec S16). Cite it in the PR description. In summary:
  - one AES key per mqweb server, created with
    `securityUtility generateAESKey --createConfigFile=<file> --key=<strongPassphrase>`;
  - the key file is included from `mqwebuser.xml` with `<include location="<file>"/>`;
  - every password in the file is encoded with
    `securityUtility encode --encoding=aes --aesConfigFile=<file> <password>`.

  Confirm the tool path on a node running MQ 10.0.1 with `ls /opt/mqm/web/bin/securityUtility`.
  S16 gives it as `MQ_INSTALLATION_PATH/web/bin`.

- [ ] **Step 2: Write the failing registry test.** Create `tests/test_mqweb_registry.py`:

  ```python
  """mqwebuser.xml registry and roles (epic .github#341 spec §2.2)."""

  from __future__ import annotations

  from pathlib import Path

  import jinja2
  import yaml

  REPO_ROOT = Path(__file__).resolve().parent.parent
  ROLE = REPO_ROOT / "ansible" / "roles" / "mqweb"
  TEMPLATE = ROLE / "templates" / "mqwebuser.xml.j2"
  DEFAULTS = ROLE / "defaults" / "main.yml"
  ENCODED = {"mqadmin": "{aes}AAA", "mqreadonly": "{aes}BBB", "mqoperator": "{aes}CCC"}


  def _render() -> str:
      env = jinja2.Environment(  # noqa: S701 - XML config, not HTML
          loader=jinja2.FileSystemLoader(str(TEMPLATE.parent)),
          undefined=jinja2.StrictUndefined,
          trim_blocks=True,
      )
      defaults = yaml.safe_load(DEFAULTS.read_text(encoding="utf-8"))
      ctx = {
          **defaults,
          "mqweb_encoded_passwords": {**ENCODED, "keystore": "{aes}KEY"},
          "mqweb_server_dir": "/var/mqm/web/installations/Installation1/servers/mqweb",
      }
      return env.get_template(TEMPLATE.name).render(**ctx)


  def test_includes_the_aes_key_file() -> None:
      # IBM S16: the key file is included from mqwebuser.xml.
      assert '<include location="/var/mqm/web/installations/Installation1/servers/mqweb/aes-key.xml"/>' in _render()


  def test_keystore_password_is_encoded_not_plaintext() -> None:
      # IBM S16 applies to EVERY password in the file, including the keystore's.
      assert 'type="PKCS12" password="{aes}KEY"' in _render()


  def test_no_anonymous_binding() -> None:
      out = _render()
      assert "EVERYONE" not in out
      assert "special-subject" not in out


  def test_each_user_has_its_ibm_role_on_console_and_rest() -> None:
      out = _render()
      for app in ("com.ibm.mq.console", "com.ibm.mq.rest"):
          block = out.split(f'id="{app}"', 1)[1].split("</enterpriseApplication>", 1)[0]
          assert '<security-role name="MQWebAdmin">' in block
          assert '<user name="mqadmin" realm="defaultRealm"/>' in block
          assert '<security-role name="MQWebAdminRO">' in block
          assert '<user name="mqreadonly" realm="defaultRealm"/>' in block
          assert '<security-role name="MQWebUser">' in block
          assert '<user name="mqoperator" realm="defaultRealm"/>' in block


  def test_registry_carries_only_encoded_passwords() -> None:
      out = _render()
      for user, enc in ENCODED.items():
          assert f'<user name="{user}" password="{enc}"/>' in out
      for user in ENCODED:
          assert f'password="{user}"' not in out


  def test_render_is_byte_identical_across_runs() -> None:
      # Review Focus 1: a re-provision must not change the file (it would restart mqweb).
      assert _render() == _render()


  def test_role_defaults_drop_console_anonymous() -> None:
      defaults = yaml.safe_load(DEFAULTS.read_text(encoding="utf-8"))
      assert "mqweb_console_anonymous" not in defaults
      assert [u["name"] for u in defaults["mqweb_users"]] == ["mqadmin", "mqreadonly", "mqoperator"]
      assert all(len(u["name"]) <= 12 for u in defaults["mqweb_users"])
  ```

- [ ] **Step 3: Run it and confirm it fails.**
  Run: `uv run pytest tests/test_mqweb_registry.py -v`
  Expected: FAIL. `EVERYONE` is present, and `mqweb_users` is missing (`KeyError` or
  `UndefinedError`).

- [ ] **Step 4: Implement the defaults and the template.**
  In `ansible/roles/mqweb/defaults/main.yml`, delete the `mqweb_console_anonymous` block and add:

  ```yaml
  # Human sample users, one per IBM role (epic .github#341 spec §2.2; IBM S8). Lab credential
  # convention: each human login's password equals its username. Encoded once with
  # securityUtility (tasks/encode.yml) and persisted, so a re-render is byte-identical.
  # LAB ONLY: never carry password = username to a real deployment (spec §8 D2).
  mqweb_users:
    - { name: mqadmin,    role: MQWebAdmin }
    - { name: mqreadonly, role: MQWebAdminRO }
    - { name: mqoperator, role: MQWebUser }
  # IBM S16: one AES key per mqweb server; every mqwebuser.xml password is encoded with it.
  mqweb_server_dir: /var/mqm/web/installations/Installation1/servers/mqweb
  mqweb_security_utility: /opt/mqm/web/bin/securityUtility
  # Real lab secret (lab-secret.sh mqweb_aes_passphrase), injected by mqlab; never committed.
  mqweb_aes_passphrase: "{{ lookup('env', 'MQWEB_AES_PASSPHRASE') }}"
  # Controller-side store for aes-key.xml and the encoded strings (set by mqlab: state("mqweb")).
  mqweb_state_dir: "{{ lookup('env', 'MQWEB_STATE_DIR') }}"
  ```

  Replace `ansible/roles/mqweb/templates/mqwebuser.xml.j2` with:

  ```jinja
  <?xml version="1.0" encoding="UTF-8"?>
  <!-- mqweb security (epic .github#341 spec §2.2): IBM's three roles (S8), one sample user per
       role in a basic registry (S10). No anonymous access. Every password in this file is
       AES-encoded with the server's one key, which is included below (S16). Each was encoded
       once and persisted, so this render is stable across re-provisions. -->
  <server>
    <featureManager>
      <feature>appSecurity-2.0</feature>
      <feature>basicAuthenticationMQ-1.0</feature>
    </featureManager>
    <include location="{{ mqweb_server_dir }}/aes-key.xml"/>
  {% for app in ['com.ibm.mq.console', 'com.ibm.mq.rest'] %}
    <enterpriseApplication id="{{ app }}">
      <application-bnd>
  {% for u in mqweb_users %}
        <security-role name="{{ u.role }}">
          <user name="{{ u.name }}" realm="defaultRealm"/>
        </security-role>
  {% endfor %}
      </application-bnd>
    </enterpriseApplication>
  {% endfor %}
    <basicRegistry id="basic" realm="defaultRealm">
  {% for u in mqweb_users %}
      <user name="{{ u.name }}" password="{{ mqweb_encoded_passwords[u.name] }}"/>
  {% endfor %}
    </basicRegistry>
    <variable name="httpHost" value="*"/>
  {% if mqweb_tls | bool %}
    <keyStore id="mqwebKeyStore" location="{{ mqweb_security_dir }}/key.p12" type="PKCS12" password="{{ mqweb_encoded_passwords['keystore'] }}"/>
    <ssl id="mqwebSSL" keyStoreRef="mqwebKeyStore" serverKeyAlias="mqweb" sslProtocol="TLSv1.3"/>
    <sslDefault sslRef="mqwebSSL"/>
  {% else %}
    <sslDefault sslRef="mqDefaultSSLConfig"/>
  {% endif %}
  {% if mqweb_json_logging | default(true) | bool %}
    <logging messageFormat="json" messageSource="message,ffdc"/>
  {% endif %}
  </server>
  ```

  **Keystore password.** `mqweb_keyrpwd` (the plaintext PKCS#12 password from `pki-distribute`) is
  no longer put in the file. It is encoded like the user passwords (Step 6), and re-encoded only
  when its SHA-256 changes, for example after a certificate rotation.

- [ ] **Step 5: Run the test and confirm it passes.**
  Run: `uv run pytest tests/test_mqweb_registry.py -v`
  Expected: PASS (7 tests).

- [ ] **Step 6: Encode-once tasks, IBM S16 to the letter.** Create
  `ansible/roles/mqweb/tasks/encode.yml`:

  ```yaml
  # IBM S16 (MQ 10.0.x "Securing your mqweb server configuration passwords by using the
  # securityUtility command"): ONE AES key per mqweb server, generated with generateAESKey,
  # included from mqwebuser.xml, and used to encode EVERY password in the file. The key is made
  # once and copied to every QM node (an encoded string must decode wherever the QM runs). Each
  # password is encoded once and persisted on the controller under state("mqweb"), so every
  # render inserts the SAME strings and mqwebuser.xml stays byte-identical (spec §2.2, Review
  # Focus 1). Fail loud on missing inputs.
  - name: require the controller state dir and the AES passphrase (exported by mqlab)
    ansible.builtin.assert:
      that:
        - mqweb_state_dir | length > 0
        - mqweb_aes_passphrase | length > 0
      fail_msg: >-
        MQWEB_STATE_DIR / MQWEB_AES_PASSPHRASE unset — run provisioning through
        `mqlab bootstrap <stack>`, which exports both (spec §2.2).
    no_log: true

  - name: ensure the controller mqweb state dir exists
    ansible.builtin.file:
      path: "{{ mqweb_state_dir }}"
      state: directory
      mode: "0700"
    delegate_to: localhost
    run_once: true
    become: false

  - name: is the AES key already generated
    ansible.builtin.stat:
      path: "{{ mqweb_state_dir }}/aes-key.xml"
    register: mqweb_key_stat
    delegate_to: localhost
    run_once: true
    become: false

  - name: generate the server's AES key ONCE (securityUtility generateAESKey, IBM S16)
    ansible.builtin.command:
      argv:
        - "{{ mqweb_security_utility }}"
        - generateAESKey
        - "--createConfigFile={{ mqweb_server_dir }}/aes-key.xml"
        - "--key={{ mqweb_aes_passphrase }}"
      creates: "{{ mqweb_server_dir }}/aes-key.xml"
    when: not mqweb_key_stat.stat.exists
    run_once: true
    become: true
    become_user: mqm
    no_log: true

  - name: read the newly generated key back
    ansible.builtin.slurp:
      src: "{{ mqweb_server_dir }}/aes-key.xml"
    register: mqweb_key_new
    when: not mqweb_key_stat.stat.exists
    run_once: true
    become: true

  - name: persist the key on the controller (0600)
    ansible.builtin.copy:
      content: "{{ mqweb_key_new.content | b64decode }}"
      dest: "{{ mqweb_state_dir }}/aes-key.xml"
      mode: "0600"
    when: not mqweb_key_stat.stat.exists
    delegate_to: localhost
    run_once: true
    become: false
    no_log: true

  - name: place the same key on THIS node (mqm, 0600)
    ansible.builtin.copy:
      src: "{{ mqweb_state_dir }}/aes-key.xml"
      dest: "{{ mqweb_server_dir }}/aes-key.xml"
      owner: mqm
      group: mqm
      mode: "0600"
    become: true
    no_log: true

  - name: which encoded user passwords already exist
    ansible.builtin.stat:
      path: "{{ mqweb_state_dir }}/{{ item.name }}.enc"
    loop: "{{ mqweb_users }}"
    loop_control: { label: "{{ item.name }}" }
    register: mqweb_enc_stat
    delegate_to: localhost
    run_once: true
    become: false

  - name: encode missing user passwords (password = username, lab convention; IBM S16)
    ansible.builtin.command:
      argv:
        - "{{ mqweb_security_utility }}"
        - encode
        - --encoding=aes
        - "--aesConfigFile={{ mqweb_server_dir }}/aes-key.xml"
        - "{{ item.item.name }}"
    loop: "{{ mqweb_enc_stat.results }}"
    loop_control: { label: "{{ item.item.name }}" }
    when: not item.stat.exists
    register: mqweb_enc_new
    changed_when: true
    run_once: true
    become: true
    become_user: mqm
    no_log: true

  - name: persist newly encoded user passwords on the controller (0600)
    ansible.builtin.copy:
      content: "{{ item.stdout | trim }}\n"
      dest: "{{ mqweb_state_dir }}/{{ item.item.item.name }}.enc"
      mode: "0600"
    loop: "{{ mqweb_enc_new.results | selectattr('stdout', 'defined') | list }}"
    loop_control: { label: "{{ item.item.item.name }}" }
    delegate_to: localhost
    run_once: true
    become: false
    no_log: true

  # The keystore password changes only when pki-distribute re-issues the store. Re-encode
  # exactly then, by comparing the SHA-256 of the current plaintext with the one recorded.
  - name: compare the keystore password with the one last encoded
    ansible.builtin.set_fact:
      mqweb_keystore_sha: "{{ mqweb_keyrpwd | hash('sha256') }}"
      mqweb_keystore_recorded: >-
        {{ lookup('ansible.builtin.file', mqweb_state_dir ~ '/keystore.sha256', errors='ignore')
           | default('', true) | trim }}
    no_log: true
    when: mqweb_tls | bool

  - name: encode the keystore password (IBM S16 — every password in the file)
    ansible.builtin.command:
      argv:
        - "{{ mqweb_security_utility }}"
        - encode
        - --encoding=aes
        - "--aesConfigFile={{ mqweb_server_dir }}/aes-key.xml"
        - "{{ mqweb_keyrpwd }}"
    register: mqweb_keystore_enc
    changed_when: true
    run_once: true
    become: true
    become_user: mqm
    no_log: true
    when: mqweb_tls | bool and mqweb_keystore_sha != mqweb_keystore_recorded

  - name: persist the encoded keystore password and its source hash (0600)
    ansible.builtin.copy:
      content: "{{ item.content }}\n"
      dest: "{{ mqweb_state_dir }}/{{ item.name }}"
      mode: "0600"
    loop:
      - { name: keystore.enc,    content: "{{ mqweb_keystore_enc.stdout | default('') | trim }}" }
      - { name: keystore.sha256, content: "{{ mqweb_keystore_sha | default('') }}" }
    loop_control: { label: "{{ item.name }}" }
    delegate_to: localhost
    run_once: true
    become: false
    no_log: true
    when: mqweb_keystore_enc is changed

  - name: load the encoded passwords for the registry render
    ansible.builtin.set_fact:
      mqweb_encoded_passwords: >-
        {{ mqweb_encoded_passwords | default({}) | combine({item:
           lookup('ansible.builtin.file', mqweb_state_dir ~ '/' ~ item ~ '.enc')}) }}
    loop: "{{ (mqweb_users | map(attribute='name') | list) + (['keystore'] if mqweb_tls | bool else []) }}"
    no_log: true
  ```

  A note on the plays' `serial: 1`: `run_once` runs once per serial batch. The `stat` and `when`
  guards make every later batch a no-op, so the key is generated once and each password is
  encoded once per lab.

  The TLS tasks that set `mqweb_keyrpwd` must run before this include. They already run before
  the registry task, which this include precedes.

  In `ansible/roles/mqweb/tasks/main.yml`, insert this before the `mqweb basic registry` task:

  ```yaml
  - name: encode the registry passwords once (stable render, #<T2>)
    ansible.builtin.include_tasks: encode.yml
  ```

- [ ] **Step 7: Export `MQWEB_STATE_DIR` from the sequencer.** In `src/mqlab/cli.py`, next to the
  existing secret export (`for secret in stack.secrets: os.environ[...]`):

  ```python
  if any(phase.name == "provision" for phase in selected):
      for secret in stack.secrets:
          os.environ[secret.upper()] = _source_secret(deps, secret)
      # mqweb encode-once store (epic .github#341 spec §2.2): the role persists each
      # securityUtility-encoded password here so a re-provision renders identically.
      os.environ["MQWEB_STATE_DIR"] = str(state("mqweb"))
  ```

  Import `state` from `mqlab.paths` if it isn't already imported. Add a test next to the existing
  secret-sourcing test in `tests/test_cli_bootstrap.py` (the one asserting
  `sourced == [...]`, around line 392). It asserts that after a provision-phase bootstrap,
  `os.environ["MQWEB_STATE_DIR"] == str(paths.state("mqweb"))`. Use the file's existing `monkeypatch`
  fixtures, and `monkeypatch.delenv("MQWEB_STATE_DIR", raising=False)` first.

- [ ] **Step 8: Remove the old admin plumbing.**
  - In `ansible/group_vars/all/admin.yml`, delete `mqweb_admin_user` and `mqweb_admin_password`
    and their comment.
  - In `ansible/group_vars/all/lab-ui.yml`, delete the `mqweb_console_anonymous` line, and reword
    the header comment to say the MQ Console now requires a login (spec §2.2).
  - In `lab/topology.yaml`, **replace** `mqweb_admin_password` with `mqweb_aes_passphrase` in
    each stack's `secrets:`. `pcmk-ubuntu` becomes `[pcmk_hacluster_password, mqweb_aes_passphrase]`.
    The existing sequencer then exports `MQWEB_AES_PASSPHRASE` from `lab-secret.sh` with no new
    code.
  - svc-sim needs nothing extra. Every stack's provision playbook imports
    `site-distributed-shared.yml` (`site-pcmk.yml`, `site-rdqm.yml`, `site-nativeha.yml`,
    `site-nativeha-ubuntu.yml`; see `src/mqlab/cli.py` `_stack_mq_boxes` docstring), so svc-sim's
    mqweb is provisioned inside a stack run that has already exported both variables. Because
    `aes-key.xml` and the `.enc` files are shared lab-wide under `state("mqweb")`, svc-sim gets
    the same key and the same encoded strings.
  - Run `grep -rn "mqweb_admin\|MQWEB_ADMIN\|mqweb_console_anonymous\|mqweb.env" src ansible lab docs/site`.
    Expect matches only in historical docs (`docs/plans`, `docs/specs`) and in test fixtures that
    build their own topology strings. Update `docs/development/lab-bringup-capture.md`'s
    `mqweb.env` mention to point at `mqlab console open` (Task 5).

- [ ] **Step 9: Run the whole suite.**
  Run: `uv run pytest -q`
  Expected: PASS. Fixture topologies in `tests/test_stacks.py`, `test_phases.py` and others still
  declare `mqweb_admin_password` on purpose. They test the generic secret mechanism, so leave them.

- [ ] **Step 10: Validate and commit.**

  ```bash
  vrg-container-run -- vrg-validate
  vrg-git add -A ansible/roles/mqweb ansible/group_vars/all lab/topology.yaml src/mqlab/cli.py tests docs/development
  vrg-commit --type feat --scope mqweb --message "IBM-role registry with encode-once passwords; retire anonymous console (#<T2>)"
  ```

---

### Task 3: `mqoperator` — OS account on every QM node, IBM-minimum grants on every QM

**Issue repo:** lab. **Blocked by:** T1 (PASS).

**Files:**

- Modify: `ansible/group_vars/all/authz.yml`
- Create: `ansible/roles/mq-pcmk-qmgr/templates/mqoperator.mqsc.j2`
- Modify (apply the template):
  - `ansible/roles/mq-pcmk-qmgr/tasks/main.yml` (pcmk, next to "apply the authorization MQSC",
    around line 134);
  - `ansible/site-rdqm.yml` (next to the authz apply, around line 387);
  - `ansible/site-nativeha-ubuntu.yml` (the `nha_steps` list around line 155);
  - `ansible/site-nativeha.yml` (a new `nativeha-on-active` step after the our-side MQSC);
  - `ansible/roles/mq-qmgr/tasks/main.yml` (svc-sim, after "define + start listener").
- Modify (add the accounts role):
  - `ansible/site-nativeha.yml`: a play `hosts: nha_rhel_crr_a:nha_rhel_crr_b`,
    `roles: [mq-authz-accounts]`;
  - `ansible/site-distributed-shared.yml`: add `mq-authz-accounts` to the svc-sim play's roles
    (line 32: `roles: [mq-authz-accounts, mq-qmgr]`).
- Test: `tests/test_mqweb_operator_authz.py`

**Interfaces:**

- Consumes: the `mqoperator` user name from Task 2's `mqweb_users`. The name must match exactly,
  in the same case (IBM S10).
- Produces: OS account `mqoperator` (group `mqoperator`) on every QM node; group var
  `authz_mqoperator_grants: list[{objtype, profile?, authadd}]`.

- [ ] **Step 1: Write the failing tests.** Create `tests/test_mqweb_operator_authz.py`:

  ```python
  """mqoperator (MQWebUser) account and IBM-minimum grants (epic .github#341 spec §2.2, S15)."""

  from __future__ import annotations

  from pathlib import Path

  import jinja2
  import yaml

  REPO_ROOT = Path(__file__).resolve().parent.parent
  ANSIBLE = REPO_ROOT / "ansible"
  AUTHZ = ANSIBLE / "group_vars" / "all" / "authz.yml"
  TEMPLATE = ANSIBLE / "roles" / "mq-pcmk-qmgr" / "templates" / "mqoperator.mqsc.j2"
  TOPOLOGY = REPO_ROOT / "lab" / "topology.yaml"


  def _authz() -> dict:
      return yaml.safe_load(AUTHZ.read_text(encoding="utf-8"))


  def test_mqoperator_account_declared() -> None:
      assert {"name": "mqoperator", "group": "mqoperator"} in _authz()["authz_accounts"]


  def test_grants_cover_ibm_minimum_for_mqwebuser() -> None:
      grants = {(g["objtype"], g.get("profile", "")): g["authadd"] for g in _authz()["authz_mqoperator_grants"]}
      assert grants[("QMGR", "")] == "CONNECT, DSP"
      assert grants[("TOPIC", "SYSTEM.ADMIN.TOPIC")] == "SUB, DSP"
      assert grants[("QUEUE", "SYSTEM.REST.REPLY.QUEUE")] == "PUT, GET, BROWSE, INQ"
      assert grants[("QUEUE", "SYSTEM.ADMIN.COMMAND.QUEUE")] == "PUT"
      assert grants[("QUEUE", "**")] == "DSP, INQ, BROWSE"


  def test_grants_are_display_only_beyond_ibm_minimum() -> None:
      forbidden = {"CHG", "CLR", "DLT", "CRT", "SET", "SETALL", "SETID", "ALTUSR", "CTRL", "CTRLX", "ALL", "ALLADM", "ALLMQI"}
      for g in _authz()["authz_mqoperator_grants"]:
          auths = {a.strip() for a in g["authadd"].split(",")}
          assert not auths & forbidden, g


  def test_template_renders_one_set_authrec_per_grant() -> None:
      env = jinja2.Environment(undefined=jinja2.StrictUndefined, trim_blocks=True)  # noqa: S701
      out = env.from_string(TEMPLATE.read_text(encoding="utf-8")).render(
          qm_name="QM1", authz_mqoperator_grants=_authz()["authz_mqoperator_grants"]
      )
      lines = [ln for ln in out.splitlines() if ln.startswith("SET AUTHREC")]
      assert len(lines) == len(_authz()["authz_mqoperator_grants"])
      assert all("GROUP('mqoperator')" in ln for ln in lines)
      assert "SET AUTHREC OBJTYPE(QMGR) GROUP('mqoperator') AUTHADD(CONNECT, DSP)" in out


  def _plays_roles(path: Path) -> set[str]:
      roles: set[str] = set()
      for play in yaml.safe_load(path.read_text(encoding="utf-8")) or []:
          if isinstance(play, dict):
              for r in play.get("roles", []) or []:
                  roles.add(r if isinstance(r, str) else r.get("role", ""))
      return roles


  def test_every_qm_play_applies_authz_accounts() -> None:
      # Review Focus 4: every stack's provision playbook, and the svc-sim play, apply
      # mq-authz-accounts, so a new arm cannot silently lack the mqoperator OS account.
      topo = yaml.safe_load(TOPOLOGY.read_text(encoding="utf-8"))
      playbooks = {REPO_ROOT / s["provision"] for s in topo["stacks"].values()}
      playbooks.add(ANSIBLE / "site-distributed-shared.yml")
      for pb in sorted(playbooks):
          assert "mq-authz-accounts" in _plays_roles(pb), f"{pb.name} lacks mq-authz-accounts"


  def test_every_qm_play_applies_mqoperator_mqsc() -> None:
      files = [
          ANSIBLE / "roles" / "mq-pcmk-qmgr" / "tasks" / "main.yml",
          ANSIBLE / "site-rdqm.yml",
          ANSIBLE / "site-nativeha-ubuntu.yml",
          ANSIBLE / "site-nativeha.yml",
          ANSIBLE / "roles" / "mq-qmgr" / "tasks" / "main.yml",
      ]
      for f in files:
          assert "mqoperator.mqsc.j2" in f.read_text(encoding="utf-8"), f.name
  ```

- [ ] **Step 2: Run them and confirm they fail.**
  Run: `uv run pytest tests/test_mqweb_operator_authz.py -v`
  Expected: FAIL. There is no `mqoperator` account, no `authz_mqoperator_grants`, no template,
  and `site-nativeha.yml` lacks `mq-authz-accounts`.

- [ ] **Step 3: Add the data.** In `ansible/group_vars/all/authz.yml`, add
  `- { name: mqoperator, group: mqoperator }` to `authz_accounts`, and append:

  ```yaml
  # mqoperator — the MQ Console/REST MQWebUser sample login (epic .github#341 spec §2.2). Under
  # MQWebUser, mqweb authorizes every request AS THIS USER (IBM S8), so these OAM grants are the
  # tier's real boundary: a write is refused by the QUEUE MANAGER even if mqweb is misconfigured.
  # The first four lines are IBM's documented MINIMUM for MQWebUser (S15, verbatim); the last is
  # display-only object access. +put on the command queue is safe: the command server authorizes
  # each command as the message's user, so an ALTER from mqoperator is refused by OAM.
  authz_mqoperator_grants:
    - { objtype: QMGR,  authadd: "CONNECT, DSP" }
    - { objtype: TOPIC, profile: "SYSTEM.ADMIN.TOPIC",         authadd: "SUB, DSP" }
    - { objtype: QUEUE, profile: "SYSTEM.REST.REPLY.QUEUE",    authadd: "PUT, GET, BROWSE, INQ" }
    - { objtype: QUEUE, profile: "SYSTEM.ADMIN.COMMAND.QUEUE", authadd: "PUT" }
    - { objtype: QUEUE, profile: "**",                         authadd: "DSP, INQ, BROWSE" }
    - { objtype: CHANNEL,  profile: "**", authadd: "DSP" }
    - { objtype: TOPIC,    profile: "**", authadd: "DSP" }
    - { objtype: NAMELIST, profile: "**", authadd: "DSP" }
  ```

  IBM documents no separate local-mode minimum; this was checked during alignment and is
  recorded in spec §2.2. S15's list is applied as written, and Task 7 proves it on the live lab.

- [ ] **Step 4: Create the template** `ansible/roles/mq-pcmk-qmgr/templates/mqoperator.mqsc.j2`:

  ```jinja
  * mqoperator (MQWebUser) grants for {{ qm_name }} — epic .github#341 spec §2.2 / IBM S15.
  * Arm-agnostic; every QM-bearing play applies it once on the running QM (HA/DR replicate it).
  {% for g in authz_mqoperator_grants %}
  {% if g.objtype == 'QMGR' %}
  SET AUTHREC OBJTYPE(QMGR) GROUP('mqoperator') AUTHADD({{ g.authadd }})
  {% else %}
  SET AUTHREC PROFILE('{{ g.profile }}') OBJTYPE({{ g.objtype }}) GROUP('mqoperator') AUTHADD({{ g.authadd }})
  {% endif %}
  {% endfor %}
  ```

  (`SET AUTHREC` takes effect immediately, so no `REFRESH SECURITY` is needed.)

- [ ] **Step 5: Wire the apply into each arm.**
  - **pcmk** (`roles/mq-pcmk-qmgr/tasks/main.yml`): add a render task alongside "render the
    authorization MQSC" (`src: mqoperator.mqsc.j2`, `dest: /var/mqm/mqoperator.mqsc`, same
    owner, mode and `run_once`). In "apply the authorization MQSC", add
    `su mqm -c '/opt/mqm/bin/runmqsc {{ qm_name }} < /var/mqm/mqoperator.mqsc'` after the
    `authz.mqsc` line, so it runs while the QM is up.
  - **rdqm** (`site-rdqm.yml`): duplicate the render-and-apply task pair for
    `mqoperator.mqsc.j2` → `/var/mqm/mqoperator.mqsc`. Use the same `when: rdqm_is_active`, the
    same `failed_when` / `until` guard, and the same `src` path prefix
    (`{{ playbook_dir }}/roles/mq-pcmk-qmgr/templates/`).
  - **nativeha-ubuntu** (`site-nativeha-ubuntu.yml`): append to the `nha_steps` expression, before
    `REFRESH SECURITY TYPE(AUTHSERV)`:

    ```yaml
    + [{'label': 'runmqsc mqoperator grants',
        'cmd': '/opt/mqm/bin/runmqsc ' ~ qm_name,
        'ok_rcs': [0, 10],
        'stdin': lookup('ansible.builtin.template',
                        playbook_dir ~ '/roles/mq-pcmk-qmgr/templates/mqoperator.mqsc.j2')}]
    ```

  - **nativeha-rhel-crr** (`site-nativeha.yml`): add a play right after `import_playbook:
    _nativeha-cluster-ha.yml`:

    ```yaml
    # mqoperator OS account on EVERY instance, both sites (epic .github#341 spec §2.2): MQWebUser
    # maps to an identical OS user ID (IBM S10); it must exist wherever the QM can run.
    - hosts: nha_rhel_crr_a:nha_rhel_crr_b
      become: true
      roles: [mq-authz-accounts]
    ```

    After the "event monitoring" task in the `hosts: nha_rhel_crr_a` play, add:

    ```yaml
    - name: mqoperator grants on the active instance (once; raft-replicated) (#<T3>)
      ansible.builtin.include_tasks: tasks/nativeha-on-active.yml
      vars:
        nha_group: nha_rhel_crr_a
        nha_qm: "{{ qm_name }}"
        nha_label: mqoperator grants
        nha_steps:
          - label: runmqsc mqoperator grants
            cmd: /opt/mqm/bin/runmqsc {{ qm_name }}
            ok_rcs: [0, 10]
            stdin: "{{ lookup('ansible.builtin.template', playbook_dir ~ '/roles/mq-pcmk-qmgr/templates/mqoperator.mqsc.j2') }}"
    ```

  - **svc-sim** (`roles/mq-qmgr/tasks/main.yml`): after "define + start listener via MQSC", add:

    ```yaml
    - name: mqoperator grants (MQWebUser, epic .github#341 spec §2.2)
      ansible.builtin.shell: |
        set -o pipefail
        printf '%s\n' "{{ lookup('ansible.builtin.template', role_path ~ '/../mq-pcmk-qmgr/templates/mqoperator.mqsc.j2', template_vars={'qm_name': qmgr_name}) }}" \
          | /opt/mqm/bin/runmqsc {{ qmgr_name }}
      args:
        executable: /bin/bash
      become: true
      become_user: mqm
      register: mqop
      changed_when: true
      failed_when: mqop.rc not in [0, 10]
    ```

    In `site-distributed-shared.yml` line 32, change `roles: [mq-qmgr]` to
    `roles: [mq-authz-accounts, mq-qmgr]`.

- [ ] **Step 6: Run the tests.**
  Run: `uv run pytest tests/test_mqweb_operator_authz.py tests/test_templates_render.py -v`
  Expected: PASS.

- [ ] **Step 7: Validate and commit.**

  ```bash
  vrg-container-run -- vrg-validate
  vrg-git add ansible tests/test_mqweb_operator_authz.py
  vrg-commit --type feat --scope authz --message "mqoperator MQWebUser account and IBM-minimum grants on every QM (#<T3>)"
  ```

---

### Task 4: mqweb on by default on every QM node

**Issue repo:** lab. **Blocked by:** T2, T3.

**Files:**

- Modify the 8 plays carrying the role:
  - `ansible/_nativeha-cluster-ha.yml`
  - `ansible/_nativeha-dr-replication.yml`
  - `ansible/_nativeha-ubuntu-cluster-ha.yml`
  - `ansible/_nativeha-ubuntu-dr-replication.yml`
  - `ansible/_pcmk-cluster-ha.yml`
  - `ansible/_pcmk-dr-replication.yml`
  - `ansible/_rdqm-cluster-ha.yml`
  - `ansible/_rdqm-dr-replication.yml`
- Modify: `ansible/roles/mq-qmgr/tasks/main.yml`, `ansible/group_vars/all/admin.yml`
- Delete: `tests/test_mqweb_disabled.py`
- Create: `tests/test_mqweb_always_on.py`

**Interfaces:**

- Produces: mqweb running on every QM node. `mqweb_enabled` no longer exists anywhere.

- [ ] **Step 1: Write the failing test.** Create `tests/test_mqweb_always_on.py`:

  ```python
  """mqweb runs on every QM node, ungated (epic .github#341 spec §2.1; replaces the #1171 gate)."""

  from __future__ import annotations

  import subprocess
  from pathlib import Path

  import yaml

  REPO_ROOT = Path(__file__).resolve().parent.parent
  ANSIBLE = REPO_ROOT / "ansible"
  STACK_PLAYS = [
      "_nativeha-cluster-ha.yml", "_nativeha-dr-replication.yml",
      "_nativeha-ubuntu-cluster-ha.yml", "_nativeha-ubuntu-dr-replication.yml",
      "_pcmk-cluster-ha.yml", "_pcmk-dr-replication.yml",
      "_rdqm-cluster-ha.yml", "_rdqm-dr-replication.yml",
  ]


  def _mqweb_entries(path: Path) -> list:
      hits = []
      for play in yaml.safe_load(path.read_text(encoding="utf-8")) or []:
          for entry in (play or {}).get("roles", []) or []:
              if entry == "mqweb" or (isinstance(entry, dict) and entry.get("role") == "mqweb"):
                  hits.append(entry)
      return hits


  def test_every_stack_play_runs_mqweb_ungated() -> None:
      for name in STACK_PLAYS:
          entries = _mqweb_entries(ANSIBLE / name)
          assert entries, f"{name}: mqweb role missing"
          for e in entries:
              assert not (isinstance(e, dict) and "when" in e), f"{name}: mqweb still gated: {e}"


  def test_svc_sim_mq_qmgr_runs_mqweb_ungated() -> None:
      tasks = yaml.safe_load((ANSIBLE / "roles" / "mq-qmgr" / "tasks" / "main.yml").read_text())
      inc = [t for t in tasks if (t.get("ansible.builtin.include_role") or {}).get("name") == "mqweb"]
      assert inc and "when" not in inc[0]


  def test_mqweb_enabled_is_gone() -> None:
      out = subprocess.run(  # noqa: S603, S607
          ["grep", "-rn", "mqweb_enabled", str(ANSIBLE)], capture_output=True, text=True, check=False
      )
      assert out.stdout == "", out.stdout
  ```

- [ ] **Step 2: Run it and confirm it fails.**
  Run: `uv run pytest tests/test_mqweb_always_on.py -v`
  Expected: FAIL. The roles are still gated, and `mqweb_enabled` is still present.

- [ ] **Step 3: Ungate.** In each of the 8 plays:
  - replace the `- role: mqweb` block's two lines (the "per-node mqweb retired" comment and
    `when: mqweb_enabled | default(false)`) with a bare `- mqweb`;
  - keep the play's `serial: 1` and its comment, since the staggered cold start is still wanted
    (#1148).

  In `roles/mq-qmgr/tasks/main.yml`, delete the `when:` on "enable the admin REST API (mqweb)" and
  rewrite its comment to say mqweb runs on every QM (epic `.github#341`). In
  `group_vars/all/admin.yml`, delete `mqweb_enabled` and its comment block.

- [ ] **Step 4: Remove the old guard.** `vrg-git rm tests/test_mqweb_disabled.py`.

- [ ] **Step 5: Run the suite.**
  Run: `uv run pytest -q`
  Expected: PASS, including `tests/test_mqweb_restart_gate.py`, which is unchanged.

- [ ] **Step 6: Validate and commit.**

  ```bash
  vrg-container-run -- vrg-validate
  vrg-git add -A ansible tests
  vrg-commit --type feat --scope mqweb --message "run mqweb on every QM node by default; remove the mqweb_enabled gate (#<T4>)"
  ```

---

### Task 5: Human access — state probe, `mqlab console open`, REST leftovers removed

**Issue repo:** lab. **Blocked by:** T4.

**Files:**

- Rename: `src/mqlab/rest.py` → `src/mqlab/mqweb_endpoints.py`; `tests/test_rest.py` →
  `tests/test_mqweb_endpoints.py`. Rename `rest_endpoints` → `mqweb_endpoints`,
  `lab_rest_endpoints` → `lab_mqweb_endpoints`, `RestError` → `EndpointError`.
- Create: `src/mqlab/mqweb_probe.py`, `tests/test_mqweb_probe.py`
- Create: `src/mqlab/console.py`, `tests/test_console.py`
- Modify: `src/mqlab/cli.py`:
  - delete `rest_app` and `rest_render` (lines 791–807);
  - delete the `pymqrest` PKI entity (line 864);
  - add a `console` Typer app with `open`.
- Modify: `tests/test_cli_pki.py` (drop `"pymqrest"` from `_FIXED_CNS`), `pyproject.toml` (drop
  `"pymqrest>=1.2,<2"`), `uv.lock` (run `uv lock`), `vergil.toml` (`port_forwards` in each lab
  profile)
- Modify: `docs/site/docs/reference/index.md` (lines 111 and 157),
  `docs/site/docs/architecture/index.md` line 136, `docs/site/docs/getting-started.md` line 447:
  replace `mqlab rest render` with `mqlab console open`.

**Interfaces:**

- Consumes: Task 1's `REST_API_VERSION`, and the `mqreadonly` / `mqreadonly` login from Task 2.
- Produces:
  - `mqweb_endpoints.lab_mqweb_endpoints() -> list[dict[str, Any]]`, the same record shape as
    before;
  - `mqweb_probe.qm_state(url: str, qm: str, *, user: str, password: str, cafile: Path, timeout: float = 5.0) -> str | None`;
  - `mqweb_probe.resolve_live(urls: list[str], qm: str, *, state_fn: Callable[[str, str], str | None]) -> str`;
  - `mqweb_probe.ProbeError`;
  - `console.FORWARD_PORT = 19443`;
  - `console.forward_argv(target_url: str) -> list[str]`;
  - CLI `mqlab console open <stack> [--site a|b]`.

- [ ] **Step 1: Rename the endpoint module** (no behaviour change):

  ```bash
  vrg-git mv src/mqlab/rest.py src/mqlab/mqweb_endpoints.py
  vrg-git mv tests/test_rest.py tests/test_mqweb_endpoints.py
  ```

  Apply the renames listed under **Files** inside both files. Update the module docstring's last
  paragraph to say it is the internal library behind `mqlab console open` (epic `.github#341`
  spec §2.3).
  Run: `uv run pytest tests/test_mqweb_endpoints.py -v` → PASS.

- [ ] **Step 2: Write the failing probe tests.** Create `tests/test_mqweb_probe.py`:

  ```python
  """Native HA follow-the-active state probe (epic .github#341 spec §2.3)."""

  from __future__ import annotations

  import pytest

  from mqlab import mqweb_probe as p

  URLS = ["https://10.10.1.91:9443", "https://10.10.1.92:9443", "https://10.10.1.93:9443"]


  def test_resolve_live_picks_running() -> None:
      states = {URLS[0]: "replica", URLS[1]: "running", URLS[2]: "replica"}
      assert p.resolve_live(URLS, "NHARCAPP", state_fn=lambda u, q: states[u]) == URLS[1]


  def test_resolve_live_skips_unreachable_candidate() -> None:
      # Review Focus 3: a down node (None) is skipped, not fatal.
      states = {URLS[0]: None, URLS[1]: "replica", URLS[2]: "running"}
      assert p.resolve_live(URLS, "NHARCAPP", state_fn=lambda u, q: states[u]) == URLS[2]


  def test_resolve_live_raises_when_none_running() -> None:
      # Review Focus 2: mid-election, never forward to a replica.
      with pytest.raises(p.ProbeError, match=r"NHARCAPP: no candidate reports 'running'.*replica"):
          p.resolve_live(URLS, "NHARCAPP", state_fn=lambda u, q: "replica")


  def test_resolve_live_single_vip_still_probed() -> None:
      with pytest.raises(p.ProbeError):
          p.resolve_live(URLS[:1], "RDQMAPP", state_fn=lambda u, q: None)


  def test_qm_state_parses_rest_payload(monkeypatch: pytest.MonkeyPatch) -> None:
      body = b'{"qmgr":[{"name":"NHARCAPP","state":"running"}]}'
      monkeypatch.setattr(p, "_get", lambda url, **kw: body)
      assert p.qm_state("https://h:9443", "NHARCAPP", user="u", password="p", cafile=None) == "running"


  def test_qm_state_returns_none_on_connection_error(monkeypatch: pytest.MonkeyPatch) -> None:
      def boom(url: str, **kw: object) -> bytes:
          raise OSError("refused")
      monkeypatch.setattr(p, "_get", boom)
      assert p.qm_state("https://h:9443", "Q", user="u", password="p", cafile=None) is None


  def test_qm_state_raises_on_auth_failure(monkeypatch: pytest.MonkeyPatch) -> None:
      # 401/403 is a misconfiguration, not "node down": fail loud, never mask it as None.
      def denied(url: str, **kw: object) -> bytes:
          raise p.HTTPError(url, 401, "Unauthorized", None, None)  # type: ignore[arg-type]
      monkeypatch.setattr(p, "_get", denied)
      with pytest.raises(p.ProbeError, match="401"):
          p.qm_state("https://h:9443", "Q", user="u", password="p", cafile=None)
  ```

- [ ] **Step 3: Run it and confirm it fails.**
  Run: `uv run pytest tests/test_mqweb_probe.py -v` → FAIL (`ModuleNotFoundError: mqlab.mqweb_probe`).

- [ ] **Step 4: Implement** `src/mqlab/mqweb_probe.py`:

  ```python
  """Find the live mqweb for a QM by asking each candidate for the QM's state (spec §2.3).

  Native HA has no VIP: the active instance's REST reports the QM ``running``, replicas report
  ``replica`` (lab runbook, observed). IBM's client model for Native HA is the same idea — a list
  of addresses, use the one that accepts (S13). An unreachable candidate is skipped (a node down
  mid-failover); an authentication failure is a misconfiguration and fails loud.
  """

  from __future__ import annotations

  import base64
  import json
  import ssl
  import urllib.request
  from collections.abc import Callable
  from pathlib import Path
  from urllib.error import HTTPError

  REST_API_VERSION = "v3"  # set from Task 1's findings (REST_API_VERSION=...)

  __all__ = ["HTTPError", "ProbeError", "qm_state", "resolve_live"]


  class ProbeError(Exception):
      """No live mqweb could be determined (fail loud, name the QM)."""


  def _get(url: str, *, user: str, password: str, cafile: Path | None, timeout: float) -> bytes:
      ctx = ssl.create_default_context(cafile=str(cafile) if cafile else None)
      # The org-CA mqweb cert is CN=mqweb with no IP SAN (#250); the chain is verified against
      # the lab CA, the hostname is not (name-based certs: lab#1498). Spec §2.3 decision.
      ctx.check_hostname = False
      token = base64.b64encode(f"{user}:{password}".encode()).decode()
      req = urllib.request.Request(url, headers={"Authorization": f"Basic {token}"})  # noqa: S310
      with urllib.request.urlopen(req, timeout=timeout, context=ctx) as resp:  # noqa: S310
          return resp.read()


  def qm_state(
      url: str, qm: str, *, user: str, password: str, cafile: Path | None, timeout: float = 5.0
  ) -> str | None:
      """The QM's state as this mqweb reports it, or None if the candidate is unreachable."""
      endpoint = f"{url}/ibmmq/rest/{REST_API_VERSION}/admin/qmgr/{qm}"
      try:
          body = _get(endpoint, user=user, password=password, cafile=cafile, timeout=timeout)
      except HTTPError as exc:
          raise ProbeError(f"{qm}: {url} answered HTTP {exc.code} — check the mqweb login") from exc
      except OSError:
          return None
      recs = json.loads(body).get("qmgr") or []
      return str(recs[0].get("state")) if recs else None


  def resolve_live(urls: list[str], qm: str, *, state_fn: Callable[[str, str], str | None]) -> str:
      """The first candidate whose mqweb reports ``qm`` running; ProbeError if none does."""
      seen: list[str] = []
      for url in urls:
          state = state_fn(url, qm)
          if state == "running":
              return url
          seen.append(f"{url}={state or 'unreachable'}")
      raise ProbeError(f"{qm}: no candidate reports 'running' ({', '.join(seen)})")
  ```

  Set `REST_API_VERSION` to the value Task 1 recorded.
  Run: `uv run pytest tests/test_mqweb_probe.py -v` → PASS.

- [ ] **Step 5: Write the failing console tests.** Create `tests/test_console.py`:

  ```python
  """mqlab console open: two-hop access to the live mqweb (epic .github#341 spec §2.3)."""

  from __future__ import annotations

  from mqlab import console as c


  def test_forward_argv_targets_live_instance_on_fixed_port() -> None:
      argv = c.forward_argv("https://10.10.1.92:9443")
      assert argv[:2] == ["systemd-run", "--user"]
      assert f"--unit={c.UNIT}" in argv
      assert f"TCP-LISTEN:{c.FORWARD_PORT},bind=127.0.0.1,reuseaddr,fork" in argv
      assert "TCP:10.10.1.92:9443" in argv


  def test_forwarder_replaces_previous_unit() -> None:
      # Review Focus 5: a re-run after failover stops the old forwarder first.
      assert c.stop_argv() == ["systemctl", "--user", "stop", c.UNIT]


  def test_banner_names_url_logins_and_rerun_quirk() -> None:
      text = c.banner("nativeha-rhel-crr", "https://10.10.1.92:9443")
      assert f"https://localhost:{c.FORWARD_PORT}/ibmmq/console/" in text
      for user in ("mqadmin", "mqreadonly", "mqoperator"):
          assert f"{user} / {user}" in text
      assert "re-run `mqlab console open nativeha-rhel-crr` after a failover" in text
  ```

- [ ] **Step 6: Run it and confirm it fails.** `uv run pytest tests/test_console.py -v` → FAIL.

- [ ] **Step 7: Implement** `src/mqlab/console.py`:

  ```python
  """Two-hop human access to the live MQ Console (epic .github#341 spec §2.3).

  Hop 1 is a STATIC vergil port forward (vergil.toml port_forwards "9443|127.0.0.1:19443"),
  workstation -> dev VM. Hop 2 is this module: a socat forwarder on the dev VM's
  127.0.0.1:19443 -> the live instance's data-plane :9443, re-pointed on every `open`.
  Accepted quirk: getting into the ephemeral lab cannot be transparent; re-run after a failover.
  """

  from __future__ import annotations

  from urllib.parse import urlsplit

  FORWARD_PORT = 19443
  UNIT = "mqlab-console-forward"
  HUMAN_USERS = ("mqadmin", "mqreadonly", "mqoperator")


  def stop_argv() -> list[str]:
      return ["systemctl", "--user", "stop", UNIT]


  def forward_argv(target_url: str) -> list[str]:
      host = urlsplit(target_url).hostname
      port = urlsplit(target_url).port or 9443
      return [
          "systemd-run", "--user", f"--unit={UNIT}", "--collect",
          "socat", f"TCP-LISTEN:{FORWARD_PORT},bind=127.0.0.1,reuseaddr,fork", f"TCP:{host}:{port}",
      ]


  def banner(stack: str, target_url: str) -> str:
      logins = "\n".join(f"  {u} / {u}" for u in HUMAN_USERS)
      return (
          f"MQ Console for {stack} -> live instance {target_url}\n"
          f"Open: https://localhost:{FORWARD_PORT}/ibmmq/console/  (cloud host: its address, same port)\n"
          f"Logins (lab convention, password = username):\n{logins}\n"
          f"Accepted quirk: re-run `mqlab console open {stack}` after a failover or DR cutover."
      )
  ```

  `socat` must be in the dev VM: add it to the `[vm.vergil-user]` (and `anonymous`) `packages` in
  `vergil.toml`, in the same PR.
  Run: `uv run pytest tests/test_console.py -v` → PASS.

- [ ] **Step 8: Wire the CLI.** In `src/mqlab/cli.py`:
  - delete the `rest_app` block (lines 791–807) and the `pymqrest` PKI entity line;
  - add:

  ```python
  console_app = typer.Typer(help="Human access to the live MQ Console (#341)", no_args_is_help=True)
  app.add_typer(console_app, name="console")


  @console_app.command("open")
  def console_open(
      stack: str = typer.Argument(..., help="stack name, e.g. nativeha-rhel-crr"),
      site: str = typer.Option("a", "--site", help="site whose endpoints to try: a or b"),
  ) -> None:
      """Resolve the live mqweb for STACK and forward localhost:19443 to it (spec §2.3)."""
      from mqlab import console, mqweb_probe
      from mqlab.mqweb_endpoints import lab_mqweb_endpoints

      recs = {r["stack"]: r for r in lab_mqweb_endpoints()}
      if stack not in recs:
          typer.echo(f"unknown stack {stack!r}; known: {', '.join(sorted(recs))}", err=True)
          raise typer.Exit(code=2)
      urls = recs[stack]["endpoints"].get(f"site-{site}")
      if not urls:
          typer.echo(f"{stack}: no site-{site} endpoints in topology", err=True)
          raise typer.Exit(code=2)
      stacks = lab_stacks()  # from mqlab.stacks
      # svc-sim is not a stack: its QM is the shared counterparty SVCQM (stacks.py, #446).
      qm = stacks[stack].qm.qm_app if stack in stacks else next(iter(stacks.values())).qm.qm_svc
      cafile = state("secrets", "pki", "ca", "app-org", "ca.crt")
      try:
          live = mqweb_probe.resolve_live(
              urls, qm,
              state_fn=lambda u, q: mqweb_probe.qm_state(
                  u, q, user="mqreadonly", password="mqreadonly", cafile=cafile
              ),
          )
      except mqweb_probe.ProbeError as exc:
          typer.echo(f"mqlab console open: {exc}", err=True)
          raise typer.Exit(code=1) from exc
      subprocess.run(console.stop_argv(), check=False)  # noqa: S603 - absent unit is fine
      subprocess.run(console.forward_argv(live), check=True)  # noqa: S603
      typer.echo(console.banner(stack, live))
  ```

  The QM name comes from the existing derivation, `mqlab.stacks.lab_stacks()[name].qm.qm_app`
  (`<short>APP`). For svc-sim it is `.qm.qm_svc` (`SVCQM`). Never add a second derivation. Import
  `lab_stacks` at the top of `console_open`. Add a CLI test in `tests/test_console.py`
  using Typer's `CliRunner`, with `lab_mqweb_endpoints`, `qm_state` and `subprocess.run`
  monkeypatched. Cover:
  - an unknown stack → exit 2;
  - `ProbeError` → exit 1, with the message on stderr;
  - success → the stop runs, then the forward, and the banner prints.

- [ ] **Step 9: Drop the `pymqrest` leftovers.** Remove `"pymqrest>=1.2,<2"` from
  `pyproject.toml`, run `uv lock`, and drop `"pymqrest"` from `_FIXED_CNS` in
  `tests/test_cli_pki.py`. Then confirm nothing remains:
  Run: `grep -rn "pymqrest\|mqlab rest\|endpoints.json\|from mqlab.rest" src tests docs/site ansible`
  Expected: no matches, apart from historical wording. Remove the lab-ui/template comment
  mentions, which Task 2 already rewrote.

- [ ] **Step 10: Add the static forward.** In `vergil.toml`:
  - `[vm.vergil-user]` → `port_forwards = ["3000|10.50.0.2:3000", "9443|127.0.0.1:19443"]`;
  - `[vm.anonymous]` → `port_forwards = ["3001|10.50.0.2:3000", "9444|127.0.0.1:19443"]`.

  The human rebuilds the dev VM once (`vrg-vm rebuild …`). Note this in the PR description.

- [ ] **Step 11: Update the site docs** (the four lines listed under **Files**), and add a "Using
  the MQ Console" subsection to `docs/site/docs/operate/index.md`. It must cover:
  - `mqlab console open <stack> [--site b]`;
  - the three logins;
  - the macOS (`https://localhost:9443`) and cloud (`https://<cloud-vm>:9443`) URLs;
  - the re-run-after-failover quirk;
  - why the browser warns about the certificate (lab CA not trusted by the workstation).

- [ ] **Step 12: Run the suite, validate and commit.**

  ```bash
  uv run pytest -q
  vrg-container-run -- vrg-validate
  vrg-git add -A src tests pyproject.toml uv.lock vergil.toml docs/site
  vrg-commit --type feat --scope console --message "mqlab console open: follow the live mqweb; remove the mqlab rest verb and pymqrest (#<T5>)"
  ```

---

### Task 6: Cold-rebuild validation, with cold-start and memory figures (operational)

**Kind:** `validation` (filed with `vrg-issue-create --kind validation`). **Blocked by:** T5. Run
with `vergil:issue-validate`.

**Precondition self-check:** T2–T5 are merged to `develop`; `vrg-vm rebuild` has run since Task 5
merged, so the new port forward exists; `grep -rn mqweb_enabled ansible` is empty.

**Procedure:**

1. Do a full cold rebuild of every stack on macOS/arm64 (`mqlab bootstrap <stack>` for each, and
   for svc-sim the shared commons), then repeat on the cloud x86 host.
2. On every QM node, record `systemctl is-active mqweb.service` (expect `active`), time-to-ready
   and the Java RSS, using the same commands as Task 1 Step 2.
3. Re-provision one live stack (`mqlab bootstrap nativeha-ubuntu --phase provision`, or the
   repo's equivalent re-provision verb) and confirm, for Review Focus 1 and spec §6 step 7:
   - `ansible` reports `mqwebuser.xml` **unchanged** on every node;
   - `systemctl show mqweb -p ActiveEnterTimestamp` is unchanged.

**Acceptance:** every bootstrap completes in one pass; every node reports `active`; the
re-provision changes nothing. Record the figures in the result comment and append them to
`docs/reference/mqweb-spike-findings.md` in Task 7's PR. **SUCCESS / FAILURE** go in the results
template.

---

### Task 7: Live-lab acceptance — roles, discovery, failover (operational plus a runbook PR)

**Kind:** `validation`. **Blocked by:** T6. The runbook rewrite lands as a docs PR inside this
task's run (or as a sibling doc task under `#1490`'s sweep, whichever the implementer prefers;
the PR must not `Close` this validation issue).

**Runbook rewrite:** in `docs/reference/mqweb-endpoint-verification.md`, replace Step 0
(`mqlab rest render`) with `mqlab console open <stack>`, replace the `MQWEB_ADMIN_*` credentials
with the role logins, and add Steps 5–7 below. Remove the "not runnable on a default bootstrap"
banner.

**Procedure** (spec §6; per stack, both sites):

1. **No anonymous access:** `curl -sk https://<live>:9443/ibmmq/rest/<REST_API_VERSION from Task 1>/admin/qmgr` with no
   credentials → `401`. Opening the Console shows the login page.
2. **Per role, through REST** (use the CSRF header on writes:
   `-H 'ibm-mq-rest-csrf-token: x'`):
   - `mqadmin`: `GET …/qmgr/<QM>` → 200; `POST …/action/qmgr/<QM>/mqsc`
     `{"type":"runCommand","parameters":{"command":"DEFINE QLOCAL(ACCEPT.TEST) REPLACE"}}` → success.
   - `mqreadonly`: the GET → 200; the same POST → 403, refused **by mqweb**.
   - `mqoperator`: the GET → 200; the same POST → the command fails with `MQRC 2035` (`AMQ8135E`
     "not authorized"), refused **by the queue manager**. Confirm an authority event for
     `mqoperator` in Loki: `{unit="mq-event-monitor"} |= "mqoperator"`, or the repo's event label.

   Repeat each case once in the Console UI, by attempting to create a local queue.
3. **Discovery:**
   - pcmk/RDQM: `mqlab console open <stack>`, then an HA failover (`pcs resource move` / RDQM
     `endmqm -r`), then re-run: it lands on the same VIP and the Console works.
   - Native HA: an in-site failover (stop the active `mqmonitor@<QM>`), then re-run
     `console open`: it lands on the new active instance.
   - Cross-site switchover (`site-nativeha-switchover.yml -e target_live=b`), then
     `console open <stack> --site b`: it lands on the site-B active instance. Fail back.
4. **TLS:** `openssl s_client -connect <live>:9443 </dev/null | openssl x509 -noout -subject -issuer`
   → `CN=mqweb`, issuer the app-org CA, TLSv1.3.
5. **No REST leftovers:** `mqlab rest` → "No such command"; `grep -rn pymqrest pyproject.toml` →
   empty.

**Acceptance:** every row of spec §6 passes on every stack, svc-sim included (on svc-sim, step 3
uses its `net-ext` endpoint only). Record SUCCESS or FAILURE per row in the result comment.

---

## Bookends (already filed)

- **Documentation-review sweep:** `lab#1490`. It runs after T7 and must reverse commit `5f6baf9`'s
  "mqweb off by default" wording across `docs/site`.
- **Follow-on brainstorm:** `.github#344`.
- **Retrospective:** `.github#343`, the terminal bookend.
