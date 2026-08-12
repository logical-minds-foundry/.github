# Arch-native box building — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restore host-arch-native fat-box building (regressed by the #70/#659
`arch: x86_64` pins) so every box builds for the host arch — arm64 on Apple
Silicon, x86_64 on the cloud — and bake the two Ubuntu arms (`mq-nativeha-ubuntu`,
`pcmk-ubuntu`) natively on both.

**Architecture:** `platforms.box_build_arch(entry, facts)` is the single authority
for a box's build arch — `entry.get("arch") or facts.arch`, so RHEL (explicit
`arch: x86_64`) is x86-pinned and an un-pinned Ubuntu box tracks the host. The
orchestrator (`cli._box_build_steps`) computes it and passes required `--arch` to
`build-fatbox.sh` (the #327 pattern); it also refuses a foreign-arch box build
(`is_foreign_box_build`, D11). The cache is uniformly `<box>-<arch>.box`
(per-host, migrated); platform *names* follow #276 (Ubuntu host-resolved, RHEL
single-name). Part B mirrors the native-HA-RHEL bake (#667/#668). All code is
unit-tested with injected facts; the two native bakes + cold rebuilds are the
acceptance.

**Tech Stack:** Python 3.12 + Typer (`mqlab`), pytest @ 100% branch coverage;
bash (`build-fatbox.sh`); Ansible (bake playbooks); libvirt/Vagrant lab; IBM MQ
9.4 Developer (Ubuntu debs, arch-specific).

**Design spec:** `epics/103-arch-native-box-building/spec.md` (Epic `.github#103`).

## Global Constraints

- **Python:** `requires-python >=3.12`; target `py312`.
- **Validation is one command:** `vrg-container-run -- vrg-validate` (ruff + mypy
  strict + pytest @ **100% branch coverage**). Authoritative per-task gate before
  every commit. Per-step red/green may run `uv run pytest <path> -v` in the worktree.
- **Git/GitHub:** `vrg-git` / `vrg-commit` (conventional commits). Raw `git`/`gh`
  denied.
- **Single authority (D1):** the build-arch rule lives only in
  `platforms.box_build_arch`; `build-fatbox.sh` consumes `--arch`, never re-derives.
- **Fail loud (D11/§10):** missing/invalid `--arch` → usage-die; a foreign-arch box
  build (e.g. RHEL on ARM) → hard refusal naming the x86 host; no silent emulation.
- **Two arch vocabularies — do not conflate:** libvirt/qemu use `aarch64`/`x86_64`
  (`<type arch=…>`, `qemu-system-<arch>`); Vagrant `box add --architecture` uses
  `arm64`/`amd64`. `--arch` carries the canonical `aarch64`/`x86_64` (matching
  `hostfacts`); `build-fatbox.sh` maps to `arm64`/`amd64` only at the `--architecture`
  seam.
- **Derive, never hardcode:** box/platform/arch come from `lab/topology.yaml` +
  `hostfacts`. No box-name, IP, or arch literal in shipped runtime code.
- **Cache is per-host, single-arch (D5):** `build/state/boxes/` is shared across
  worktrees on one host only. Migration is per-host; no cross-host reconciliation.
- **Platform-name asymmetry (D4):** cache filenames uniform `-<arch>`; RHEL keeps its
  single `mq-*-rhel9` platform name (node pins + baked-box tests unchanged); Ubuntu
  fat boxes become host-resolved.

## Task dependency graph

```text
              agnostic-code (PR-workable, land in mq-resiliency-lab-for-linux)
  T1 platforms.box_build_arch + is_foreign_box_build (authority + D11 predicate)
       │
       ├─► T2 orchestrator: pass --arch, refuse foreign build ─► T3 build-fatbox.sh consumes --arch
       │                                                              │
       ├─► T4 box.py arch-aware cache + `mqlab build migrate` rename ─┤
       │                                                              │
       └─► T5 topology un-pin Ubuntu fat boxes + arch-aware MQ acquisition (D3/D6/D10)
                    │                                                 │
                    └────────────┬────────────────────────────────────┘
                                 ▼
        T6 bake-nativeha-ubuntu (playbook+guard+repoint+boot test)
        T7 bake-pcmk-ubuntu     (playbook+guard+repoint cluster-only+boot test)
                                 │
        ┌────────────────────────┴───────────────────────────┐
   arm64-native (Apple Silicon)                       x86-native (cloud agent)
   D-arm  deploy: migrate + bake+cache arm64 arms      D-x86  deploy: migrate + bake+cache x86 arms
   V-arm  validate: arm64 cold rebuild boots baked     V-x86  validate: x86 cold rebuild boots baked
```

---

## Task 1: `platforms.box_build_arch` + `is_foreign_box_build` — the authority

**Platform:** agnostic-code · **Repo:** `mq-resiliency-lab-for-linux`

**Files:**

- Modify: `src/mqlab/platforms.py` (add after `build_domain_virt`, ~line 122)
- Test: `tests/test_platforms.py`

**Interfaces:**

- Consumes: a box-registry `entry` (`dict[str, Any]`, the `boxes[<name>]` value with
  an optional `"arch"`), `HostFacts` (`arch`), module constants `X86_64`, `AARCH64`.
- Produces:
  - `box_build_arch(entry: dict[str, Any], facts: HostFacts) -> str` — the build/guest
    arch: `entry.get("arch") or facts.arch`. Pure, display-safe.
  - `is_foreign_box_build(entry: dict[str, Any], facts: HostFacts) -> bool` — `True`
    when the entry pins an arch that is not the host's (`entry.get("arch")` set and
    `!= facts.arch`). Pure predicate; the orchestrator raises on it (D11).

