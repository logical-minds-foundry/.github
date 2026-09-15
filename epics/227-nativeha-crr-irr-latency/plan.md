# Dual-mechanism Native HA (CRR async vs IRR strict-sync), latency-tunable — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the RHEL Native HA arm from one queue manager to two parallel independent stacks — CRR (async) and IRR (strict-sync) — add a `tc netem` cross-region latency knob, and build a purpose-built benchmark harness that quantifies the sync-vs-async cost as a curve.

**Architecture:** Two independent Native HA stacks (12 nodes total) that differ only by names and a single `SyncConsistency` config value. The shared `ansible/roles/mq-nativeha/` role already takes zero hardcoded literals; the CRR↔IRR difference is one conditional line in `crr.yml` driven by a new `replication_mode` var. The composing playbooks (which today hardcode group names and WAN IPs) are parameterized so **one** provision/switchover pair drives both stacks. A thin `mqlab netem` Typer verb shells a `tc` script on the libvirt host (`virbr-wan` only). A new benchmark client + a `mqlab bench` sweep verb produce a structured results artifact and a comparison report.

**Tech Stack:** IBM MQ 10.0 Native HA (CRR/IRR); Ansible (roles + composing playbooks); Python 3.14 / Typer (`mqlab` CLI); Vagrant + libvirt (VMs); `tc netem` (latency); Prometheus textfile metrics; pytest (100% branch coverage on `src/`).

**Spec:** `epics/227-nativeha-crr-irr-latency/spec.md` (in this repo, `logical-minds-foundry/.github`). Executors read the spec and this plan together.

## Global Constraints

- **MQ version:** single-source pin `lab/mq-version` = `10.0.0.0`. Never hardcode a version; read the pin. Role floor asserts `>= 9.4.4`; raise/gate only if Task 1 proves IRR needs a higher floor.
- **Substrate:** RHEL Native HA on **cloud x86** (native MQ, no emulation). RHEL stacks require x86_64 (`rhel_stack_unsupported_reason`); do not attempt on aarch64.
- **100% branch coverage on `src/`** is enforced by validation (`pytest --cov=src --cov-branch --cov-fail-under=100`). Every new Python branch needs a test. (`clients/` is **not** under `--cov=src`.)
- **Validation is `vrg-container-run -- vrg-validate` only.** It runs: markdownlint, shellcheck, yamllint, actionlint, ansible-lint, `ruff check`/`ruff format --check` on `src/ tests/`, `mypy src/`, `ty check src tests`, the pytest line above, and the dependency audit. Do not run individual linters as the acceptance signal.
- **Git/GitHub:** use `vrg-git` / `vrg-gh` / `vrg-commit` only. Commit from inside the worktree.
- **No hardcoded `build/<X>` paths** — use `mqlab build path <bucket>`.
- **`async` is byte-for-byte the current CRR** — the refactor must not change the rendered `qm.ini` or behaviour for the CRR stack (regression-guarded).
- **Naming:** CRR stack `nativeha-rhel-crr` / short `NHARC` / QM `NHARCAPP`; IRR stack `nativeha-rhel-irr` / short `NHARI` / QM `NHARIAPP`. QM names derive from `short` — never hardcode a QM literal in new code.
- **Do not bluff IRR mechanics.** Where the exact IRR stanza/port/`SyncConsistency` placement or version floor is not yet confirmed, it is Task 1's output; downstream tasks apply what Task 1 recorded, not a guess.

---

## Prerequisite gate (human/ops, before Task 5 bring-up, Task 8, and the validation)

**Resize the cloud VM to carry 12 nodes (≥64 GiB target).** The current box runs 6 nodes comfortably; two full stacks need roughly double. This is a manual platform action performed *after* the code lands and *before* the two-stack bring-up / experiment / cold-rebuild validation. Tasks 1–4, 6, and 7 (code + unit tests) do **not** need the larger VM; Task 5's live bring-up, Task 8's experiment, and validation `#1098` do.

---

## File Structure

**Ansible (shared role — one conditional line):**

- Modify: `ansible/roles/mq-nativeha/tasks/crr.yml` — add `replication_mode`-gated `SyncConsistency=Strict` to the `NativeHARecoveryGroup` block.
- Modify: `ansible/roles/mq-nativeha/tasks/main.yml` — version-floor gate (only if Task 1 requires).

