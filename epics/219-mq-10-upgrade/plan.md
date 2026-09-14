# MQ 9.4.5 → 10.0 Native HA CRR Upgrade — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver a tested, documented IBM MQ 9.4.5 → 10.0 rolling-upgrade runbook for a Native HA cross-region (CRR) cluster, and collapse the scattered MQ version pins into one authoritative source so the upgrade is a one-value change.

**Architecture:** Three workstreams. (A) A single plain-text MQ version pin at `lab/mq-version`, read by the Python module, the shell fetcher, and Ansible — killing the hand-synced literals and the stale SUT manifests. (B) A generic, anonymized CRR upgrade runbook grounded in IBM docs (Recovery-site-first; replicas-first/active-last; quiesce-and-assert; honest back-out; post-upgrade DR failover/failback validation). (C) A manual, AI-executed test of the runbook against the full 6-node RHEL CRR topology, cold-rebuilt at 9.4.5 and upgraded to 10.0, captured as an evidence report; then the lab default is rebaselined to 10.0 and cold-rebuilt from zero.

**Tech Stack:** Python 3.14 (`mqlab`, tested with `uv run pytest`), Bash, Ansible, IBM MQ Advanced for Developers (Native HA + CRR), RHEL 9.6, Vagrant/libvirt lab.

**Spec:** `epics/219-mq-10-upgrade/spec.md` (in `logical-minds-foundry/.github`). The plan argues from the spec; executors read both.

## Global Constraints

- **MQ-only upgrade.** MQ 10.0 supports RHEL 9; the lab runs RHEL 9.6 → **no OS move**.
- **Maintenance-window philosophy.** The application is quiesced; this is not a zero-downtime upgrade. **No DR switchover as part of the upgrade**; the end-state Live/Recovery role assignment is **identical to the start**.
- **Ordering rules (verbatim, from IBM):** across sites, **Recovery group upgraded before Live group**, Recovery always at a version **≥** Live; within a group, **replicas first, active last**, gating each on healthy `REPLICA` status.
- **Operational validation is a real DR drill:** after the upgrade, fail over Live→Recovery, verify, then fail back Recovery→Live, verify — returning roles to the start.
- **Runbook is generic and anonymized** — no client-identifiable material.
- **No upgrade/downgrade automation** — the procedure is manual (AI-executed).
- **Version pin is plain text** (no XML; no YAML parsing in shell). Author JSON/YAML only where a consumer already speaks it.
- **`DEFAULT_MQ_VERSION` name is load-bearing** — `src/mqlab/cli.py` imports it; keep the symbol, change only its source.
- **Validation command is the only one:** `vrg-container-run -- vrg-validate`. Unit tests run under it; locally, `uv run pytest`. `uv` is a build/dev tool, never runtime.
- **Commits:** `vrg-commit --type <type> --scope <scope> --message <msg>`; every commit message ends with `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`.
- **Cold rebuild is the acceptance gate** for provisioning changes; lint-green ≠ done.

## File Structure

- `lab/mq-version` — **new.** The single authoritative MQ version pin: one line, the bare 4-part version string (initially `9.4.5.0`). One responsibility: name the MQ version.
- `src/mqlab/manifest.py` — **modify.** `DEFAULT_MQ_VERSION` becomes a read of `lab/mq-version` (via `repo_root()`), not a literal.
- `scripts/fetch-mq.sh` — **modify.** `VER` reads `lab/mq-version`, not a literal.
- `ansible/group_vars/all/versions.yml` — **modify.** `mq_version` sourced from `lab/mq-version` via `lookup('file', …)`.
- `ansible/roles/mq-nativeha/tasks/main.yml` — **modify.** The floor assertion becomes a real version comparison that accepts 10.0.
- `manifests/distributed-rdqm-rhel/default.yaml`, `manifests/distributed-pcmk-ubuntu/default.yaml` — **delete** (stale SUT manifests, after confirming they are unreferenced).
- `tests/test_mq_version_pin.py` — **new.** Guards the single-source invariant.
- `docs/reference/nativeha-mq-upgrade-runbook.md` — **new.** The CRR upgrade runbook (workstream B deliverable).
- `docs/reports/YYYY-MM-DD-nativeha-mq-10-upgrade.md` — **new** (authored during the validation task; workstream C evidence).
- `build/refs/ibm-docs/ibm-mq/10.0.x/…` — **new** (cached IBM pages from the spike; gitignored).