- [ ] **Step 1: Write the failing tests**

Add to `tests/test_platforms.py` (fixtures `X86_KVM`, `X86_NOKVM`, `ARM_KVM` exist):

```python
def test_box_build_arch_rhel_is_x86_on_any_host():
    rhel = {"box": "rhel/9.6-x86_64", "arch": "x86_64"}
    assert p.box_build_arch(rhel, X86_KVM) == "x86_64"
    assert p.box_build_arch(rhel, ARM_KVM) == "x86_64"  # still x86 on the Mac


def test_box_build_arch_unpinned_ubuntu_tracks_host():
    ubuntu = {"box": "cloud-image/ubuntu-24.04"}  # no arch pin
    assert p.box_build_arch(ubuntu, X86_KVM) == "x86_64"
    assert p.box_build_arch(ubuntu, ARM_KVM) == "aarch64"


def test_is_foreign_box_build_true_for_rhel_on_arm():
    rhel = {"box": "rhel/9.6-x86_64", "arch": "x86_64"}
    assert p.is_foreign_box_build(rhel, ARM_KVM) is True
    assert p.is_foreign_box_build(rhel, X86_KVM) is False


def test_is_foreign_box_build_false_for_unpinned_ubuntu():
    ubuntu = {"box": "cloud-image/ubuntu-24.04"}
    assert p.is_foreign_box_build(ubuntu, ARM_KVM) is False
    assert p.is_foreign_box_build(ubuntu, X86_KVM) is False
```

- [ ] **Step 2: Run to verify they fail** — `uv run pytest tests/test_platforms.py -k "box_build_arch or foreign" -v` → FAIL (`AttributeError: … has no attribute 'box_build_arch'`).

- [ ] **Step 3: Implement** — in `src/mqlab/platforms.py`, after `build_domain_virt`:

```python
def box_build_arch(entry: dict[str, Any], facts: HostFacts) -> str:
    """The build/guest arch for a fat/base box (design D1).

    A box that pins its arch (RHEL: ``arch: x86_64``) keeps it on every host; an
    un-pinned box (host-resolved Ubuntu) tracks the host. Pure and display-safe —
    never raises — mirroring resolve(). build-fatbox.sh consumes the result via
    --arch; it does not re-derive it.
    """
    return entry.get("arch") or facts.arch


def is_foreign_box_build(entry: dict[str, Any], facts: HostFacts) -> bool:
    """True when this box pins an arch other than the host's — a build that would be
    fully emulated (e.g. the RHEL box on Apple Silicon). The orchestrator refuses it
    (design D11); this predicate stays pure so status paths can call it safely."""
    pinned = entry.get("arch")
    return bool(pinned) and pinned != facts.arch
```

- [ ] **Step 4: Run to verify they pass** — `uv run pytest tests/test_platforms.py -k "box_build_arch or foreign" -v` → PASS.

- [ ] **Step 5: Full gate** — `vrg-container-run -- vrg-validate` → PASS (both branches of each function covered).

- [ ] **Step 6: Commit** — `vrg-commit --type feat --scope platforms --message "box_build_arch + is_foreign_box_build — host-arch box-build authority (#103)"`

---

## Task 2: Orchestrator passes `--arch` and refuses foreign builds

**Platform:** agnostic-code · **Repo:** `mq-resiliency-lab-for-linux`

**Files:**

- Modify: `src/mqlab/cli.py` — `_box_build_steps` (~line 1022), the `platforms` import
  (~line 52), a box-registry accessor
- Test: `tests/test_cli_vm.py`

**Interfaces:**

- Consumes: `platforms.box_build_arch`, `platforms.is_foreign_box_build` (T1);
  `hostfacts` (`cli.probe`); the `boxes:` registry from `lab/topology.yaml`.