**Ansible (composing playbooks — parameterize, don't fork):**

- Modify: `ansible/site-nativeha.yml`, `ansible/_nativeha-cluster-ha.yml`, `ansible/_nativeha-dr-replication.yml`, `ansible/site-nativeha-switchover.yml` — de-hardcode live/recovery group names and `peer_wan_addrs`; drive them from stack-scoped extra-vars.

**Topology + CLI wiring:**

- Modify: `lab/topology.yaml` — full CRR rename (stack/short/QM + `nha_rhel_crr_a/b` groups + `nha-rhel-crr-*` nodes); add 6 IRR nodes, `nha_rhel_irr_a/b` groups, the `nativeha-rhel-irr` stack entry.
- Modify: `src/mqlab/cli.py` — new `netem` and `bench` Typer sub-apps; pass the stack's live/recovery groups + `replication_mode` as extra-vars to the provision playbook.
- Modify: `src/mqlab/parity.py` — capability-matrix rows for the two stacks.
- Modify: `src/mqlab/clusterboard.py` — Grafana board selectors for the two stacks.
- Modify: `ansible/observability.yml` — group-membership conditionals + QM fallback for the new groups.

**New host script + client + tests:**

- Create: `lab/scripts/net-latency.sh` — the `tc netem` recipe on `virbr-wan` (set/clear/show).
- Create: `clients/bench_client.py` — purpose-built benchmark client.
- Create: `lab/scripts/bench-sweep.sh` — orchestrates a `{mode}×{latency}` sweep, emits the results artifact.
- Create: `tests/test_cli_netem.py`, `tests/test_cli_bench.py`, and extend `tests/test_stacks.py`.

---

## Task 1: IRR-setup facts spike (gates the build)

**Kind:** investigation → a facts report committed as a doc. No production code. Blocks Tasks 3, 4, 5.

**Files:**

- Create: `docs/reports/<YYYY-MM-DD>-mq10-nativeha-irr-setup-facts-spike.md`
- Cache (gitignored): `build/refs/ibm-docs/ibm-mq/10.0.x/<slug>/` via `tools/ibm_doc_cache.py`

**Deliverable:** a report that answers, each with a cited IBM 10.0 source (`content.txt` + `source_url` from `meta.json`), the questions the build depends on:

- [ ] **Step 1: Cache the IRR *setup/config* reference(s).** The lab has the IRR *upgrade* doc but not setup. Run, for each relevant page:

```bash
python3 tools/ibm_doc_cache.py "https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=..."
```

  Target the Native HA IRR configuration/creation pages and the `qm.ini` Native HA stanza reference.

- [ ] **Step 2: Record the exact IRR config delta vs CRR.** Answer in the report:
  1. Is IRR configured with the same `NativeHALocalInstance` / `NativeHARecoveryGroup` stanzas as CRR, plus `SyncConsistency=Strict`? **Which stanza does `SyncConsistency` belong in, and what is its exact key/values?**
  2. Does IRR use the **same replication port 9415**, or a different one?
  3. Does strict-sync impose a **start/stop ordering** constraint (e.g. a Strict Live instance refusing to start, or blocking commits, when the Recovery group is unreachable)? This determines whether the `nha_start` Recovery-first ordering is sufficient or needs augmenting.
  4. Is there a **minimum MQ version** for IRR strict-sync above the current 9.4.4 floor?
  5. Is IRR available in the lab's **IBM MQ Advanced for Developers** edition (the tarball `install-RedHat.yml` installs)? *(Low risk — CRR already runs on this edition — but confirm and cite.)*

- [ ] **Step 3: Write the report** mirroring `docs/reports/2026-09-14-mq10-nativeha-crr-upgrade-facts-spike.md` structure (cached-sources table, terminology-pinned section, findings). State plainly where IBM's docs are silent.

- [ ] **Step 4: Validate + commit.**

```bash
vrg-container-run -- vrg-validate
cd <worktree> && vrg-git add docs/reports/<file>.md && \
  vrg-commit --type docs --scope report --message "IRR setup facts spike — MQ 10.0 Native HA in-region replication (#227)"
```

**Acceptance:** every Step-2 question is answered with a cited IBM 10.0 source, or explicitly marked "docs silent — decide empirically in Task N". Downstream tasks reference this report by path.

---

## Task 2: Rename the CRR stack identity (`nativeha-rhel` → `nativeha-rhel-crr`, `NHAR` → `NHARC`)

**Rationale:** give the existing arm an explicit CRR identity paired with the IRR stack — a **full rename**: stack key, `short`, QM, **and** node/group names. Renaming the nodes/groups serves the **arm-coexistence constraint** (each arm's nodes are uniquely named so CRR and IRR can run together — which this epic exercises); it is not gratuitous churn. The rename is **atomic** — every reference moves in this one task, including the composing playbooks' still-hardcoded group literals, so the CRR stack works after Task 2; Task 4 later replaces those literals with parameterized extra-vars. `async` behaviour is byte-for-byte unchanged.

**Files:**

- Modify: `lab/topology.yaml` — stack key `nativeha-rhel`→`nativeha-rhel-crr`, `short` NHAR→NHARC, `alloc.app_unit` app-nhar→app-nhar-crr, `cluster_group`/`groups`/`dr_groups` and the group definitions `nha_rhel_a/b`→`nha_rhel_crr_a/b`, and the 6 node entries `nha-rhel-{a,b}{1,2,3}`→`nha-rhel-crr-*` (keep their existing IPs)
- Modify: `ansible/site-nativeha.yml`, `ansible/_nativeha-cluster-ha.yml`, `ansible/_nativeha-dr-replication.yml`, `ansible/site-nativeha-switchover.yml` — group literals `nha_rhel_a/b`→`nha_rhel_crr_a/b` (in `hosts:`/`delegate_to`/preflight), node names in the inline `nha_site_nodes`, and `qm_app | default('NHARAPP')`→`default('NHARCAPP')`
- Modify: `src/mqlab/parity.py:43` (`"nativeha-rhel"`→`"nativeha-rhel-crr"`)
- Modify: `src/mqlab/clusterboard.py` (board map + `nha_rhel_a/b` / `NHARAPP` selectors)
- Modify: `ansible/observability.yml:17-44` (group conditionals `nha_rhel_a/b`→`nha_rhel_crr_a/b` + `NHARAPP` fallback→`NHARCAPP`)
- Test: `tests/test_stacks.py`

- [ ] **Step 1: Write the failing test.** In `tests/test_stacks.py`, extend the seeded `TOPO` literal to key the stack `nativeha-rhel-crr` with `short: NHARC`, and assert the derived names:

```python
def test_crr_stack_names(monkeypatch, tmp_path):
    monkeypatch.setenv("MQLAB_REPO_ROOT", str(tmp_path))
    _seed(tmp_path)  # TOPO now keys the stack 'nativeha-rhel-crr', short 'NHARC'
    stacks = lab_stacks()
    assert "nativeha-rhel-crr" in stacks
    assert stacks["nativeha-rhel-crr"].short == "NHARC"
    assert stacks["nativeha-rhel-crr"].qm.qm_app == "NHARCAPP"
    assert stacks["nativeha-rhel-crr"].groups == ["nha_rhel_crr_a", "nha_rhel_crr_b"]
```

- [ ] **Step 2: Run it — expect FAIL** (`KeyError: 'nativeha-rhel-crr'`):

```bash
vrg-container-run -- pytest tests/test_stacks.py::test_crr_stack_names -v
```

- [ ] **Step 3: Rename in `lab/topology.yaml`.** Stack key → `nativeha-rhel-crr`, `short` → `NHARC`, `alloc.app_unit` → `app-nhar-crr`; `cluster_group`/`groups`/`dr_groups` and the group definitions `nha_rhel_a/b` → `nha_rhel_crr_a/b`; the 6 node entries `nha-rhel-{a,b}{1,2,3}` → `nha-rhel-crr-*` (keep their existing IPs — only the names change).

- [ ] **Step 4: Update every remaining reference atomically** (so the CRR stack still works after this task):
  - **Composing playbooks** — `ansible/site-nativeha.yml`, `_nativeha-cluster-ha.yml`, `_nativeha-dr-replication.yml`, `site-nativeha-switchover.yml`: group literals `nha_rhel_a/b` → `nha_rhel_crr_a/b` (in `hosts:`, `delegate_to`, and the `groups['nha_rhel_a'][0]` preflight), the node names in the inline `nha_site_nodes`, and `qm_app | default('NHARAPP')` → `default('NHARCAPP')`. (Task 4 later replaces these group literals with extra-vars.)
  - `src/mqlab/parity.py:43`: key `"nativeha-rhel"` → `"nativeha-rhel-crr"`.
  - `src/mqlab/clusterboard.py`: board-map key → `"nativeha-rhel-crr"`; `nha_rhel_a/b` selectors → `nha_rhel_crr_a/b`; `NHARAPP` → `NHARCAPP` (prefer deriving from the stack).
  - `ansible/observability.yml:17-44`: group conditionals `nha_rhel_a/b` → `nha_rhel_crr_a/b`; `NHARAPP` fallback → `NHARCAPP`.

- [ ] **Step 5: Run the full unit suite — expect PASS** (catch every other reference via coverage):

```bash
vrg-container-run -- pytest -q
```

- [ ] **Step 6: Grep for stragglers** and fix any remaining `NHARAPP` / `"nativeha-rhel"` literals that should be the CRR names (skip docs/history):

```bash
grep -rn "NHARAPP\|nativeha-rhel\b" src/ ansible/ lab/ | grep -v nativeha-rhel-
```

- [ ] **Step 7: Validate + commit.**

```bash
vrg-container-run -- vrg-validate
vrg-commit --type refactor --scope nativeha --message "rename the CRR Native HA stack to nativeha-rhel-crr / NHARCAPP (#227)"
```

**Acceptance:** `mqlab status` / `mqlab qm ...` resolve `nativeha-rhel-crr` with QM `NHARCAPP`; validation green; no orphaned `NHARAPP` reference outside docs/history.

---

## Task 3: `replication_mode` in the shared role (the one-line CRR↔IRR difference)

**Files:**

- Modify: `ansible/roles/mq-nativeha/tasks/crr.yml:29-41`
- Modify: `ansible/roles/mq-nativeha/tasks/main.yml:18-28` (version floor — only if Task 1 requires)

- [ ] **Step 1: Add the `replication_mode` var, defaulting to `async`.** In `ansible/roles/mq-nativeha/defaults/main.yml` add:

```yaml
# Replication mode for the cross-region/in-region group relationship.
# 'async' = CRR (today's behaviour, byte-for-byte). 'strict' = IRR (SyncConsistency=Strict).
replication_mode: async
```

- [ ] **Step 2: Gate `SyncConsistency=Strict` in `crr.yml`.** In the `NativeHARecoveryGroup` block (`crr.yml:33-41`), append the strict line **only** when `replication_mode == 'strict'`, using the exact key/placement Task 1 confirmed. Best-known form (replace with Task 1's confirmed form):

```yaml
      NativeHARecoveryGroup:
         GroupName={{ peer_group_name }}
         ReplicationAddress={% for a in peer_wan_addrs %}{{ a }}(9415){{ "," if not loop.last }}{% endfor %}

         Enabled={{ recovery_enabled }}
{% if replication_mode == 'strict' %}
         SyncConsistency=Strict
{% endif %}
```

- [ ] **Step 3: Verify `async` is byte-for-byte unchanged.** Render the block both ways and confirm `async` matches the pre-change output exactly. Since the role is not unit-tested in `src/`, verify by rendering the template locally:

```bash
# from ansible/, render crr.yml's blockinfile content with replication_mode=async and =strict,
# diff async output against git HEAD's rendered output — async must be identical.
```

  Document the two rendered blocks in the commit body as evidence.

- [ ] **Step 4: Version floor (conditional on Task 1).** If Task 1 found IRR needs MQ > 9.4.4, gate `main.yml`'s assertion on `replication_mode` (strict → higher floor). If not, leave `main.yml` unchanged and note "no floor change needed per Task 1".

- [ ] **Step 5: Validate + commit.**

```bash
vrg-container-run -- vrg-validate   # ansible-lint + yamllint cover this change
vrg-commit --type feat --scope nativeha --message "parameterize Native HA replication mode (async CRR | strict IRR) (#227)"
```

**Acceptance:** `ansible-lint` green; `async` render identical to HEAD (evidence in commit body); `strict` adds exactly the `SyncConsistency` line Task 1 confirmed. Behavioural proof is deferred to Task 5 bring-up + validation `#1098`.

---

## Task 4: Parameterize the composing playbooks (one codebase drives both stacks)

**Rationale:** today `site-nativeha.yml` / `_nativeha-cluster-ha.yml` / `_nativeha-dr-replication.yml` / `site-nativeha-switchover.yml` hardcode `nha_rhel_a` / `nha_rhel_b` in `hosts:`/`delegate_to`/preflight, inline `nha_site_nodes` with literal IPs, and hardcode `peer_wan_addrs`. To serve both stacks from one codebase (spec §3.3, binding decision #4), drive these from stack-scoped extra-vars supplied by `mqlab`.

**Files:**

- Modify: `ansible/site-nativeha.yml`, `ansible/_nativeha-cluster-ha.yml`, `ansible/_nativeha-dr-replication.yml`, `ansible/site-nativeha-switchover.yml`
- Modify: `src/mqlab/cli.py` (pass the extra-vars) + wherever the provision step is built (`src/mqlab/phases.py`)
- Test: `tests/test_cli_bootstrap.py` (assert the extra-vars are passed)

- [ ] **Step 1: Define the extra-vars contract.** The provision/switchover playbooks consume:
  - `nha_live_group` (e.g. `nha_rhel_a`), `nha_recovery_group` (e.g. `nha_rhel_b`)
  - `nha_recovery_wan_addrs`, `nha_live_wan_addrs` (lists, from topology `net-wan` NICs of the recovery/live nodes)
  - `replication_mode` (`async` | `strict`)

  Build these in `mqlab` from the stack definition (`stack.groups`, node NICs) and pass via `-e`.

- [ ] **Step 2: Write the failing test.** In `tests/test_cli_bootstrap.py`, assert the provision command carries the group + mode extra-vars for the CRR stack:

```python
def test_bootstrap_passes_replication_extravars(monkeypatch, tmp_path):
    _seed(monkeypatch, tmp_path)  # topology with nativeha-rhel-crr
    runner = RecordingRunner(results=[ScriptedResult([])] * 8)
    monkeypatch.setattr(cli, "build_deps", lambda verb, ts: _deps(runner))
    result = CliRunner().invoke(cli.app, ["bootstrap", "nativeha-rhel-crr", "--no-dr"])
    assert result.exit_code == 0
    provision = next(c for c in runner.recorded if "site-nativeha.yml" in " ".join(c.argv))
    joined = " ".join(provision.argv)
    assert "replication_mode=async" in joined
    assert "nha_live_group=nha_rhel_crr_a" in joined  # post-Task-2 rename
```

- [ ] **Step 3: Run it — expect FAIL** (extra-vars not present):

```bash
vrg-container-run -- pytest tests/test_cli_bootstrap.py::test_bootstrap_passes_replication_extravars -v
```

- [ ] **Step 4: Implement the extra-vars in `mqlab`.** Where the provision `CommandStep` is built (phase `build_steps`), derive and append `-e replication_mode=<stack mode> -e nha_live_group=<...> -e nha_recovery_group=<...> -e nha_recovery_wan_addrs=<json> -e nha_live_wan_addrs=<json>` from the stack. Add a `replication_mode` field to the `Stack`/topology model (default `async`) and read it in `stacks.py` (mirror how `mechanism`/`os` are read).

- [ ] **Step 5: De-hardcode the playbooks.** Replace literal `nha_rhel_a`/`nha_rhel_b` with `"{{ nha_live_group }}"`/`"{{ nha_recovery_group }}"` in `hosts:`/`delegate_to`/preflight; build `nha_site_nodes` from `groups[nha_live_group]` + hostvars (hb/data NICs) instead of inline literals; replace the literal `peer_wan_addrs` with `nha_recovery_wan_addrs`/`nha_live_wan_addrs`. Keep the Recovery-first-then-Live restart ordering.

- [ ] **Step 6: Run the CRR regression.** `pytest -q` green; then render-diff the CRR provision to confirm identical `qm.ini` output vs HEAD (async unchanged).

- [ ] **Step 7: Validate + commit.**

```bash
vrg-container-run -- vrg-validate
vrg-commit --type refactor --scope nativeha --message "drive Native HA composing playbooks from stack-scoped extra-vars (#227)"
```

**Acceptance:** `bootstrap nativeha-rhel-crr` passes `replication_mode=async` + the derived group/WAN vars; the CRR `qm.ini` render is unchanged; validation green.

---

## Task 5: Add the IRR stack (`nativeha-rhel-irr` / `NHARI` / `NHARIAPP`)

**Prerequisite:** Tasks 1, 3, 4 landed. Live bring-up needs the resized VM (prerequisite gate).

**Files:**

- Modify: `lab/topology.yaml` — 6 new nodes (`nha-rhel-irr-a1..3`, `-b1..3`) on a fresh octet block (`.7x` free), `nha_rhel_irr_a/b` groups, the `nativeha-rhel-irr` stack entry (`replication_mode: strict`, `short: NHARI`, `provision: ansible/site-nativeha.yml`, `alloc.exporter_app_port: 9165`, `app_unit: app-nhar-irr`, reuse box `mq-nativeha-rhel9`).
- Modify: `src/mqlab/parity.py`, `src/mqlab/clusterboard.py`, `ansible/observability.yml` (per Task-2 pattern, add the IRR groups/QM).
- Test: `tests/test_stacks.py`

- [ ] **Step 1: Write the failing test.** Assert the IRR stack resolves with strict mode and its own names:

```python
def test_irr_stack(monkeypatch, tmp_path):
    monkeypatch.setenv("MQLAB_REPO_ROOT", str(tmp_path))
    _seed(tmp_path)  # TOPO now includes nativeha-rhel-irr
    s = lab_stacks()["nativeha-rhel-irr"]
    assert s.short == "NHARI"
    assert s.qm.qm_app == "NHARIAPP"
    assert s.replication_mode == "strict"
    assert s.groups == ["nha_rhel_irr_a", "nha_rhel_irr_b"]
```

- [ ] **Step 2: Run it — expect FAIL:**

```bash
vrg-container-run -- pytest tests/test_stacks.py::test_irr_stack -v
```

- [ ] **Step 3: Add the 6 IRR nodes to `lab/topology.yaml`** on a fresh octet block, mirroring the `nha-rhel-a1`/`b1` shape. Both stacks attach to the **same** `net-wan` (10.99.0.0/24) with distinct IPs (e.g. a-nodes `10.99.0.71/.72/.73`, b-nodes `10.99.0.74/.75/.76`) so one netem knob shapes both. Example:

```yaml
  nha-rhel-irr-a1:
    platform: mq-nativeha-rhel9
    cpus: 2
    memory: 2048
    nics: { net-mgmt: 10.50.0.71, net-data-a: 10.10.1.71, net-hb-a: 172.16.1.71, net-wan: 10.99.0.71, net-ext: 10.60.0.71 }
  # ... a2/a3, b1/b2/b3 following the site-B shape (no net-ext on b-nodes)
```

- [ ] **Step 4: Add the groups + stack entry:**

```yaml
  nha_rhel_irr_a: [nha-rhel-irr-a1, nha-rhel-irr-a2, nha-rhel-irr-a3]
  nha_rhel_irr_b: [nha-rhel-irr-b1, nha-rhel-irr-b2, nha-rhel-irr-b3]
```

```yaml
  nativeha-rhel-irr:
    mechanism: native-ha
    os: rhel
    short: NHARI
    replication_mode: strict
    cluster_group: nha_rhel_irr_a
    groups: [nha_rhel_irr_a, nha_rhel_irr_b]
    dr_groups: [nha_rhel_irr_b]
    provision: ansible/site-nativeha.yml
    secrets: [mqweb_admin_password]
    qm: {}
    alloc: { exporter_app_port: 9165, app_unit: app-nhar-irr, svc_port: 1414 }
    verbs: { ...mirror nativeha-rhel-crr, qm_app default NHARIAPP... }
```

- [ ] **Step 5: Extend the hardcoded touch-points** (parity.py row, clusterboard.py board + selectors for `nha_rhel_irr_*` / `NHARIAPP`, observability.yml group conditionals + QM fallback).

- [ ] **Step 6: Run unit suite — expect PASS:** `vrg-container-run -- pytest -q`.

- [ ] **Step 7: Validate + commit** (code only; live bring-up in Step 8):

```bash
vrg-container-run -- vrg-validate
vrg-commit --type feat --scope nativeha --message "add the IRR Native HA stack (nativeha-rhel-irr / NHARIAPP, strict-sync) (#227)"
```

- [ ] **Step 8: Live bring-up smoke (needs resized VM).** After the VM resize, bring up both stacks and confirm the IRR pair forms strict-sync:

```bash
mqlab bootstrap nativeha-rhel-crr
mqlab bootstrap nativeha-rhel-irr
mqlab qm status nativeha-rhel-irr    # dspmq -o nativeha shows healthy IRR state
```

  Confirm a persistent PUT to `NHARIAPP` is present on the Recovery group before commit returns (the strict-sync sanity check — one observation, not the swept RPO proof). Record the transcript on the epic.

**Acceptance:** both stacks resolve and bring up; `NHARIAPP` forms a strict-sync Live+Recovery pair; an app reaches `NHARCAPP` and `NHARIAPP` by name; validation green.

---

## Task 6: `mqlab netem` delay knob

**Files:**

- Create: `lab/scripts/net-latency.sh` (the `tc` recipe; owns the attach-point decision)
- Modify: `src/mqlab/cli.py` (new `netem` Typer sub-app)
- Test: `tests/test_cli_netem.py`

- [ ] **Step 1: Write `lab/scripts/net-latency.sh`** — determine the working attach point empirically (root qdisc on `virbr-wan`, per-tap `netem`, or `ifb` for ingress) and encapsulate it. Interface: `net-latency.sh set <delay>` / `clear` / `show`; target the cross-region plane only; never touch `virbr-hb-*`. Verify symmetric RTT with a probe (`ping` across `virbr-wan`) — one-way `<delay>` → RTT ≈ 2×`<delay>`. Pass `shellcheck` (part of validation).

- [ ] **Step 2: Write the failing test.** Mirror `tests/test_cli_dr.py`'s RecordingRunner pattern:

```python
def test_netem_set_shells_script(monkeypatch, tmp_path):
    _seed(monkeypatch, tmp_path)
    runner = RecordingRunner(results=[ScriptedResult([])])
    monkeypatch.setattr(cli, "build_deps", lambda verb, ts: _deps(runner))
    result = CliRunner().invoke(cli.app, ["netem", "set", "--delay", "10ms"])
    assert result.exit_code == 0
    cmd = runner.recorded[-1]
    assert cmd.argv[0] == "bash"
    assert cmd.argv[1].endswith("/lab/scripts/net-latency.sh")
    assert cmd.argv[2:] == ["set", "10ms"]

def test_netem_clear_and_show(monkeypatch, tmp_path):
    # analogous: ["netem","clear"] -> [...,"clear"]; ["netem","show"] -> [...,"show"]
    ...

def test_netem_script_failure_propagates(monkeypatch, tmp_path):
    _seed(monkeypatch, tmp_path)
    runner = RecordingRunner(results=[ScriptedResult([], exit_code=7)])
    monkeypatch.setattr(cli, "build_deps", lambda verb, ts: _deps(runner))
    result = CliRunner().invoke(cli.app, ["netem", "set", "--delay", "10ms"])
    assert result.exit_code == 7
```

- [ ] **Step 3: Run — expect FAIL** (`No such command 'netem'`):

```bash
vrg-container-run -- pytest tests/test_cli_netem.py -v
```

- [ ] **Step 4: Implement the `netem` sub-app** in `src/mqlab/cli.py`, mirroring `pki`/`dr`:

```python
netem_app = typer.Typer(help="inject WAN latency on the cross-region plane (tc netem)", no_args_is_help=True)
app.add_typer(netem_app, name="netem")
_NET_LATENCY_SCRIPT = "net-latency.sh"

def _netem(action: str, *args: str) -> None:
    argv = ["bash", str(lab_script(_NET_LATENCY_SCRIPT)), action, *args]
    _execute(f"netem-{action}", [CommandStep(f"netem {action}", Command(argv))], step_mode=False)

@netem_app.command("set")
def netem_set(delay: str = typer.Option(..., "--delay", help="one-way delay, e.g. 10ms")) -> None:
    _netem("set", delay)

@netem_app.command("clear")
def netem_clear() -> None:
    _netem("clear")

@netem_app.command("show")
def netem_show() -> None:
    _netem("show")
```

- [ ] **Step 5: Run — expect PASS** (all branches; 100% coverage):

```bash
vrg-container-run -- pytest tests/test_cli_netem.py -v && vrg-container-run -- pytest --cov=src --cov-branch --cov-fail-under=100 -q
```

- [ ] **Step 6: Validate + commit.**

```bash
vrg-container-run -- vrg-validate
vrg-commit --type feat --scope netem --message "add mqlab netem: tunable WAN latency on the cross-region plane (#227)"
```

**Acceptance:** `mqlab netem set --delay 10ms` raises cross-site RTT to ≈20 ms **measured both directions by probe** (not merely `tc qdisc show`) and leaves `virbr-hb-*` untouched; `clear` restores `fq_codel`; script failure surfaces as the CLI's non-zero exit; validation green.

---

## Task 7: Purpose-built benchmark client

**Files:**

- Create: `clients/bench_client.py` (not under `--cov=src`; still unit-test its pure logic)
- Modify: `ansible/roles/app-requester/` or a new `bench` deploy path (deploy the client to `app-client`)
- Test: `tests/test_bench_client.py` (pure-logic tests)

- [ ] **Step 1: Design the client.** Reuse the isolated pieces of `clients/app_requester.py` — `RoundTripStats`, `render_prom`, `write_textfile`/`publish_metrics`, and the `_RECONNECT_REASONS` reconnect logic. Add: warmup phase, fixed-N persistent messages under syncpoint, commit-latency percentile capture (p50/p95/p99/max), steady-state throughput measurement, and a **structured results writer** (JSONL: one record per `{mode, delay, msg_size, offered_rate}` point with achieved rate, depth-trend flag, and the percentiles). CLI knobs: `--qm`, `--conn`, `--channel`, `--msg-size`, `--offered-rate`, `--warmup-seconds`, `--measure-seconds`, `--count`, `--results <path>`, TLS opts.

- [ ] **Step 2: Write failing tests for the pure logic** (percentile math + JSONL record shape), e.g.:

```python
def test_percentiles_from_samples():
    from clients.bench_client import percentiles
    p = percentiles([1,2,3,...,100], (50,95,99))
    assert p[50] == ...  # exact expected
```

- [ ] **Step 3: Run — expect FAIL; Step 4: implement; Step 5: run — PASS.**

```bash
vrg-container-run -- pytest tests/test_bench_client.py -v
```

- [ ] **Step 6: Deploy path.** Add an Ansible task/unit to install `bench_client.py` on `app-client` (mirror `app-requester` role), parameterized by target QM (`NHARCAPP` | `NHARIAPP`). Do **not** entangle it with the always-on `app-requester` stream.

- [ ] **Step 7: Validate + commit.**

```bash
vrg-container-run -- vrg-validate
vrg-commit --type feat --scope bench --message "purpose-built Native HA benchmark client (persistent throughput/latency, percentiles) (#227)"
```

**Acceptance:** the client runs a controlled warmup+measure cycle against a QM by name, emits the JSONL results record and the Prometheus textfile; percentile math unit-tested; validation green.

---

## Task 8: Sweep runner + first-pass experiment + comparison report

**Prerequisite:** Tasks 5–7 landed; resized VM; both stacks up.

**Files:**

- Create: `lab/scripts/bench-sweep.sh` (orchestrates the matrix; heavy logic here, not `--cov=src`)
- Modify: `src/mqlab/cli.py` (thin `bench` Typer verb shelling the sweep script)
- Create: `docs/reports/<YYYY-MM-DD>-crr-vs-irr-latency-comparison.md` (the report) + the JSONL results artifact alongside it in `docs/reports/`
- Test: `tests/test_cli_bench.py`

- [ ] **Step 1: Write `lab/scripts/bench-sweep.sh`** — for each `mode ∈ {crr, irr}` and each one-way `delay ∈ {0, 5, 10, 20, 40} ms` (alignment defaults; tunable in a dry-run): `mqlab netem set --delay <d>`, run `bench_client.py` against the mode's QM with **2 KiB persistent** messages, **30 s warmup / 120 s steady-state measure**, percentiles at **80 % of the measured ceiling**, append the JSONL record, then `mqlab netem clear`. Both stacks may be measured in parallel (no contention) or one-at-a-time — a flag.

- [ ] **Step 2–5: TDD the thin `bench` verb** (mirror Task 6): `mqlab bench sweep [--parallel] [--out <path>]` shells `bench-sweep.sh`; assert argv + failure propagation via RecordingRunner; 100% coverage.

- [ ] **Step 6: Run the first-pass experiment** on the resized lab; collect the JSONL results artifact.

- [ ] **Step 7: Write the comparison report** at `docs/reports/<YYYY-MM-DD>-crr-vs-irr-latency-comparison.md` — the headline deliverable. Plot/tabulate IRR-vs-CRR sustained-throughput ceiling and commit-latency percentiles vs injected latency; identify the knee (if within 0–40 ms) where strict-sync falls off; state the fidelity claim (relative, native x86, one representative size/rate — first pass, not the full surface). Data vs judgment clearly separated.

- [ ] **Step 8: Validate + commit** the report + results.

```bash
vrg-container-run -- vrg-validate
vrg-commit --type feat --scope bench --message "sweep runner + first-pass CRR-vs-IRR latency comparison report (#227)"
```

**Acceptance:** the sweep produces the results artifact and the written comparison report showing the IRR-vs-CRR overhead across the latency sweep — reported as an outcome, not a threshold.

---

## Operational task: cold-rebuild validation (`mq-resiliency-lab-for-linux#1098`)

Not PR-workable — run via `issue-validate` after Tasks 5/8 merge and the VM is resized. A full cold rebuild of both stacks on cloud x86 comes up green in one pass; `NHARCAPP` (async) and `NHARIAPP` (strict-sync) both form, both reachable by name, the `mqlab netem` knob measurably shapes `virbr-wan` and not the HA nets. `Outcome: SUCCESS` closes it. Blocked-by the implementation tasks.

---

## Task → spec mapping & sequencing

- Task 1 (spike) → spec §9.1; gates 3/4/5.
- Task 2 (rename) → spec §9.2. Independent; can run in parallel with Task 1.
- Task 3 (role mode) → spec §9.3; needs Task 1.
- Task 4 (playbook parameterize) → spec §9.3; needs Task 2.
- Task 5 (IRR stack) → spec §9.4; needs 1, 3, 4 + resized VM for bring-up.
- Task 6 (netem) → spec §9.5. Independent of the stack work; can run any time.
- Task 7 (bench client) → spec §9.6. Independent.
- Task 8 (sweep + report) → spec §9.7; needs 5, 6, 7 + resized VM.
- Validation `#1098` → needs 5, 8, resized VM.

## Self-review notes

- **Spec coverage:** every spec §9 task maps to a plan task (above); the netem goal-not-mechanism decision is Task 6 Step 1; the throughput-ceiling definition (spec §3.5) is applied in Task 7/8; the "async byte-for-byte" constraint is a gate in Tasks 3 & 4.
- **Spike-gated honesty:** the exact IRR `SyncConsistency` placement/port/floor is Task 1 output; Tasks 3/5 apply what Task 1 records, with the best-known form marked for replacement.
- **Coverage constraint:** `src/` changes (Tasks 2,4,5,6,8) carry full pytest branch coverage; `clients/`+`lab/scripts/`+`ansible/` are validated by lint + render-diff + the cold-rebuild `#1098`.
- **Open for alignment:** benchmark numbers (msg-size, offered-rate, sweep points, steady-state window, the ~80% percentile-report fraction) — deliberately unfixed here (spec §6).
