# Multi-version OS axis — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build each MQ stack on a chosen OS major version (Ubuntu 24/26, RHEL 9/10),
selected by a build file and pinned per running instance. Defaults are Ubuntu 26 and
RHEL 10, subject to the IBM-support gate. Shared and infra nodes move to Ubuntu 26.

**Architecture:** A version layer (`src/mqlab/versions.py` plus
`src/mqlab/instances.py`) sits **in front of** the existing host resolver
(`platforms.resolve` → `build/work/lab/topology.resolved.yaml` → the dumb-consumer
`lab/Vagrantfile`):

- It reads the committed catalog `lab/versions.yaml`, an optional build file and the
  per-stack instance records.
- It hands `platforms.resolve` a concrete box for every node.
- Topology names box **roles**, and box names are generated as
  `<role>-<os><major>`.
- The shell builders become dumb: mqlab passes them every input as flags, and their
  `case` tables are deleted.
- Phase 1 performs this refactor at today's versions with no behavior change.
  Phases 2 and 3 add Ubuntu 26 and RHEL 10.

**Tech Stack:** Python 3 + Typer (`mqlab`, pytest at 100% branch coverage), PyYAML,
bash box builders (`lab/boxes/`), Vagrant + vagrant-libvirt, Ansible, IBM MQ 10.0
Developer edition, Ubuntu cloud images, RHEL DVD + kickstart.

**Spec:** `epics/280-os-version-axis/spec.md` (this directory). Executors read both.

**Repos:**

- Implementation tasks land in `logical-minds-foundry/mq-resiliency-lab-for-linux`.
- This plan and the spec live in `logical-minds-foundry/.github`.

## Global Constraints

- **Version tokens** (`ubuntu24`, `2404`, `noble`, `rhel9`, `9.6`, `el9`, and the 26
  and 10 forms) appear only in:
  - `lab/versions.yaml` and generated box names;
  - per-major kickstart files;
  - `ansible/**/vars/<Distribution>-<major>.yml`;
  - the `box gc` retired-names table.
- **Box naming:** `<role>-<os><major>`. Roles are `infra`, `obs`, `mq-client`,
  `san`, `mq-nativeha`, `pcmk` and `mq-rdqm`. RHEL base boxes are
  `rhel/<major>-x86_64`. Caches are `build/state/boxes/<box>-<arch>.box`.
- **Logical names never carry a version.** This covers stacks, nodes, inventory
  groups, QM names, playbooks and dashboards. Playbooks may carry the OS **family**
  (`bake-nativeha-rhel.yml`), because a family is not a version token.
- **Selection is file-based only:** `mqlab bootstrap <stack> --config <file>`. There
  is no `--os` flag.
- **Fail loud.** There are no silent fallbacks to a default for a running instance or
  an unmatched OS. mqlab's errors name fully-qualified `mqlab` commands (layered
  error vocabulary).
- **`build/` paths** go only through `mqlab.paths` (`state()`, `work()`, `cache()`);
  never hard-code a `build/<X>` path.
- **Infra OS** is `ubuntu:26` after Phase 2 (`ubuntu:24` during Phase 1). It covers
  obs, infra-svc, infra-client, svc-sim, app-client, mon-probe, san-a and san-b, and
  is never selectable.
- **Support gate:** a stack's `default` must never point at an entry carrying
  `ibm_support.status: unsupported`.
- **RDQM is RHEL 9 only.** RHEL on aarch64 stays refused.
- **Validation gate is exactly one command:**
  `vrg-container-run -- vrg-validate`.
  - Dev-loop tests: `uv run pytest …` (a build tool only; never embedded in shipped
    code).
  - Coverage: `uv run pytest --cov=src --cov-branch --cov-fail-under=100`.
- **Cold-rebuild acceptance gate:** bring-up and provisioning changes are accepted
  only after a full cold rebuild proves them in one pass. Lint-green is not done.
  The human operates the lab.
- **One GitHub issue per task**, on `feature/<issue>-<slug>` off `develop`, committed
  with `vrg-commit`, with a PR into `develop`.

## Review Focus

1. **Two stacks up at different majors** (`nativeha-ubuntu` on 24 and `pcmk-ubuntu`
   on 26). The render must give each stack's nodes their own record's boxes, and the
   shared nodes the infra boxes. Pinned by
   `test_render_combines_records_across_stacks` (Task 3).
2. **Resuming with the same build file** (`bootstrap s --from provision --config
   same.yaml`) is accepted. A *different* file is refused, naming
   `mqlab teardown s`. Pinned by `test_config_matching_record_accepted` and
   `test_config_conflicting_record_refused` (Task 3).
3. **A record survives a default flip.** A stack recorded at `ubuntu:24` keeps
   resolving to 24 after the catalog default moves to 26. Pinned by
   `test_record_pins_through_default_change` (Task 3).
4. **Stale Vagrant `box_meta` after the rename.** A guest whose cached box_meta names
   a retired box (`mq-nativeha-ubuntu`) is reconciled before `vagrant up`, so the old
   box doesn't boot. Pinned by `test_reconcile_forgets_retired_box_names` (Task 4).
5. **`commons up` with no stack running** resolves the shared nodes to the infra OS,
   with no record lookup. Pinned by `test_commons_render_uses_infra_without_records`
   (Task 2).

## Task dependency graph

```text
Phase 0   T0a (IBM support research)     T0b (live spike: ubuntu-26.04 box + x86-64-v3)
                     \                          /
Phase 1   T1 (catalog + resolver, pure)
           ├─► T4 (builders take catalog inputs; renames; FLEET from catalog; gc retired names)
           │     └─► T2 (topology → roles; render via version layer)
           │           └─► T3 (--config + instance records + refusal + teardown)
           ├─► T5 (Ansible per-version vars indirection)
           └─► T6 (baked SAN box; retire sandeb)   [needs T4]
          T7 (version-token guardrail)  [needs T2,T3,T5,T6]
          D1 deploy: bake+cache Phase-1 boxes  [needs T7]
          V1 validate: cold rebuild × 4 stacks @ ubuntu24/rhel9  [needs D1]
Phase 2   T8 (Ubuntu 26 entry, role fix-ups, infra→26)  [needs V1, T0a, T0b]
          D2 deploy: bake ubuntu26 boxes  [needs T8]
          V2 validate: both Ubuntu stacks @26 + @24-on-26-commons  [needs D2]
          T9 flip Ubuntu stack defaults → 26 (if IBM-supported)  [needs V2]
Phase 3   T10 (RHEL 10 entry, base box, vars, v3 gate)  [needs V1, T0a, T0b]
          D3 deploy: bake rhel/10 + mq-nativeha-rhel10  [needs T10; human stages RHEL 10 DVD]
          V3 validate: nativeha-rhel-crr@10  [needs D3]
          T11 flip nativeha-rhel-crr default → rhel:10 (if IBM-supported)  [needs V3]
Closing   docs review (#1263) → follow-on brainstorm MQ axis (.github#282) → retrospective (.github#283)
```

Phases 2 and 3 are independent of each other once V1 is green. Either may proceed
while the other is paused on a spike finding.

---

## Phase 0: Spikes

### Task T0a: OS/MQ support matrix research (docs PR)

**Files:**

- Create: `docs/reference/os-version-support-matrix.md`

**Interfaces:**

- Produces:
  - For each of {Ubuntu 26.04, RHEL 10} × {MQ 10.0 server, Native HA, RDQM}: a cited
    status of `supported` / `unsupported` / `unknown`.
  - The RHEL 10 point release to pin.
  - On Ubuntu 26.04: the package names and availability for `pacemaker`, `pcs`,
    `fence-agents-virsh`, `drbd-utils`, `targetcli-fb` and `linux-modules-extra-*`.

  T8, T10, T9 and T11 consume these.