- Produces: `_box_build_steps(...)` argv now carries `--arch <aarch64|x86_64>` for each
  fat box; a foreign-arch build raises `StepFailedError` naming the x86 host before any
  step is scheduled. A helper `_box_registry() -> dict[str, dict]` returns
  `topology["boxes"]`.

- [ ] **Step 1: Write the failing tests** — add to `tests/test_cli_vm.py` (inject facts for determinism; CI is x86):

```python
def test_box_build_steps_passes_arch_for_ubuntu_fat_box(monkeypatch, tmp_path):
    monkeypatch.setenv("MQLAB_REPO_ROOT", str(tmp_path))
    monkeypatch.setattr(cli, "_box_registry", lambda: {"mq-ubuntu2404": {"box": "cloud-image/ubuntu-24.04"}})
    facts = HostFacts(arch=AARCH64, kvm=True, distro_family="apt", in_vergil=True)
    steps = cli._box_build_steps({"mq-ubuntu2404": "lab/boxes/build-fatbox.sh"}, {}, facts)
    argv = steps[0].command.argv
    assert "--box" in argv and "mq-ubuntu2404" in argv
    assert argv[argv.index("--arch") + 1] == "aarch64"


def test_box_build_steps_refuses_rhel_on_arm(monkeypatch, tmp_path):
    monkeypatch.setenv("MQLAB_REPO_ROOT", str(tmp_path))
    monkeypatch.setattr(cli, "_box_registry", lambda: {"mq-rdqm-rhel9": {"box": "rhel/9.6-x86_64", "arch": "x86_64"}})
    facts = HostFacts(arch=AARCH64, kvm=True, distro_family="apt", in_vergil=True)
    with pytest.raises(cli.StepFailedError, match="x86"):
        cli._box_build_steps({"mq-rdqm-rhel9": "lab/boxes/build-fatbox.sh"}, {}, facts)
```

- [ ] **Step 2: Run to verify they fail** — `uv run pytest tests/test_cli_vm.py -k "box_build_steps and (arch or refuse)" -v` → FAIL.

- [ ] **Step 3: Extend the import** (~line 52):

```python
from mqlab.platforms import (
    PlatformError, box_build_arch, build_domain_virt, ensure_resolved, is_foreign_box_build,
)
```

- [ ] **Step 4: Add the registry accessor** near `_resolved_nodes`:

```python
def _box_registry() -> dict[str, dict]:
    """The `boxes:` registry from lab/topology.yaml (box name -> entry)."""
    return _load_topology().get("boxes", {})
```

(Reuse whatever `_load_topology()`/topology loader `_resolved_nodes` already uses; if
none is exposed, read `lab/topology.yaml` via the existing `paths` helper.)

- [ ] **Step 5: Thread arch + refusal into `_box_build_steps`** — replace the argv build:

```python
    domain_type, cpu_mode = build_domain_virt(facts)
    registry = _box_registry()
    steps: list[CommandStep] = []
    for name, script in sorted(needed.items()):
        if name in present:
            continue
        entry = registry.get(name, {})
        # DEPRECATED, not removed (#103 D11): emulated cross-arch box builds (e.g. the
        # RHEL box on Apple Silicon) are refused here. The emulated build path in
        # build-fatbox.sh + build_domain_virt's TCG branch is retained for a future
        # standalone non-HA/DR RHEL lab; re-enable by lifting this guard.
        if is_foreign_box_build(entry, facts):
            raise StepFailedError(
                f"box {name} pins arch {entry['arch']} but this host is {facts.arch}: "
                f"emulated cross-arch box builds are disabled (#103 D11). Build it on the x86 host."
            )
        argv = ["bash", str(repo_root() / script)]
        if script.endswith("build-fatbox.sh"):
            argv += ["--box", name, "--arch", box_build_arch(entry, facts)]
        argv += ["--domain-type", domain_type, "--cpu-mode", cpu_mode]
        if force:
            argv.append("--rebuild-box")
        steps.append(CommandStep(f"box {name}", Command(argv)))  # noqa: S607
    return steps
```

(The base-OS `build-box.sh` still gets only the virt flags — it is x86-only and
unchanged; `--arch` is appended only on the `build-fatbox.sh` branch.)

- [ ] **Step 6: Run to verify they pass** — `uv run pytest tests/test_cli_vm.py -k box_build_steps -v` → PASS (existing arg-passing tests still green).

- [ ] **Step 7: Full gate** — `vrg-container-run -- vrg-validate` → PASS.

- [ ] **Step 8: Commit** — `vrg-commit --type feat --scope cli --message "pass --arch to build-fatbox.sh; refuse foreign-arch box builds (#103)"`

---

## Task 3: `build-fatbox.sh` consumes `--arch`

