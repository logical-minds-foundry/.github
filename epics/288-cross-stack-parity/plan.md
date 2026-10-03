# Cross-Stack Bootstrap Parity — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bring `pcmk-ubuntu`, `rdqm-rhel` and `nativeha-rhel-crr` to epic #275's cold-bootstrap bar. Measure first with the existing perf instrumentation, fix by data one change per issue, and prove it with 5-run streaks per stack and platform.

**Architecture:** Mostly box-bake and provision changes in Ansible, plus one small CLI addition (`mqlab status --check`), framed in epic #280's role/catalog model. Operational tasks carry the measurement:

- **W1** baselines and the **W4** streaks run on macOS (maintainer's host) and x86 cloud (cloud-agent validation tickets);
- two human checkpoints set targets and gate the **W3** data-driven items.

**Tech Stack:** Python 3.14 (`src/mqlab`, `uv run` in the dev loop only), pytest (100% branch coverage), Ansible roles/playbooks, Vagrant + libvirt, `lab/versions.yaml` (from #280).

**Spec:** `epics/288-cross-stack-parity/spec.md` (this repo; it travels with this plan).

## Global Constraints

- **Starts after epic #280 completes.** Use #280's names verbatim:
  - box roles `infra`, `obs`, `mq-client`, `san`, `mq-nativeha`, `pcmk`, `mq-rdqm`;
  - boxes `<role>-<os><major>` (e.g. `pcmk-ubuntu24`, `mq-client-ubuntu24`, `san-ubuntu26`, `mq-rdqm-rhel9`, `mq-nativeha-rhel10`);
  - RHEL kickstarts `lab/boxes/rhel/ks.cfg` plus optional `ks-<major>.cfg`;
  - per-version Ansible vars `ansible/roles/<role>/vars/<Distribution>-<major>.yml`.
  
  If #280's merged shape differs, re-shape the affected task at the start; never hand-patch around it.
- **Version tokens** (`ubuntu24`, `2404`, `rhel9`, `9.6`, `el9`, 26/10 forms) appear only where #280 allows: `lab/versions.yaml`, generated box names, per-major kickstarts, `vars/<Distribution>-<major>.yml`, the box-gc retired-names table. New roles here are version-agnostic.
- **All work** is in `logical-minds-foundry/mq-resiliency-lab-for-linux`: a feature branch off `develop`, committed with `vrg-commit`, PR via `vrg-pr-workflow report-ready`. One task = one issue = one PR.
- **Validation gate:** exactly `vrg-container-run -- vrg-validate`, green, 100% branch coverage. Dev-loop tests use `uv run pytest …`; never embed `uv run` in shipped code.
- **Fail loud;** no silent fallbacks. mqlab errors name fully-qualified `mqlab` commands. `build/` paths go through `mqlab.paths` only.
- **Every bake change** flips the affected boxes' manifest hashes. State which boxes need a rebake in each PR. **Acceptance is a cold rebuild in one pass, on both arches** where the box is built for both.
- **Measurement hygiene:**
  - no validation containers, bakes or other heavy work on the Vergil VM during a measured macOS run;
  - cloud measured runs are detached (`setsid nohup`);
  - `MQLAB_ENV` is never set (auto-detect, #1245);
  - every measured run pins an explicit OS version (each stack's #280 default; `rdqm-rhel` on RHEL 9).
- **One lever at a time** for W3 tuning. Each lever is kept or reverted on perf-report evidence.
- **Targets:**
  - provisional at the W1 review (ceiling 1200 s);
  - final at the pre-streak checkpoint: **post-fix run wall-clock × 1.20, rounded up to the minute, capped at 1200 s**. Not applied to DR measurement runs.
- **A clean run** = exit 0, plus `uv run mqlab status <stack> --check` exit 0 (Task 1), plus a perf report. Any failure resets the streak count and is classified, never retried until it passes.

## Review Focus

Inputs and conditions the spec implies that no unit test exercises; each is pinned in its owning task:

1. **`mqlab status --check` on a partially provisioned stack** (cluster up, QM not created) must exit non-zero, not 0. This is the false-positive case. Pinned in Task 1.
2. **A RHEL major whose live probe shows a unit the role doesn't know** (e.g. RHEL 10 has no `insights-client`). The role must skip absent units, not fail the bake, and must fail if a unit it is required to mask remains enabled. Pinned in Task 5.
3. **A per-run package task on a box baked before this epic** (an old box missing the newly baked package). It must still install the package; the skip-if-baked guard must not skip on an old box. Pinned in Tasks 2, 3 and 6.
4. **`forks` raised on a host with fewer cores than forks** (macOS Vergil VM 24 vCPU vs cloud 16). It must not oversubscribe controller SSH so that plays go UNREACHABLE; #590's timeouts must still hold. Pinned in Task 7's measurement.
5. **An Ubuntu bake for the `san` role** (from #280 T6) must also pick up `fwupd-off` and stay covered by `bake-dirs-guard`. Pinned in Task 4.

---

## Task dependency graph

```text
#280 complete
  ├─ V1a pcmk-ubuntu macOS baseline ─┐
  ├─ V1b pcmk-ubuntu cloud baseline  ├─► C1 baseline review (maintainer) ─► W3: T5, T6, T7(+V7), T8?, T9?
  ├─ V1c rdqm-rhel cloud baseline    │
  ├─ V1d nativeha-rhel-crr baseline ─┘
  ├─ W2: T1 status --check, T2 pcmk pkgs baked, T3 mq-client venv baked, T4 fwupd-off
  └─ T10 pcmk qm-status verb (blocked by V1b's live verification)
W2 + W3 merged ─► V4a..V4d streaks (run 1 = C2 pre-streak checkpoint) ─► V5a..V5d DR measurements
  ─► #1296 docs review ─► #290 retrospective
```

W1 baselines run on post-#280 develop at a commit pinned **before** any W2 change merges. W2 implementation proceeds in parallel.

---

## Phase W1 — Baseline (operational)

### Operational V1a–V1d (validation): Baseline cold runs

One validation ticket per row, created with `vrg-issue-create --kind validation`, each pinned to the same pre-W2 develop SHA.

| ticket | stack | platform | OS (explicit) | runner |
|---|---|---|---|---|
| V1a | `pcmk-ubuntu --no-dr` | macOS arm64 | #280 default for pcmk-ubuntu | maintainer's host (check-in) |
| V1b | `pcmk-ubuntu --no-dr` | x86 cloud | #280 default | cloud agent |
| V1c | `rdqm-rhel --no-dr` | x86 cloud | `rhel:9` | cloud agent |
| V1d | `nativeha-rhel-crr --no-dr` | x86 cloud | #280 default (expected `rhel:10`) | cloud agent |

**Every ticket's procedure** (#1267's format):

1. **Preconditions:**
   - commit pinned;
   - boxes rebaked to REUSE **outside** the timed run (record bake times);
   - cloud tickets: SSD boot disk proof (`lsblk ROTA=0`);
   - macOS tickets (V1a, V4a): the `reserve huge pages` preflight step passes, and `MemAvailable` is recorded (`grep MemAvailable /proc/meminfo`) with commons up (spec §5 headroom);
   - `env -u MQLAB_ENV` resolves the expected env;
   - a `--config` file with `os: <explicit>` when the target isn't the default.
2. **Cold:** `uv run mqlab teardown <stack> --commons`.
3. **Timed run:** `uv run mqlab bootstrap <stack> --no-dr [--config f.yaml]`, detached, with no `MQLAB_ENV`.
4. **Record:** exit, wall-clock, every phase (including `prereq:*`), milestones, iowait mean/peak, and the **top 15 `profile_tasks` entries** from the run log (the per-stack bring-up hotspots).
5. **Extra checks:**
   - **V1a/V1b:** SAN boot checks on `san-a` (`systemd-analyze`, `systemd-analyze blame | head`, `snap list`, `systemctl is-enabled cloud-config cloud-final apt-daily.timer fwupd-refresh.timer`), to confirm #280 T6's baked SAN box carries #275's hygiene.
   - **V1b, pcmk probe verification for T10:** after `provision` completes, run `pcs resource disable mq_group --wait`, then `pcs status resources; echo rc=$?`, then `pcs resource enable mq_group --wait`. Record whether `pcs status resources` exits 0 with `mq_group` stopped. This is read-mostly and restores the group.
   - **V1c/V1d, RHEL live hygiene probe** on one MQ node, read-only:

     ```text
     systemctl list-unit-files --state=enabled
     systemctl list-timers --all
     rpm -q subscription-manager insights-client cloud-init kexec-tools dnf-automatic fwupd
     ls /etc/dnf/plugins/ /etc/motd.d/ 2>&1
     cat /proc/cmdline
     systemd-analyze; systemd-analyze blame | head -15
     ```

6. **Result:** `Outcome: SUCCESS` means the run completed and the data was posted (a failing bootstrap that produces a perf report is still a SUCCESSful *measurement*: record the failure and file a fix).

### Checkpoint C1: Baseline review (maintainer, recorded as an epic comment)

- [ ] Read V1a–V1d: name each stack's bottleneck from phases and `profile_tasks`.
- [ ] Record each stack's **provisional target** (ceiling 1200 s).
- [ ] Decide go/no-go for each W3 item (T5–T9) and file any **new bottleneck** as its own task under the epic.
- [ ] Record T10's verdict from V1b's probe verification.

---

## Phase W2 — Known wins (code; independent of W1)

### Task 1: `mqlab status <stack> --check` (scriptable clean-run check)

**Files:**

- Modify: `src/mqlab/cli.py`, the `status` command: add `--check`.
- Test: `tests/test_cli_status.py`.

**Interfaces:**

- Consumes: the existing `_probe_all(deps, stack) -> dict` and `build_states` / `first_unsatisfied(stack, states)` used by bootstrap, plus each stack's `qm-status` verb. Bootstrap's observe probe only checks Prometheus targets, so `--check` adds a separate **obs health probe** (new).
- Produces:
  - `_probe_obs_health(deps) -> list[str]`, returning the names of failed obs checks (empty = healthy). Over the existing ssh-to-obs path, read-only: `systemctl is-active` for each obs unit (`opensearch`, `opensearch-dashboards`, `data-prepper`, `loki`, `prometheus`, `grafana-server`, `alloy`) and `curl -fsS localhost:9200/_cluster/health`, requiring `status == "green"`.
  - `mqlab status <stack> --check` exits **0 iff every phase is satisfied (including `provision`, i.e. the stack's `qm-status` verb passes) AND `_probe_obs_health` returns no failures**. Otherwise it exits 1, printing the first unsatisfied phase or the failed obs checks.
  - Without `--check`, behaviour is unchanged.

- [ ] **Step 1: Write the failing tests**

```python
def test_status_check_exits_zero_when_all_phases_satisfied(fake_probes_all_up, runner):
    result = runner.invoke(app, ["status", "nativeha-ubuntu", "--check"])
    assert result.exit_code == 0

def test_status_check_exits_nonzero_when_qm_not_up(fake_probes_qm_down, runner):
    # Review Focus 1: cluster/VMs up, QM not provisioned -> must not pass.
    result = runner.invoke(app, ["status", "pcmk-ubuntu", "--check"])
    assert result.exit_code == 1
    assert "provision" in result.output

def test_status_without_check_keeps_exit_zero(fake_probes_qm_down, runner):
    assert runner.invoke(app, ["status", "pcmk-ubuntu"]).exit_code == 0

def test_status_check_fails_when_obs_degraded_but_targets_up(fake_probes_all_up, fake_obs_health, runner):
    # Prometheus targets present (observe phase "satisfied") but OpenSearch yellow / Dashboards down.
    fake_obs_health.failures = ["opensearch _cluster/health=yellow", "opensearch-dashboards inactive"]
    result = runner.invoke(app, ["status", "nativeha-ubuntu", "--check"])
    assert result.exit_code == 1
    assert "opensearch" in result.output

def test_obs_health_probe_reports_each_failed_unit(fake_ssh_obs):
    fake_ssh_obs.inactive = {"loki"}
    fake_ssh_obs.cluster_status = "green"
    assert _probe_obs_health(fake_ssh_obs.deps) == ["loki inactive"]

def test_status_check_requires_stack_name(runner):
    result = runner.invoke(app, ["status", "--check"])
    assert result.exit_code == 2
    assert "mqlab status <stack> --check" in result.output
```

- [ ] **Step 2:** `uv run pytest tests/test_cli_status.py -q`. Expected: FAIL (no `--check` option).
- [ ] **Step 3:** Implement `--check`.
  - Reuse `_probe_all` and `first_unsatisfied`, then run `_probe_obs_health`. Its ssh call is bounded by the same timeouts as other read-only probes, and a probe that errors counts as a failed check, never as a pass.
  - Exit with `typer.Exit(1)`, printing `mqlab status <stack> --check: <phase> not satisfied` or `… obs unhealthy: <failures>`.
  - Require a stack name with `--check` (exit 2 with a usage error naming the full command).
- [ ] **Step 4:** Run the tests: PASS. Then `vrg-container-run -- vrg-validate`: green.
- [ ] **Step 5:** `vrg-commit --type feat --scope mqlab --message "mqlab status <stack> --check: scriptable clean-run check (#<T1>)"`.

### Task 2: Bake the pcmk cluster packages; no per-run apt refresh

**Files:**

- Modify: `ansible/bake-pcmk-ubuntu.yml`. Add an install play (before the final hygiene/guard plays) for `pacemaker`, `corosync`, `pcs`, `resource-agents-base`, `resource-agents-extra`, `fence-agents-virsh` and `open-iscsi`, then disable and stop `pcsd`, `corosync` and `pacemaker` at bake.
- Modify: `ansible/roles/pcmk-cluster/tasks/install-Debian.yml`, `ansible/roles/pcmk-stonith/tasks/install-Debian.yml` and `ansible/roles/iscsi-initiator/tasks/install-Debian.yml`. Use `update_cache: false` and `state: present`, behind a `package_facts` skip-if-installed guard. Keep an install path for old boxes (Review Focus 3).
- Modify: `docs/development/box-bake-manifest.md` (the `pcmk` role rows).
- Test: `tests/test_pcmk_baked.py` (new).

**Interfaces:**

- Consumes: #280's `pcmk` role (box `pcmk-<os><major>`, bake stem `pcmk-ubuntu`).
- Produces: no new names. Flips the `pcmk-*` boxes' manifest hash.

- [ ] **Step 1: Write the failing tests**

```python
import yaml
from pathlib import Path
ANSIBLE = Path(__file__).resolve().parents[1] / "ansible"
PKGS = {"pacemaker", "corosync", "pcs", "resource-agents-base", "resource-agents-extra",
        "fence-agents-virsh", "open-iscsi"}

def _tasks(path):
    data = yaml.safe_load((ANSIBLE / path).read_text())
    return data if isinstance(data, list) else []

def _installed_names(plays):
    names = set()
    for play in plays:
        for t in play.get("tasks", []) + play.get("pre_tasks", []):
            apt = t.get("ansible.builtin.apt") or t.get("apt") or {}
            n = apt.get("name") or []
            names |= set([n] if isinstance(n, str) else n)
    return names

def test_pcmk_bake_installs_cluster_packages():
    assert PKGS <= _installed_names(_tasks("bake-pcmk-ubuntu.yml"))

def test_pcmk_bake_leaves_cluster_daemons_disabled():
    text = (ANSIBLE / "bake-pcmk-ubuntu.yml").read_text()
    for svc in ("pcsd", "corosync", "pacemaker"):
        assert svc in text and "enabled: false" in text

def test_per_run_installs_never_refresh_apt_cache():
    for role in ("pcmk-cluster", "pcmk-stonith", "iscsi-initiator"):
        for t in _tasks(f"roles/{role}/tasks/install-Debian.yml"):
            apt = t.get("ansible.builtin.apt") or t.get("apt") or {}
            if apt:
                assert apt.get("update_cache", False) is False, role
                assert apt.get("state", "present") == "present", role

def test_per_run_installs_still_install_on_old_boxes():
    # Review Focus 3: the guard skips only when the package is installed, never unconditionally.
    for role in ("pcmk-cluster", "pcmk-stonith", "iscsi-initiator"):
        text = (ANSIBLE / f"roles/{role}/tasks/install-Debian.yml").read_text()
        assert "ansible_facts.packages" in text, role
```

- [ ] **Step 2:** `uv run pytest tests/test_pcmk_baked.py -q`: FAIL.
- [ ] **Step 3:** Implement the bake play and the guarded per-run tasks. The guard is `when: "'<pkg>' not in ansible_facts.packages"` after one `ansible.builtin.package_facts:`.
- [ ] **Step 4:** PASS, then `vrg-validate` green. The PR notes say `pcmk-*` boxes need a rebake on both arches, and that acceptance is V4a/V4b.
- [ ] **Step 5:** `vrg-commit --type fix --scope box --message "bake pcmk cluster packages; no per-run apt refresh (#<T2>)"`.

### Task 3: Bake the app-client pymqi venv into the `mq-client` role

**Files:**

- Create: `ansible/roles/mq-client/tasks/install.yml` with the venv creation (`creates:` guard), `pip install pymqi` (`state: present`) and an `import pymqi` check. Follow #1227's `mq-inter-qm/tasks/install.yml` pattern, including the `/opt/mqm/inc/cmqc.h` precondition.
- Modify: `ansible/roles/mq-client/tasks/main.yml` to `import_tasks: install.yml` instead of the inline venv/pip tasks.
- Modify: `ansible/roles/mq-client/defaults/main.yml`: `mq_client_venv: /home/vagrant/mqvenv` (the single definition).
- Modify: `ansible/bake-mq-ubuntu.yml` (the `mq-client` role's bake). Include `mq-client` with `tasks_from: install` after `mq-install`.
- Modify: `ansible/site-distributed-shared.yml`. Delete the duplicate venv tasks.
- Modify: `docs/development/box-bake-manifest.md`.
- Test: `tests/test_mq_client_venv_baked.py` (new).

**Interfaces:**

- Consumes: #280's `mq-client` role (box `mq-client-<os><major>`).
- Produces: the variable `mq_client_venv`. Flips the `mq-client-*` hash.

- [ ] **Step 1: Write the failing tests**

```python
def test_mq_client_bake_builds_venv_after_mq_install():
    plays = _tasks("bake-mq-ubuntu.yml")
    roles = [t.get("ansible.builtin.include_role", {}) for p in plays for t in p.get("tasks", [])]
    names = [(r.get("name"), r.get("tasks_from")) for r in roles if r]
    assert names.index(("mq-install", None)) < names.index(("mq-client", "install"))

def test_site_distributed_shared_has_no_duplicate_venv_build():
    text = (ANSIBLE / "site-distributed-shared.yml").read_text()
    assert "python3 -m venv" not in text and "pip" not in text

def test_venv_path_defined_once():
    hits = [p for p in ANSIBLE.rglob("*.yml") if "/home/vagrant/mqvenv" in p.read_text()]
    assert hits == [ANSIBLE / "roles/mq-client/defaults/main.yml"]

def test_per_run_install_is_noop_safe_and_old_box_safe():
    text = (ANSIBLE / "roles/mq-client/tasks/install.yml").read_text()
    assert "creates:" in text and "import pymqi" in text and "state: present" in text
```

- [ ] **Step 2:** FAIL. **Step 3:** Implement. **Step 4:** PASS; `vrg-validate` green.
- [ ] **Step 5:** `vrg-commit --type feat --scope box --message "bake the app-client pymqi venv into the mq-client role (#<T3>)"`.

### Task 4: `fwupd-off` in every Ubuntu bake (including `san`)

**Files:**

- Create: `ansible/roles/fwupd-off/tasks/main.yml`. Mask `fwupd-refresh.timer`, `fwupd-refresh.service` and `fwupd.service`, skipping units that are absent (`systemctl list-unit-files`). Assert they're masked, failing the bake otherwise.
- Modify: every Ubuntu bake playbook: `bake-obs.yml`, `bake-infra.yml`, `bake-mq-ubuntu.yml`, `bake-nativeha-ubuntu.yml`, `bake-pcmk-ubuntu.yml`, and **`bake-san.yml` (#280 T6)**. Include the role in the same play as `cloud-init-trim`/`snapd-off` (the second-to-last play), keeping `bake-dirs-guard` last.
- Modify: `docs/development/box-model.md` and `box-bake-manifest.md`.
- Test: `tests/test_fwupd_baked_off.py` (new).

**Interfaces:**

- Consumes: the list of Ubuntu bakes. Derive it the same way `tests/test_apt_autoupdate_baked_off.py` does, so `bake-san.yml` is included automatically.
- Produces: no names. Flips every Ubuntu box hash; RHEL is unchanged.

- [ ] **Step 1: Write the failing tests**

```python
def test_every_ubuntu_bake_includes_fwupd_off(ubuntu_bakes):
    assert "bake-san.yml" in {p.name for p in ubuntu_bakes}   # Review Focus 5
    for p in ubuntu_bakes:
        assert "fwupd-off" in p.read_text(), p.name

def test_no_rhel_bake_includes_fwupd_off(rhel_bakes):
    for p in rhel_bakes:
        assert "fwupd-off" not in p.read_text()

def test_bake_dirs_guard_stays_last(ubuntu_bakes):
    for p in ubuntu_bakes:
        plays = yaml.safe_load(p.read_text())
        assert "bake-dirs-guard" in yaml.safe_dump(plays[-1]), p.name

def test_role_masks_and_asserts():
    text = (ANSIBLE / "roles/fwupd-off/tasks/main.yml").read_text()
    assert "fwupd-refresh.timer" in text and "masked" in text and "ansible.builtin.fail" in text
```

- [ ] **Step 2:** FAIL. **Step 3:** Implement. **Step 4:** PASS; `vrg-validate` green. The PR notes say every Ubuntu box needs a rebake.
- [ ] **Step 5:** `vrg-commit --type fix --scope box --message "mask fwupd refresh in every Ubuntu bake (#<T4>)"`.

### Task 10: pcmk `qm-status` verb checks `mq_group` (verify, then fix)

**Blocked-by:** V1b (its probe-verification record).

**Files:**

- Modify: `lab/topology.yaml`, `stacks.pcmk-ubuntu.verbs.qm-status`.
- Test: `tests/test_cli_bootstrap.py` / `tests/test_cli_status.py` (pcmk status shape).

- [ ] **Step 0: Gate.** If V1b recorded that `pcs status resources` exits **non-zero** with `mq_group` stopped, close this task with that evidence (not needed). Otherwise continue.
- [ ] **Step 1: Failing test.** The pcmk `qm-status` verb is a command that fails unless `mq_group` is Started, e.g. `{cmd: "pcs resource status mq_group | grep -q 'Started'"}`. Confirm the exact `pcs resource status` output format from V1b's capture. With `fake_probes` returning "Stopped", the provision state is unsatisfied.
- [ ] **Step 2:** FAIL. **Step 3:** Change the verb. **Step 4:** PASS; `vrg-validate` green.
- [ ] **Step 5:** `vrg-commit --type fix --scope topology --message "pcmk qm-status checks mq_group Started, not cluster liveness (#<T10>)"`.

---

## Phase W3 — Data-gated (each blocked by C1)

### Task 5: `rhel-hygiene` role (version-agnostic, every RHEL major)

**Blocked-by:** V1c, V1d, and C1's go decision.

**Files:**

- Create: `ansible/roles/rhel-hygiene/tasks/main.yml`:
  - mask the listed units **if present** (`rhel_hygiene_mask_units`);
  - set `enabled=0` in the listed dnf plugin confs if present (`rhel_hygiene_disable_dnf_plugins`);
  - remove the listed motd hooks if present;
  - **assert** each listed unit that exists is `masked`, failing the bake otherwise.
- Create: `ansible/roles/rhel-hygiene/defaults/main.yml` with lists common to every major, e.g. `dnf-makecache.timer`, `dnf-makecache.service`, `rhsmcertd.service`, `insights-client.timer`, `insights-client-boot.service`, `kdump.service`, plus plugins `subscription-manager` and `product-id`, **as confirmed by V1c/V1d's probes.**
- Create: `ansible/roles/rhel-hygiene/vars/RedHat-9.yml` and `RedHat-10.yml`, only for per-major differences the probes found. Version tokens are allowed only here.
- Modify: `ansible/bake-mq-rdqm.yml` and `ansible/bake-nativeha-rhel.yml`. Include `rhel-hygiene` (the last play before any guard).
- Modify: `lab/boxes/rhel/ks.cfg` (and `ks-10.cfg` if #280 created it): `%addon com_redhat_kdump --disable` + `%end`, **only if** V1c/V1d show kdump enabled and the crashkernel reservation present in `/proc/cmdline`.
- Modify: `docs/development/box-model.md` and `box-bake-manifest.md`.
- Test: `tests/test_rhel_hygiene_baked.py` (new).

- [ ] **Step 1: Write the failing tests**

```python
def test_both_rhel_bakes_include_rhel_hygiene(rhel_bakes):
    for p in rhel_bakes:
        assert "rhel-hygiene" in p.read_text(), p.name

def test_no_ubuntu_bake_includes_rhel_hygiene(ubuntu_bakes):
    for p in ubuntu_bakes:
        assert "rhel-hygiene" not in p.read_text()

def test_role_skips_absent_units_but_asserts_present_ones():
    # Review Focus 2: absent unit -> skip; present-but-not-masked -> fail the bake.
    text = (ANSIBLE / "roles/rhel-hygiene/tasks/main.yml").read_text()
    assert "list-unit-files" in text and "ansible.builtin.fail" in text and "masked" in text

def test_role_is_version_agnostic():
    for p in (ANSIBLE / "roles/rhel-hygiene").rglob("*.yml"):
        if "/vars/" in str(p):
            continue
        assert not re.search(r"rhel ?(9|10)|el(9|10)|\b9\.6\b", p.read_text()), p
```

- [ ] **Step 2:** FAIL. **Step 3:** Implement from the probe data. **Step 4:** PASS; `vrg-validate` green. The PR notes say `mq-rdqm-rhel9` and `mq-nativeha-rhel<N>` need a rebake, and that acceptance is V4c/V4d.
- [ ] **Step 5:** `vrg-commit --type feat --scope box --message "rhel-hygiene: mask dnf/rhsm/insights/kdump on every RHEL major (#<T5>)"`.

### Task 6: Skip redundant `dnf` no-ops on baked RHEL boxes

**Blocked-by:** C1.

**Files:**

- Modify: the per-run `acl` / `libicu` dnf tasks in `ansible/_rdqm-cluster-ha.yml`, `ansible/_rdqm-dr-replication.yml`, `ansible/_nativeha-cluster-ha.yml`, `ansible/_nativeha-dr-replication.yml`, `ansible/roles/mq-nativeha/tasks/install-RedHat.yml`, `ansible/roles/mq-nativeha/tasks/tls.yml` and `ansible/roles/node-exporter/tasks/main.yml`. Put each behind one `package_facts` + `when: "'<pkg>' not in ansible_facts.packages"`.
- Test: `tests/test_rhel_dnf_noops.py` (new).

- [ ] **Step 1: Failing test.** Every `ansible.builtin.dnf`/`package` task for `acl`/`libicu` in those files has a `when:` referencing `ansible_facts.packages` (Review Focus 3: old boxes still install).
- [ ] **Step 2:** FAIL. **Step 3:** Implement. **Step 4:** PASS; `vrg-validate` green.
- [ ] **Step 5:** `vrg-commit --type fix --scope ansible --message "skip per-run dnf no-ops on baked RHEL boxes (#<T6>)"`.

### Task 7: Ansible `forks` (one lever, measured)

**Blocked-by:** C1.

**Files:**

- Modify: `ansible/ansible.cfg` `[defaults]`: `forks = 20`, with a comment citing this task and #590 (the SSH timeouts stay).
- Test: `tests/test_ansible_cfg.py`. It parses `ansible.cfg`; `forks` is an integer ≥ 10, and `timeout`/`ConnectTimeout` are unchanged.

- [ ] **Step 1–4:** Test → FAIL → change → PASS; `vrg-validate` green.
- [ ] **Step 5:** `vrg-commit --type fix --scope ansible --message "forks=20 for multi-host plays (#<T7>)"`. `vrg-commit` doesn't accept `perf`.

### Operational V7 (validation): forks keep/revert measurement

**Blocked-by:** T7 merged.

- One cold `pcmk-ubuntu --no-dr` run on **each platform**, compared with `mqlab perf diff` against V1a/V1b. Report `provision` and `observe` deltas and any UNREACHABLE (Review Focus 4).
- **Keep** if neither platform regresses and no UNREACHABLE appears. Otherwise file a revert task.

### Task 8 (conditional): pcmk boot-batch parallelism

**Blocked-by:** C1. **Only if** V1a/V1b show `vms` ≥ 40% of pcmk wall-clock.

- Hypothesis: `_batch_shares_box` (`src/mqlab/phases.py`) serialises same-box batches because of a box-volume staging race (#859). With #1248's REUSE registration, the base volume already exists after the first boot.
- Change: allow parallel `vagrant up` for a same-box batch when every guest's box base volume already exists in the pool (checked via `virsh vol-list`).
- Tests cover both branches. A measured keep/revert run follows, as for V7.

### Task 9 (conditional): Stack-specific milestones

**Blocked-by:** C1. **Only if** C1 finds `profile_tasks` insufficient.

- Extend the readiness-task milestone map in `src/mqlab/perfrun.py` with:
  - `drbd_connected`: the DRBD connect wait task;
  - `pcs_cluster_online`: `pcmk-cluster : wait for all nodes online`;
  - `rdqm_qm_created`: the `rdqm-qm-create` step;
  - `crr_group_formed`: the `crr.yml` recovery-group task.
- Tests pin the task names to the role files, as #1205 did.

---

## Phase W4 — Acceptance (operational)

### Operational V4a–V4d (validation): Streaks

**Blocked-by:** T1–T4, T10 (or its close-as-not-needed), and every W3 task C1 approved (with V7's keep decision).

| ticket | stack | platform | OS |
|---|---|---|---|
| V4a | `pcmk-ubuntu --no-dr` | macOS arm64 (check-in per run) | #280 default |
| V4b | `pcmk-ubuntu --no-dr` | x86 cloud | #280 default |
| V4c | `rdqm-rhel --no-dr` | x86 cloud | `rhel:9` |
| V4d | `nativeha-rhel-crr --no-dr` | x86 cloud | #280 default |

- **Pinned:** the develop SHA after the last W2/W3 merge, with "UNBLOCKED @ &lt;sha&gt;" posted when it's known.
- **Boxes:** rebaked outside the timed runs (record times). Every changed box is verified on both arches **before** its streak (#1265 lesson).
- **Run 1 = Checkpoint C2:** its wall-clock × 1.20, rounded up to the minute and capped at 1200 s, is the **target**, recorded in the ticket before run 2.
- **Per run:** exit, wall-clock, every phase, the milestones, `uv run mqlab status <stack> --check` (Task 1) exit code, and iowait.
- **Comparison table** against that row's V1 baseline.
- **SUCCESS:** 5 consecutive clean runs within the target. **FAILURE:** 3 failures in total (classify each; new bugs become tasks).

### Operational V5a–V5d (validation): DR measurements (cloud, one run each)

**Blocked-by:** V4b–V4d SUCCESS.

- `pcmk-ubuntu`, `rdqm-rhel`, `nativeha-rhel-crr` and `nativeha-ubuntu`, **without `--no-dr`**, on x86 cloud, each at its explicit OS.
- Record the same data as V1, plus the DR-side `profile_tasks` top 15 and DRBD / CRR sync times.
- **No target applies.** Name each DR bottleneck and file follow-ups.

---

## Closing bookends (already filed)

- `logical-minds-foundry/.github#289`: documentation (this spec + plan PR).
- `logical-minds-foundry/mq-resiliency-lab-for-linux#1296`: docs review. Sweep the site and dev docs for the new roles (`fwupd-off`, `rhel-hygiene`), the baked pcmk/mq-client packages, `mqlab status --check`, forks, per-stack targets and measured results.
- `logical-minds-foundry/.github#290`: retrospective (terminal).

## Self-Review

- **Spec coverage:**

  | spec section | covered by |
  |---|---|
  | §1 streaks | V4a–d |
  | §1 clean check | Task 1 + V4 |
  | §1 targets | C1 + C2 |
  | §1 DR | V5a–d |
  | §1 cold rebuild on both arches | Global Constraints + V4 |
  | §2 milestones | T9 (conditional) |
  | §2 hygiene | Global Constraints |
  | §3 gap 1 | T2 |
  | §3 gaps 2–3 | **#280 T6** (baked SAN box, SAN deb cache retired); verified in V1a/V1b |
  | §3 gap 4 | T8 |
  | §3 gap 5 | T10 |
  | §3 gap 6 | T5 |
  | §3 gap 7 | T6 |
  | §3 gap 8 | T3 |
  | §3 gap 9 | T7 + V7 |
  | §3 gap 10 | T4 |
  | §4 W1 | V1a–d |
  | §5 cross-arch risk | V4 preconditions |

- **Spec delta to confirm at alignment:** the spec's W2 "baked SAN box" and gap 3 (san-b deb cache) are **already delivered by #280 T6**. This plan doesn't duplicate them; it verifies them (V1a/V1b) and extends hygiene to `san` (T4).
- **Placeholders:** `<T1>`…`<T10>` are issue numbers, filled when tasks are filed. RHEL hygiene unit lists are gated on V1c/V1d's probe by design (T5 Step 3), not TBD.
- **Names:** `mq_client_venv`, `rhel_hygiene_mask_units`, `rhel_hygiene_disable_dnf_plugins` and `mqlab status --check` are used consistently. #280's role/box names are used verbatim.
