# HA-only (no-DR) bootstrap — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `mqlab bootstrap <stack> --no-dr` — bring up only a stack's HA site, skipping the DR (site-B) guests and DR provisioning — to run a lighter footprint under load.

**Architecture:** A stateless boolean flag threaded from the `bootstrap` CLI command through `_bootstrap_run` into the four phase builders. An explicit `dr_groups` topology marker (mirrored on the `Stack` dataclass) names each stack's DR groups; an effective-members helper subtracts them. `vms`/`observe` enumerate effective (site-A) members when the flag is set; `provision` passes `-e dr_enabled=false`; `net` is untouched. The provision playbook gates its two site-B touch-points on `dr_enabled`.

**Tech Stack:** Python 3.12, Typer + Rich CLI, Ansible (provisioning), pytest (100% branch coverage), Vergil-managed repo (`vrg-*` wrappers).

## Global Constraints

- Repo `mq-resiliency-lab-for-linux` is **Vergil-managed**: feature branch off `develop`, PR into `develop`, both protected. Commit with `vrg-commit`; git/gh via `vrg-git`/`vrg-gh`.
- **Validation is one command:** `vrg-container-run -- vrg-validate` (ruff, ansible-lint, `uv run pytest` at **100% branch coverage**). Never run linters individually.
- **Fail loud, no silent failures.** No swallowed exceptions or masking fallbacks.
- **Provisioning changes require the cold-rebuild acceptance gate** — proven by the validation task (Task 7), not by CI.
- `--no-dr` is **stateless**: it shapes only the phases it runs and never tears down guests (spec §3.6).
- Default behaviour is unchanged: everything keys on `dr_enabled | default(true)`, so an ordinary bootstrap (no flag) is byte-for-byte identical.

---

## Wave 1 — the mechanism, wired for `nativeha-ubuntu` (one PR, repo `mq-resiliency-lab-for-linux`)

### Task 1: `Stack.dr_groups` + effective-members helpers (`stacks.py`)

**Files:**
- Modify: `src/mqlab/stacks.py` (the `Stack` dataclass ~line 118-128; `lab_stacks()` ~line 206-218; add helpers near `stack_members` ~line 251)
- Test: `tests/test_stacks.py`

**Interfaces:**
- Produces: `Stack.dr_groups: list[str]`; `stack_dr_hosts(name: str) -> list[str]`; `stack_members_effective(name: str, *, no_dr: bool) -> list[str] | None`.

- [ ] **Step 1: Write the failing tests**

```python
# tests/test_stacks.py
from mqlab.stacks import lab_stacks, stack_members, stack_members_effective, stack_dr_hosts

def test_nativeha_ubuntu_declares_dr_groups():
    assert lab_stacks()["nativeha-ubuntu"].dr_groups == ["nha_ubuntu_b"]

def test_stack_dr_hosts_are_site_b():
    assert stack_dr_hosts("nativeha-ubuntu") == ["nha-ubuntu-b1", "nha-ubuntu-b2", "nha-ubuntu-b3"]

def test_effective_members_excludes_dr_when_no_dr():
    full = stack_members("nativeha-ubuntu")
    eff = stack_members_effective("nativeha-ubuntu", no_dr=True)
    assert eff == [m for m in full if not m.startswith("nha-ubuntu-b")]
    assert "nha-ubuntu-b1" not in eff

def test_effective_members_is_full_when_not_no_dr():
    assert stack_members_effective("nativeha-ubuntu", no_dr=False) == stack_members("nativeha-ubuntu")

def test_effective_members_unknown_stack_is_none():
    assert stack_members_effective("does-not-exist", no_dr=True) is None
```

- [ ] **Step 2: Run to verify failure**

Run: `uv run pytest tests/test_stacks.py -k "dr_groups or dr_hosts or effective_members" -v`
Expected: FAIL (`dr_groups` attribute / helpers missing).

