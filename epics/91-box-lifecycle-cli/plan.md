# mqlab box-lifecycle CLI — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give `mqlab` first-class control of the baked-image layer — `mqlab box status/build/rebuild/clean` per-box and `--all` — so the box-rebake-in-place tier is a clean command, plus a verify-and-guide + host-side auto-stage for the credentialed RHEL DVD and a cold-boot staleness nudge.

**Architecture:** Six code tasks landing in **`mq-resiliency-lab-for-linux`** (this plan lives in `.github`). Task 1 introduces `src/mqlab/box.py` (the five-box fleet model) and the read-only `mqlab box status` decision surface. Tasks 2–3 add the mutating verbs (`build`/`rebuild`, then `clean`) and refactor bootstrap's `_ensure_local_boxes` to share the `box build` core. Task 4 adds the in-VM RHEL-DVD verify-and-guide at the base-box locus. Task 5 adds the host-side idempotent rsync auto-stage + its `vergil.toml` post-build-hook declaration (activation gated on a cross-org `vrg-vm` capability). Task 6 adds the cold-boot staleness stamp + nudge. One operational validation task closes the loop with a live-lab exercise + cold rebuild. The shell builders (`build-fatbox.sh`/`build-box.sh`) stay the single source of truth for bake/staleness decisions — the CLI orchestrates and renders, it does not reimplement them.

**Tech Stack:** `mqlab` (Python 3 + Typer, pytest @ 100% branch coverage); the existing bash builders `lab/boxes/build-fatbox.sh` + `lab/boxes/rhel96/build-box.sh` + `lab/boxes/_manifest-hash.sh` (each with a `--dry-run` REUSE/BUILD/STALE/FORCE decision surface); libvirt/Vagrant lab; `vergil.toml` VM profile; rsync (host-side staging).

## Global Constraints

- **Manage, don't reimplement.** The shell builders are the single source of truth for bake + staleness decisions; the CLI shells out and parses, it never duplicates manifest-hash/age logic (avoids a second source of truth that would drift).
- **Fail loud.** No swallowed failures; a failed bake, a missing / checksum-mismatched DVD, or a builder non-zero exit surfaces with the underlying tool's own message and a non-zero exit.
- **Derive, never hardcode.** The fleet, box names, cache paths, and platforms come from the existing topology / `_LOCAL_BOX_BUILDERS` resolution — no new box-name or path literal scattered in shipped code; one fleet definition in `box.py`.
- **Never fabricate the DVD checksum.** The pinned SHA-256 per RHEL version is authoritative external data the operator supplies from Red Hat's published checksum; tests verify the *mechanism* against a computed fixture hash, never a bluffed real value.
- **Layered error vocabulary.** `mqlab`'s own messages name fully-qualified `mqlab` commands; the underlying builders speak for themselves.
- **`uv` is a build tool, not runtime.** Never embed `uv run` in shipped CLI/scripts; the dev-loop test command is `uv run pytest …` only.
- **Validation gate is exactly one command:** `vrg-container-run -- vrg-validate`. No individual linters/formatters run outside it.
- **Coverage floor: 100% branch** for `mqlab` — `uv run pytest --cov=src --cov-branch --cov-fail-under=100`; `# pragma: no cover` only for genuinely unreachable guards.
- **Cold-rebuild acceptance gate** applies to the box-layer behavior: the closing validation task proves it one-pass on a full VM cold rebuild; lint-green ≠ done. The human operates the lab.
- **Each task = one GitHub issue** in the lab repo, on `feature/<issue>-<slug>` off `develop`; commit with `vrg-commit`; PR into `develop`.

## Task dependency graph

```
T1 (box.py fleet + `box status`) ──┬─▶ T2 (build/rebuild + bootstrap refactor) ──▶ T4 (DVD verify-and-guide)
                                   ├─▶ T3 (clean)
                                   └─▶ T6 (cold-boot nudge)
T5 (host-side rsync + vergil.toml hook) ── no code blocker (parallel);
     activation blocked-by cross-org vrg-vm post-build hook (referenced by comment in #91)
Validation (operational) ── blocked-by T2, T3, T4, T6 (and T5 where the hook exists)
```