All file paths below are relative to the lab repo `logical-minds-foundry/mq-resiliency-lab-for-linux`, **except** this plan and the spec, which live in `.github`.

---

### Task 0 (Spike): Pin the IBM CRR upgrade facts + confirm media

**Type:** research spike. Output is a short findings note committed under `docs/reports/`, plus IBM pages cached under `build/refs/`. De-risks the unknowns in spec §8 before the runbook (Task 3) and the pin bump (Task 5) are written. No production code.

**Files:**

- Create: `docs/reports/YYYY-MM-DD-mq-10-upgrade-spike.md`
- Cache (gitignored): `build/refs/ibm-docs/ibm-mq/10.0.x/…` via `tools/ibm_doc_cache.py`

**Interfaces:**

- Produces: the confirmed facts consumed by Task 3 (runbook) and Task 5 (10.0 version string / tarball). Specifically: the exact 10.0 LTS 4-part version string; the developer-tarball filename(s) and base URL for `fetch-mq.sh`; the within-group and cross-site ordering rules with the exact `dspmq` status gates; and the **point of no return** for a major-version Native HA/CRR migration (is any in-place downgrade supported, or is back-out restore-from-backup?).

- [ ] **Step 1: Cache the authoritative IBM pages**

Run:

```bash
python3 tools/ibm_doc_cache.py "https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=irr-upgrading-native-ha-configurations"
python3 tools/ibm_doc_cache.py "https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=migrating-native-ha-queue-manager"
python3 tools/ibm_doc_cache.py "https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=migration-queue-manager-methods"
```

(The IRR topic is confirmed to exist; discover and cache the sibling Native-HA migration and queue-manager-migration-methods topics for 10.0 from its navigation. Cite `content.txt` with the `source_url` from `meta.json`.)

- [ ] **Step 2: Confirm the 10.0 developer media**

Determine the exact 10.0 developer tarball name(s) matching `fetch-mq.sh`'s pattern `${VER}-IBM-MQ-Advanced-for-Developers-{UbuntuLinuxX64|UbuntuLinuxARM64|LinuxX64}.tar.gz` and the download base. If the naming or base URL changed at 10.0, record the exact values Task 5 must use. Check the IBM developer download page and, if needed, the Passport/Fix Central paths noted in the spec sources.

- [ ] **Step 3: Confirm the 9.4.5 media is staged (test precondition)**

Run:

```bash
ls -la "$(mqlab build path cache)/mq/" 2>/dev/null || ls -la build/cache/mq/
```

Record whether `9.4.5.0-IBM-MQ-Advanced-for-Developers-LinuxX64.tar.gz` is present (the RHEL CRR test needs it). If absent, note whether `scripts/fetch-mq.sh` can still fetch it from IBM; if it cannot, raise to the human — this is the spec §8 media-availability precondition.

- [ ] **Step 4: Write the findings note**

Capture, in `docs/reports/<date>-mq-10-upgrade-spike.md`, with citations: the 10.0 version string; the developer-tarball name/URL; the within-group + cross-site ordering with the exact `dspmq` gates; and the **point of no return / back-out mechanism** (state plainly if in-place downgrade is unsupported and back-out is restore-from-backup). This note is the source of truth Task 3 cites.

- [ ] **Step 5: Commit**

```bash
vrg-commit --type docs --scope upgrade --message "spike — IBM MQ 10.0 Native HA CRR upgrade facts + media check"
```

---

### Task 1: The MQ version pin + Python/shell consumers

**Files:**

- Create: `lab/mq-version`
- Modify: `src/mqlab/manifest.py:68-70`
- Modify: `scripts/fetch-mq.sh:7`
- Test: `tests/test_mq_version_pin.py`

**Interfaces:**

- Produces: `lab/mq-version` (bare 4-part version string, single line, trailing newline). `manifest.DEFAULT_MQ_VERSION: str` — unchanged name and type, now sourced from the pin. `manifest.mq_version_pin_path() -> Path` — helper returning `repo_root() / "lab" / "mq-version"`, consumed by the test.
- Consumes: `repo_root()` from `mqlab.paths` (already imported in `manifest.py`).