**Platform:** agnostic-code · **Repo:** `mq-resiliency-lab-for-linux`

**Files:**

- Modify: `lab/boxes/build-fatbox.sh` — arg loop (~L48), usage (~L34), cache
  (`CACHE`/`HASH_FILE` L95-96), base-box add (L166), base-img find (L173-174), the
  build-domain XML (`<os><type arch=…>` L223), the emulator element
- Test: `tests/test_build_fatbox_usage.py` (new; mirrors `tests/test_build_box_usage.py`)

**Interfaces:**

- Consumes: `--arch <aarch64|x86_64>` from T2 (required).
- Produces: an arch-correct guest domain + base-box add + cache filename
  `<box>-<arch>.box` / `<box>-<arch>.manifest-hash`; usage-die on missing/invalid
  `--arch`.

- [ ] **Step 1: Write the failing usage test** — create `tests/test_build_fatbox_usage.py`:

```python
"""build-fatbox.sh must require --arch and die loudly when it is missing/invalid —
mqlab always supplies it (#103). Exercises only the early arg-validation path."""
from __future__ import annotations
import subprocess
from pathlib import Path

SCRIPT = Path(__file__).resolve().parents[1] / "lab" / "boxes" / "build-fatbox.sh"

def _run(*a: str) -> subprocess.CompletedProcess[str]:
    return subprocess.run(["bash", str(SCRIPT), *a], capture_output=True, text=True, check=False)  # noqa: S603

def test_missing_arch_dies():
    r = _run("--box", "mq-ubuntu2404", "--domain-type", "kvm", "--cpu-mode", "host-passthrough")
    assert r.returncode != 0
    assert "--arch" in (r.stderr + r.stdout)

def test_invalid_arch_dies():
    r = _run("--box", "mq-ubuntu2404", "--arch", "bogus", "--domain-type", "kvm", "--cpu-mode", "host-passthrough")
    assert r.returncode != 0
```

- [ ] **Step 2: Run to verify it fails** — `uv run pytest tests/test_build_fatbox_usage.py -v` → FAIL (arch not yet required).

- [ ] **Step 3: Add `--arch` to the arg loop + validation** — add `ARCH=""` beside
  `BOX=""`; add `--arch) ARCH="${2:-}"; shift ;;` to the `while` case; after the
  `--box` case-map, add:

```bash
case "$ARCH" in
  aarch64|x86_64) ;;
  *) echo "ERROR: --arch must be 'aarch64' or 'x86_64' (got '${ARCH}')" >&2; usage; exit 2 ;;
esac
# Vagrant's `box add --architecture` speaks arm64/amd64, not aarch64/x86_64.
case "$ARCH" in
  aarch64) VAGRANT_ARCH=arm64 ;;
  x86_64)  VAGRANT_ARCH=amd64 ;;
esac
```

Update the `usage()` heredoc to list `--arch <aarch64|x86_64>` as REQUIRED.

- [ ] **Step 4: Arch the cache filename** (L95-96):

```bash
CACHE="$CACHE_DIR/${BOX}-${ARCH}.box"
HASH_FILE="$CACHE_DIR/${BOX}-${ARCH}.manifest-hash"
```

- [ ] **Step 5: Arch the base-box add** (L166) — add `--architecture "$VAGRANT_ARCH"`:

```bash
vagrant box list | grep -q "^${BASE_BOX} " \
  || vagrant box add --provider libvirt --architecture "$VAGRANT_ARCH" "$BASE_BOX"
```

- [ ] **Step 6: Arch the base-img find** (L173-174) — scope the `find` to the arch by
  preferring the `--architecture` path libvirt lays down; if the box dir is
  arch-partitioned, filter on `"$VAGRANT_ARCH"`, else the single provider dir is
  correct. (Keep the existing `tail -n1` fallback.)

- [ ] **Step 7: Arch the build-domain guest** (L223) — replace the hardcoded literal:

```bash
  <os><type arch='@ARCH@' machine='@MACHINE@'>hvm</type></os>
```

and substitute in the `sed` that renders the domain: `@ARCH@` → `$ARCH`, `@MACHINE@`
→ `q35` for x86_64 and `virt` for aarch64. Point the `<emulator>` at
`/usr/bin/qemu-system-${ARCH}`. (If the domain XML is a heredoc rather than a `.tpl`,
interpolate `$ARCH`/machine/emulator directly.)

- [ ] **Step 8: Run to verify the usage test passes** — `uv run pytest tests/test_build_fatbox_usage.py -v` → PASS.

- [ ] **Step 9: Full gate** — `vrg-container-run -- vrg-validate` → PASS (shellcheck-clean if run).

- [ ] **Step 10: Commit** — `vrg-commit --type feat --scope box-build --message "build-fatbox.sh: arch-aware guest domain + cache via required --arch (#103)"`