- **T1** — foundational, no blockers.
- **T2, T3, T6** — each blocked-by **T1**.
- **T4** — blocked-by **T2** (it hooks the `build` core's RHEL base path).
- **T5** — parallel; its *deliverable* (script + declaration) has no code blocker, but the hook only *fires* once the cross-org `vrg-vm` capability lands.
- **Validation** (operational) — blocked-by T2, T3, T4, T6.

---

### Task 1: `box.py` fleet model + read-only `mqlab box status`

Introduce the fleet definition and the `box` sub-app with its read-only verb. No mutation, no bootstrap change — pure foundation the other verbs build on.

**Files:**
- Create: `src/mqlab/box.py` — the five-box fleet model + monkeypatchable orchestration seams.
- Modify: `src/mqlab/cli.py` — `box_app = typer.Typer(...)`, `app.add_typer(box_app, name="box")`, and the `status` handler.
- Create: `tests/test_cli_box.py`.

**Interfaces:**
- Consumes: `cli._LOCAL_BOX_BUILDERS`, `cli.parse_box_list`, `cli._resolved_nodes`, `cli._vagrant_env`, `cli.repo_root`.
- Produces:
  - `box.FLEET: dict[str, BoxSpec]` where `BoxSpec` carries `name`, `builder` (script path relative to repo root), `cache_artifact` (e.g. `mq-rdqm-rhel9.box` / `rhel-9.6-x86_64-libvirt.box`), and `has_manifest_hash: bool` (False for the base box).
  - `box.box_decision(name: str) -> BoxDecision` with fields `name`, `cached: bool`, `age_days: int | None`, `hash_match: bool | None` (None when `has_manifest_hash` is False), `registered: bool`, `action: str` (one of `REUSE`/`BUILD`/`STALE`/`FORCE-BUILD`), obtained by running the box's builder with `--dry-run` and parsing its decision line.
  - `box.render_status(names: list[str]) -> str` — the table.

- [ ] **Step 1: Write the failing test for the fleet definition.** In `tests/test_cli_box.py`:

```python
from mqlab import box

def test_fleet_has_five_local_boxes():
    assert set(box.FLEET) == {
        "rhel/9.6-x86_64", "mq-rdqm-rhel9",
        "obs-ubuntu2404", "infra-ubuntu2404", "mq-ubuntu2404",
    }

def test_base_box_has_no_manifest_hash():
    assert box.FLEET["rhel/9.6-x86_64"].has_manifest_hash is False
    assert box.FLEET["mq-rdqm-rhel9"].has_manifest_hash is True
```

- [ ] **Step 2: Run it, verify it fails** — `uv run pytest tests/test_cli_box.py -v` → FAIL (`box` module / `FLEET` not defined).

- [ ] **Step 3: Implement the fleet model** in `src/mqlab/box.py` — a frozen `BoxSpec` dataclass and `FLEET` built from `cli._LOCAL_BOX_BUILDERS` (fat boxes → `build-fatbox.sh`, `has_manifest_hash=True`; `rhel/9.6-x86_64` → `build-box.sh`, `has_manifest_hash=False`), with each box's `cache_artifact` name. Keep the mapping derived from `_LOCAL_BOX_BUILDERS`, not a second literal list.

- [ ] **Step 4: Run tests, verify pass** — `uv run pytest tests/test_cli_box.py -v` → PASS.

- [ ] **Step 5: Write the failing test for `box_decision` parsing.** Monkeypatch the builder runner seam to return a canned `--dry-run` line and assert the parsed `BoxDecision`:

```python
def test_box_decision_parses_reuse(monkeypatch):
    monkeypatch.setattr(box, "_run_builder_dry_run",
                        lambda name: "action: REUSE (age 3d, hash match)")
    monkeypatch.setattr(box, "_cache_present", lambda name: True)
    monkeypatch.setattr(box, "_registered_boxes", lambda: {"mq-rdqm-rhel9": ""})
    d = box.box_decision("mq-rdqm-rhel9")
    assert d.action == "REUSE"
    assert d.registered is True
```

- [ ] **Step 6: Run it, verify it fails**, then **implement** `_run_builder_dry_run` (shell out to `bash <builder> --box <name> --dry-run …` for fat boxes, `build-box.sh --dry-run` for the base — supplying `--domain-type`/`--cpu-mode` from the existing `build_domain_virt(probe())` seam), `_cache_present` (stat `build/state/boxes/<artifact>`), `_registered_boxes` (`parse_box_list` over `vagrant box list`), and `box_decision`/`render_status`. Run: `uv run pytest tests/test_cli_box.py -v` → PASS.

- [ ] **Step 7: Wire the `status` verb.** In `cli.py` add `box_app` + `app.add_typer(box_app, name="box")` and:

```python
@box_app.command("status")
def box_status(boxes: list[str] = typer.Argument(None)) -> None:
    """Show the baked-box fleet: cache/age/hash/registration + REUSE/BUILD/STALE/FORCE decision."""
    names = boxes or list(box.FLEET)
    typer.echo(box.render_status(names))
```

Add a test asserting `mqlab box status` (via Typer's `CliRunner`) lists all five boxes and a decision column, mirroring `tests/test_cli_build.py`'s runner usage.

- [ ] **Step 8: Validate + commit** — `vrg-container-run -- vrg-validate`; `vrg-commit --type feat --scope mqlab --message "box.py fleet model + read-only \`mqlab box status\` (#<issue>)" --body "Epic logical-minds-foundry/.github#91"`.

**Acceptance:** `mqlab box status` renders the five-box fleet with cache/age/hash(N-A for base)/registration + decision, read-only; fleet + parsing unit-tested at 100% branch.

---

### Task 2: `box build` / `box rebuild` + share the core with bootstrap

Add the mutating ensure/force-bake verbs and refactor `_ensure_local_boxes` to call the same core, so bootstrap and the CLI share one path.

**Files:**
- Modify: `src/mqlab/box.py` — the build/rebuild core.
- Modify: `src/mqlab/cli.py` — `box build` / `box rebuild` handlers; refactor `_ensure_local_boxes` to delegate.
- Modify: `tests/test_cli_box.py`, `tests/test_cli_bootstrap.py`.

**Interfaces:**
- Consumes: Task 1's `box.FLEET`; `cli._box_build_steps`, `cli.run_steps`, `cli.build_deps`.
- Produces:
  - `box.build_boxes(names: list[str], *, force: bool) -> None` — for each name, build a `CommandStep` (reusing `_box_build_steps`, adding `--rebuild-box` when `force`) and run it via `run_steps`; raises `typer.Exit(code)` on `StepFailedError`.
  - `_ensure_local_boxes(guests)` now computes the needed box set and calls `box.build_boxes(sorted(needed), force=False)` (plus the existing DVD staging).

- [ ] **Step 1: Write the failing test** — `box build mq-rdqm-rhel9` issues a non-force build step; `box rebuild mq-rdqm-rhel9` adds `--rebuild-box`:

```python
def test_build_boxes_force_adds_rebuild_flag(monkeypatch):
    captured = []
    monkeypatch.setattr(box, "run_steps", lambda steps, **kw: captured.extend(steps))
    box.build_boxes(["mq-rdqm-rhel9"], force=True)
    argv = captured[0].command.argv
    assert "--box" in argv and "mq-rdqm-rhel9" in argv and "--rebuild-box" in argv
```

- [ ] **Step 2: Run it, verify it fails** (`build_boxes` undefined), then **implement** `box.build_boxes` in `box.py` reusing `_box_build_steps` (extended to accept a `force` flag that appends `--rebuild-box`). Run → PASS.

- [ ] **Step 3: Wire the verbs** in `cli.py`:

```python
@box_app.command("build")
def box_build(boxes: list[str] = typer.Argument(None), all_: bool = typer.Option(False, "--all")) -> None:
    """Ensure each box is present: REUSE a valid cache, else bake."""
    box.build_boxes(_select_boxes(boxes, all_), force=False)

@box_app.command("rebuild")
def box_rebuild(boxes: list[str] = typer.Argument(None), all_: bool = typer.Option(False, "--all")) -> None:
    """Force a fresh bake (--rebuild-box), overwriting the cache — the box-rebake-in-place tier."""
    box.build_boxes(_select_boxes(boxes, all_), force=True)
```

Add `_select_boxes(names, all_)` — returns `list(box.FLEET)` for `--all`, the validated names otherwise, and `typer.Exit(2)` with a fail-loud message if neither is given or a name is unknown.

- [ ] **Step 4: Write the failing test for the bootstrap refactor** — `_ensure_local_boxes` delegates to `box.build_boxes` with `force=False`:

```python
def test_ensure_local_boxes_delegates_to_build_core(monkeypatch):
    calls = {}
    monkeypatch.setattr(cli.box, "build_boxes", lambda names, *, force: calls.update(names=names, force=force))
    monkeypatch.setattr(cli, "_needed_local_boxes", lambda g: {"mq-rdqm-rhel9": "…/build-fatbox.sh"})
    monkeypatch.setattr(cli, "_guests_need_dvd", lambda g: False)
    cli._ensure_local_boxes(["rdqm-a1"])
    assert calls == {"names": ["mq-rdqm-rhel9"], "force": False}
```

- [ ] **Step 5: Run it, verify it fails**, then **refactor** `_ensure_local_boxes` to compute `needed` and call `box.build_boxes(sorted(needed), force=False)`, preserving the DVD staging step. Run `uv run pytest tests/test_cli_box.py tests/test_cli_bootstrap.py --cov=src --cov-branch --cov-fail-under=100 -v` → PASS.

- [ ] **Step 6: Validate + commit** — `vrg-container-run -- vrg-validate`; `vrg-commit --type feat --scope mqlab --message "box build/rebuild + share the ensure-box core with bootstrap (#<issue>)" --body "Epic logical-minds-foundry/.github#91"`.

**Acceptance:** `box build`/`rebuild` work per-box and `--all`; `rebuild` forces a re-bake; bootstrap uses the shared core with no behavior regression; 100% branch.

---

### Task 3: `box clean` — pristine cache removal + deregister

**Files:**
- Modify: `src/mqlab/box.py` — the clean core.
- Modify: `src/mqlab/cli.py` — the `clean` handler.
- Modify: `tests/test_cli_box.py`.

**Interfaces:**
- Consumes: Task 1's `box.FLEET`; `cli.state`, `cli._vagrant_env`.
- Produces: `box.clean_boxes(names: list[str]) -> list[str]` — for each box, remove `build/state/boxes/<artifact>` and its `<name>.manifest-hash` (when present) and `vagrant box remove <name>` (ignore "not installed"); returns the list of removed paths/registrations for the caller to echo.

- [ ] **Step 1: Write the failing test** — clean removes the cache artifact + manifest-hash and deregisters:

```python
def test_clean_removes_cache_and_deregisters(monkeypatch, tmp_path):
    (tmp_path / "mq-rdqm-rhel9.box").write_text("x")
    (tmp_path / "mq-rdqm-rhel9.manifest-hash").write_text("h")
    monkeypatch.setattr(box, "_boxes_cache_dir", lambda: tmp_path)
    removed_regs = []
    monkeypatch.setattr(box, "_vagrant_box_remove", lambda n: removed_regs.append(n))
    removed = box.clean_boxes(["mq-rdqm-rhel9"])
    assert not (tmp_path / "mq-rdqm-rhel9.box").exists()
    assert not (tmp_path / "mq-rdqm-rhel9.manifest-hash").exists()
    assert removed_regs == ["mq-rdqm-rhel9"]
```

- [ ] **Step 2: Run it, verify it fails**, then **implement** `clean_boxes` + `_boxes_cache_dir` + `_vagrant_box_remove`. Run → PASS.

- [ ] **Step 3: Wire the verb with the `--all` confirm guard** in `cli.py`:

```python
@box_app.command("clean")
def box_clean(boxes: list[str] = typer.Argument(None), all_: bool = typer.Option(False, "--all"),
              yes: bool = typer.Option(False, "--yes-rebake-all")) -> None:
    """Make pristine: remove the durable .box (+ .manifest-hash) and deregister. Next build re-bakes."""
    if all_ and not yes:
        typer.echo("refusing to clean --all (forces a full fleet re-bake). Re-run with --all --yes-rebake-all.", err=True)
        raise typer.Exit(code=2)
    removed = box.clean_boxes(_select_boxes(boxes, all_))
    typer.echo("removed: " + ", ".join(removed))
```

- [ ] **Step 4: Test the guard** — `box clean --all` without `--yes-rebake-all` exits 2 and removes nothing; with it, cleans all five. Run `uv run pytest tests/test_cli_box.py --cov=src --cov-branch --cov-fail-under=100 -v` → PASS.

- [ ] **Step 5: Validate + commit** — `vrg-container-run -- vrg-validate`; `vrg-commit --type feat --scope mqlab --message "box clean: pristine cache removal + deregister, --all confirm-guarded (#<issue>)" --body "Epic logical-minds-foundry/.github#91"`.

**Acceptance:** `box clean <box>` removes the durable cache + manifest-hash and deregisters; `--all` is confirm-guarded; 100% branch.

---

### Task 4: RHEL DVD verify-and-guide (in-VM, pinned SHA-256)

Guard the RHEL base-box BUILD path (the sole DVD consumer) with a presence + checksum preflight and fail-loud guidance.

**Files:**
- Modify: `src/mqlab/box.py` — a `verify_rhel_dvd()` preflight + the pinned-SHA config, invoked from the build core before the RHEL base BUILD path.
- Create: `tests/test_box_dvd.py`.
- Modify: `docs/development/box-model.md` — note the mqlab-side verify-and-guide (its own §6 doc sweep is the closing #666 gate; this is the code-adjacent note).

**Interfaces:**
- Consumes: Task 2's build core; the canonical ISO path `state("rhel-9.6-x86_64-dvd.iso")` (mirroring `stage-rhel-iso.sh`'s default).
- Produces:
  - `box.RHEL_DVD_SHA256: dict[str, str]` — RHEL version → pinned checksum (operator-supplied from Red Hat's published checksum; **not fabricated**).
  - `box.verify_rhel_dvd(version: str) -> None` — raises `typer.Exit(2)` with fail-loud guidance (version, Red Hat download URL, destination path) on MISSING; raises on checksum FAIL; returns on PASS. Called by `build_boxes` when a selected box's base is `rhel/9.6-x86_64` and the decision is BUILD/FORCE-BUILD (not REUSE — a cached box needs no DVD).

- [ ] **Step 1: Write the failing tests** in `tests/test_box_dvd.py` against a fixture ISO (a small temp file with a known computed sha256 — the *mechanism*, never a real DVD hash):

```python
import hashlib
from mqlab import box

def test_verify_dvd_missing_raises(monkeypatch, tmp_path):
    monkeypatch.setattr(box, "_rhel_dvd_path", lambda: tmp_path / "absent.iso")
    with pytest.raises(SystemExit):
        box.verify_rhel_dvd("9.6")

def test_verify_dvd_checksum_mismatch_raises(monkeypatch, tmp_path):
    iso = tmp_path / "dvd.iso"; iso.write_bytes(b"wrong")
    monkeypatch.setattr(box, "_rhel_dvd_path", lambda: iso)
    monkeypatch.setattr(box, "RHEL_DVD_SHA256", {"9.6": "0"*64})
    with pytest.raises(SystemExit):
        box.verify_rhel_dvd("9.6")

def test_verify_dvd_pass(monkeypatch, tmp_path):
    iso = tmp_path / "dvd.iso"; iso.write_bytes(b"content")
    monkeypatch.setattr(box, "_rhel_dvd_path", lambda: iso)
    monkeypatch.setattr(box, "RHEL_DVD_SHA256", {"9.6": hashlib.sha256(b"content").hexdigest()})
    box.verify_rhel_dvd("9.6")  # returns, no raise
```

- [ ] **Step 2: Run them, verify they fail** (`verify_rhel_dvd` undefined), then **implement** `verify_rhel_dvd`, `_rhel_dvd_path` (defaults to `state("rhel-9.6-x86_64-dvd.iso")`, honoring `MQLAB_RHEL_ISO`/`RHEL_ISO` like `stage-rhel-iso.sh`), and the `RHEL_DVD_SHA256` config (seed the `9.6` entry from Red Hat's published DVD checksum — an operator step, documented as such; leave a clear fail-loud error if a version is unpinned). Run → PASS.

- [ ] **Step 3: Hook it into the build core** — in `box.build_boxes`, before invoking a BUILD/FORCE-BUILD step whose box's base is `rhel/9.6-x86_64`, call `verify_rhel_dvd(...)`. Add a test asserting a BUILD decision for a RHEL box calls `verify_rhel_dvd` and a REUSE decision does not.

- [ ] **Step 4: Run tests + coverage** — `uv run pytest tests/test_box_dvd.py tests/test_cli_box.py --cov=src --cov-branch --cov-fail-under=100 -v` → PASS.

- [ ] **Step 5: Validate + commit** — `vrg-container-run -- vrg-validate`; `vrg-commit --type feat --scope mqlab --message "RHEL DVD verify-and-guide (pinned SHA-256) at the base-box build path (#<issue>)" --body "Epic logical-minds-foundry/.github#91"`.

**Acceptance:** a missing / checksum-mismatched RHEL DVD is caught before the expensive BUILD with actionable fail-loud guidance; a REUSE needs no DVD; 100% branch.

---

### Task 5: Host-side RHEL DVD auto-stage — rsync script + `vergil.toml` post-build hook

Deliver the credential-free convenience: a one-time-configured local DVD archive that a VM build auto-stages. The script + declaration land here; the hook *fires* once the cross-org `vrg-vm` capability exists (referenced by comment in #91).

**Files:**
- Create: `lab/scripts/stage-rhel-dvd-from-archive.sh` — idempotent rsync from a configured source dir → `build/state/`.
- Modify: `vergil.toml` — declare the VM-specific post-build hook that runs the script (inert until `vrg-vm` supports it).
- Modify: `docs/development/box-model.md` — document the one-time-download + static-archive + auto-stage flow.

**Interfaces:**
- Consumes: an operator-configured source directory (env `MQLAB_RHEL_DVD_ARCHIVE`, else a documented default under the operator's home archive); the build/ state bucket path.
- Produces: an idempotent staging script + the `vergil.toml` hook declaration. No Python surface (host-side shell + config).

- [ ] **Step 1: Write `stage-rhel-dvd-from-archive.sh`** — resolve the source dir (`MQLAB_RHEL_DVD_ARCHIVE` or the documented default), resolve the destination (`build/state/` via the main-worktree git-common-dir resolution used by `stage-rhel-iso.sh`), then `rsync -a --ignore-existing <src>/*.iso <dst>/` for every RHEL DVD in the archive. `set -euo pipefail`; a `--dry-run` flag that prints the planned rsync without copying; loud error if the source dir is absent. Model structure on `lab/scripts/stage-rhel-iso.sh` (credential-less, idempotent).

- [ ] **Step 2: Prove idempotency + dry-run** — run the script twice against a temp source with a dummy `.iso`; assert the second run copies nothing (`--ignore-existing`), and `--dry-run` copies nothing. (Shell-level check; `vrg-validate`'s shellcheck covers lint.)

- [ ] **Step 3: Declare the `vergil.toml` post-build hook** — add the VM-specific post-build customization entry for `[vm.vergil-user]` naming `lab/scripts/stage-rhel-dvd-from-archive.sh` as the end-of-build hook, with a comment that it is inert until the cross-org `vrg-vm` post-build-hook capability lands (referenced by comment in epic #91). Keep it a declaration only — do not invent a `vergil.toml` key `vrg-vm` does not yet read; use the shape the cross-org issue specifies, or a clearly-commented placeholder pending that issue.

- [ ] **Step 4: Document the flow** in `docs/development/box-model.md` — the one-time manual RHEL download, the static archive dir, the auto-stage on VM rebuild, and the `mqlab box` verify-and-guide backstop.

- [ ] **Step 5: Validate + commit** — `vrg-container-run -- vrg-validate`; `vrg-commit --type feat --scope boxes --message "host-side RHEL DVD auto-stage: idempotent rsync + vergil.toml post-build hook (#<issue>)" --body "Epic logical-minds-foundry/.github#91; activation blocked-by the cross-org vrg-vm post-build hook"`.

**Acceptance:** the idempotent rsync script stages RHEL DVD(s) from a configured archive; the `vergil.toml` hook is declared (activation gated on the external `vrg-vm` capability); the flow is documented.

> **Sequencing note:** Step 3's exact `vergil.toml` shape depends on the cross-org `vrg-vm` post-build-hook issue. If that issue has not defined the key when this task runs, land the script + docs + a clearly-commented placeholder declaration, and complete the wiring in a small follow-up once the capability ships. Do not block the whole task on the external dependency.

---

### Task 6: Cold-boot staleness nudge — stamp + surfaces

**Files:**
- Modify: `src/mqlab/buildenv.py` — write `state/.cold-boot-stamp` once when `state/` is first initialized.
- Create: `src/mqlab/coldboot.py` — the age read + banded NOTICE render (tunable constants).
- Modify: `src/mqlab/cli.py` — surface the nudge in the `box status` header, `doctor`, and the `bootstrap` preflight.
- Modify: `tests/test_cli_box.py`, `tests/test_cli_doctor.py`, and add `tests/test_coldboot.py`.

**Interfaces:**
- Consumes: `cli.state`, `buildenv.ensure`.
- Produces:
  - `buildenv.ensure(...)` writes `state/.cold-boot-stamp` with the current UTC time **iff** it does not already exist (write-once; a fresh `/vergil` has no stamp → new stamp).
  - `coldboot.cold_boot_age_days() -> int | None` (None if no stamp) and `coldboot.nudge() -> str | None` — the banded message: None below the quiet threshold, an informational line in the mid band, a louder NOTICE at/above the loud threshold. Thresholds are module constants (`QUIET_DAYS`, `LOUD_DAYS`) with placeholder defaults, **never blocking**.

- [ ] **Step 1: Write the failing test for the write-once stamp:**

```python
from mqlab import buildenv

def test_ensure_writes_cold_boot_stamp_once(tmp_path, monkeypatch):
    monkeypatch.setattr(buildenv, "_now_iso", lambda: "2026-07-16T00:00:00Z")
    buildenv.ensure(tmp_path)
    stamp = tmp_path / "build" / "state" / ".cold-boot-stamp"
    assert stamp.read_text().strip() == "2026-07-16T00:00:00Z"
    monkeypatch.setattr(buildenv, "_now_iso", lambda: "2026-08-01T00:00:00Z")
    buildenv.ensure(tmp_path)  # must NOT overwrite
    assert stamp.read_text().strip() == "2026-07-16T00:00:00Z"
```

- [ ] **Step 2: Run it, verify it fails**, then **implement** the write-once stamp in `buildenv.ensure` (guarded on `not stamp.exists()`; inject `_now_iso` as a seam). Run → PASS.

- [ ] **Step 3: Write the failing test for the banded nudge:**

```python
from mqlab import coldboot

def test_nudge_banding(monkeypatch):
    monkeypatch.setattr(coldboot, "cold_boot_age_days", lambda: 5)
    assert coldboot.nudge() is None                      # quiet band
    monkeypatch.setattr(coldboot, "cold_boot_age_days", lambda: coldboot.LOUD_DAYS)
    assert "NOTICE" in coldboot.nudge()                  # loud band, never raises
```

- [ ] **Step 4: Run it, verify it fails**, then **implement** `coldboot.cold_boot_age_days` (parse the stamp, compute whole-day age; None if absent) and `coldboot.nudge` (banded, NOTICE-only). Run → PASS.

- [ ] **Step 5: Surface it in three places** — prepend `coldboot.nudge()` (when non-None) to the `box status` output header; emit it in the `doctor` command; and print it in the `bootstrap` preflight (before the phases run). Add tests asserting each surface shows the loud message when `nudge()` returns one and stays silent when it returns None — and that none of them raises/blocks on an old stamp.

- [ ] **Step 6: Run tests + coverage** — `uv run pytest tests/test_coldboot.py tests/test_cli_box.py tests/test_cli_doctor.py --cov=src --cov-branch --cov-fail-under=100 -v` → PASS.

- [ ] **Step 7: Validate + commit** — `vrg-container-run -- vrg-validate`; `vrg-commit --type feat --scope mqlab --message "cold-boot staleness nudge: write-once stamp + banded NOTICE across status/doctor/bootstrap (#<issue>)" --body "Epic logical-minds-foundry/.github#91"`.

**Acceptance:** a write-once `state/.cold-boot-stamp` anchors the age; the nudge shows in `box status`, `doctor`, and `bootstrap` preflight, loud but never blocking; thresholds are tunable constants; 100% branch.

---

## Operational task (filed under #91; run via issue-validate — NOT PR-workable)

- **Validation (live-lab + cold rebuild).** Exercise the box verbs on the running lab and prove the behavior end-to-end: `mqlab box status` reflects reality; `mqlab box rebuild <box>` re-bakes one box in place without touching the data disk; `mqlab box clean <box>` + `build` round-trips; the RHEL DVD verify-and-guide fires on a missing/mismatched ISO; the cold-boot nudge appears. Then a full VM cold rebuild proves the box tooling one-pass (the cold-rebuild acceptance gate). Blocked-by T2, T3, T4, T6. Run with `issue-validate`.

## Bookend & follow-on tasks

- **`.github#92` Documentation** (this task) — publishes this spec + plan; its PR is the opening bookend.
- **`.github#93` Follow-on brainstorm** (existing, closing) — CI/automated box-library builds + publishing. The cold-boot nudge (T6) is the manual bridge until this lands.
- **`.github#<new>` Closing brainstorm — automate the credentialed RHEL fetch** (to be filed under #91) — explore option C (credentialed download, arbitrary future RHEL versions 9.7/9.8/10) as a follow-on epic/challenge. Distinct topic from #93.
- **`mq-resiliency-lab-for-linux#666` Documentation review** (existing, closing) — site docs reflect the box-lifecycle verbs + cold-boot cadence, extending `docs/development/box-model.md`'s rebuild-tiers section with the box-rebake-in-place tier + the CLI. Final close gate.

## Cross-org dependencies (referenced by comment in #91 — cannot be epic sub-issues)

Epics cannot hold sub-issues in another **org**, so these are filed as proper issues in the target repo's standing ad-hoc epic and referenced from #91 by comment:

- **Tactical — `vrg-vm` post-build hook (Vergil tooling).** Add a VM-build post-build customization-script hook, the activation mechanism for Task 5. Verified absent in `vergil.toml` today.
- **Strategic — first-class cross-org tasks in the epic framework (plugin tooling).** Teach the `epic-create` skill to create a task in a remote repo and reference it from an epic in another org. Recurring need; filed as an idea/brainstorm.

## Self-Review

- **Spec coverage:** §4/§5 command surface + architecture → T1 (status + fleet/seams), T2 (build/rebuild + bootstrap share), T3 (clean); §6 DVD → T4 (in-VM verify) + T5 (host-side auto-stage); §7 cold-boot → T6; §8 testing/fail-loud/coverage/cold-rebuild → every task's TDD steps + the operational validation; §9 dependencies/follow-ons/cross-org → Bookend + Cross-org sections. All covered.
- **Placeholders:** none of the vague kind — file paths, TDD test bodies, and commands are concrete. The two deliberately-external unknowns are named honestly: the pinned DVD SHA-256 (operator-supplied from Red Hat, never fabricated) and the `vergil.toml` hook shape (depends on the cross-org issue; land a commented placeholder + small follow-up rather than block).
- **Type consistency:** `box.FLEET`/`BoxSpec`/`box_decision`/`BoxDecision`/`render_status`/`build_boxes`/`clean_boxes`/`verify_rhel_dvd`/`RHEL_DVD_SHA256` used consistently across T1–T4; `_select_boxes` shared by build/rebuild/clean; `coldboot.cold_boot_age_days`/`nudge`/`QUIET_DAYS`/`LOUD_DAYS` consistent in T6; reused cli symbols (`_LOCAL_BOX_BUILDERS`, `_box_build_steps`, `_ensure_local_boxes`, `_needed_local_boxes`, `parse_box_list`, `_resolved_nodes`, `state`, `run_steps`, `build_deps`) match `cli.py`.