- [ ] **Step 1: Create the pin file**

`lab/mq-version` — exactly one line:

```text
9.4.5.0
```

- [ ] **Step 2: Write the failing test**

`tests/test_mq_version_pin.py`:

```python
"""The single-source MQ version invariant (epic .github#219)."""

import re
from pathlib import Path

from mqlab import manifest
from mqlab.paths import repo_root


def test_pin_file_is_a_bare_version() -> None:
    text = (repo_root() / "lab" / "mq-version").read_text()
    assert text.endswith("\n"), "pin file must end with a newline"
    assert re.fullmatch(r"\d+\.\d+\.\d+\.\d+", text.strip()), text


def test_manifest_default_matches_pin() -> None:
    pin = (repo_root() / "lab" / "mq-version").read_text().strip()
    assert manifest.DEFAULT_MQ_VERSION == pin


def test_fetch_script_has_no_hardcoded_version() -> None:
    src = (repo_root() / "scripts" / "fetch-mq.sh").read_text()
    assert not re.search(r'VER="\d+\.\d+\.\d+\.\d+"', src), (
        "fetch-mq.sh must read lab/mq-version, not hardcode VER"
    )
    assert "lab/mq-version" in src
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `uv run pytest tests/test_mq_version_pin.py -v`
Expected: FAIL — `test_manifest_default_matches_pin` (still a literal that happens to match is fine, but `test_fetch_script_has_no_hardcoded_version` FAILS on the current `VER="9.4.5.0"`).

- [ ] **Step 4: Re-thread `manifest.py`**

Replace the literal at `src/mqlab/manifest.py:68-70`:

```python
def mq_version_pin_path() -> Path:
    """The single authoritative MQ version pin (epic .github#219)."""
    return repo_root() / "lab" / "mq-version"


# Canonical MQ-for-Developers version — read from the one pin every consumer
# shares (lab/mq-version); no more hand-synced literals. cli.py imports this name.
DEFAULT_MQ_VERSION = mq_version_pin_path().read_text().strip()
```

Add `from pathlib import Path` to the runtime imports (it is currently only under `TYPE_CHECKING`). Keep `repo_root` (already imported).

- [ ] **Step 5: Re-thread `fetch-mq.sh`**

Replace `scripts/fetch-mq.sh:7`:

```bash
VER="$(cat "$(cd "$(dirname "$0")/.." && pwd)/lab/mq-version")"
```

(The script already computes the repo root the same way for `DEST`.)

- [ ] **Step 6: Run the tests to verify they pass**

Run: `uv run pytest tests/test_mq_version_pin.py -v`
Expected: PASS (all three).

- [ ] **Step 7: Full validation**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS (ruff/mypy/pytest). Fix any typing fallout from the new `Path` import.

- [ ] **Step 8: Commit**

```bash
vrg-commit --type refactor --scope versioning --message "single MQ version pin (lab/mq-version); manifest + fetch read it"
```

---

### Task 2: Ansible consumers — group_vars source + version-aware NHA floor

**Files:**

- Modify: `ansible/group_vars/all/versions.yml:5`
- Modify: `ansible/roles/mq-nativeha/tasks/main.yml` (the "assert MQ >= 9.4.4" task)
- Delete: `manifests/distributed-rdqm-rhel/default.yaml`, `manifests/distributed-pcmk-ubuntu/default.yaml`
- Test: extend `tests/test_mq_version_pin.py`

**Interfaces:**

- Consumes: `lab/mq-version` (Task 1).
- Produces: `mq_version` Ansible fact sourced from the pin; an NHA floor assertion that accepts any MQ `>= 9.4.4` (so 10.0 passes).

- [ ] **Step 1: Write the failing tests**

Append to `tests/test_mq_version_pin.py`:

```python
def test_group_vars_sources_pin() -> None:
    y = (repo_root() / "ansible" / "group_vars" / "all" / "versions.yml").read_text()
    assert "lookup('file'" in y and "lab/mq-version" in y
    assert not re.search(r'mq_version:\s*"\d+\.\d+\.\d+\.\d+"', y)