- [ ] **Step 1:** Fetch IBM's MQ 10.0 system requirements pages through the repo tool
  and cite the cached `content.txt` with its `source_url`:
  `python3 tools/ibm_doc_cache.py "https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=…"`.
  IBM *Support* pages (the SPCR/system-requirements PDFs) are not reachable by the
  tool. List them as human-fetch items in the doc. Do not paraphrase them from memory.
- [ ] **Step 2:** For the Ubuntu package questions, record `apt-cache policy <pkg>`
  output from a 26.04 guest. T0b provides the guest; until it exists, mark these
  "pending T0b".
- [ ] **Step 3:** Write the matrix. Every cell is either **data** (with a citation) or
  **judgment** (labelled as such). Add a "Defaults decision" section that applies spec
  §4.1: a version may become a default only if it is cited as `supported`.
- [ ] **Step 4:** `vrg-container-run -- vrg-validate`, then commit:
  `vrg-commit --type docs --scope reference --message "OS/MQ support matrix for the OS-axis epic"`.

### Task T0b: Live spike — Ubuntu 26.04 base box + x86-64-v3 exposure (report PR)

**Files:**

- Create: `docs/reports/2026-10-os-axis-spike.md`

**Interfaces:**

- Produces:
  - The Vagrant Cloud box name and pinned version for Ubuntu 26.04 (amd64 + arm64),
    or a "not available" finding with the fallback chosen.
  - Whether a 26.04 guest boots under our vagrant-libvirt on both hosts.
  - Whether a KVM guest with `cpu_mode: host-passthrough` on the cloud host exposes
    `avx2 bmi1 bmi2 fma movbe f16c abm xsave`.
  - Whether TCG `cpu_mode: maximum` exposes the same flags.

  T8 and T10 consume these.

- [ ] **Step 1 (human-run):** On each host:
  `vagrant box add cloud-image/ubuntu-26.04 --provider libvirt --architecture <amd64|arm64>`.
  Record the version printed, or the error.
- [ ] **Step 2 (human-run):** Boot it with a throwaway one-node Vagrantfile in
  `$(mqlab build path temp)/spike-2604/`. Record `cat /etc/os-release`,
  `uname -r` and the boot time.
- [ ] **Step 3 (human-run):** In a KVM guest on the cloud host, run
  `grep -o -w -E 'avx2|bmi1|bmi2|fma|movbe|f16c|abm|xsave' /proc/cpuinfo | sort -u`.
  Repeat in a TCG guest.
- [ ] **Step 4:** In the same 26.04 guest, run `apt-cache policy` for each T0a Step 2
  package. Paste the output into the report and into T0a's pending cells.
- [ ] **Step 5:** `vrg-container-run -- vrg-validate`, then commit:
  `vrg-commit --type docs --scope reports --message "OS-axis spike: ubuntu-26.04 box + x86-64-v3"`.

**Go/no-go:**

- No 26.04 libvirt box → Phase 2 pauses (comment on the epic). Phase 3 proceeds.
- No v3 flags under KVM on the cloud host → Phase 3 pauses. Phase 2 proceeds.

---

## Phase 1: Resolver refactor at today's versions

### Task T1: The catalog and the version resolver (pure)

**Files:**

- Create: `lab/versions.yaml`
- Create: `src/mqlab/versions.py`
- Modify: `src/mqlab/paths.py` (add `versions_catalog_path()`)
- Test: `tests/test_versions.py`

**Interfaces:**

- Consumes: `mqlab.hostfacts.HostFacts` (`arch`, `kvm`), and `paths.repo_root()`.
- Produces (exact; later tasks rely on these):

```python
class VersionError(RuntimeError): ...

@dataclass(frozen=True, order=True)
class OsRef:
    family: str          # "ubuntu" | "rhel"
    major: int
    @classmethod
    def parse(cls, text: str) -> "OsRef": ...   # "rhel:9" -> OsRef("rhel", 9); bad text -> VersionError
    @property
    def token(self) -> str: ...                 # "rhel9", "ubuntu26"
    def __str__(self) -> str: ...               # "rhel:9"

@dataclass(frozen=True)
class OsEntry:
    ref: OsRef
    base_box: str               # "cloud-image/ubuntu-24.04" | "rhel/9-x86_64"
    base_box_version: str | None
    point: str | None           # RHEL point release, e.g. "9.6"
    iso: str | None             # RHEL DVD filename, e.g. "rhel-9.6-x86_64-dvd.iso"
    arch_pin: str | None        # "x86_64" for RHEL; None (host-tracking) for Ubuntu
    requires: tuple[str, ...]   # e.g. ("x86-64-v3",)
    ibm_unsupported_source: str | None  # citation URL when IBM does not support MQ here

@dataclass(frozen=True)
class BoxEntry:
    name: str           # "mq-nativeha-ubuntu24"
    role: str           # "mq-nativeha"
    os: OsEntry
    bake_stem: str      # "nativeha-ubuntu" -> ansible/bake-nativeha-ubuntu.yml
    mq_bearing: bool

@dataclass(frozen=True)
class BuildFile:
    os: OsRef | None

@dataclass(frozen=True)
class Catalog:
    oses: dict[OsRef, OsEntry]
    roles: dict[str, dict[str, Any]]
    infra: OsRef
    stacks: dict[str, dict[str, Any]]   # name -> {"supported": [OsRef], "default": OsRef}
    def box(self, role: str, ref: OsRef) -> BoxEntry: ...
    def stack_os(self, stack: str, build: BuildFile | None, facts: HostFacts) -> OsRef: ...
    def all_boxes(self, facts: HostFacts) -> list[BoxEntry]: ...   # every buildable (role, os) on this host

def load_catalog(path: Path | None = None) -> Catalog: ...
def load_build_file(path: Path) -> BuildFile: ...
```

- [ ] **Step 1: Write `lab/versions.yaml` at today's versions.** Phase 1 changes no
  behavior, so infra is `ubuntu:24` and every stack defaults to its current OS. Bake
  stems are today's playbook stems, except `san`, which Task T6 adds.

```yaml
# lab/versions.yaml — the ONLY place OS version tokens are written by hand (epic .github#280).
os:
  ubuntu:
    24: { base_box: cloud-image/ubuntu-24.04, base_box_version: "20260518.0.0" }
  rhel:
    9:  { base_box: rhel/9-x86_64, point: "9.6", iso: rhel-9.6-x86_64-dvd.iso, arch: x86_64 }
roles:
  infra:       { bake: { ubuntu: infra } }
  obs:         { bake: { ubuntu: obs } }
  mq-client:   { bake: { ubuntu: mq-ubuntu }, mq_bearing: true }
  mq-nativeha: { bake: { ubuntu: nativeha-ubuntu, rhel: nativeha-rhel }, mq_bearing: true }
  pcmk:        { bake: { ubuntu: pcmk-ubuntu } }
  mq-rdqm:     { bake: { rhel: mq-rdqm }, mq_bearing: true }
infra: ubuntu:24
stacks:
  nativeha-ubuntu:   { supported: [ubuntu:24], default: ubuntu:24 }
  pcmk-ubuntu:       { supported: [ubuntu:24], default: ubuntu:24 }
  nativeha-rhel-crr: { supported: [rhel:9],    default: rhel:9 }
  rdqm-rhel:         { supported: [rhel:9],    default: rhel:9 }
```