- [ ] **Step 3: Implement**

Add the field to the dataclass (after `groups: list[str]`):

```python
    dr_groups: list[str]
```

Read it in `lab_stacks()` (in the `Stack(...)` construction):

```python
            dr_groups=list(cfg.get("dr_groups") or []),
```

Add the helpers near `stack_members`:

```python
def stack_dr_hosts(name: str) -> list[str]:
    """Hosts contributed by a stack's `dr_groups` (the DR / site-B members).

    Empty for a stack that declares no `dr_groups` or is unknown. Read the same
    pure-membership way as `stack_members`."""
    data = _topology()
    stacks = data.get("stacks") or {}
    all_groups: dict[str, list[str]] = {
        g: list(hosts) for g, hosts in (data.get("groups") or {}).items()
    }
    hosts: list[str] = []
    for g in (stacks.get(name) or {}).get("dr_groups", []):
        for host in all_groups.get(g, []):
            if host not in hosts:
                hosts.append(host)
    return hosts


def stack_members_effective(name: str, *, no_dr: bool) -> list[str] | None:
    """`stack_members`, minus the `dr_groups` hosts when `no_dr` — the guests a
    bring-up actually starts. Full membership (or None for an unknown stack) when
    `no_dr` is False."""
    members = stack_members(name)
    if members is None or not no_dr:
        return members
    dr = set(stack_dr_hosts(name))
    return [m for m in members if m not in dr]
```

- [ ] **Step 4: Run to verify pass**

Run: `uv run pytest tests/test_stacks.py -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd <worktree> && vrg-git add src/mqlab/stacks.py tests/test_stacks.py
vrg-commit --type feat --scope bootstrap --message "Stack.dr_groups + effective-members helpers (#188)"
```

---

### Task 2: topology marker for `nativeha-ubuntu` (`lab/topology.yaml`)

**Files:**
- Modify: `lab/topology.yaml` (the `stacks: nativeha-ubuntu:` block — add `dr_groups`)

**Interfaces:**
- Consumes: nothing. Produces: the data Task 1's helpers read (its tests already assert this value, so they now pass against real data).

- [ ] **Step 1: Add the marker**

In the `nativeha-ubuntu` stack block, directly under its `groups:` line:

```yaml
    dr_groups: [nha_ubuntu_b]     # DR site — skipped by `bootstrap --no-dr` (#188)
```

- [ ] **Step 2: Verify Task 1's data-bound tests pass**

Run: `uv run pytest tests/test_stacks.py -v`
Expected: PASS (the `== ["nha_ubuntu_b"]` / site-B host assertions now bind to real topology).

- [ ] **Step 3: Commit**

```bash
vrg-git add lab/topology.yaml
vrg-commit --type feat --scope bootstrap --message "declare dr_groups for nativeha-ubuntu (#188)"
```

---

### Task 3: `--no-dr` flag + fail-loud guard (`cli.py`)

**Files:**
- Modify: `src/mqlab/cli.py` (the `bootstrap` command ~line 2094; `_bootstrap_run` ~line 2010)
- Test: `tests/test_cli_bootstrap.py`

**Interfaces:**
- Consumes: `Stack.dr_groups` (Task 1).
- Produces: `bootstrap(..., no_dr: bool)`; `_bootstrap_run(stack_name, *, only, from_phase, step, no_dr: bool = False)`.

- [ ] **Step 1: Write the failing test (guard)**

```python
# tests/test_cli_bootstrap.py
from typer.testing import CliRunner
from mqlab import cli

def test_no_dr_on_stack_without_dr_groups_fails_loud(monkeypatch):
    # a stack whose dr_groups is empty cannot honor --no-dr
    result = CliRunner().invoke(cli.app, ["bootstrap", "rdqm-rhel", "--no-dr"])
    assert result.exit_code != 0
    assert "declares no DR site" in result.output
```