def test_nativeha_floor_is_version_aware() -> None:
    t = (repo_root() / "ansible" / "roles" / "mq-nativeha" / "tasks" / "main.yml").read_text()
    assert "is version(" in t, "floor must be a version comparison, not substring match"
    assert "'9.4.5' in mqver.stdout" not in t


def test_stale_sut_manifests_removed() -> None:
    root = repo_root() / "manifests"
    assert not (root / "distributed-rdqm-rhel" / "default.yaml").exists()
    assert not (root / "distributed-pcmk-ubuntu" / "default.yaml").exists()
```

- [ ] **Step 2: Run to verify they fail**

Run: `uv run pytest tests/test_mq_version_pin.py -v -k "group_vars or floor or stale"`
Expected: FAIL (all three).

- [ ] **Step 3: Source `mq_version` from the pin**

`ansible/group_vars/all/versions.yml:5` becomes:

```yaml
mq_version: "{{ lookup('file', playbook_dir + '/../lab/mq-version') | trim }}"
```

(Playbooks run from `ansible/`, so `playbook_dir/../lab/mq-version` resolves to the repo pin. Leave the `grafana_version` line untouched — obs drift is out of scope.)

- [ ] **Step 4: Make the NHA floor version-aware**

Replace the assert task in `ansible/roles/mq-nativeha/tasks/main.yml`:

```yaml
- name: assert MQ >= 9.4.4 CD (Native HA off-container floor)
  ansible.builtin.assert:
    that:
      - "mqver.stdout is version('9.4.4', '>=')"
    fail_msg: "MQ level {{ mqver.stdout }} is below the 9.4.4 Native HA floor"
    success_msg: "MQ level OK for Native HA: {{ mqver.stdout }}"