---

## Task 4: `box.py` arch-aware cache + `mqlab build migrate` rename

**Platform:** agnostic-code · **Repo:** `mq-resiliency-lab-for-linux`

**Files:**

- Modify: `src/mqlab/box.py` — `BoxSpec` (add `arch`), `_build_fleet` (compute arch),
  `cache_artifact` (`<name>-<arch>.box`), `render_status` (arch column)
- Modify: the `mqlab build migrate` implementation (find via
  `grep -rn "def .*migrate" src/mqlab`) — rename legacy `<box>.box` → `<box>-x86_64.box`
- Test: `tests/test_box.py`, `tests/test_build_migrate.py` (or the existing migrate test)

**Interfaces:**

- Consumes: `platforms.box_build_arch` (T1), `hostfacts.probe`, the `boxes:` registry.
- Produces: `BoxSpec.arch: str`; `cache_artifact == f"{name}-{arch}.box"`; the base box
  becomes `rhel-9.6-x86_64-libvirt.box` unchanged (already arch-tagged). Migration
  renames every legacy `build/state/boxes/<box>.box` (+ `.manifest-hash`) to the
  `-x86_64` form, idempotently.

- [ ] **Step 1: Write failing tests** — in `tests/test_box.py`, assert the fleet's
  fat boxes carry `arch` and an arch-suffixed `cache_artifact` for injected x86 facts
  (`mq-ubuntu2404` → `arch="x86_64"`, `cache_artifact="mq-ubuntu2404-x86_64.box"`); for
  injected arm64 facts, `mq-ubuntu2404` → `aarch64`/`…-aarch64.box`, while
  `mq-rdqm-rhel9` stays `x86_64`/`…-x86_64.box` on both. In `tests/test_build_migrate.py`,
  seed a tmp `boxes/` with `mq-ubuntu2404.box` + `.manifest-hash` and assert migrate
  renames them to `-x86_64` and is a no-op on a second run.

- [ ] **Step 2: Run to verify they fail** → FAIL.

- [ ] **Step 3: Implement** — add `arch: str` to `BoxSpec`; in `_build_fleet` derive
  `arch = platforms.box_build_arch(registry_entry, facts)` per box (facts injected,
  default `probe()`), set `cache_artifact=f"{name}-{arch}.box"`; the base box keeps its
  literal artifact. Add the `ARCH` column to `render_status`. In `build migrate`, for
  each fleet fat box, if the legacy `<name>.box` exists and `<name>-x86_64.box` does
  not, `rename` both the `.box` and `.manifest-hash`.

- [ ] **Step 4: Run to verify they pass** → PASS.

- [ ] **Step 5: Full gate** — `vrg-container-run -- vrg-validate` → PASS.

- [ ] **Step 6: Commit** — `vrg-commit --type feat --scope box --message "box.py arch-aware cache + build migrate renames to <box>-<arch>.box (#103)"`

---

## Task 5: Un-pin the Ubuntu fat boxes + arch-aware MQ acquisition (D3/D6/D10)

**Platform:** agnostic-code · **Repo:** `mq-resiliency-lab-for-linux`

**Files:**

- Modify: `lab/topology.yaml` — drop `arch: x86_64` from `obs-ubuntu2404` (L60),
  `infra-ubuntu2404` (L70), `mq-ubuntu2404` (L82) and update their pin-rationale
  comments to "host-resolved (#103) — arch tracks the host"
- Modify: `src/mqlab/manifest.py` — make `_ARCH_SUFFIX`/`tarball_name`/`setup_platforms`
  resolve the Ubuntu fat-box suffix by **host arch**, not a box-name literal (D10)