*(Note: pick a stack whose real `dr_groups` is empty in Wave 1 — every stack except `nativeha-ubuntu`. If `rdqm-rhel` is arch-gated first, monkeypatch `_gate_stack_host_arch` to a no-op, or assert on the guard message via a direct `_bootstrap_run` call as the existing bootstrap tests do.)*

- [ ] **Step 2: Run to verify failure**

Run: `uv run pytest tests/test_cli_bootstrap.py::test_no_dr_on_stack_without_dr_groups_fails_loud -v`
Expected: FAIL (`--no-dr` unknown option).

- [ ] **Step 3: Implement**

Add the option to `bootstrap`:

```python
    no_dr: Annotated[
        bool,
        typer.Option("--no-dr", help="skip the DR site — bring up the HA site only"),
    ] = False,
```

Pass it through:

```python
    _bootstrap_run(stack_name, only=only, from_phase=from_phase, step=step, no_dr=no_dr)
```

In `_bootstrap_run`, add `no_dr: bool = False` to the signature, and the guard right after `stack = _lookup_stack_or_exit(stack_name)`:

```python
    if no_dr and not stack.dr_groups:
        deps_renderer_or_typer_error(f"{stack.name} declares no DR site to skip (no dr_groups)")
        raise typer.Exit(code=2)
```