```

(`dspmqver -b -f 2` returns the bare 4-part version, so Ansible's `version` test compares correctly: `10.0.0.0 >= 9.4.4` is true.)

- [ ] **Step 5: Confirm the stale manifests are unreferenced, then delete**

Run:

```bash
grep -rn "distributed-rdqm-rhel\|distributed-pcmk-ubuntu" src/ ansible/ lab/ scripts/ tools/ | grep -v _shared
```

Expected: no code loads `manifests/distributed-*/default.yaml` (the live overlay is `manifests/_shared/observability.yaml`). If the grep is clean, delete both files:

```bash
vrg-git rm manifests/distributed-rdqm-rhel/default.yaml manifests/distributed-pcmk-ubuntu/default.yaml
```

If the grep finds a real consumer, STOP and raise it — do not delete.

- [ ] **Step 6: Run tests + validate**

Run: `uv run pytest tests/test_mq_version_pin.py -v` then `vrg-container-run -- vrg-validate`
Expected: PASS. `vrg-validate` runs `ansible-lint`; fix any lint on the edited role/vars.

- [ ] **Step 7: Commit**

```bash
vrg-commit --type refactor --scope versioning --message "Ansible reads the MQ pin; NHA floor is version-aware; drop stale SUT manifests"
```

---

### Task 3: The Native HA CRR upgrade runbook (docs)

**Type:** documentation. No unit tests; validated by `vrg-validate` (markdown/style) and human review. Grounded entirely in Task 0's cached IBM facts.

**Files:**

- Create: `docs/reference/nativeha-mq-upgrade-runbook.md`

**Interfaces:**

- Consumes: Task 0's findings note (ordering rules, `dspmq` gates, point of no return, back-out mechanism).

- [ ] **Step 1: Draft the runbook to the spec §5 structure**

Sections, in order, each with concrete MQSC/CLI commands (not prose gestures):

1. **Scope & headline rules** — Recovery-site-first; replicas-first/active-last; maintenance window; roles return to start.
2. **Pre-flight & back-out** — pre-checks (versions in both groups, `dspmq -o nativeha` health, disk, media staged, entitlement, recorded starting roles); the **point of no return** and the **true back-out mechanism** from Task 0.
3. **Quiesce & assert** — stop listener(s); drain; assert via `DISPLAY CONN`, `DISPLAY CHSTATUS`, transmit-queue depths zero.
4. **Upgrade the Recovery group first** — per-instance replica-first/active-last, with the exact `dspmq` REPLICA gate between instances.
5. **Upgrade the Live group** — same within-group rule; QM down during the Live-active step.
6. **Post-upgrade sanity gate** — `dspmqver` 10.0 on all six; both groups quorate; objects intact; test message round-trips.
7. **Post-upgrade DR validation** — fail over Live→Recovery (reference the CRR switchover procedure: `ansible/site-nativeha-switchover.yml` / the `dr-cutover` verb / `docs/reference/nativeha-crr-setup-guide.md`), verify; fail back Recovery→Live; verify roles == start.
8. **Go-live** — re-enable listener(s).
9. **Applicability** — a short note stating the procedure applies equally to the Ubuntu Native HA CRR arm; it is tested on RHEL (spec §6), and the Native HA formation is OS-agnostic (the `mq-nativeha` role is shared verbatim across RHEL and Ubuntu), so the same steps hold, differing only in the per-OS package manager for the MQ install.

Keep it generic and anonymized (queue manager, application, counterparty as generic roles).

- [ ] **Step 2: Cross-link and cite**

Link the runbook from the relevant site/reference index if one exists (the doc-review bookend #1069 will complete the site-docs sweep). Cite the IBM sources by their cached `source_url`.

- [ ] **Step 3: Validate**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS (markdown lint / link checks). Fix any findings.

- [ ] **Step 4: Commit**

```bash
vrg-commit --type docs --scope upgrade --message "Native HA CRR MQ 9.4.5→10.0 rolling-upgrade runbook"
```

---

### Task 4: [VALIDATION — operational, not PR-workable] Execute the CRR upgrade + capture evidence

**Type:** `--kind validation`. Filed as an operational task; run with `issue-validate`, **not** `issue-implement`. Closes only on `Outcome: SUCCESS` recorded as a comment. **Blocked-by** Tasks 1, 2, 3 merged (the centralized pin + runbook must be on `develop`); informed by Task 0.

**Precondition self-check (run first; if unmet, comment "blocked: preconditions not met" and stop):**

- Tasks 1–3 merged to `develop`.
- `build/cache/mq/9.4.5.0-IBM-MQ-Advanced-for-Developers-LinuxX64.tar.gz` present (spec §8 media precondition; Task 0 Step 3), **and** the 10.0 developer tarball fetchable/staged per Task 0 Step 2.

**Procedure (the human operates the lab; the AI drives the runbook and records evidence):**

- [ ] **Step 1: Cold-rebuild the full RHEL CRR topology at 9.4.5**

Bring up the RHEL Native HA arm with **DR enabled** (no `--no-dr`) from a cold rebuild, at the pinned 9.4.5. This exercises the Task 1–2 re-threading end-to-end (spec §4.4 acceptance rides here). Record the six instances' `dspmqver` (all 9.4.5.0) and the starting Live/Recovery role assignment.

- [ ] **Step 2: Stage the 10.0 developer media**

Ensure the 10.0 `LinuxX64` developer tarball is in `build/cache/mq/` (via the Task 0 filename/URL). Do **not** flip `lab/mq-version` yet — the upgrade installs 10.0 alongside/over 9.4.5 on each node per the runbook; the default pin flip is Task 5.

- [ ] **Step 3: Execute the runbook against the live cluster**

Follow `docs/reference/nativeha-mq-upgrade-runbook.md` exactly: quiesce & assert → upgrade the **Recovery group** to 10.0 (replicas first, active last, gating on REPLICA) → upgrade the **Live group** → sanity gate → **DR failover Live→Recovery, verify → failback Recovery→Live, verify** → go-live. Capture command output at each gate.

- [ ] **Step 4: Author the evidence report**

`docs/reports/<date>-nativeha-mq-10-upgrade.md` — the spec §6 evidence list: starting roles; clean quiesce; per-instance `dspmqver` 9.4.5→10.0 across both groups; `dspmq -o nativeha` health at each step with the Recovery ≥ Live invariant holding; the two DR drills succeeding on 10.0; final all-10.0 state with roles identical to start; the point of no return as observed. Include any runbook corrections discovered during execution (fold fixes back into Task 3's runbook via a follow-up PR if needed).

- [ ] **Step 5: Record the outcome**

Comment `Outcome: SUCCESS` (or `FAILURE` with detail) on the validation issue per the operational-task template. On SUCCESS the task closes.

---

### Task 5: Rebaseline the lab default to MQ 10.0

**Files:**

- Modify: `lab/mq-version`
- Modify: `scripts/fetch-mq.sh` comment header (if it names 9.4.5)
- Test: `tests/test_mq_version_pin.py` (unchanged — the regex invariant already covers any 4-part version)

**Interfaces:**

- Consumes: Task 0's confirmed 10.0 version string; Task 4 SUCCESS (only rebaseline after the upgrade is proven). **Blocked-by** Task 4.

- [ ] **Step 1: Flip the pin**

`lab/mq-version` → the confirmed 10.0 LTS 4-part string (e.g. `10.0.0.0`; use Task 0's exact value).

- [ ] **Step 2: Run the invariant tests**

Run: `uv run pytest tests/test_mq_version_pin.py -v`
Expected: PASS — `DEFAULT_MQ_VERSION` and the Ansible lookup now resolve to 10.0 with no code change (this is the payoff of Task 1–2).

- [ ] **Step 3: Refresh media + validate**

Run: `scripts/fetch-mq.sh` (fetches the 10.0 tarballs) then `vrg-container-run -- vrg-validate`.
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
vrg-commit --type feat --scope versioning --message "rebaseline the lab default to IBM MQ 10.0"
```