(`pcmk` is `mq_bearing: false` deliberately, matching today's `_manifest-hash.sh`.)

- [ ] **Step 2: Write failing tests** in `tests/test_versions.py`:

```python
import pytest
from mqlab.hostfacts import HostFacts
from mqlab.versions import BuildFile, OsRef, VersionError, load_build_file, load_catalog

X86 = HostFacts(arch="x86_64", kvm=True, in_vergil=True)   # match HostFacts' real fields
ARM = HostFacts(arch="aarch64", kvm=True, in_vergil=True)

def test_osref_parse_and_token():
    ref = OsRef.parse("rhel:9")
    assert (ref.family, ref.major, ref.token, str(ref)) == ("rhel", 9, "rhel9", "rhel:9")

@pytest.mark.parametrize("bad", ["rhel", "rhel:", "rhel:nine", "fedora:40", ":9", "rhel:9:1"])
def test_osref_parse_rejects(bad):
    with pytest.raises(VersionError, match="expected <family>:<major>"):
        OsRef.parse(bad)

def test_box_name_is_role_plus_token():
    cat = load_catalog()
    assert cat.box("mq-nativeha", OsRef("ubuntu", 24)).name == "mq-nativeha-ubuntu24"
    assert cat.box("mq-rdqm", OsRef("rhel", 9)).bake_stem == "mq-rdqm"

def test_box_unknown_family_for_role():
    with pytest.raises(VersionError, match="role 'mq-rdqm' has no bake for ubuntu"):
        load_catalog().box("mq-rdqm", OsRef("ubuntu", 24))

def test_stack_default_when_no_build_file():
    assert load_catalog().stack_os("rdqm-rhel", None, X86) == OsRef("rhel", 9)

def test_stack_rejects_unsupported_version(tmp_path):
    cat = load_catalog()
    with pytest.raises(VersionError, match=r"rdqm-rhel supports \[rhel:9\]; got rhel:10"):
        cat.stack_os("rdqm-rhel", BuildFile(os=OsRef("rhel", 10)), X86)

def test_stack_rejects_family_mismatch():
    with pytest.raises(VersionError, match="nativeha-ubuntu is an ubuntu stack; got rhel:9"):
        load_catalog().stack_os("nativeha-ubuntu", BuildFile(os=OsRef("rhel", 9)), X86)

def test_rhel_refused_on_aarch64():
    with pytest.raises(VersionError, match="RHEL needs an x86_64 host"):
        load_catalog().stack_os("rdqm-rhel", None, ARM)

def test_build_file_unknown_key(tmp_path):
    f = tmp_path / "b.yaml"; f.write_text("os: rhel:9\nmq: 9\n")
    with pytest.raises(VersionError, match="unknown key 'mq'"):
        load_build_file(f)

def test_build_file_not_a_mapping(tmp_path):
    f = tmp_path / "b.yaml"; f.write_text("- os\n")
    with pytest.raises(VersionError, match="must be a mapping"):
        load_build_file(f)

def test_default_must_not_be_ibm_unsupported(tmp_path):
    # Copy the catalog, mark ubuntu:24 unsupported, expect load to refuse the default.
    ...  # write the YAML with ibm_support: {status: unsupported, source: https://example}
    with pytest.raises(VersionError, match="default ubuntu:24 for nativeha-ubuntu is IBM-unsupported"):
        load_catalog(tmp_path / "versions.yaml")
```

  Use the real `HostFacts` constructor signature from `src/mqlab/hostfacts.py`.
  Write the unsupported-default fixture out in full: copy Step 1's YAML and add
  `ibm_support: { status: unsupported, source: "https://example.invalid" }` to the
  `ubuntu: 24` entry.

- [ ] **Step 3:** Run `uv run pytest tests/test_versions.py -v`. Expected: it fails
  with `ModuleNotFoundError: mqlab.versions`.
- [ ] **Step 4: Implement `src/mqlab/versions.py`.**
  - `load_catalog` validates the whole file eagerly. It checks unknown top-level keys,
    every stack default being in its `supported` list, and the support gate.
  - `stack_os` applies, in order: the family check, the supported check, then the host
    gates (RHEL needs x86_64; `requires: x86-64-v3` is checked in T10, so Phase 1 has
    no such entry).
  - `all_boxes(facts)` returns the infra box for each infra role (`infra`, `obs`,
    `mq-client`, plus `san` from T6). For each stack's roles it returns every
    `supported` OS, de-duplicated and skipping RHEL on aarch64. Stack roles come from
    `topology.yaml` nodes in T2; in T1, pass `stack_roles: dict[str, set[str]]` as a
    parameter and give `all_boxes(facts, stack_roles)` that signature.
  - Every error message names the fix, e.g. `"… — edit your --config file or omit it
    to use the default (see lab/versions.yaml)"`.
- [ ] **Step 5:** Run the tests and the coverage command. Expected: they pass, with
  100% branch coverage on `versions.py`.
- [ ] **Step 6:** Commit:
  `vrg-commit --type feat --scope versions --message "OS version catalog + resolver (pure)"`.

### Task T4: Builders take catalog inputs; box renames; fleet from catalog

**Files:**

- Modify: `lab/boxes/build-fatbox.sh`. Delete the `case "$BOX"` table at `:76-87`.
  Add the required flags `--base-kind`, `--base-box`, `--base-box-version`, `--bake`,
  `--dvd` and `--os-pin`, and pass them through to `_manifest-hash.sh`.
- Modify: `lab/boxes/_manifest-hash.sh`. Delete the `case "$BOX"` at `:67-75`. Take
  `<box> --bake-stem S --mq-bearing 0|1 --os-pin P`, and digest `os_pin=P`.
- Move: `lab/boxes/rhel96/` → `lab/boxes/rhel/`. `build-box.sh` takes
  `--major N --point X.Y --iso <file>`. Keep `ks.cfg`, and add a `ks-<major>.cfg`
  only in T10 if RHEL 10 differs.
- Modify: `lab/scripts/stage-rhel-iso.sh` and `scripts/push-rhel-iso.sh` to take
  `--iso <file>` (no 9.6 literal).
- Modify: `src/mqlab/box.py`.
  - `_build_fleet` derives from `versions.load_catalog().all_boxes(...)` plus one RHEL
    base box per catalog RHEL major.
  - Delete `_BASE_BOX`, `_BASE_ARTIFACT` and `_RHEL_VERSION`.
  - `build_boxes` invokes the builders with flags taken from each `BoxEntry`.
  - `verify_rhel_dvd(version)` takes the `point` from the `OsEntry`.
  - Add `src/mqlab/retired_boxes.py` (`RETIRED_BOX_NAMES`) and teach `clean`/`gc` to remove them.
- Modify: `src/mqlab/cli.py`.
  - Delete `_LOCAL_BOX_BUILDERS` (`:1116-1125`); `box.FLEET` no longer derives from it.
  - `mqlab box build` gains `--config <file>`, which builds that build file's boxes
    for every stack.
- Modify: `lab/topology.yaml`. **Mechanical interim rename:** the `boxes:` registry
  keys and node `platform:` values become the new names (`mq-client-ubuntu24`,
  `infra-ubuntu24`, `obs-ubuntu24`, `mq-nativeha-ubuntu24`, `pcmk-ubuntu24`,
  `mq-rdqm-rhel9`, `mq-nativeha-rhel9`; base `rhel/9-x86_64`; platform keys
  `ubuntu24-arm64` and `ubuntu24-x86_64`). The registry itself is removed in T2.
- Modify: `src/mqlab/manifest.py` (`_OS_PREFIX` keys follow the rename) and
  `src/mqlab/platforms.py:60-62` (`default_platform` returns `ubuntu24-*`).
- Modify: `src/mqlab/cli.py` `_reconcile_box_meta`. It treats any `box_meta` naming a
  `RETIRED_BOX_NAMES` entry as repointed.
- Modify: `docs/development/box-model.md` and `docs/development/box-bake-manifest.md`
  (naming rule, flags, retired names).
- Test: `tests/test_box_build.py`, `tests/test_cli_box.py`,
  `tests/test_build_fatbox_usage.py`, `tests/test_build_box_usage.py`,
  `tests/test_box_dvd.py`, `tests/test_manifest.py` and `tests/test_platforms.py`.
  Old box names become the new names, and new tests are added below.

**Interfaces:**

- Consumes: `versions.Catalog.all_boxes`, `BoxEntry`, `OsEntry` (T1).
- Produces:
  - `box.FLEET: dict[str, BoxSpec]`, keyed by generated box name.
  - `mqlab.retired_boxes.RETIRED_BOX_NAMES: tuple[str, ...]`, in its own module so
    T7's guardrail can exempt the file as a whole. It equals
    `("obs-ubuntu2404", "infra-ubuntu2404", "mq-ubuntu2404", "mq-nativeha-ubuntu", "pcmk-ubuntu", "rhel/9.6-x86_64")`.
    `box.py` and `cli.py` import it.
  - `build-fatbox.sh` and `_manifest-hash.sh` flag interfaces as listed above.

- [ ] **Step 1: Write failing tests.**

```python
def test_fleet_derives_from_catalog(x86_facts):
    names = set(box._build_fleet(facts=x86_facts))
    assert {"mq-nativeha-ubuntu24", "mq-client-ubuntu24", "infra-ubuntu24", "obs-ubuntu24",
            "pcmk-ubuntu24", "mq-rdqm-rhel9", "mq-nativeha-rhel9", "rhel/9-x86_64"} <= names
    assert not names & set(RETIRED_BOX_NAMES)

def test_build_passes_catalog_flags(fake_runner, x86_facts):
    box.build_boxes(["mq-nativeha-ubuntu24"], force=False)
    argv = fake_runner.last_argv()
    assert argv[argv.index("--base-box") + 1] == "cloud-image/ubuntu-24.04"
    assert argv[argv.index("--bake") + 1] == "nativeha-ubuntu"
    assert argv[argv.index("--os-pin") + 1] == "cloud-image/ubuntu-24.04@20260518.0.0"

def test_manifest_hash_flips_on_os_pin(tmp_path):
    a = run_hash("mq-nativeha-ubuntu24", "--bake-stem", "nativeha-ubuntu", "--mq-bearing", "1", "--os-pin", "x@1")
    b = run_hash("mq-nativeha-ubuntu24", "--bake-stem", "nativeha-ubuntu", "--mq-bearing", "1", "--os-pin", "x@2")
    assert a != b

def test_build_fatbox_requires_bake_flag():
    r = run_fatbox("--box", "x", "--arch", "x86_64", "--domain-type", "kvm", "--cpu-mode", "host-passthrough", "--dry-run")
    assert r.returncode == 2 and "--bake is required" in r.stderr

def test_reconcile_forgets_retired_box_names(fake_box_meta):     # Review Focus 4
    fake_box_meta({"nha-ubuntu-a1": "mq-nativeha-ubuntu"})
    forgotten = cli._stale_box_meta_guests(["nha-ubuntu-a1"])
    assert forgotten == ["nha-ubuntu-a1"]

def test_gc_removes_retired_names(fake_vagrant_boxes):
    fake_vagrant_boxes(["mq-nativeha-ubuntu", "mq-nativeha-ubuntu24"])
    assert box.clean_retired() == ["mq-nativeha-ubuntu"]
```

  `run_hash`/`run_fatbox` are the existing subprocess helpers in
  `tests/test_build_fatbox_usage.py`. Reuse them. `_stale_box_meta_guests` is the pure
  helper `_reconcile_box_meta` already uses, or extract it if it's inline.

- [ ] **Step 2:** `uv run pytest tests/test_box_build.py tests/test_cli_box.py tests/test_build_fatbox_usage.py -v`.
  Expected: these fail.
- [ ] **Step 3:** Implement the shell flag changes. Each missing required flag exits 2
  with `ERROR: --<flag> is required`. `--dvd` and `--base-box-version` may be the
  literal `none`.
- [ ] **Step 4:** Implement the `box.py`/`cli.py`/`manifest.py`/`platforms.py`
  changes, the topology rename and the `lab/boxes/rhel/` move. Add a `mqlab box gc`
  path, `clean_retired()`, which `vagrant box remove`s retired names. The existing
  `gc_orphaned_images` in-use guard still protects any volume backing a live overlay.
- [ ] **Step 5:** Run the full suite plus coverage. Expected: it passes, at 100%.
- [ ] **Step 6:** Update the two dev docs, then commit:
  `vrg-commit --type refactor --scope box --message "builders take catalog inputs; <role>-<os><major> box names"`.

### Task T2: Topology names roles; render goes through the version layer

**Files:**

- Modify: `lab/topology.yaml`.
  - Delete the `boxes:` registry.
  - Each node `platform: X` becomes `box: <role>`. The `san-a`/`san-b` nodes get no
    role until T6, so give them `box: base` here. That's a pseudo-role resolving to
    the infra OS's bare `base_box`.
  - Each stack's `os:` becomes `os_family:`.
- Modify: `src/mqlab/versions.py`. Add the following, which reads records through
  `mqlab.instances` (T3). In T2, `instances.read_record` returns `None` for every
  stack until T3 lands it; stub it in `instances.py` with the T3 signature.

```python
def node_boxes(topo: dict[str, Any], catalog: Catalog, facts: HostFacts) -> dict[str, BoxEntry]:
    """node -> BoxEntry: stack nodes from their stack's record (else stack default),
    shared nodes and SANs from catalog.infra."""
```

- Modify: `src/mqlab/platforms.py`. `resolve(topo, facts, boxes: dict[str, BoxEntry])`
  takes the selection instead of reading `platform:`. `ResolvedNode` gets
  `box_version: str | None` (from `BoxEntry.os.base_box_version` only for `base`, else
  `None`), and `dvd` (path under the libvirt pool from `OsEntry.iso`).
  `ensure_resolved` computes `node_boxes(...)` and passes it in. Delete
  `default_platform`. The `platform` field becomes `os: str` (e.g. `"ubuntu:24"`).
- Modify: `lab/Vagrantfile`. Delete the `box-versions.json` block (`:18-21`, `:35-36`).
  Use `node.vm.box_version = n["box_version"] if n["box_version"]`.
- Modify: `src/mqlab/paths.py` (delete `box_versions_path`) and `src/mqlab/buildenv.py:210`.
- Modify: `src/mqlab/fleet.py`. `lab_guests()` returns guest → box name via
  `node_boxes`.
- Modify: `src/mqlab/phases.py:321-349`. `_guest_box` uses `node_boxes`, and the
  `_HOST_RESOLVED_BASE_BOX` sentinel is kept for role `base`.
- Modify: `src/mqlab/manifest.py`. Replace `_OS_PREFIX` with a family map
  `{"ubuntu": "UbuntuLinux", "rhel": "Linux"}`.
  `tarball_name(mq_version, entry: BoxEntry, facts)`.
- Modify: `src/mqlab/cli.py`. `_commons_mq_platforms` and `_stack_mq_platforms`
  return `set[BoxEntry]` (rename them to `_commons_mq_boxes` and `_stack_mq_boxes`).
  Update `artifact.ensure_mq_tarballs_for_platforms` to
  `ensure_mq_tarballs_for_boxes`.
- Modify: `src/mqlab/stacks.py`. `rhel_stack_unsupported_reason` reads `os_family`
  (it stays as the pre-bake gate), and dashboard folder labels read `os_family`.
- Modify: `src/mqlab/clusterboard.py:1446-1450`. The OS label comes from the
  resolved node's `os` (`"Ubuntu 24"`, `"RHEL 9"`), with no literal.
- Test: `tests/test_platforms.py`, `tests/test_fleet.py`, `tests/test_manifest.py`,
  `tests/test_cli_bootstrap.py`, `tests/test_cli_vm.py`, `tests/test_cli_commons.py`,
  `tests/test_clusterboard.py` and `tests/test_versions.py`.

**Interfaces:**

- Consumes: T1's `Catalog`, `BoxEntry` and `OsRef`; T4's renamed boxes.
- Produces:
  - `versions.node_boxes(topo, catalog, facts) -> dict[str, BoxEntry]`.
  - `platforms.resolve(topo, facts, boxes)`.
  - `ResolvedNode.os: str` and `ResolvedNode.box_version: str | None`.
  - `manifest.tarball_name(mq_version, entry, facts)`.

- [ ] **Step 1: Write failing tests.**

```python
def test_node_boxes_defaults(x86_facts):
    nb = versions.node_boxes(topology.load(), versions.load_catalog(), x86_facts)
    assert nb["nha-ubuntu-a1"].name == "mq-nativeha-ubuntu24"
    assert nb["rdqm-a1"].name == "mq-rdqm-rhel9"
    assert nb["infra-svc"].name == "infra-ubuntu24"

def test_commons_render_uses_infra_without_records(x86_facts, no_records):   # Review Focus 5
    nb = versions.node_boxes(topology.load(), versions.load_catalog(), x86_facts)
    assert {nb[n].os.ref for n in ("obs", "svc-sim", "app-client", "mon-probe", "infra-client")} \
        == {versions.load_catalog().infra}

def test_resolved_topology_has_no_platform_key(tmp_build, x86_facts):
    text = platforms.ensure_resolved(facts=x86_facts).read_text()
    assert "platform:" not in text and "box: mq-nativeha-ubuntu24" in text

def test_tarball_name_from_family(x86_facts):
    e = versions.load_catalog().box("mq-rdqm", versions.OsRef("rhel", 9))
    assert manifest.tarball_name("10.0.0.0", e, x86_facts).endswith("-LinuxX64.tar.gz")

def test_topology_has_no_boxes_registry():
    assert "boxes" not in yaml.safe_load((repo_root() / "lab/topology.yaml").read_text())
```

- [ ] **Step 2:** Run them. Expected: they fail.
- [ ] **Step 3:** Implement as listed under Files. Keep `ResolvedNode`'s other provider
  mechanics byte-for-byte identical. Compare a before/after render of
  `topology.resolved.yaml` on x86 facts: the only diffs allowed are the
  `platform`→`os` key and `box_version`.
- [ ] **Step 4:** Full suite plus coverage. Expected: it passes, at 100%.
- [ ] **Step 5:** Commit:
  `vrg-commit --type refactor --scope topology --message "topology names box roles; render via the version layer"`.

### Task T3: `--config` build files, instance records, refusal rules

**Files:**

- Create/complete: `src/mqlab/instances.py`.
- Modify: `src/mqlab/paths.py`. Add `instances_dir() -> Path` (`state("instances")`).
- Modify: `src/mqlab/cli.py`.
  - `bootstrap` gains `--config PATH`.
  - `_bootstrap_run` resolves and checks the record **before** `_prepare_lab()`
    renders.
  - `_teardown_run` deletes the record after every destroy succeeds.
  - `status` prints the stack's OS.
  - Every stack-scoped command except teardown calls `instances.require_record_if_live`.
    That covers `bootstrap --from/--only`, `status <stack>`, `qm-*`, `dr-*` and
    `diagnostics`.
- Modify: `src/mqlab/versions.py`. `node_boxes` reads real records.
- Test: `tests/test_instances.py`, `tests/test_cli_bootstrap.py`,
  `tests/test_cli_teardown.py` and `tests/test_cli_status.py`.

**Interfaces:**

- Consumes: T1's `BuildFile`, `OsRef`, `Catalog.stack_os`, `load_build_file`; T2's
  `node_boxes`.
- Produces:

```python
@dataclass(frozen=True)
class InstanceRecord:
    stack: str
    os: OsRef
    build_file: str | None      # the --config path as given, or None
    created: str                # ISO-8601 UTC

def read_record(stack: str) -> InstanceRecord | None: ...
def write_record(rec: InstanceRecord) -> None: ...
def delete_record(stack: str) -> None: ...          # missing record is not an error
def reconcile(stack: str, requested: OsRef, *, live: bool, build_file: str | None) -> InstanceRecord:
    """No record + not live -> write and return a new record.
    No record + live        -> VersionError(run `mqlab teardown <stack>` and re-bootstrap).
    Record == requested     -> return it.
    Record != requested     -> VersionError(stack is running <rec.os>; requested <requested>;
                                run `mqlab teardown <stack>` first)."""
def require_record_if_live(stack: str, *, live: bool) -> InstanceRecord | None: ...
```

  A record **without** `--config` on a resume is reused as-is: `requested` defaults to
  the record's OS when one exists, otherwise to the stack default. Records are JSON,
  written atomically (temp file plus `os.replace`).

- [ ] **Step 1: Write failing tests.**

```python
def test_first_bootstrap_writes_record(tmp_state):
    rec = instances.reconcile("nativeha-ubuntu", OsRef("ubuntu", 24), live=False, build_file=None)
    assert instances.read_record("nativeha-ubuntu") == rec

def test_config_matching_record_accepted(tmp_state):           # Review Focus 2
    instances.reconcile("s", OsRef("ubuntu", 24), live=False, build_file="a.yaml")
    assert instances.reconcile("s", OsRef("ubuntu", 24), live=True, build_file="a.yaml").os == OsRef("ubuntu", 24)

def test_config_conflicting_record_refused(tmp_state):         # Review Focus 2
    instances.reconcile("s", OsRef("ubuntu", 24), live=False, build_file=None)
    with pytest.raises(VersionError, match=r"running ubuntu:24; requested ubuntu:26; run `mqlab teardown s` first"):
        instances.reconcile("s", OsRef("ubuntu", 26), live=True, build_file="b.yaml")

def test_live_without_record_refused(tmp_state):
    with pytest.raises(VersionError, match=r"running without a version record; run `mqlab teardown s`"):
        instances.require_record_if_live("s", live=True)

def test_teardown_deletes_record(cli_harness):
    cli_harness.bootstrap("nativeha-ubuntu"); cli_harness.teardown("nativeha-ubuntu")
    assert instances.read_record("nativeha-ubuntu") is None

def test_teardown_works_without_record(cli_harness, live_domains):
    cli_harness.teardown("nativeha-ubuntu")       # no VersionError

def test_record_pins_through_default_change(tmp_state, catalog_with_default):   # Review Focus 3
    instances.reconcile("nativeha-ubuntu", OsRef("ubuntu", 24), live=False, build_file=None)
    nb = versions.node_boxes(topology.load(), catalog_with_default("nativeha-ubuntu", "ubuntu:26"), x86)
    assert nb["nha-ubuntu-a1"].name == "mq-nativeha-ubuntu24"

def test_render_combines_records_across_stacks(tmp_state, catalog_24_26):      # Review Focus 1
    instances.write_record(InstanceRecord("nativeha-ubuntu", OsRef("ubuntu", 24), None, "t"))
    instances.write_record(InstanceRecord("pcmk-ubuntu", OsRef("ubuntu", 26), None, "t"))
    nb = versions.node_boxes(topology.load(), catalog_24_26, x86)
    assert (nb["nha-ubuntu-a1"].name, nb["pcmk-a1"].name) == ("mq-nativeha-ubuntu24", "pcmk-ubuntu26")

def test_bootstrap_config_refused_before_any_lab_io(cli_harness, fake_runner):
    (cli_harness.tmp / "b.yaml").write_text("os: rhel:10\n")
    result = cli_harness.invoke(["bootstrap", "rdqm-rhel", "--config", str(cli_harness.tmp / "b.yaml")])
    assert result.exit_code == 2 and fake_runner.calls == []
```

  `catalog_24_26` and `catalog_with_default` are fixtures that write a temporary
  `versions.yaml` with `ubuntu: {24: …, 26: …}`. Point `load_catalog` at it via
  monkeypatching `paths.versions_catalog_path`.

- [ ] **Step 2:** Run them. Expected: they fail.
- [ ] **Step 3:** Implement `instances.py`, then the CLI wiring. A `VersionError` is
  caught at the CLI boundary, printed to stderr, and exits 2 **before** any `virsh` or
  `vagrant` call. `live` comes from the existing `_probe_states` / `is_live` over
  `stack_members(stack)`.
- [ ] **Step 4:** Full suite plus coverage. Expected: it passes, at 100%.
- [ ] **Step 5:** Commit:
  `vrg-commit --type feat --scope bootstrap --message "--config build files + per-stack instance records"`.

### Task T5: Ansible per-version vars indirection

**Files:**

- Create: `ansible/tasks/os-vars.yml`, the shared include:

```yaml
# Load <role>/vars/<Distribution>-<major>.yml; FAIL (no fallback) when absent (epic .github#280).
- name: "load per-OS-version vars for {{ os_vars_role }}"
  ansible.builtin.include_vars:
    file: "{{ lookup('ansible.builtin.first_found', params) }}"
  vars:
    params:
      files: ["{{ ansible_distribution }}-{{ ansible_distribution_major_version }}.yml"]
      paths: ["{{ playbook_dir }}/roles/{{ os_vars_role }}/vars"]
```

  `first_found` raises when nothing matches, and that is the fail-loud behavior we
  want. Do **not** add `skip: true` or an `errors: ignore`.

- Create: `ansible/roles/rdqm-install/vars/RedHat-9.yml`. It defines `rhel_el_tag: el9`,
  `rdqm_dvd_repo_name: "RHEL {{ rhel_point }} DVD"`, `drbd_kmod_dir: el9/kmod-drbd-9`,
  `pacemaker_dir: el9/pacemaker-2` and `drbd_utils_dir: el9/drbd-utils-9`.
- Modify: `ansible/roles/rdqm-install/tasks/install.yml:31-36,65-89`. Include
  `os-vars.yml` with `os_vars_role: rdqm-install` and replace the literals with those
  vars. The shell `${KREL%%.el9*}` becomes `${KREL%%.{{ rhel_el_tag }}*}`.
- Create: `ansible/roles/mq-nativeha/vars/RedHat-9.yml` and
  `ansible/roles/mq-nativeha-spike/vars/RedHat-9.yml` (the DVD repo name).
- Modify: `ansible/roles/mq-nativeha/tasks/install-RedHat.yml:33-38` and
  `ansible/roles/mq-nativeha-spike/tasks/main.yml:30-35`.
- Modify: `ansible/group_vars/all/versions.yml`. Add `rhel_point`, passed in by mqlab
  through extra-vars from `OsEntry.point` for RHEL nodes. Or derive it from
  `ansible_distribution_version` when it isn't passed, because the guest already
  knows its point release. Prefer the fact; drop the extra-var.
- Modify: the noble-assuming *comments* in `pcmk-stonith/tasks/install-Debian.yml:1`,
  `motd-off/defaults/main.yml:3-4`, `snapd-off/defaults/main.yml:11-12`,
  `grafana-image-renderer/defaults/main.yml:14`, `data-prepper/defaults/main.yml:94`,
  `bake-nativeha-ubuntu.yml:5`, `bake-pcmk-ubuntu.yml:5` and the `host-resolver`
  comments. Reword them version-neutrally ("Ubuntu LTS ships …"). Any genuinely
  version-specific *value* moves to a `vars/Ubuntu-24.yml` in that role, loaded the
  same way.
- Test: `tests/test_ansible_os_vars.py`.

**Interfaces:**

- Produces: the `os-vars.yml` include contract. Callers set `os_vars_role`, and the
  role must ship a `vars/<Distribution>-<major>.yml` for every OS in the catalog that
  runs it. T8 and T10 add the 26 and 10 files.

- [ ] **Step 1: Write failing tests.** They are static, following the existing
  `tests/test_*_baked.py` pattern:

```python
ROLES_WITH_OS_VARS = {"rdqm-install": ["RedHat-9"], "mq-nativeha": ["RedHat-9"], "mq-nativeha-spike": ["RedHat-9"]}

@pytest.mark.parametrize("role,files", ROLES_WITH_OS_VARS.items())
def test_role_ships_vars_for_each_catalog_os(role, files):
    for f in files:
        assert (ANSIBLE / "roles" / role / "vars" / f"{f}.yml").is_file()

def test_os_vars_include_has_no_fallback():
    text = (ANSIBLE / "tasks/os-vars.yml").read_text()
    assert "first_found" in text and "skip:" not in text and "ignore" not in text

def test_rdqm_install_has_no_el9_literal():
    assert "el9" not in (ANSIBLE / "roles/rdqm-install/tasks/install.yml").read_text()
```

  Also derive `ROLES_WITH_OS_VARS`'s expected files from `lab/versions.yaml`. For each
  RHEL major in the catalog, each listed role must ship `RedHat-<major>.yml`. That way
  T10 adding `rhel: 10` makes this test fail until `RedHat-10.yml` exists.

- [ ] **Step 2:** Run them. Expected: they fail. Implement. Then run
  `vrg-container-run -- vrg-validate` (it includes ansible-lint).
- [ ] **Step 3:** Commit:
  `vrg-commit --type refactor --scope ansible --message "per-OS-version vars indirection (fail-loud include)"`.

### Task T6: Baked SAN box; retire the SAN deb cache (supersedes `.github#108`)

**Files:**

- Create: `ansible/bake-san.yml`. It runs the **install half** of `drbd-san` and
  `iscsi-target` (packages: `drbd-utils`, `targetcli-fb`,
  `linux-modules-extra-{{ ansible_kernel }}`), plus `entropy`, `node-exporter`,
  `alloy` install-half, `snapd-off`, `motd-off`, `cloud-init-trim` and
  `apt-autoupdate-off`. Copy `ansible/bake-infra.yml` for the common baked set.
- Modify: `lab/versions.yaml`. Add `san: { bake: { ubuntu: san } }` under `roles:`.
- Modify: `src/mqlab/versions.py`. Add `san` to the infra roles in `all_boxes`, and
  have `node_boxes` resolve role `san` from infra. Delete the `base` pseudo-role from
  T2 if nothing else uses it (nothing else should).
- Modify: `lab/topology.yaml`. `san-a`/`san-b` get `box: san`. Update the stale
  "SAN targets are NOT repointed … (#108)" comments.
- Modify: `ansible/roles/drbd-san/tasks/main.yml:20-60` and
  `ansible/roles/iscsi-target/tasks/install-Debian.yml:5-45`. Delete the
  `/var/cache/san-debs` copy branches and the network fallback. Replace them with a
  skip-if-baked guard (`ansible.builtin.package_facts` and
  `when: "'drbd-utils' not in ansible_facts.packages"`), followed by a plain `apt`
  install so the roles keep working at bake time.
- Delete: `src/mqlab/sandeb.py`, `tests/test_sandeb.py` and
  `docs/development/san-deb-cache.md`.
- Modify: `src/mqlab/cli.py`. Delete `_ensure_san_debs_for_stack` and the `"san"`
  prereq kind (`:281-298`, `:363-364`).
- Modify: `src/mqlab/phases.py`. Remove `"san"` from every `Phase.ensure`, and drop
  the `_HOST_RESOLVED_BASE_BOX` sentinel if no `base` role remains.
- Modify: `src/mqlab/buildenv.py` and `src/mqlab/paths.py`. Delete
  `san_deb_cache_dir`.
- Test: `tests/test_box_build.py` (fleet contains `san-ubuntu24`),
  `tests/test_cli_bootstrap.py` (no san prereq), and a new `tests/test_san_baked.py`.

**Interfaces:**

- Consumes: T1, T2, T4.
- Produces: role `san` → box `san-<infra token>`, e.g. `san-ubuntu24` in Phase 1 and
  `san-ubuntu26` after T8.

- [ ] **Step 1: Write failing tests.**

```python
def test_san_nodes_resolve_to_baked_san_box(x86_facts):
    nb = versions.node_boxes(topology.load(), versions.load_catalog(), x86_facts)
    assert nb["san-a"].name == nb["san-b"].name == f"san-{versions.load_catalog().infra.token}"

def test_no_san_deb_cache_remains():
    assert not (SRC / "mqlab/sandeb.py").exists()
    for role in ("drbd-san", "iscsi-target"):
        assert "san-debs" not in "".join(p.read_text() for p in (ANSIBLE / "roles" / role).rglob("*.yml"))

def test_bootstrap_declares_no_san_prereq():
    assert all("san" not in p.ensure for p in phases.PHASES)
```

- [ ] **Step 2:** Run them. Expected: they fail. Implement. Then run the full suite,
  coverage and `vrg-container-run -- vrg-validate`.
- [ ] **Step 3:** Commit:
  `vrg-commit --type feat --scope box --message "bake the SAN targets (san box); retire the SAN deb cache"`.
  The body must state that it supersedes `.github#108`, with the reason from spec
  §4.7.1.

### Task T7: Version-token guardrail

**Files:**

- Create: `tests/test_version_token_guardrail.py`. Model it on the logsearch-ref
  guardrail added in f0b86c2, including its `__pycache__`/`.pyc` exclusion.

**Interfaces:**

- Consumes: the end state of T2, T3, T5 and T6. It must pass on `develop` the day it
  lands.

- [ ] **Step 1: Write the guardrail.**

```python
# OS-version tokens only: deliberately NOT bare "9.6"/"2404", which collide with
# unrelated software versions and ports.
TOKENS = re.compile(
    r"\b(ubuntu-?2[46](\.?04)?|noble|resolute|rhel-?(9|10)(\.\d+)?|rhel/(9|10)|el(9|10))\b"
)
SCAN = ["src", "lab", "ansible", "scripts"]
EXEMPT = {
    Path("lab/versions.yaml"),            # the catalog
    Path("src/mqlab/retired_boxes.py"),   # the retired-names table (spec §4.9)
}
EXEMPT_GLOBS = ["lab/boxes/rhel/ks*.cfg", "ansible/**/vars/*-[0-9]*.yml"]

def test_no_os_version_tokens_outside_catalog():
    hits = []
    for root in SCAN:
        for p in (REPO / root).rglob("*"):
            if not p.is_file() or "__pycache__" in p.parts or p.suffix == ".pyc":
                continue
            rel = p.relative_to(REPO)
            if rel in EXEMPT or any(rel.match(g) for g in EXEMPT_GLOBS):
                continue
            for n, line in enumerate(p.read_text(errors="ignore").splitlines(), 1):
                if TOKENS.search(line):
                    hits.append(f"{rel}:{n}: {line.strip()}")
    assert not hits, "OS version tokens outside lab/versions.yaml (epic .github#280):\n" + "\n".join(hits)
```

- [ ] **Step 2:** Run `uv run pytest tests/test_version_token_guardrail.py -v`. Fix
  every remaining hit at its source. Never widen `EXEMPT` to silence a real hit.
- [ ] **Step 3:** Run `vrg-container-run -- vrg-validate`, then commit:
  `vrg-commit --type test --scope guardrail --message "version-token guardrail (scoped, spec §6)"`.

### Operational D1 (deployment): Bake and cache the Phase-1 boxes

Created with `--kind deployment`, `--blocked-by T7`.

- **Precondition:** `develop` contains T1–T7.
- **Procedure:** On each host (the cloud x86 host and the macOS arm64 host), run
  `mqlab box build --all`.
- **Acceptance:** `mqlab box status` shows every fleet box at REUSE.
  `mqlab box gc` has removed the retired names, and `vagrant box list` shows none of
  them.

### Operational V1 (validation): Phase-1 regression cold rebuild

Created with `--kind validation`, `--blocked-by D1`.

- **Rows (one cold rebuild each, full DR, existing end-to-end check, human dashboard
  review):**
  - `nativeha-ubuntu` @ ubuntu24
  - `pcmk-ubuntu` @ ubuntu24 (this proves the **baked SAN box**)
  - `nativeha-rhel-crr` @ rhel9 (cloud x86)
  - `rdqm-rhel` @ rhel9 (cloud x86)
- **Also check:**
  - `build/state/instances/<stack>.json` exists after bootstrap and is gone after
    teardown.
  - A second `bootstrap --config` naming a different OS is refused.
- **Acceptance:** all four rows green; `Outcome: SUCCESS`.

---

## Phase 2: Ubuntu 26

### Task T8: Ubuntu 26 catalog entry, role fix-ups, infra → 26

**Files:**

- Modify: `lab/versions.yaml`.
  - Add `ubuntu: 26: { base_box: cloud-image/ubuntu-26.04, base_box_version: "<T0b pin>" }`,
    filling in the version string T0b recorded.
  - If T0a cites IBM as not supporting MQ 10 on 26.04, add
    `ibm_support: { status: unsupported, source: <T0a URL> }`.
  - Set `infra: ubuntu:26`.
  - Set `nativeha-ubuntu` and `pcmk-ubuntu` to `supported: [ubuntu:24, ubuntu:26]`,
    and **keep `default: ubuntu:24`** (T9 flips it).
- Create: `ansible/roles/<role>/vars/Ubuntu-26.yml` for every role that T5 gave an
  `Ubuntu-24.yml`, with values from T0a/T0b (package-name changes, if any).
- Modify: the bake playbooks and roles only where T0a/T0b found a 26.04 package or
  behavior difference. Each change gets a comment citing the T0b report section.
- Modify: `src/mqlab/versions.py`. When a selection lands on an
  `ibm_unsupported_source` entry, `stack_os` prints
  `WARNING: <os> is not IBM-supported for MQ (source: <url>) — lab-only` to stderr.
- Test: `tests/test_versions.py`.

**Interfaces:**

- Consumes: T0a and T0b findings; the T5 vars contract.

- [ ] **Step 1: Write failing tests.**

```python
def test_infra_is_ubuntu26():
    assert versions.load_catalog().infra == OsRef("ubuntu", 26)

def test_ubuntu_stacks_accept_26():
    cat = versions.load_catalog()
    assert cat.stack_os("pcmk-ubuntu", BuildFile(os=OsRef("ubuntu", 26)), X86) == OsRef("ubuntu", 26)

def test_unsupported_selection_warns(capsys, catalog_26_unsupported):
    catalog_26_unsupported.stack_os("pcmk-ubuntu", BuildFile(os=OsRef("ubuntu", 26)), X86)
    assert "not IBM-supported" in capsys.readouterr().err
```

  T5's catalog-derived vars test now requires the `Ubuntu-26.yml` files.

- [ ] **Step 2:** Run them. Expected: they fail. Implement. Then run the full suite,
  coverage and `vrg-container-run -- vrg-validate` (the guardrail must stay green).
- [ ] **Step 3:** Commit:
  `vrg-commit --type feat --scope versions --message "Ubuntu 26: catalog entry, role fix-ups, infra on 26"`.

### Operational D2 (deployment): Bake the Ubuntu 26 boxes

Created with `--kind deployment`, `--blocked-by T8`.

- **Procedure:** On each host, run `mqlab box build --all`. This bakes `infra`,
  `obs`, `mq-client` and `san` at ubuntu26, plus `mq-nativeha-ubuntu26` and
  `pcmk-ubuntu26`.
- **Acceptance:** `mqlab box status` shows them all at REUSE.

### Operational V2 (validation): Ubuntu 26 rows

Created with `--kind validation`, `--blocked-by D2`.

- **Rows:**
  - `nativeha-ubuntu` with `--config` set to `os: ubuntu:26`
  - `pcmk-ubuntu` with `--config` set to `os: ubuntu:26`
  - `nativeha-ubuntu` @ ubuntu24 (the default), now running on **ubuntu26 shared
    nodes**: the mixed-version regression (spec §6)
  - `pcmk-ubuntu` @ ubuntu24 (the default) against the **ubuntu26 SANs and shared
    nodes**: cluster initiators on 24, DRBD-backed SAN targets on 26 (spec §6)
- **For each row:** full DR, the end-to-end check, and a human dashboard review that
  confirms the OS labels read correctly.
- **Acceptance:** all rows green; `Outcome: SUCCESS`.

### Task T9: Flip the Ubuntu stack defaults to 26

**Files:**

- Modify: `lab/versions.yaml` (`default: ubuntu:26` for `nativeha-ubuntu` and
  `pcmk-ubuntu`).
- Test: `tests/test_versions.py`.

- [ ] **Step 1:** Write a failing test:
  `assert load_catalog().stack_os("nativeha-ubuntu", None, X86) == OsRef("ubuntu", 26)`.
- [ ] **Step 2:** Flip the defaults. If T0a cited 26.04 as IBM-unsupported,
  `load_catalog` refuses this, and that is the gate working. In that case, close T9
  as **not done**, with a comment citing T0a and the support-gate rule (spec §4.1).
  Don't merge a broken catalog.
- [ ] **Step 3:** Run `vrg-container-run -- vrg-validate`, then commit:
  `vrg-commit --type feat --scope versions --message "Ubuntu stacks default to 26"`.

---

## Phase 3: RHEL 10

### Task T10: RHEL 10 catalog entry, base box, vars, x86-64-v3 gate

**Files:**

- Modify: `lab/versions.yaml`.
  - Add `rhel: 10: { base_box: rhel/10-x86_64, point: "<T0a pin>", iso: "rhel-<T0a pin>-x86_64-dvd.iso", arch: x86_64, requires: [x86-64-v3] }`,
    plus `ibm_support` if T0a says so.
  - Set `nativeha-rhel-crr` to `supported: [rhel:9, rhel:10]`, and **keep
    `default: rhel:9`** (T11 flips it).
- Create: `lab/boxes/rhel/ks-10.cfg`, only if a RHEL 10 kickstart differs from
  `ks.cfg` (the 10.x Anaconda changes). `build-box.sh --major 10` selects
  `ks-<major>.cfg` when present and `ks.cfg` otherwise. That is a deliberate choice of
  input, not a silent fallback, and it's documented in `build-box.sh`'s header.
- Create: `ansible/roles/mq-nativeha/vars/RedHat-10.yml`. Do not create one for
  `rdqm-install`: RDQM isn't supported on RHEL 10, and the catalog never resolves
  `mq-rdqm` on rhel10.
- Modify: `ansible/roles/host-resolver/*`. Act on T0a/T0b's systemd-resolved finding
  for RHEL 10.
- Modify: `src/mqlab/hostfacts.py`. Add `x86_64_v3: bool`. Under KVM, it reads
  `/proc/cpuinfo` flags ⊇ `{avx2, bmi1, bmi2, fma, movbe, f16c, abm, xsave}`. Under
  TCG it is True only if T0b confirmed `cpu_mode: maximum` exposes them; otherwise
  it's False.
- Modify: `src/mqlab/versions.py`. `stack_os` enforces `requires: [x86-64-v3]` with
  `VersionError("rhel:10 needs an x86-64-v3 CPU; this host lacks <missing flags> — use os: rhel:9 in your --config")`.
- Modify: `src/mqlab/doctor.py`. Report `x86-64-v3: yes/no` informationally.
- Test: `tests/test_versions.py`, `tests/test_hostfacts.py`,
  `tests/test_build_box_usage.py` and `tests/test_box_build.py` (the fleet gains
  `rhel/10-x86_64` and `mq-nativeha-rhel10` only on a v3 host).

**Interfaces:**

- Consumes: T0a (point release and support), T0b (v3 findings), the T5 vars contract.
- Produces: `HostFacts.x86_64_v3: bool`.

- [ ] **Step 1: Write failing tests.**

```python
def test_rhel10_refused_without_v3():                               # Review Focus (T10)
    facts = replace(X86, x86_64_v3=False)
    with pytest.raises(VersionError, match="rhel:10 needs an x86-64-v3 CPU"):
        load_catalog().stack_os("nativeha-rhel-crr", BuildFile(os=OsRef("rhel", 10)), facts)

def test_rdqm_still_rhel9_only():
    with pytest.raises(VersionError, match=r"rdqm-rhel supports \[rhel:9\]; got rhel:10"):
        load_catalog().stack_os("rdqm-rhel", BuildFile(os=OsRef("rhel", 10)), replace(X86, x86_64_v3=True))

def test_hostfacts_v3_from_cpuinfo(tmp_path):
    cpuinfo = tmp_path / "cpuinfo"; cpuinfo.write_text("flags\t: avx2 bmi1 bmi2 fma movbe f16c abm xsave sse4_2\n")
    assert hostfacts._x86_64_v3(cpuinfo, kvm=True) is True

def test_build_box_selects_major_kickstart(tmp_path):
    r = run_build_box("--major", "10", "--point", "10.0", "--iso", "x.iso", "--dry-run")
    assert "ks-10.cfg" in r.stdout or "ks.cfg" in r.stdout
```

- [ ] **Step 2:** Run them. Expected: they fail. Implement. Then run the full suite,
  coverage and `vrg-container-run -- vrg-validate`.
- [ ] **Step 3:** Commit:
  `vrg-commit --type feat --scope versions --message "RHEL 10: catalog entry, base box, vars, x86-64-v3 gate"`.

### Operational D3 (deployment): Bake RHEL 10

Created with `--kind deployment`, `--blocked-by T10`.

- **Precondition (human-attested):** the RHEL 10 DVD
  (`rhel-<point>-x86_64-dvd.iso`, from T0a's pin) has been staged with
  `lab/scripts/stage-rhel-iso.sh --iso <file>`. This is licensed media and the agent
  never fetches it.
- **Procedure:** On the cloud x86 host, run
  `mqlab box build rhel/10-x86_64 mq-nativeha-rhel10`.
- **Acceptance:** both show REUSE in `mqlab box status`.

### Operational V3 (validation): RHEL 10 row

Created with `--kind validation`, `--blocked-by D3`.

- **Row:** `nativeha-rhel-crr` with `--config` set to `os: rhel:10`, on the cloud x86
  host. Full DR, the end-to-end check including native-HA switchover and CRR, and a
  human dashboard review.
- **Acceptance:** green; `Outcome: SUCCESS`.

### Task T11: Flip `nativeha-rhel-crr`'s default to RHEL 10

The same shape as T9: a failing test, the flip, and the same IBM-support-gate
not-done path. Commit with
`vrg-commit --type feat --scope versions --message "nativeha-rhel-crr defaults to RHEL 10"`.

Note: once this flips, a default `bootstrap nativeha-rhel-crr` on a non-v3 host fails
loudly and names `os: rhel:9`. That is intended (spec §4.3).

---

## Closing bookends (already filed)

- **Docs review:** `mq-resiliency-lab-for-linux#1263`, blocked by T9, T11, V2 and V3.
  A sweep covering `docs/site` plus the dev and reference docs. It spawns per-repo
  doc tasks as needed.
- **Follow-on brainstorm, MQ major axis:** `.github#282`. It reuses `versions.yaml`
  (adding an `mq:` section), `BuildFile` (adding an `mq:` key), `node_boxes` and the
  records.
- **Retrospective:** `.github#283` (terminal).