- Modify: `scripts/fetch-mq.sh` — fetch the host-arch Ubuntu deb tarball for the fat
  boxes (audit against `manifest`, per #276 §7)
- Test: `tests/test_manifest.py`, `tests/test_platforms.py`

**Interfaces:**

- Consumes: `box_build_arch` (T1); `hostfacts`.
- Produces: an un-pinned Ubuntu fat-box registry entry resolves its arch from facts
  (so `_provider`/`box_build_arch` yield the host arch); acquisition stages the
  host-arch Ubuntu tarball (`UbuntuLinuxARM64` on Apple Silicon, `UbuntuLinuxX64` on the
  cloud) into `build/mq/`.

- [ ] **Step 1: Write failing tests** — in `tests/test_platforms.py`, assert an
  un-pinned Ubuntu fat-box entry resolves `arch == facts.arch` under both fact sets. In
  `tests/test_manifest.py`, assert the fat-box MQ tarball for injected arm64 facts is
  the `UbuntuLinuxARM64` name and for x86 facts the `UbuntuLinuxX64` name.

- [ ] **Step 2: Run to verify they fail** → FAIL (today the fat box maps to a single
  x86_64 suffix).

- [ ] **Step 3: Un-pin the topology** — remove the three `arch: x86_64` lines; confirm
  `_provider` fills `arch = box.get("arch", facts.arch)` (add the `.get` default if the
  resolver currently requires `box["arch"]`), so an un-pinned Ubuntu box tracks the
  host and RHEL (still pinned) does not.

- [ ] **Step 4: Arch-aware acquisition** — change the Ubuntu fat-box suffix resolution
  so it flows through the host-arch/`box_build_arch` path rather than the box-name
  literal in `_ARCH_SUFFIX`; a fat Ubuntu box on arm64 stages `UbuntuLinuxARM64`. Keep
  the RHEL/`LinuxX64` entries literal. Audit `scripts/fetch-mq.sh` for the same rule.

- [ ] **Step 5: Run to verify they pass** → PASS.

- [ ] **Step 6: Full gate** — `vrg-container-run -- vrg-validate` → PASS.

- [ ] **Step 7: Commit** — `vrg-commit --type feat --scope topology --message "un-pin Ubuntu fat boxes to host-resolved; arch-aware MQ acquisition (#103)"`

---

## Task 6: Bake `mq-nativeha-ubuntu` — playbook, guard, repoint, boot test

**Platform:** agnostic-code · **Repo:** `mq-resiliency-lab-for-linux`

**Files:**

- Create: `ansible/bake-nativeha-ubuntu.yml` (mirror `ansible/bake-nativeha-rhel.yml`:
  install-body-only, node-exporter baked+enabled, alloy install-half inert)
- Modify: the Ubuntu Native-HA install role's `install-Debian`/`install.yml` — add the
  `stat`-marker skip-if-baked guard (mirror `roles/mq-nativeha/tasks/install-RedHat.yml`
  #668)
- Modify: `lab/boxes/build-fatbox.sh` — add `mq-nativeha-ubuntu) BASE_KIND=ubuntu;
  BASE_BOX="cloud-image/ubuntu-24.04"; BAKE=nativeha-ubuntu ;;` to the `--box` case-map
  - the usage list
- Modify: `src/mqlab/cli.py` `_LOCAL_BOX_BUILDERS` — register `mq-nativeha-ubuntu` →
  `build-fatbox.sh` (fleet + box CLI pick it up via `_build_fleet`)
- Modify: `lab/topology.yaml` — add a host-resolved `mq-nativeha-ubuntu` `boxes:` entry
  (no `arch:` pin, no dvd, no `extra_disk`); repoint the `nha-ubuntu-*` nodes' `platform`
  to it
- Test: `tests/test_topology_nativeha.py` — add
  `test_nativeha_ubuntu_nodes_boot_the_baked_fat_box` (mirror the RHEL one: assert
  `platform == "mq-nativeha-ubuntu"`, no `extra_disk`)

**Interfaces:**

- Consumes: the arch-aware builder (T3), un-pinned host-resolved boxes + acquisition
  (T5), arch-aware fleet (T4).
- Produces: a buildable `mq-nativeha-ubuntu` box (per-arch cache) and `nha-ubuntu-*`
  nodes that boot it; the baked MQ install is skipped on a baked node.

> **Merged ≠ runnable:** this PR repoints to a box that D-arm/D-x86 build later — see
> the Operational-tasks note. The boot test here is a topology assertion (passes in-PR);
> a real cold rebuild of the arm requires the per-host deployment first.

- [ ] **Step 1: Write the failing boot test** — add
  `test_nativeha_ubuntu_nodes_boot_the_baked_fat_box` asserting the `nha-ubuntu-*` nodes
  resolve `platform == "mq-nativeha-ubuntu"`. Run → FAIL (nodes still host-resolved base).

- [ ] **Step 2: Author `bake-nativeha-ubuntu.yml`** — copy `bake-nativeha-rhel.yml`,
  switch the install include to the Ubuntu Native-HA install path (`install-Debian` /
  the deb install body), keep node-exporter baked+enabled and alloy install-half inert.
  `ansible-playbook --syntax-check ansible/bake-nativeha-ubuntu.yml`.

- [ ] **Step 3: Add the skip-if-baked guard** to the Ubuntu Native-HA install tasks —
  the `stat: /opt/mqm/inc/cmqc.h` marker gating the tar copy/unpack, mirroring
  `install-RedHat.yml` (#668), so a baked node skips and an un-baked base-box node still
  installs.

- [ ] **Step 4: Register the box** — add the `--box` case-map entry + usage line in
  `build-fatbox.sh`; add `mq-nativeha-ubuntu` to `_LOCAL_BOX_BUILDERS`. Add its
  `_ARCH_SUFFIX`/host-arch acquisition coverage if a new key is needed (per T5).

- [ ] **Step 5: Add the `boxes:` entry + repoint** — host-resolved `mq-nativeha-ubuntu`
  (no `arch:`), then repoint `nha-ubuntu-a*/b*` `platform:` to it.

- [ ] **Step 6: Run to verify the boot test passes** → PASS.

- [ ] **Step 7: Full gate** — `vrg-container-run -- vrg-validate` → PASS.

- [ ] **Step 8: Commit** — `vrg-commit --type feat --scope boxes --message "bake mq-nativeha-ubuntu + skip-if-baked the Ubuntu Native-HA install; repoint nha-ubuntu nodes (#103)"`

---

## Task 7: Bake `pcmk-ubuntu` (cluster nodes only) — playbook, guard, repoint, boot test

**Platform:** agnostic-code · **Repo:** `mq-resiliency-lab-for-linux`

**Files:**

- Create: `ansible/bake-pcmk-ubuntu.yml` (mirror T6; the Pacemaker/MQ cluster-node
  install body only — **not** the SAN target install)
- Modify: the Pacemaker MQ install role — add the skip-if-baked guard (as T6 Step 3)
- Modify: `lab/boxes/build-fatbox.sh` — add `pcmk-ubuntu) … BAKE=pcmk-ubuntu ;;`
- Modify: `src/mqlab/cli.py` `_LOCAL_BOX_BUILDERS` — register `pcmk-ubuntu`
- Modify: `lab/topology.yaml` — host-resolved `pcmk-ubuntu` `boxes:` entry; repoint the
  **cluster** nodes only (`pcmk-a1..3`, `pcmk-b1..3`) — leave `san-a`/`san-b`
  host-resolved on the base box (D8)
- Test: `tests/test_topology_pcmk.py` (or the pcmk topology test) —
  `test_pcmk_ubuntu_cluster_nodes_boot_the_baked_fat_box` (assert `pcmk-a*/b*` →
  `pcmk-ubuntu`, and `san-a`/`san-b` are **not** repointed)

**Interfaces:**

- Consumes: T3/T4/T5 as in T6.
- Produces: a `pcmk-ubuntu` box and repointed cluster nodes; SAN nodes unchanged.

> **Merged ≠ runnable:** as T6 — the repoint precedes the per-host bake (D-arm/D-x86);
> the boot test is a topology assertion, not a live boot.

- [ ] **Step 1: Write the failing boot test** — assert `pcmk-a1..3`/`pcmk-b1..3` resolve
  `platform == "pcmk-ubuntu"` **and** `san-a`/`san-b` still resolve to the host-resolved
  base Ubuntu platform. Run → FAIL.

- [ ] **Step 2: Author `bake-pcmk-ubuntu.yml`** — cluster-node MQ+Pacemaker install body
  only; `--syntax-check`.

- [ ] **Step 3: Confirm (or extend) the skip-if-baked guard** on the pcmk MQ install
  path. The pcmk cluster nodes install MQ via `roles/mq-install` (`tasks/main.yml`),
  which **already** carries the `cmqc.h` skip-if-baked guard (#648/#659) *and* the
  arch-derived tarball — so verify a baked `pcmk-ubuntu` node short-circuits it; add a
  guard only if the pcmk path routes through a separate install task that lacks one.

- [ ] **Step 4: Register the box** — case-map + `_LOCAL_BOX_BUILDERS` + acquisition
  coverage.

- [ ] **Step 5: Add the `boxes:` entry + repoint the cluster nodes only** — verify the
  SAN nodes are untouched.

- [ ] **Step 6: Run to verify the boot test passes** → PASS.

- [ ] **Step 7: Full gate** — `vrg-container-run -- vrg-validate` → PASS.

- [ ] **Step 8: Commit** — `vrg-commit --type feat --scope boxes --message "bake pcmk-ubuntu cluster nodes + skip-if-baked; repoint pcmk cluster nodes, SAN nodes unchanged (#103)"`

---

## Operational tasks (filed under #103; run via issue-deploy / issue-validate — NOT PR-workable)

Seeded at task-filing time (step 9 of epic-create) with `--blocked-by` the impl tasks.
Each carries a **platform** the human routes to the agent on that host.

> **Merged ≠ runnable (dual-arch consequence).** Merging T6/T7 repoints the
> `nha-ubuntu`/`pcmk` cluster nodes to boxes that **do not exist until these
> deployments run** — so a fresh cold rebuild of those arms fails on `develop` until
> **D-arm** (arm64) *and* **D-x86** (x86) have each baked+cached on their host. This is
> the framework's merged-vs-deployed model (a deployment's closure *is* the "usable"
> signal); the `Blocked-by` edges (T6/T7 → D-arm/D-x86 → V-arm/V-x86) sequence it, and
> `epic-implement` will not surface a validation as runnable until its deployment closes.
> Unlike the x86-only #88 (one task repointed *and* proved boot), the two-host bake here
> is intrinsically operational and split.

- **D-arm — Deployment (arm64-native, Apple Silicon).** Run `mqlab build migrate` on
  the Mac (rename its x86 cache — no-op if none), then bake + cache the **arm64**
  `mq-nativeha-ubuntu` and `pcmk-ubuntu` boxes (`build-fatbox.sh … --arch aarch64`, via
  `mqlab box build`). Precondition self-check: host arch is `aarch64`. Blocked-by
  **T6, T7** (+ T3/T4/T5). Run with `issue-deploy`.
- **D-x86 — Deployment (x86-native, cloud agent).** Same on the cloud x86 host
  (`--arch x86_64`), plus re-bake any RHEL box whose cache the migration renamed.
  Precondition: host arch is `x86_64`, native KVM. Blocked-by **T6, T7**. Run with
  `issue-deploy`.
- **V-arm — Validation (arm64-native).** A one-shot **cold rebuild** on Apple Silicon:
  `mqlab rebuild nativeha-ubuntu` and `mqlab rebuild pcmk-ubuntu` come up in one pass
  booting the **baked arm64** boxes natively (no emulation, no per-run MQ install on the
  baked cluster nodes; transcript proves the `cmqc.h` guard short-circuits). Blocked-by
  **D-arm**. Run with `issue-validate`.
- **V-x86 — Validation (x86-native).** The same cold rebuild on the cloud x86 host
  booting the **baked x86** boxes natively. Blocked-by **D-x86**. Run with
  `issue-validate`.

**Both validations green = Part A restored (arch-native building) AND Part B baked**
(the spec's §11 acceptance). The cold-rebuild acceptance gate applies to both.

## Bookend tasks (already created)

- **`.github#106`** documentation (this spec + plan) — closed by the docs PR.
- **`.github#107`** brainstorm(closing): decommission the `pcmk-rhel` arm.
- **`.github#108`** brainstorm(closing): SAN-hosts epic (`san-a` + `san-b`).
- **`mq-resiliency-lab-for-linux#698`** docs-review(closing): site docs reflect
  arch-native box building (final gate; may spawn per-repo doc tasks).

---

## Self-Review

**Spec coverage:**

- D1 `box_build_arch` → T1. ✓
- D2 orchestrator `--arch` → T2 + T3. ✓
- D3 un-pin Ubuntu fat boxes → T5. ✓
- D4 uniform cache filename / #276 platform names → T3 (cache) + T4 (fleet) + T5/T6/T7
  (Ubuntu host-resolved entries; RHEL single-name untouched). ✓
- D5 per-host single-arch cache + migration → T4. ✓
- D6 per-arch MQ media (install) → T5. ✓
- D7 bake the two Ubuntu arms, cluster nodes only → T6 + T7. ✓
- D8 SAN nodes host-resolved → T7 Step 1/5 (explicit assertion). ✓
- D9 `pcmk-rhel` decommission → bookend `#107` (out of scope here). ✓
- D10 arch-aware acquisition → T5 Step 4 (+ per-box coverage in T6/T7). ✓
- D11 refuse + deprecate RHEL-on-ARM → T1 (`is_foreign_box_build`) + T2 (orchestrator
  raise). ✓
- §11 acceptance (both native cold rebuilds) → V-arm + V-x86. ✓

**Placeholder scan:** the `@ARCH@`/`@MACHINE@` are intentional template tokens;
integration tasks (T4–T7) intentionally describe the change against exact files/lines
rather than repeating full Ansible/YAML bodies (house style, matching
`epics/88-nativeha-rhel/plan.md`) — the seams, signatures, and assertions are concrete.

**Type consistency:** `box_build_arch(entry, facts) -> str` and
`is_foreign_box_build(entry, facts) -> bool` are used identically in T1 (def), T2
(orchestrator), T4 (fleet). `--arch` values are the canonical `aarch64`/`x86_64`
everywhere; the `arm64`/`amd64` mapping is isolated to `build-fatbox.sh`'s
`--architecture` seam (Global Constraints).

**Open items carried from spec §12** (resolve during the tasks that touch them, flag in
alignment): the exact host-resolved Ubuntu fat-box topology entry shape (this plan uses
a **single host-resolved entry**, arch omitted — T5/T6/T7); the precise `_ARCH_SUFFIX`
seam (T5); the mixed old/new cache in migrate (T4); phased-startup/machine-id parity for
the Ubuntu arms (fold into T6/T7 if the cold rebuild surfaces it).