---

### Task 6: [VALIDATION — operational, not PR-workable] Cold rebuild at 10.0 from zero

**Type:** `--kind validation`. Run with `issue-validate`. The closing cold-rebuild acceptance gate — proves the centralized pin builds the whole RHEL CRR arm at 10.0 from a cold start. **Blocked-by** Task 5 merged.

**Precondition self-check:** Task 5 merged to `develop`; the 10.0 developer tarball staged in `build/cache/mq/`.

- [ ] **Step 1: Cold rebuild the RHEL Native HA CRR arm from zero at 10.0**

Full cold rebuild (DR enabled) from the rebaselined pin. Confirm all six instances come up on 10.0 (`dspmqver`), both groups reach quorum (`dspmq -o nativeha`), and provisioning is one-pass (no manual fix-ups) — the cold-rebuild doctrine.

- [ ] **Step 2: Record the outcome**

Comment `Outcome: SUCCESS` (or `FAILURE` with detail). On SUCCESS the task closes; the lab now defaults to 10.0.

---

## Self-Review

**Spec coverage:**

- §4 version centralization → Tasks 1–2 (pin, consumers, floor, stale-manifest removal); acceptance (§4.4) rides Task 4 Step 1 and the closing Task 6.
- §5 runbook (all sub-sections, ordering rules, quiesce/assert, back-out, DR validation) → Task 3, grounded by Task 0.
- §6 test & evidence (full RHEL CRR, DR enabled, both-group upgrade, failover/failback, report) → Task 4; rebaseline + from-zero 10.0 build → Tasks 5–6.
- §8 risks: rolling rules & point of no return → Task 0; media availability → Task 0 Step 3 + Task 4 precondition; real back-out mechanism → Task 0 Step 4 + Task 3 §2; DR runbook dependency → Task 3 §7 / Task 4 Step 3; cold-rebuild cost → Tasks 4 & 6 (human operates the lab).

**Placeholder scan:** dates are written `YYYY-MM-DD` (resolved when each doc lands) and the 10.0 version string is deliberately confirmed in Task 0 before use in Task 5 — not placeholders but sequenced discoveries. No "TODO/TBD/handle appropriately" steps.

**Type consistency:** `DEFAULT_MQ_VERSION: str` and `mq_version_pin_path() -> Path` are defined in Task 1 and consumed (unchanged) by the tests and by `cli.py`; the pin-file path string `lab/mq-version` is identical across `manifest.py`, `fetch-mq.sh`, `versions.yml`, and every test.

**Operational-task note:** Tasks 4 and 6 are `--kind validation` (not PR-workable) and are filed with `--blocked-by` their prerequisites; Tasks 0, 1, 2, 3, 5 are PR-workable.