*(Use the module's existing fail-loud idiom — `typer.echo(..., err=True)` + `typer.Exit`, matching how `_validate_phase_name` reports; do not invent a new error path.)*

- [ ] **Step 4: Run to verify pass**

Run: `uv run pytest tests/test_cli_bootstrap.py -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-git add src/mqlab/cli.py tests/test_cli_bootstrap.py
vrg-commit --type feat --scope bootstrap --message "bootstrap --no-dr flag + fail-loud guard (#188)"
```

---

### Task 4: phases honor `no_dr` (`phases.py`)

**Files:**
- Modify: `src/mqlab/phases.py` — the four builders (`_net_build_steps` 181, `_vms_build_steps` 298, `_provision_build_steps` 432, `_observe_build_steps` 504); the `Phase` invocation in the run loop (~line 621); the shared member lookup (~line 145)
- Modify: `src/mqlab/cli.py` — `_bootstrap_run` passes `no_dr` into the phase run
- Test: `tests/test_phases.py`

**Interfaces:**
- Consumes: `stack_members_effective` (Task 1); the `no_dr` bool (Task 3).
- Produces: builders that accept `*, no_dr: bool = False`; `provision` step whose ansible cmd includes `dr_enabled=false` only under `no_dr`.

**Threading:** the run loop calls each phase's `build_steps(stack, deps)`. Extend to `build_steps(stack, deps, no_dr=no_dr)` and give every builder the keyword-only `no_dr: bool = False`. `net` accepts-and-ignores it (keeps a uniform signature). `_bootstrap_run` threads `no_dr` into whatever runs the selected phases.

- [ ] **Step 1: Write the failing tests**

```python
# tests/test_phases.py
from mqlab import phases

def _stack(name="nativeha-ubuntu"):
    from mqlab.stacks import lab_stacks
    return lab_stacks()[name]

def test_vms_no_dr_brings_up_site_a_only(fake_deps):
    steps = phases._vms_build_steps(_stack(), fake_deps, no_dr=True)
    argv = " ".join(a for s in steps for a in getattr(s.command, "argv", []))
    assert "nha-ubuntu-a1" in argv
    assert "nha-ubuntu-b1" not in argv

def test_vms_full_brings_up_both_sites(fake_deps):
    steps = phases._vms_build_steps(_stack(), fake_deps, no_dr=False)
    argv = " ".join(a for s in steps for a in getattr(s.command, "argv", []))
    assert "nha-ubuntu-b1" in argv

def test_provision_no_dr_emits_dr_enabled_false(fake_deps):
    steps = phases._provision_build_steps(_stack(), fake_deps, no_dr=True)
    provision = next(s for s in steps if s.label.endswith("provision"))
    assert "dr_enabled=false" in " ".join(provision.command.argv)

def test_provision_full_omits_dr_enabled(fake_deps):
    steps = phases._provision_build_steps(_stack(), fake_deps, no_dr=False)
    provision = next(s for s in steps if s.label.endswith("provision"))
    assert not any("dr_enabled" in a for a in provision.command.argv)

def test_observe_no_dr_limits_to_site_a(fake_deps):
    steps = phases._observe_build_steps(_stack(), fake_deps, no_dr=True)
    limit = next(s for s in steps if "observability.yml" in " ".join(s.command.argv))
    argv = " ".join(limit.command.argv)
    assert "nha-ubuntu-a1" in argv and "nha-ubuntu-b1" not in argv
```

*(Match `fake_deps` and the command-introspection style to the existing `tests/test_phases.py` fixtures — reuse them rather than inventing new ones.)*

- [ ] **Step 2: Run to verify failure**

Run: `uv run pytest tests/test_phases.py -k "no_dr or both_sites" -v`
Expected: FAIL (builders reject `no_dr`).

- [ ] **Step 3: Implement**

- Give each builder `*, no_dr: bool = False`. Change the shared member lookup so `vms`/`observe` use `stack_members_effective(stack.name, no_dr=no_dr)` instead of `stack_members(stack.name)`. For `observe`, that effective list is the `--limit nodes` value (spec §3.3 — `observability.yml` keys on `nha_ubuntu_b` membership, so the limit must be effective members).
- In `_provision_build_steps`, append `dr_enabled=false` to the provision ansible cmd under `no_dr`:

```python
    dr_vars = ["-e", "dr_enabled=false"] if no_dr else []
    cmd = Command(
        ["ansible-playbook", Path(stack.provision).name, *_qm_extra_vars(stack), *dr_vars],
        cwd=ansible,
    )
```

- `net` builder: add the keyword, ignore it (`# noqa: ARG001` as its siblings do).
- Run loop (~621) + `_bootstrap_run`: thread `no_dr` into `phase.build_steps(stack, deps, no_dr=no_dr)`.

- [ ] **Step 4: Run to verify pass**

Run: `uv run pytest tests/test_phases.py -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-git add src/mqlab/phases.py src/mqlab/cli.py tests/test_phases.py
vrg-commit --type feat --scope bootstrap --message "phases honor --no-dr: site-A members + dr_enabled var (#188)"
```

---

### Task 5: provision-playbook DR gates (`site-nativeha-ubuntu.yml`)

**Files:**
- Modify: `ansible/site-nativeha-ubuntu.yml` (the authz play `hosts:` ~line 58; the DR `import_playbook` ~line 241)

**Interfaces:**
- Consumes: the `dr_enabled` extra-var (Task 4). Default `true` preserves current behaviour exactly.

- [ ] **Step 1: Gate the authz play host pattern**

```yaml
- name: authz service accounts on every nativeha-ubuntu node (HA + DR)
  hosts: "nha_ubuntu_a{{ ':nha_ubuntu_b' if (dr_enabled | default(true) | bool) else '' }}"
```

- [ ] **Step 2: Gate the DR import**

```yaml
- import_playbook: _nativeha-ubuntu-dr-replication.yml
  when: dr_enabled | default(true) | bool
```

- [ ] **Step 3: Lint + a default-run parse check**

Run: `vrg-container-run -- vrg-validate` (ansible-lint covers the playbook). Confirm no syntax/lint regression; the default (`dr_enabled` unset → true) renders `hosts: nha_ubuntu_a:nha_ubuntu_b` and runs the DR import, unchanged.

- [ ] **Step 4: Commit**

```bash
vrg-git add ansible/site-nativeha-ubuntu.yml
vrg-commit --type feat --scope bootstrap --message "gate nativeha-ubuntu DR touch-points on dr_enabled (#188)"
```

---

### Task 6: full validation + report-ready (Wave 1 PR)

- [ ] **Step 1:** `vrg-container-run -- vrg-validate` — green (100% branch coverage, ruff, ansible-lint).
- [ ] **Step 2:** Manual sanity: `uv run mqlab bootstrap nativeha-ubuntu --no-dr --help` shows the flag; `uv run mqlab bootstrap <no-dr-groups-stack> --no-dr` errors loud.
- [ ] **Step 3:** `vrg-pr-workflow report-ready --issue <Wave-1 task #> --title … --summary … --notes …` (human runs `vrg-submit-pr`). PR into `develop`, `Closes` the Wave-1 implementation task.

---

## Wave 2 — the remaining HADR stacks (one task/PR each, repo `mq-resiliency-lab-for-linux`)

Each stack repeats the *stack-local* half of Wave 1 (the mechanism already exists):

- **Task W2-a — `pcmk-ubuntu`:** add `dr_groups: [<its site-B group>]` to its topology block; gate its provision playbook's site-B touch-points on `dr_enabled | default(true)`. Verify against the real playbook (`stack.provision`), which differs from nativeha's (pacemaker/SAN shape). Tests mirror Task 1/4 for this stack.
- **Task W2-b — `rdqm-rhel`** and **Task W2-c — `nativeha-rhel`:** same, per stack. **x86-only** (`rhel_stack_unsupported_reason` gates them on aarch64), so their acceptance runs on an x86 host; unit tests still run anywhere.

Each is `Blocked-by` the Wave-1 task (they consume its mechanism). File born-linked under the epic.

---

## Task 7 (operational) — cold-rebuild validation

- **Kind:** `validation` (run with `issue-validate`, closes on `Outcome: SUCCESS`). `Blocked-by` the Wave-1 task (needs it merged to `develop`).
- **Procedure:** on a cold lab, `mqlab commons up` then `mqlab bootstrap nativeha-ubuntu --no-dr`.
- **Acceptance:** exactly three QM guests (`nha-ubuntu-a1/a2/a3`), no `b*`; Native HA `QUORUM(3/3)` with an `Active`; HA + app/inter-QM MQSC applied; **no** DR replication play executed (grep the run for `_nativeha-ubuntu-dr-replication`); one pass, no manual intervention. Record SUCCESS/FAILURE per the scaffold.

---

## Closing bookends (already seeded)

- **#993 — documentation-review sweep** (member repo): reflect `--no-dr` in `docs/site/…` (operator bring-up runbook); spawn per-repo doc tasks as the sweep finds drift. Runs **before** the retrospective.
- **#190 — retrospective** (`.github`, terminal): authored with `epic-retrospective`; records the DR-add-later idempotency follow-on (spec §6). Its merge closes the epic.

---

## Self-review

- **Spec coverage:** §3.1 flag → Task 3; §3.2 `dr_groups`/effective members → Task 1-2; §3.3 phase table (incl. observe `--limit`) → Task 4; §3.4 provision gates → Task 5; §3.5 fail-loud guard → Task 3; §3.6 composition → Task 4 (shapes only phases run; no teardown); §4 waves → W2 + Task 7; §5 tests → Task 1/3/4; §5 acceptance → Task 7. No uncovered requirement.
- **Type consistency:** `dr_groups: list[str]`, `stack_dr_hosts(name)->list[str]`, `stack_members_effective(name,*,no_dr)->list[str]|None`, builders `(...,*,no_dr:bool=False)`, `_bootstrap_run(...,no_dr:bool=False)` — consistent across tasks.
- **Placeholders:** the one intentional lookup the implementer must confirm against live code is the exact fail-loud idiom in `cli.py` (Task 3 Step 3) and the `fake_deps`/command-introspection fixture names in `tests/test_phases.py` (Task 4) — both explicitly flagged to reuse existing patterns, not invent.
