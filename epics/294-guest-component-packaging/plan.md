# Guest component packaging (M1) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Every guest component is a standalone uv project under `components/`,
built and tested on one pinned CPython 3.14 build, and installed from staged,
hash-pinned, offline artifacts into its own venv under
`/opt/logical-minds-foundry/<name>/`, baked into the fat boxes. The dark collector
dashboards come back.

**Architecture:**

- `lab/versions.yaml` gains a `runtime:` pin and a per-role `components:` list.
- `mqlab component build` is the only artifact producer. It refuses a dirty tree, a
  stale lock or a failing test (run on the cached pinned interpreter), then stages a
  pure wheel plus hash-pinned requirements and dependency artifacts under
  `$(mqlab build path cache)/components/<name>/<version>+<tree>/`.
- Two Ansible roles, `runtime-install` and `component-install`, are the only install
  path. Both the box bakes and `mqlab component install` use them.
- The box builder folds the runtime pin and each baked component's git tree hash
  into the manifest hash, and builds missing artifacts itself.

**Tech Stack:** Python 3.14 + Typer (`mqlab`, pytest at 100% branch coverage), uv,
hatchling, python-build-standalone CPython 3.14.8, pip, Ansible, systemd, bash box
builders (`lab/boxes/`), Vagrant + vagrant-libvirt, IBM MQ 10.0 client SDK, `pymqi`
1.12.13.

**Spec:** `epics/294-guest-component-packaging/spec.md` (this directory). Executors
read both.

**Repos:**

- Implementation tasks land in `logical-minds-foundry/mq-resiliency-lab-for-linux`.
- This plan and the spec live in `logical-minds-foundry/.github`.

## Global Constraints

- **Runtime pin:** CPython `3.14.8`, python-build-standalone release `20261003`;
  sha256 x86_64 `371b6c281bbb09b29279e9e3a2996bab4ae2ea03cca52bf869f8bd89286b0ae8`,
  aarch64 `abc0c8dd54a144909a5e4905737bc33f244cb17ce8afc4a7b0791ee1e006ae23`
  (`install_only` artifacts). It is written by hand **only** in `lab/versions.yaml`.
- **Guest interpreter path:** `/opt/vergil/cpython-3.14/` (minor-keyed; patch bumps
  replace it in place).
- **Component layout:** `/opt/logical-minds-foundry/<name>/venv`, `venv.prev`,
  `INSTALLED.json`; config at `/etc/opt/logical-minds-foundry/<name>/<unit>.env`;
  static units at `/usr/lib/systemd/system/`.
- **Component names:** `mq-resiliency-observability` (import `mqro`) and
  `mq-resiliency-clients` (import `mqrc`). Each is the directory name, the future
  repo name and the `/opt` directory name.
- **Components never import `mqlab`, and `mqlab` never imports components.**
  Components have no `vergil.toml` and are never workspace members.
- **Component `requires-python = "==3.14.*"`.** The backend is hatchling.
- **Guests never run `uv`.** Guests install with the venv's own `pip`, always with
  `--no-index --find-links deps/` and `--require-hashes`.
- **No fallbacks:** none to the system Python, to PyPI, or to an older artifact.
  mqlab errors name fully-qualified `mqlab` commands. Underlying tools speak for
  themselves.
- **`build/` paths go only through `mqlab.paths`** (`cache()`, `work()`, …). Never
  hard-code a `build/<X>` path.
- **Ansible's interpreter stays `/usr/bin/python3`.** The pinned interpreter is never
  Ansible's.
- **Operator-facing names are unchanged:** `lab-cluster-state`,
  `lab-nativeha-state`, `lab-loglifecycle-state`, `lab-rdqm-state`, `mq-bench`,
  `mq-app-requester.service`, `mq-svc-responder@.service`, and the textfile drop
  paths.
- **Validation:** `vrg-container-run -- vrg-validate` covers mqlab only. Component
  tests run through `mqlab component build <name>`.
- **Commits:** `vrg-commit --type <t> --scope <s> --message <m>`; git operations use
  `vrg-git`.
- **T6 through T9 are one chain** (spec §8). Develop is not guaranteed to provision
  between T6 and T9. Do not cold-rebuild per task. V1/V2 validate the chain.

## Review Focus

1. **Old units in `/etc/systemd/system` shadow the new component units.** Today the
   roles render `lab-cluster-state.*`, `lab-nativeha-state.*`, `lab-rdqm-state.*`
   and `mq-app-requester.service` into `/etc/systemd/system/`
   (`cluster-state/tasks/main.yml:21`, `nativeha-state/tasks/main.yml:38`,
   `rdqm-state/tasks/main.yml:33`, `app-requester/tasks/main.yml:44`). systemd
   prefers `/etc` over `/usr/lib`, so a guest that keeps the old file runs the old
   `/usr/bin/python3` `ExecStart`. Each rewire role must delete the `/etc` copies and
   `daemon-reload`. Pinned by `test_rewired_roles_remove_etc_unit_copies` (Task T7)
   and `test_client_roles_remove_etc_unit_copies` (Task T9).
2. **A component edited but not rebuilt** must not reuse a stale box. Pinned by
   `test_manifest_hash_flips_on_component_tree_change` and
   `test_box_build_builds_missing_component_artifact` (Task T5).
3. **A failed install leaves the running version untouched.** A `selfcheck` failure
   in `venv.new` must not swap, and must fail the play. Pinned by
   `test_component_install_swaps_only_after_selfcheck` (Task T5) plus the V1 manual
   check.
4. **Building with uncommitted component changes** is refused, with the dirty paths
   named. Pinned by `test_build_refuses_dirty_component` (Task T4).
5. **A host whose interpreter doesn't match the pin** (an older baked box after a pin
   bump) fails `mqlab component install` before touching the venv, naming
   `mqlab box build <box>`. Pinned by
   `test_component_install_asserts_runtime_pin_first` (Task T5).

## Task dependency graph

```text
T1  docs: spec + plan (.github#295)
T2  spike: runtime feasibility (report)                      [T1]
T3  boundary foundations + runtime pin + roles.components    [T2]
 ├─► T4  mqlab component build + status                      [T3]
 │     └─► T5  runtime-install + component-install roles,
 │             mqlab component install, box-builder wiring   [T4]
 ├─► T6  carve mq-resiliency-observability                   [T3]   ─┐
 └─► T8  carve mq-resiliency-clients                         [T3]    │ chain
        T7  rewire collector roles + bakes                   [T5,T6] │ (develop may not
        T9  rewire client roles, playbooks, dr-run.sh        [T5,T8] ─┘  provision T6..T9)
V1  validate: cold rebuild, Ubuntu stacks (arm64 local)      [T7,T9]
V2  validate: cold rebuild, RHEL stacks (x86 cloud)          [T7,T9]
Closing  docs review (#1348) · follow-ons (.github#296, #298) · retrospective (.github#297)
```

---

## Task T2: Spike — runtime feasibility (report PR)

**Files:**

- Create: `docs/reports/2026-10-guest-runtime-spike.md`

**Interfaces:**

- Produces a go/no-go per row (consumed by T3 to T9):
  1. The 3.14.8 tarball runs on RHEL 9.6 x86_64, Ubuntu 24.04 x86_64 and Ubuntu
     24.04 aarch64 (`python3 -c 'import ssl, sqlite3, ctypes; print(ssl.OPENSSL_VERSION)'`).
  2. `pymqi==1.12.13` builds from sdist in a venv on that interpreter against
     `/opt/mqm`, with `--no-build-isolation` and pip-installed `setuptools` +
     `wheel`, and `python -c 'import pymqi'` succeeds with `LD_LIBRARY_PATH=/opt/mqm/lib64`.
  3. A full offline install (`--no-index --find-links deps/ --require-hashes`)
     succeeds on RHEL with the network to PyPI unreachable.
  4. The exact `setuptools` and `wheel` versions that built pymqi, for T8's
     `sdist-build` group.

- [ ] **Step 1 (human-run):** On a running `mq-rdqm`/`nha-rhel-crr` node (x86 cloud)
  and an `app-client` (arm64 local), copy the tarball from
  `https://github.com/astral-sh/python-build-standalone/releases/download/20261003/`.
  Verify it with `sha256sum` against the Global Constraints, then unpack it into
  `/opt/vergil/cpython-3.14/`. Run row 1's command and record the output.
- [ ] **Step 2 (human-run):** On each node with `/opt/mqm/inc/cmqc.h`:
  `/opt/vergil/cpython-3.14/bin/python3 -m venv /tmp/spike-venv`, then
  `/tmp/spike-venv/bin/pip install setuptools wheel`, then
  `/tmp/spike-venv/bin/pip install --no-build-isolation pymqi==1.12.13`. Record the
  full output and `pip freeze`.
- [ ] **Step 3 (human-run):** On the RHEL node, do a dev-VM `pip download` of the
  same set into `deps/` with hashes, copy it over, block the network
  (`ip route add blackhole 151.101.0.0/16` or equivalent; record what you used), and
  repeat Step 2 with `--no-index --find-links deps/ --require-hashes`.
- [ ] **Step 4:** Write the report. Each row is **data** (pasted output) or a
  **finding**. Add a go/no-go table.
- [ ] **Step 5:** `vrg-container-run -- vrg-validate`, then
  `vrg-commit --type docs --scope reports --message "guest runtime spike: pbs 3.14.8 + pymqi on RHEL/Ubuntu"`.

**Go/no-go:** any row that fails stops the epic. Comment on `.github#294`; the spec
is revised before T3 (spec §11). There is no workaround by fallback.

---

## Task T3: Boundary foundations, runtime pin, `roles.components`

**Files:**

- Create: `components/README.md` (the component contract: spec §5.4 verbatim rules
  plus the layout block)
- Modify: `lab/versions.yaml` (add `runtime:`; add `components:` to roles `pcmk`,
  `san`, `mq-nativeha`, `mq-rdqm` → `[mq-resiliency-observability]` and
  `mq-client` → `[mq-resiliency-clients]`)
- Modify: `src/mqlab/versions.py` (`RuntimePin`, `Catalog.runtime`,
  `_ROLE_KEYS += components`, `BoxEntry.components`)
- Modify: `pyproject.toml` (`[tool.ruff] extend-exclude = ["components"]`)
- Modify: `.ansible-lint.yml` (`exclude_paths: + components/`)
- Create: `tests/test_component_boundary.py`
- Modify: `tests/test_versions.py` (runtime + components parsing)

**Interfaces:**

- Produces:

  ```python
  # src/mqlab/versions.py
  ARCHES = ("x86_64", "aarch64")

  @dataclass(frozen=True)
  class RuntimePin:
      version: str            # "3.14.8"
      pbs_release: str        # "20261003"
      sha256: dict[str, str]  # arch -> hex digest; keys == ARCHES

      @property
      def minor(self) -> str: ...        # "3.14"
      @property
      def token(self) -> str: ...        # "3.14.8+20261003"
      def tarball(self, arch: str) -> str:
          # "cpython-3.14.8+20261003-<arch>-unknown-linux-gnu-install_only.tar.gz"
      def url(self, arch: str) -> str:
          # "https://github.com/astral-sh/python-build-standalone/releases/download/<release>/<tarball>"

  @dataclass(frozen=True)
  class Catalog:  # existing, gains:
      runtime: RuntimePin

  @dataclass(frozen=True)
  class BoxEntry:  # existing, gains:
      components: tuple[str, ...] = ()
  ```

  `Catalog.box()` copies `roles[role]["components"]` into `BoxEntry.components`.

- [ ] **Step 1: Write the failing catalog tests** in `tests/test_versions.py`:

  ```python
  def test_runtime_pin_parsed(catalog_path_with_runtime):
      cat = load_catalog(catalog_path_with_runtime)
      assert cat.runtime.token == "3.14.8+20261003"
      assert cat.runtime.minor == "3.14"
      assert cat.runtime.tarball("aarch64") == (
          "cpython-3.14.8+20261003-aarch64-unknown-linux-gnu-install_only.tar.gz")

  GOOD_SHA = {"x86_64": "a" * 64, "aarch64": "b" * 64}

  @pytest.mark.parametrize("bad", [
      {"python": {"version": "3.14", "pbs_release": "20261003", "sha256": GOOD_SHA}},   # not x.y.z
      {"python": {"version": "3.14.8", "pbs_release": "2026-10-03", "sha256": GOOD_SHA}},  # not 8 digits
      {"python": {"version": "3.14.8", "pbs_release": "20261003", "sha256": {"x86_64": "a" * 64}}},  # arch missing
      {"python": {"version": "3.14.8", "pbs_release": "20261003", "sha256": {"x86_64": "ab", "aarch64": "b" * 64}}},  # bad hex
      {},  # python missing
  ])
  def test_runtime_pin_rejects_malformed(write_catalog, bad):
      path = write_catalog(runtime=bad)   # fixture: today's catalog with runtime replaced
      with pytest.raises(VersionError, match=r"runtime\.python.*lab/versions\.yaml"):
          load_catalog(path)

  def test_role_components_flow_to_box_entry(catalog): ...
      assert catalog.box("pcmk", OsRef("ubuntu", 24)).components == ("mq-resiliency-observability",)
      assert catalog.box("infra", OsRef("ubuntu", 24)).components == ()

  def test_role_components_must_be_known_dirs(tmp_path): ...
      # a components entry naming a directory absent under components/ -> VersionError
  ```

  `write_catalog` is a new `conftest.py` fixture. It loads the committed
  `lab/versions.yaml`, applies keyword overrides to the top-level keys, and writes
  the result under `tmp_path`.

- [ ] **Step 2:** Run `vrg-container-run -- vrg-validate`. Expected: the new tests
  FAIL.
- [ ] **Step 3: Implement.** Add `"runtime"` to `_TOP_KEYS` and `"components"` to
  `_ROLE_KEYS`. `_runtime(raw)` validates `version` against `^\d+\.\d+\.\d+$`,
  `pbs_release` against `^\d{8}$`, and `sha256` keys `== set(ARCHES)` with values
  against `^[0-9a-f]{64}$`. `_roles` validates `components` as a list of strings,
  each `(repo_root() / "components" / name / "pyproject.toml").is_file()`. Every
  error ends with `_CATALOG_FIX`.
- [ ] **Step 4: Add the pin to `lab/versions.yaml`:**

  ```yaml
  # The ONE pinned guest Python runtime (epic .github#294). Contract = the minor
  # (3.14); a patch bump is this block only. sha256 = the install_only tarballs, from
  # https://github.com/astral-sh/python-build-standalone/releases/download/20261003/SHA256SUMS
  runtime:
    python:
      version: "3.14.8"
      pbs_release: "20261003"
      sha256:
        x86_64: "371b6c281bbb09b29279e9e3a2996bab4ae2ea03cca52bf869f8bd89286b0ae8"
        aarch64: "abc0c8dd54a144909a5e4905737bc33f244cb17ce8afc4a7b0791ee1e006ae23"
  ```

  The `components:` role keys are added in T6 and T8, once the directories exist.
  The known-dir check would refuse them here.
- [ ] **Step 5: Write `tests/test_component_boundary.py`:**

  ```python
  """components/ is NOT part of mqlab (epic .github#294 spec §5.1)."""
  import ast, tomllib
  from pathlib import Path

  ROOT = Path(__file__).resolve().parents[1]
  COMPONENTS = ROOT / "components"

  def _imports(py: Path) -> set[str]:
      tree = ast.parse(py.read_text())
      out: set[str] = set()
      for node in ast.walk(tree):
          if isinstance(node, ast.Import):
              out |= {a.name.split(".")[0] for a in node.names}
          elif isinstance(node, ast.ImportFrom) and node.module and node.level == 0:
              out.add(node.module.split(".")[0])
      return out

  def _component_import_names() -> set[str]:
      names = set()
      for pp in COMPONENTS.glob("*/pyproject.toml"):
          names |= {p.name for p in (pp.parent / "src").iterdir() if p.is_dir()}
      return names

  def test_mqlab_never_imports_a_component():
      comps = _component_import_names()
      bad = {str(p): _imports(p) & comps for p in (ROOT / "src/mqlab").rglob("*.py")}
      assert not {k: v for k, v in bad.items() if v}

  def test_components_never_import_mqlab():
      bad = [str(p) for p in COMPONENTS.rglob("*.py") if "mqlab" in _imports(p)]
      assert not bad

  def test_mqlab_wheel_packages_only_src_mqlab():
      cfg = tomllib.loads((ROOT / "pyproject.toml").read_text())
      assert cfg["tool"]["hatch"]["build"]["targets"]["wheel"]["packages"] == ["src/mqlab"]

  def test_root_tooling_excludes_components():
      cfg = tomllib.loads((ROOT / "pyproject.toml").read_text())
      assert "components" in cfg["tool"]["ruff"]["extend-exclude"]
      lint = (ROOT / ".ansible-lint.yml").read_text()
      assert "components/" in lint
  ```

  With no components yet, the first two tests pass vacuously. T6 and T8 make them
  bite.
- [ ] **Step 6:** Add the ruff `extend-exclude` and the `.ansible-lint.yml` entry.
  Write `components/README.md` with the contract: rules 1–6 from spec §5.4, the
  layout block, "how to add a component", and that `vrg-validate` does not cover
  components.
- [ ] **Step 7:** `vrg-container-run -- vrg-validate`. Expected: PASS at 100%.
- [ ] **Step 8:** `vrg-commit --type feat --scope components --message "component boundary, runtime pin, roles.components"`.

---

## Task T4: `mqlab component build` + `status`

**Files:**

- Create: `src/mqlab/runtime.py`, `src/mqlab/component.py`
- Modify: `src/mqlab/cli.py` (add `component_app` with `build` and `status`)
- Create: `tests/test_runtime.py`, `tests/test_component.py`, `tests/test_cli_component.py`

**Interfaces:**

- Consumes: `RuntimePin`, `load_catalog()` (T3); `mqlab.paths.cache`, `repo_root`;
  `mqlab.runner.Command`, `SubprocessRunner`.
- Produces:

  ```python
  # src/mqlab/runtime.py
  class RuntimePinError(Exception)
  def host_arch() -> str                           # platform.machine() normalised: arm64->aarch64
  def runtime_dir() -> Path                        # paths.cache("runtime")
  def ensure_tarball(pin: RuntimePin, arch: str, *, fetch: Callable[[str, Path], None] = _urlretrieve) -> Path
      # cached at runtime_dir()/pin.tarball(arch); sha256-verified on EVERY call;
      # a mismatch deletes the file and raises RuntimePinError naming the expected and
      # actual digests (never re-downloads silently).
  def ensure_interpreter(pin: RuntimePin, arch: str | None = None) -> Path
      # unpacks to runtime_dir()/f"cpython-{pin.token}-{arch}"/ ; returns .../python/bin/python3.14

  # src/mqlab/component.py
  class ComponentError(Exception)
  COMPONENTS_DIRNAME = "components"
  @dataclass(frozen=True)
  class Artifact:
      name: str; version: str; tree: str; path: Path
  def known_components() -> list[str]              # dirs under components/ with a pyproject.toml
  def component_dir(name: str) -> Path             # ComponentError if unknown, listing known
  def tree_hash(name: str, *, runner=...) -> str   # git rev-parse HEAD:components/<name>
  def dirty_paths(name: str, *, runner=...) -> list[str]   # git status --porcelain -- components/<name>
  def version(name: str) -> str                    # [project].version via tomllib
  def artifact_dir(name: str, version: str, tree: str) -> Path   # paths.cache("components", name, f"{version}+{tree}")
  def find_artifact(name: str, tree: str) -> Artifact | None     # a dir *+<tree> containing BUILD.json
  def current(name: str) -> Artifact | None        # reads paths.cache("components", name, "CURRENT")
  def build(name: str, *, runner: CommandRunner, on_line: Callable[[str], None]) -> Artifact
  def ensure_built(name: str, *, runner, on_line) -> Artifact   # find_artifact(tree_hash) or build()
  ```

- [ ] **Step 1: Failing tests for `runtime.py`** (`tests/test_runtime.py`), with
  `fetch` injected writing known bytes:
  - `test_ensure_tarball_downloads_then_verifies`
  - `test_ensure_tarball_sha_mismatch_deletes_and_raises` (message contains both
    digests and `lab/versions.yaml`)
  - `test_ensure_tarball_reverifies_cached_file` (a corrupted cached file is caught on
    the second call)
  - `test_host_arch_normalises_arm64`
  - `test_ensure_interpreter_unpacks_once` (unpacks a tiny fake tarball containing
    `python/bin/python3.14`; the second call doesn't re-extract)
- [ ] **Step 2: Failing tests for `component.py`** (`tests/test_component.py`), with a
  fake runner recording `Command`s and a `tmp_path` repo containing
  `components/demo/pyproject.toml`:
  - `test_build_refuses_dirty_component`: `dirty_paths` returns
    `["components/demo/x.py"]`, so `ComponentError` names that path and "commit".
  - `test_build_runs_steps_in_order`: asserts the exact argv sequence:
    1. `git rev-parse HEAD:components/demo`
    2. `git status --porcelain -- components/demo`
    3. `uv --directory components/demo lock --check`
    4. `uv --directory components/demo run --frozen --python <interp> pytest`
    5. `uv --directory components/demo build --wheel --out-dir <stage>`
    6. `uv --directory components/demo export --frozen --no-dev --no-emit-project --all-extras --format requirements-txt -o <stage>/requirements.txt`
    7. (only if the `sdist-build` dependency group exists) `uv --directory components/demo export --frozen --only-group sdist-build --no-emit-project --format requirements-txt -o <stage>/build-requirements.txt`
  - `test_build_stops_on_failing_tests`: runner exit 1 at step 4 means `ComponentError`
    and nothing staged.
  - `test_build_stages_deps_from_lock`: given a fixture `uv.lock` with a `pymqi`
    sdist entry (url + `sha256:`), `deps/` receives that file (fetch injected) and its
    hash is verified.
  - `test_build_writes_build_json_and_current`: `BUILD.json` keys are `name`,
    `version`, `tree`, `commit`, `runtime` (`pin.token`), `tests` (`"passed"`) and
    `built_at`, and `CURRENT` holds the dir name.
  - `test_build_stages_systemd_units`: `components/demo/systemd/*` is copied to
    `<artifact>/systemd/`. A component without `systemd/` stages none, without error.
  - `test_ensure_built_reuses_matching_tree` and `test_ensure_built_builds_when_missing`.
  - `test_unknown_component_lists_known`.
- [ ] **Step 3:** Run `vrg-container-run -- vrg-validate`. Expected: FAIL.
- [ ] **Step 4: Implement `runtime.py` and `component.py`.** Staging rules:
  - Build into `artifact_dir(...).with_suffix(".partial")`, then `rename` on
    success, so a failed build never leaves a half artifact that `find_artifact` would
    match.
  - `deps/` is populated by parsing `components/<name>/uv.lock` (`tomllib`): for
    every package named in `requirements.txt`, and in `build-requirements.txt` if
    present, fetch its `sdist` (if any) and every `wheels[]` entry that a CPython 3.14
    Linux guest could install, verifying each entry's `hash`. A wheel qualifies when:
    - its python tag is `py3`, `cp314` or `cp3x` + `abi3`;
    - **and** its platform tag is `any` or starts with `manylinux` and ends in
      `x86_64` or `aarch64`.
  - `pip` on the guest picks the right file.
  - `systemd/` is copied from the component into `<artifact>/systemd/`.
- [ ] **Step 5: CLI** in `cli.py`, following the `box_app` pattern:

  ```python
  component_app = typer.Typer(
      help="lab guest components: build (test + stage) / status / install", no_args_is_help=True)
  app.add_typer(component_app, name="component")

  @component_app.command("build")
  def component_build(names: list[str] | None = typer.Argument(None), all_: bool = typer.Option(False, "--all")) -> None:
      """Test on the pinned runtime and stage an installable artifact (the ONLY producer)."""

  @component_app.command("status")
  def component_status(host: list[str] | None = typer.Option(None, "--host")) -> None:
      """Built artifacts (CURRENT, tree, tests) and, with --host, each host's INSTALLED.json."""
  ```

  `status --host` reads `INSTALLED.json` with
  `ansible <host> -i <inventory_path()> -m ansible.builtin.slurp -a src=/opt/logical-minds-foundry/<name>/INSTALLED.json`
  and prints one row per host and component: installed version and tree, and whether
  that matches `CURRENT`.
- [ ] **Step 6:** `tests/test_cli_component.py` covers: no args and no `--all` exits 2,
  listing the known components; `ComponentError` exits 1 with the message; and the
  status table rendering, including a host with no `INSTALLED.json`, which shows
  `not installed`.
- [ ] **Step 7:** `vrg-container-run -- vrg-validate`. Expected: PASS at 100%.
- [ ] **Step 8:** `vrg-commit --type feat --scope component --message "mqlab component build/status: pinned-runtime test gate + staged artifacts"`.

---

## Task T5: Install roles, `mqlab component install`, box-builder wiring

**Files:**

- Create: `ansible/roles/runtime-install/{tasks/main.yml,defaults/main.yml}`
- Create: `ansible/roles/component-install/{tasks/main.yml,defaults/main.yml}`
- Create: `ansible/component-install.yml` (play: `hosts: "{{ target }}"`, roles:
  runtime-install, component-install)
- Modify: `src/mqlab/component.py` (`install_vars(names) -> Path`),
  `src/mqlab/cli.py` (`component install`)
- Modify: `src/mqlab/box.py` (`builder_args`: `--runtime-pin`, `--components`;
  `_build_plan`/`_build_steps`: `ensure_built` before a BUILD)
- Modify: `lab/boxes/build-fatbox.sh` (accept the two flags; pass them to
  `_manifest-hash.sh`; add `-e @<install-vars>` to `BAKE_EXTRA_VARS`)
- Modify: `lab/boxes/_manifest-hash.sh` (digest `--runtime-pin` and `--components`)
- Create: `tests/test_install_roles.py`, `tests/test_box_components.py`; extend
  `tests/test_manifest_hash*.py` (whichever file covers `_manifest-hash.sh` today)

**Interfaces:**

- Consumes: `Artifact`, `ensure_built`, `ensure_tarball`, `RuntimePin`,
  `BoxEntry.components`.
- Produces:
  - `install_vars(names: list[str], arch: str | None = None) -> Path` writes
    `paths.work("components", "install-vars.json")`:

    ```json
    {"runtime_python": {"version": "3.14.8", "pbs_release": "20261003", "minor": "3.14",
                        "token": "3.14.8+20261003", "tarball_dir": "<cache>/runtime",
                        "tarballs": {"x86_64": "<name>", "aarch64": "<name>"},
                        "sha256": {"x86_64": "...", "aarch64": "..."}},
     "component_artifacts": {"<name>": "<cache>/components/<name>/<version>+<tree>"}}
    ```

  - Role variables: `runtime_python` (above); `component_name` (string);
    `component_artifacts` (map); `component_start_units` (bool, default `false`;
    `true` from `mqlab component install`).
  - Builder flags: `--runtime-pin 3.14.8+20261003` and
    `--components "<name>@<tree>[,<name>@<tree>…]"` (empty string when none).

- [ ] **Step 1: Write `runtime-install`** (`tasks/main.yml`):

  ```yaml
  # Lay down the pinned guest interpreter (epic .github#294 spec §5.2/§5.7). Idempotent:
  # a matching /opt/vergil/cpython-<minor>/PIN is a no-op. NEVER Ansible's interpreter.
  - name: map the guest arch to a runtime tarball
    ansible.builtin.set_fact:
      _rt_arch: "{{ {'x86_64': 'x86_64', 'aarch64': 'aarch64', 'arm64': 'aarch64'}[ansible_architecture] }}"
      _rt_root: "/opt/vergil/cpython-{{ runtime_python.minor }}"
  - name: read the installed runtime PIN
    ansible.builtin.slurp: {src: "{{ _rt_root }}/PIN"}
    register: _rt_pin
    failed_when: false
  - name: install the pinned runtime
    when: (_rt_pin.content | default('') | b64decode | trim) != runtime_python.token
    become: true
    block:
      - name: copy the pinned tarball
        ansible.builtin.copy:
          src: "{{ runtime_python.tarball_dir }}/{{ runtime_python.tarballs[_rt_arch] }}"
          dest: "/var/tmp/{{ runtime_python.tarballs[_rt_arch] }}"
          checksum: "{{ omit }}"
          mode: "0644"
      - name: verify its sha256 against the pin
        ansible.builtin.stat: {path: "/var/tmp/{{ runtime_python.tarballs[_rt_arch] }}", checksum_algorithm: sha256}
        register: _rt_tar
        failed_when: _rt_tar.stat.checksum != runtime_python.sha256[_rt_arch]
      - name: unpack beside, then swap into place
        ansible.builtin.shell: |
          set -euo pipefail
          rm -rf "{{ _rt_root }}.new" && mkdir -p "{{ _rt_root }}.new"
          tar -xzf "/var/tmp/{{ runtime_python.tarballs[_rt_arch] }}" -C "{{ _rt_root }}.new" --strip-components=1
          echo "{{ runtime_python.token }}" > "{{ _rt_root }}.new/PIN"
          rm -rf "{{ _rt_root }}.old"; [ -d "{{ _rt_root }}" ] && mv "{{ _rt_root }}" "{{ _rt_root }}.old"
          mv "{{ _rt_root }}.new" "{{ _rt_root }}" && rm -rf "{{ _rt_root }}.old"
        args: {executable: /bin/bash}
  ```

- [ ] **Step 2: Write `component-install`** (`tasks/main.yml`):

  ```yaml
  # Install ONE staged component artifact (spec §5.7). The only install path: bake and
  # `mqlab component install` both run this. Live venv untouched unless selfcheck passes.
  - name: read the runtime PIN
    ansible.builtin.slurp: {src: "/opt/vergil/cpython-{{ runtime_python.minor }}/PIN"}
    register: _pin
    failed_when: false
  - name: precondition — runtime matches the pin (checked BEFORE any change)
    ansible.builtin.assert:
      that:
        - _pin.content is defined
        - (_pin.content | b64decode | trim) == runtime_python.token
      fail_msg: >-
        the guest runtime on {{ inventory_hostname }} is not {{ runtime_python.token }}
        — rebuild its box (mqlab box build <box>) and re-provision
  - name: set component facts
    ansible.builtin.set_fact:
      _c_root: "/opt/logical-minds-foundry/{{ component_name }}"
      _c_etc: "/etc/opt/logical-minds-foundry/{{ component_name }}"
      _c_src: "{{ component_artifacts[component_name] }}"
      _c_stage: "/var/tmp/component-{{ component_name }}"
  - name: clear the guest staging dir
    become: true
    ansible.builtin.file: {path: "{{ _c_stage }}", state: absent}
  - name: copy the staged artifact (trailing slash = contents)
    become: true
    ansible.builtin.copy: {src: "{{ _c_src }}/", dest: "{{ _c_stage }}/", mode: preserve}
  - name: component dirs (/opt payload, /etc/opt deployer config)
    become: true
    ansible.builtin.file: {path: "{{ item }}", state: directory, mode: "0755"}
    loop: ["{{ _c_root }}", "{{ _c_etc }}"]
  - name: build venv.new (pinned runtime, venv pip, offline, hash-pinned)
    become: true
    ansible.builtin.shell: |
      set -euo pipefail
      rm -rf "{{ _c_root }}/venv.new"
      /opt/vergil/cpython-{{ runtime_python.minor }}/bin/python3 -m venv "{{ _c_root }}/venv.new"
      P="{{ _c_root }}/venv.new/bin/pip"
      F="--no-index --find-links {{ _c_stage }}/deps"
      if [ -s "{{ _c_stage }}/build-requirements.txt" ]; then
        $P install $F --require-hashes -r "{{ _c_stage }}/build-requirements.txt"
      fi
      if [ -s "{{ _c_stage }}/requirements.txt" ]; then
        $P install $F --require-hashes --no-build-isolation -r "{{ _c_stage }}/requirements.txt"
      fi
      $P install --no-index --no-deps "{{ _c_stage }}"/*.whl
    args: {executable: /bin/bash}
    environment: {LD_LIBRARY_PATH: /opt/mqm/lib64}
  - name: selfcheck from venv.new (BEFORE the swap)
    become: true
    ansible.builtin.command: "{{ _c_root }}/venv.new/bin/{{ component_name }}-selfcheck"
    environment: {LD_LIBRARY_PATH: /opt/mqm/lib64}
    changed_when: false
  - name: swap venv.new into place, keep venv.prev
    become: true
    ansible.builtin.shell: |
      set -euo pipefail
      cd "{{ _c_root }}"
      rm -rf venv.prev
      if [ -d venv ]; then mv venv venv.prev; fi
      mv venv.new venv
    args: {executable: /bin/bash}
  - name: record INSTALLED.json
    become: true
    ansible.builtin.copy:
      dest: "{{ _c_root }}/INSTALLED.json"
      mode: "0644"
      content: >-
        {{ (lookup('file', _c_src + '/BUILD.json') | from_json)
           | combine({'installed_at': now(utc=true).isoformat(),
                      'runtime_installed': runtime_python.token})
           | to_nice_json }}
  - name: list the component's static units
    ansible.builtin.set_fact:
      _c_units: "{{ query('fileglob', _c_src + '/systemd/*') | map('basename') | list }}"
  - name: install the component's static units
    become: true
    ansible.builtin.copy:
      src: "{{ _c_src }}/systemd/{{ item }}"
      dest: "/usr/lib/systemd/system/{{ item }}"
      mode: "0644"
    loop: "{{ _c_units }}"
  - name: daemon-reload
    become: true
    ansible.builtin.systemd: {daemon_reload: true}
  - name: which of this component's units are enabled (live lab only)
    when: component_start_units | bool
    ansible.builtin.command: "systemctl is-enabled {{ item }}"
    loop: "{{ _c_units }}"
    register: _c_enabled
    changed_when: false
    failed_when: false   # is-enabled exits non-zero for "disabled"/"static"; read stdout below
  - name: restart the enabled ones (a failed restart FAILS the play)
    when:
      - component_start_units | bool
      - item.stdout | default('') == 'enabled'
    become: true
    ansible.builtin.systemd: {name: "{{ item.item }}", state: restarted}
    loop: "{{ _c_enabled.results | default([]) }}"
    loop_control: {label: "{{ item.item }}"}
  ```

  The `failed_when: false` on `is-enabled` is not a swallowed error. `is-enabled` uses
  its exit code to say "not enabled", and the task reads `stdout` instead. A unit that
  fails to restart fails the play through `ansible.builtin.systemd`. During a bake,
  `component_start_units` is false, so units stay inert.

- [ ] **Step 3: Failing structural tests** (`tests/test_install_roles.py`, parsing
  the YAML in the style of `tests/test_responder_venv_baked.py`):
  - `test_component_install_asserts_runtime_pin_first`: the first two tasks slurp
    `PIN` and `assert` it equals `runtime_python.token`, before any task with
    `become: true`.
  - `test_component_install_copies_with_builtin_only`: no `ansible.posix.*` modules
    (the collection isn't guaranteed in the bake's `ANSIBLE_COLLECTIONS_PATH`).
  - `test_component_install_swaps_only_after_selfcheck`: the selfcheck task index is
    less than the swap task index, and the swap task contains `mv venv.new venv`.
  - `test_component_install_never_uses_uv_or_pypi`: no task text contains `uv ` or
    `pypi`, and every `pip install` carries `--no-index`.
  - `test_runtime_install_verifies_sha_before_unpack`.
  - `test_runtime_install_never_sets_ansible_python_interpreter`.
- [ ] **Step 4: `mqlab component install`:**

  ```python
  @component_app.command("install")
  def component_install(name: str, host: list[str] = typer.Option(..., "--host"),
                        version: str | None = typer.Option(None, "--version")) -> None:
      """Install a BUILT component onto running hosts (same role the bake uses)."""
  ```

  It resolves the artifact (`--version` dir, or `ensure_built`), writes
  `install_vars`, then runs
  `ansible-playbook component-install.yml -i <inventory_path()> -e target=<hosts> -e component_name=<name> -e component_start_units=true -e @<install-vars>`
  with `cwd=repo_root()/"ansible"`. Tests cover argv construction, plus "not built
  and dirty", which exits 1 naming `commit` then `mqlab component build <name>`.
- [ ] **Step 5: Box-builder wiring (spec §5.8).**
  - In `box.py` `builder_args` for a fat box, append
    `"--runtime-pin", catalog.runtime.token, "--components", ",".join(f"{n}@{tree_hash(n)}" for n in spec.components)`.
    `BoxSpec` gains `components: tuple[str, ...]` from `BoxEntry`.
  - In the build path, before any step that BAKES a box with components, call
    `ensure_built(n)` for each component and `install_vars(...)`.
  - In `build-fatbox.sh`:
    - parse `--runtime-pin` and `--components` (both REQUIRED, like `--os-pin`; an
      empty `--components ""` is allowed);
    - pass both to `_manifest-hash.sh`;
    - when non-empty, append `-e @"$MAIN_ROOT/build/work/components/install-vars.json"`
      and `-e baked_components=<names>` to `BAKE_EXTRA_VARS`. Match the script's
      existing `MAIN_ROOT` idiom, which is used for `ANSIBLE_COLLECTIONS_PATH`.
  - In `_manifest-hash.sh`, add both values to the digest input with a header comment
    in the house style. Explain why the hash keys on the **tree hash, not the
    artifact**, citing #649/#1324/#1087 as the precedent.
  - **Record what the box baked (spec §5.8).** After a successful bake,
    `build-fatbox.sh` writes `<box>-<arch>.components.json` beside the cached `.box`
    and `.manifest-hash`, with `{"runtime": "<pin>", "components": {"<name>": "<tree>"}}`.
    `mqlab box status` shows it in a `components` column.
- [ ] **Step 6: Failing tests for the wiring** (`tests/test_box_components.py`, plus
  the manifest-hash test file):
  - `test_manifest_hash_flips_on_component_tree_change`: same args except
    `--components demo@aaa` vs `demo@bbb` gives different digests.
  - `test_manifest_hash_flips_on_runtime_pin_change`.
  - `test_manifest_hash_requires_runtime_pin_flag`: missing gives exit 2.
  - `test_box_build_builds_missing_component_artifact`: with `ensure_built`
    monkeypatched, a BUILD of `pcmk-ubuntu24` calls it for
    `mq-resiliency-observability` before the builder step.
  - `test_builder_args_carry_components_and_pin`.
  - `test_box_build_records_baked_components`: the builder's post-bake step writes
    `<box>-<arch>.components.json` with the pin and the tree hashes.
  - `test_box_status_shows_baked_components`.
- [ ] **Step 7:** Implement until green. Run `vrg-container-run -- vrg-validate`.
  Expected: PASS at 100%.
- [ ] **Step 8:** `vrg-commit --type feat --scope component --message "runtime/component install roles, mqlab component install, box-builder wiring"`.

The bake playbooks start including these roles in T7 and T9. That's when
`roles.components` is populated.

---

## Task T6: Carve out `mq-resiliency-observability` (chain start)

**Files:**

- Create: `components/mq-resiliency-observability/` with:
  - `pyproject.toml`
  - `uv.lock`
  - `README.md`
  - `src/mqro/{__init__.py,clusterstate.py,nativehastate.py,rdqmstate.py,loglifecycle.py,selfcheck.py}`
  - `tests/`, holding the moved `test_clusterstate.py`, `test_nativehastate.py`,
    `test_rdqmstate.py` and `test_loglifecycle.py`, plus a new `test_selfcheck.py`
  - `systemd/`, holding `lab-cluster-state.{service,timer}`,
    `lab-nativeha-state.{service,timer}` and `lab-rdqm-state.{service,timer}`
- Delete: `src/mqlab/{clusterstate,nativehastate,rdqmstate,loglifecycle}.py`,
  `tests/test_{clusterstate,nativehastate,rdqmstate,loglifecycle}.py`
- Modify: `lab/versions.yaml` (`components: [mq-resiliency-observability]` on `pcmk`,
  `san`, `mq-nativeha`, `mq-rdqm`)

**Interfaces:**

- Produces:
  - Entry points (`[project.scripts]`):
    - `lab-cluster-state = "mqro.clusterstate:main"`
    - `lab-nativeha-state = "mqro.nativehastate:main"`
    - `lab-loglifecycle-state = "mqro.loglifecycle:main"`
    - `lab-rdqm-state = "mqro.rdqmstate:main"`
    - `mq-resiliency-observability-selfcheck = "mqro.selfcheck:main"`
  - Unit env files (rendered by T7) and their variables:
    - `lab-cluster-state.env`: `ROLE`
    - `lab-nativeha-state.env`: `QM`
    - `lab-rdqm-state.env`: `QM`

- [ ] **Step 1:** `git mv` the four modules to
  `components/mq-resiliency-observability/src/mqro/`, and their four tests to the
  component's `tests/`. Rewrite imports:
  - `rdqmstate`: `from mqlab.clusterstate import …` becomes `from mqro.clusterstate import …`.
  - `loglifecycle`: **delete** the two-way fallback (`loglifecycle.py:38-47`) and
    replace it with `from mqro import nativehastate`. This also removes the untested
    `# pragma: no cover` branch.
  - The tests: `from mqlab import X` becomes `from mqro import X`.
- [ ] **Step 2: `pyproject.toml`:**

  ```toml
  [project]
  name = "mq-resiliency-observability"
  version = "0.1.0"
  requires-python = "==3.14.*"
  dependencies = []

  [project.scripts]
  lab-cluster-state = "mqro.clusterstate:main"
  lab-nativeha-state = "mqro.nativehastate:main"
  lab-loglifecycle-state = "mqro.loglifecycle:main"
  lab-rdqm-state = "mqro.rdqmstate:main"
  mq-resiliency-observability-selfcheck = "mqro.selfcheck:main"

  [dependency-groups]
  dev = ["pytest", "pytest-cov", "ruff"]

  [build-system]
  requires = ["hatchling"]
  build-backend = "hatchling.build"

  [tool.hatch.build.targets.wheel]
  packages = ["src/mqro"]

  [tool.ruff]
  target-version = "py314"
  line-length = 100

  [tool.pytest.ini_options]
  testpaths = ["tests"]
  addopts = "--cov=mqro --cov-branch --cov-report=term-missing"
  ```

  The version starts at `0.1.0`, the same as the retired `mqro` repo's `VERSION`, so
  M3's reconciliation starts from a shared baseline. Then run
  `uv --directory components/mq-resiliency-observability lock`.
- [ ] **Step 3: Write the failing `tests/test_selfcheck.py`:**

  ```python
  from mqro import selfcheck

  def test_selfcheck_imports_every_module(capsys):
      assert selfcheck.main([]) == 0
      out = capsys.readouterr().out
      for mod in ("clusterstate", "nativehastate", "rdqmstate", "loglifecycle"):
          assert f"mqro.{mod}" in out

  def test_selfcheck_fails_on_wrong_minor(monkeypatch):
      monkeypatch.setattr(selfcheck.sys, "version_info", (3, 12, 3, "final", 0))
      assert selfcheck.main([]) == 1

  def test_selfcheck_fails_loud_on_import_error(monkeypatch, capsys):
      monkeypatch.setattr(selfcheck, "_modules", lambda: ["mqro.nope"])
      assert selfcheck.main([]) == 1
      assert "mqro.nope" in capsys.readouterr().err
  ```

- [ ] **Step 4: Implement `src/mqro/selfcheck.py`:**

  ```python
  """Prove this install can import everything on the interpreter it runs on (spec §5.4 r4)."""
  from __future__ import annotations

  import importlib
  import importlib.metadata
  import pkgutil
  import sys

  import mqro

  REQUIRED_MINOR = (3, 14)

  def _modules() -> list[str]:
      return [m.name for m in pkgutil.walk_packages(mqro.__path__, "mqro.")]

  def main(argv: list[str] | None = None) -> int:
      if tuple(sys.version_info[:2]) != REQUIRED_MINOR:
          print(f"selfcheck: interpreter {sys.version.split()[0]} is not CPython "
                f"{REQUIRED_MINOR[0]}.{REQUIRED_MINOR[1]}", file=sys.stderr)
          return 1
      failed = 0
      for name in _modules():
          try:
              importlib.import_module(name)
              print(f"ok {name}")
          except Exception as exc:  # report every failure, then fail
              print(f"FAIL {name}: {exc!r}", file=sys.stderr)
              failed += 1
      ver = importlib.metadata.version("mq-resiliency-observability")
      print(f"mq-resiliency-observability {ver} on {sys.version.split()[0]}")
      return 1 if failed else 0
  ```

  `except Exception` here is not swallowing: every failure is printed and the exit
  code is 1.

- [ ] **Step 5: Give each collector module a `main()`** for the entry points, if it
  doesn't already have one. Wrap the existing `if __name__ == "__main__":` body. Then
  confirm the moved tests still pass:
  `uv --directory components/mq-resiliency-observability run --frozen pytest`.
- [ ] **Step 6: Static units** in `systemd/`, for example `lab-cluster-state.service`:

  ```ini
  # Component-owned, static (epic .github#294 spec §5.4 r3). Site values come from the
  # deployer-owned EnvironmentFile; never edit this file on a host.
  [Unit]
  Description=Render lab_cluster_state textfile (mq-resiliency-observability)

  [Service]
  Type=oneshot
  EnvironmentFile=/etc/opt/logical-minds-foundry/mq-resiliency-observability/lab-cluster-state.env
  ExecStart=/opt/logical-minds-foundry/mq-resiliency-observability/venv/bin/lab-cluster-state --role ${ROLE}
  ```

  - `lab-nativeha-state.service` keeps **both** `ExecStart` lines (`lab-nativeha-state
    --qm ${QM}` and `lab-loglifecycle-state --qm ${QM}`; #810) and reads
    `lab-nativeha-state.env`.
  - `lab-rdqm-state.service` uses `ExecStart=…/venv/bin/lab-rdqm-state --qm ${QM}`,
    with no `PYTHONPATH`.
  - Copy the three `.timer` files verbatim from the current `.timer.j2` templates
    (they have no Jinja variables; confirm each with `grep '{{'`).
- [ ] **Step 7:** Add `components: [mq-resiliency-observability]` to the four roles in
  `lab/versions.yaml`.
- [ ] **Step 8:** `mqlab component build mq-resiliency-observability` (requires T4)
  succeeds. `vrg-container-run -- vrg-validate` passes at 100% with the moved modules
  gone from `src/mqlab`, and the boundary test (T3) now bites.
- [ ] **Step 9:** `vrg-commit --type refactor --scope components --message "carve mq-resiliency-observability out of src/mqlab"`.
  The PR body states that **develop does not provision collectors until T7**.

---

## Task T7: Rewire the collector roles and bakes (chain)

**Files:**

- Modify: `ansible/roles/{cluster-state,nativeha-state,rdqm-state}/tasks/main.yml`
- Delete: `ansible/roles/{cluster-state,nativeha-state,rdqm-state}/templates/*.j2`
- Create: `ansible/roles/{cluster-state,nativeha-state,rdqm-state}/templates/<unit>.env.j2`
- Modify: `ansible/bake-pcmk-ubuntu.yml`, `bake-san.yml`, `bake-nativeha-ubuntu.yml`,
  `bake-nativeha-rhel.yml`, `bake-mq-rdqm.yml` (include `runtime-install` +
  `component-install` with `component_name: mq-resiliency-observability`)
- Create: `tests/test_collector_roles_rewired.py`

**Interfaces:**

- Consumes:
  - the T5 roles;
  - the variables `cluster_state_role`, `nativeha_qm` and `rdqm_qm` (unchanged in
    `ansible/observability.yml:26-53`);
  - the unit and env names from T6.

- [ ] **Step 1: Write the failing structural tests:**
  - `test_rewired_roles_remove_etc_unit_copies`: each role has a task with
    `state: absent` over `/etc/systemd/system/lab-<x>-state.{service,timer}`,
    followed by `daemon_reload`.
  - `test_rewired_roles_remove_old_payloads`: the absent-list covers
    `/usr/local/bin/lab-cluster-state`, `/usr/local/bin/lab-nativeha-state`,
    `/usr/local/bin/nativehastate.py`, `/usr/local/bin/lab-loglifecycle-state` and
    `/usr/local/lib/lab-rdqm-state`.
  - `test_rewired_roles_never_copy_source`: no task `src:` contains `src/mqlab` or
    `clients/`.
  - `test_collector_roles_assert_baked_component`: the first task asserts
    `/opt/logical-minds-foundry/mq-resiliency-observability/INSTALLED.json` exists,
    with a `fail_msg` naming `mqlab box build`.
  - `test_bakes_install_observability_component`: each of the five bake playbooks
    includes `runtime-install` and then `component-install` with
    `component_name: mq-resiliency-observability`.
- [ ] **Step 2: Rewrite `cluster-state/tasks/main.yml`:**

  ```yaml
  # Per-run half (epic .github#294): the component is BAKED; this only renders the
  # deployer-owned env, retires the pre-#294 payloads, and enables the timer.
  - name: look for the baked observability component
    ansible.builtin.stat: {path: /opt/logical-minds-foundry/mq-resiliency-observability/INSTALLED.json}
    register: _baked
  - name: the observability component is baked on this box
    ansible.builtin.assert:
      that: _baked.stat.exists
      fail_msg: >-
        mq-resiliency-observability is not baked on {{ inventory_hostname }} — rebuild the
        box (mqlab box build <box>) or install it live (mqlab component install
        mq-resiliency-observability --host {{ inventory_hostname }})
  - name: retire pre-#294 payloads and /etc unit copies (they shadow /usr/lib units)
    become: true
    ansible.builtin.file: {path: "{{ item }}", state: absent}
    loop:
      - /usr/local/bin/lab-cluster-state
      - /etc/systemd/system/lab-cluster-state.service
      - /etc/systemd/system/lab-cluster-state.timer
  - name: render the env file
    become: true
    ansible.builtin.template:
      src: lab-cluster-state.env.j2
      dest: /etc/opt/logical-minds-foundry/mq-resiliency-observability/lab-cluster-state.env
      mode: "0644"
  - name: daemon-reload, enable + start the timer
    become: true
    ansible.builtin.systemd: {name: lab-cluster-state.timer, enabled: true, state: started, daemon_reload: true}
  ```

  `lab-cluster-state.env.j2` contains `ROLE={{ cluster_state_role }}`. The parent
  directory `/etc/opt/logical-minds-foundry/mq-resiliency-observability/` is created
  by `component-install` (T5) at bake time.
- [ ] **Step 3:** Apply the same shape to `nativeha-state`:
  - the env is `QM={{ nativeha_qm | default('NHARCAPP') }}`;
  - the absent-list also covers `/usr/local/bin/lab-nativeha-state`,
    `/usr/local/bin/nativehastate.py`, `/usr/local/bin/lab-loglifecycle-state` and
    the `/etc` units.

  And to `rdqm-state`:
  - the env is `QM={{ rdqm_qm | default('RDQMAPP') }}`;
  - the absent-list adds `/usr/local/lib/lab-rdqm-state` and its `/etc` units.
- [ ] **Step 4:** Add to each of the five bake playbooks, after the MQ install role
  and before the hygiene plays:

  ```yaml
  - name: pinned guest runtime (epic .github#294)
    ansible.builtin.include_role: {name: runtime-install}
  - name: observability component (baked, units inert)
    ansible.builtin.include_role: {name: component-install}
    vars: {component_name: mq-resiliency-observability}
  ```

- [ ] **Step 5:** `vrg-container-run -- vrg-validate`. Expected: PASS at 100%.
- [ ] **Step 6:** `vrg-commit --type feat --scope ansible --message "collectors run from the baked mq-resiliency-observability component"`.

---

## Task T8: Carve out `mq-resiliency-clients` (chain)

**Files:**

- Create: `components/mq-resiliency-clients/` with:
  - `pyproject.toml`
  - `uv.lock`
  - `README.md`
  - `src/mqrc/`, holding `app_requester.py`, `bench_client.py`, `svc_responder.py`,
    `authz_probe.py`, `dlq_probe.py`, `reconnect_probe.py`, `dr_flow.py`,
    `dr_responder.py`, `dr_mqi.py`, `dr_baseline.py`, `dr_forced.py`, `header.py`,
    `dr/` (from `src/mqlab/dr/`) and `selfcheck.py`
  - `tests/`
  - `systemd/`, holding `mq-app-requester.service` and `mq-svc-responder@.service`
- Delete: `clients/`, `src/mqlab/dr/`, `src/mqlab/header.py`, and their tests under
  `tests/` (moved)
- Modify: `lab/versions.yaml` (`components: [mq-resiliency-clients]` on `mq-client`)

**Interfaces:**

- Produces:
  - Entry points:
    - `mq-app-requester = "mqrc.app_requester:main"`
    - `mq-bench = "mqrc.bench_client:main"`
    - `mq-svc-responder = "mqrc.svc_responder:main"`
    - `mq-authz-probe = "mqrc.authz_probe:main"`
    - `mq-dlq-probe = "mqrc.dlq_probe:main"`
    - `mq-reconnect-probe = "mqrc.reconnect_probe:main"`
    - `mq-dr-flow = "mqrc.dr_flow:main"`
    - `mq-dr-responder = "mqrc.dr_responder:main"`
    - `mq-dr-baseline = "mqrc.dr_baseline:main"`
    - `mq-dr-forced = "mqrc.dr_forced:main"`
    - `mq-resiliency-clients-selfcheck = "mqrc.selfcheck:main"`
  - Env files (rendered by T9) and their variables:
    - `mq-app-requester.env`: `QM`, `CONN`, `CHANNEL`, `RATE`, `MSG_SIZE`,
      `TEXTFILE` and `TLS_ARGS` (empty, or `--keyrepo … --certlabel …`)
    - `mq-svc-responder.env`: `QM`, `CHANNEL`, `CONN`, `KEYREPO` and `CERTLABEL`

- [ ] **Step 1:** `git mv` the client modules into `src/mqrc/`, `src/mqlab/dr` into
  `src/mqrc/dr`, and `src/mqlab/header.py` into `src/mqrc/header.py`. Move
  `tests/test_{app_requester,bench_client,authz_probe,dlq_probe,header,dr_*}.py` into
  the component's `tests/`. Rewrite imports:
  - `import dr_mqi` / `import app_requester` (sibling imports) become
    `from mqrc import dr_mqi` / `from mqrc import app_requester`;
  - `mqlab.dr` / `mqlab.header` become `mqrc.dr` / `mqrc.header`.

  Keep `tests/test_responder_venv_baked.py` in mqlab for now; T9 rewrites it.
- [ ] **Step 2: `pyproject.toml`.** Same shape as T6, with these differences:

  ```toml
  [project]
  name = "mq-resiliency-clients"
  version = "0.1.0"
  requires-python = "==3.14.*"
  dependencies = []

  [project.optional-dependencies]
  mqi = ["pymqi==1.12.13"]   # compiled on the guest at install; NEVER on the dev VM

  [dependency-groups]
  dev = ["pytest", "pytest-cov", "ruff"]
  # Build requirements of our sdist deps (pymqi has no [build-system]; its setup.py
  # imports distutils, removed in 3.12, so setuptools' shim must be present). Exported
  # by `mqlab component build` to build-requirements.txt. Versions from the T2 spike.
  sdist-build = ["setuptools==<T2 row 4>", "wheel==<T2 row 4>"]
  ```

  Replace `<T2 row 4>` with the exact versions T2 recorded **before** running
  `uv --directory components/mq-resiliency-clients lock`. This step is blocked on T2's
  report, not a placeholder.
- [ ] **Step 3:** Each client module gets a `main(argv: list[str] | None = None) -> int`
  if it lacks one. Keep pymqi imports **lazy** (inside functions), as
  `app_requester.run()` already does (`tests/test_app_requester.py:3-4`). That means
  moving `dr_mqi.py:45`'s top-level `import pymqi` into its functions, so the tests and
  the selfcheck's module walk can run without the extra.
- [ ] **Step 4: Failing tests:**
  - `tests/test_svc_responder.py` (new; the responder had none). Use a fake `pymqi`
    module in the style of `test_app_requester.py:109`'s `_fake_pymqi`. Cover: one
    request on `--in-queue` produces one reply to the request's reply-to queue with
    the correlation id copied; `--seconds` expiry exits 0; an MQ error is logged and
    exits non-zero.
  - `tests/test_selfcheck.py`: as in T6, plus
    `test_selfcheck_requires_pymqi_when_installed_with_mqi`. With
    `importlib.metadata.requires` reporting the `mqi` extra as installed and a missing
    `pymqi`, the result is 1, with `pymqi` in stderr.
- [ ] **Step 5: `selfcheck.py`** is the T6 implementation with `mqro` replaced by
  `mqrc`, plus a final check:

  ```python
  def _mqi_installed() -> bool:
      try:
          importlib.metadata.version("pymqi")
      except importlib.metadata.PackageNotFoundError:
          return False
      return True
  # in main(): if _mqi_installed(): import pymqi (the real binding) — failure => exit 1
  ```

  On guests the `mqi` extra is installed, so the real binding is proven. On the dev
  VM it's absent, so it's skipped.
- [ ] **Step 6: Units:**

  ```ini
  # mq-app-requester.service — component-owned, static (spec §5.4 r3)
  [Unit]
  Description=MQ app requester (steady request/reply load + round-trip textfile)
  After=network-online.target

  [Service]
  User=vagrant
  EnvironmentFile=/etc/opt/logical-minds-foundry/mq-resiliency-clients/mq-app-requester.env
  Environment=LD_LIBRARY_PATH=/opt/mqm/lib64
  ExecStart=/opt/logical-minds-foundry/mq-resiliency-clients/venv/bin/mq-app-requester --qm ${QM} --conn ${CONN} --channel ${CHANNEL} --rate ${RATE} --msg-size ${MSG_SIZE} --textfile ${TEXTFILE} $TLS_ARGS
  Restart=always

  [Install]
  WantedBy=multi-user.target
  ```

  `mq-svc-responder@.service` keeps `User=mqm`, `LD_LIBRARY_PATH`, and the `%i`
  in-queue: `--qm ${QM} --in-queue %i --channel ${CHANNEL} --conn ${CONN}
  --keyrepo ${KEYREPO} --certlabel ${CERTLABEL}`. Copy every other directive
  (`Restart`, `After`, `WantedBy`) verbatim from the current `.j2` templates; diff
  them before committing.
- [ ] **Step 7:** Add `components: [mq-resiliency-clients]` to `mq-client` in
  `lab/versions.yaml`.
- [ ] **Step 8:** `mqlab component build mq-resiliency-clients` succeeds, with no
  pymqi on the dev VM. `vrg-container-run -- vrg-validate` passes at 100%.
- [ ] **Step 9:** `vrg-commit --type refactor --scope components --message "carve mq-resiliency-clients (clients + dr + header) into a component"`.

---

## Task T9: Rewire the client roles, playbooks and `dr-run.sh` (chain end)

**Files:**

- Modify: `ansible/roles/mq-client/tasks/main.yml:40,57-72` (drop `python3-venv`
  from the apt line; keep `python3-dev`/`gcc` only if T2 showed pymqi needs them,
  since the runtime tarball ships its own headers; delete the mqvenv create + pip)
- Modify: `ansible/roles/mq-inter-qm/tasks/{install.yml,main.yml}` (delete the rvenv
  create/pip and the `svc_responder.py` copy; render `mq-svc-responder.env`)
- Modify: `ansible/roles/app-requester/tasks/main.yml` (render the env; delete the
  script copy and the `/etc` unit)
- Modify: `ansible/roles/bench-client/{tasks/main.yml,templates/mq-bench.j2}` (replace
  the wrapper with a `/usr/local/bin/mq-bench` symlink to the venv entry point)
- Modify: `ansible/site-pcmk-authz-validate.yml`,
  `site-nativeha-ubuntu-authz-validate.yml`, `site-rdqm-authz-validate.yml` (probe
  invocations become entry points; drop `probe_src`/`probe_dest` copies)
- Modify: `ansible/site-distributed-shared.yml:57-63` (drop the duplicate shell venv
  create)
- Modify: `ansible/bake-mq-ubuntu.yml` (add `runtime-install` + `component-install`
  for `mq-resiliency-clients`; remove the rvenv bake)
- Modify: `lab/scripts/dr-run.sh:29-39`
- Rewrite: `tests/test_responder_venv_baked.py` → `tests/test_clients_component_baked.py`
- Create: `tests/test_client_roles_rewired.py`

**Interfaces:**

- Consumes: the T5 roles and the T8 entry points, units and env variables.

- [ ] **Step 1: Write the failing structural tests:**
  - `test_client_roles_remove_etc_unit_copies`:
    `/etc/systemd/system/mq-app-requester.service` and
    `/etc/systemd/system/mq-svc-responder@.service` are `state: absent`.
  - `test_no_role_or_playbook_references_mqvenv_or_rvenv`: grep `ansible/` and
    `lab/scripts/` for `mqvenv`, `rvenv`, `clients/` and `app_requester.py`, which
    returns nothing.
  - `test_client_roles_remove_old_payloads`: `/home/vagrant/mqvenv`,
    `/home/vagrant/*.py` (an explicit list), `/var/mqm/rvenv` and
    `/var/mqm/svc_responder.py` are `state: absent`.
  - `test_mq_ubuntu_bake_installs_clients_component`.
  - `test_authz_validate_plays_call_entry_points`: each probe task's command starts
    with `/opt/logical-minds-foundry/mq-resiliency-clients/venv/bin/mq-`.
- [ ] **Step 2:** Rewrite the roles following T7's shape: assert baked, retire old
  payloads and `/etc` units, render the env, then enable/start. For `app-requester`,
  `TLS_ARGS` renders as
  `{% if app_requester_tls | bool %}--keyrepo {{ app_requester_keyrepo }} --certlabel {{ app_requester_certlabel }}{% endif %}`.
- [ ] **Step 3:** In the three authz-validate playbooks, replace each
  `{{ mqvenv }}/bin/python {{ probe_dest }} …` with
  `/opt/logical-minds-foundry/mq-resiliency-clients/venv/bin/mq-authz-probe …`, and
  `dlq_probe.py` with `mq-dlq-probe`. Delete the stage-probe copy tasks
  (`site-pcmk-authz-validate.yml:399-411,616-630` and the matching blocks in the other
  two). Keep every argument verbatim.
- [ ] **Step 4:** In `dr-run.sh`:

  ```bash
  CLIENTS=/opt/logical-minds-foundry/mq-resiliency-clients/venv/bin
  vagrant ssh svc-sim -c \
    "LD_LIBRARY_PATH=/opt/mqm/lib64 $CLIENTS/mq-dr-responder --qm ${QM_SVC:-SVCQM} --conn 'localhost(1414)' \
     --in-queue SVC.REQUEST --out-queue APP.REPLY --seconds $((SECONDS_RUN + 10)) \
     --ledger ~/dr-ledgers/svc.jsonl" &
  ```

  Make the same change for `mq-dr-flow` on app-client. svc-sim now has the component
  (role `mq-client` → `mq-resiliency-clients`), which fixes `#293` inventory row 11.
- [ ] **Step 5:** `vrg-container-run -- vrg-validate`. Expected: PASS at 100%.
- [ ] **Step 6:** `vrg-commit --type feat --scope ansible --message "clients run from the baked mq-resiliency-clients component; retire mqvenv/rvenv"`.
  The PR body states that the T6..T9 chain is now complete and V1/V2 are unblocked.

---

## Operational V1 (validation): Cold rebuild, Ubuntu stacks (arm64 local)

Created with `--kind validation`, `--blocked-by T7`, `--blocked-by T9`.

- **Precondition:** develop contains T3 to T9; the macOS arm64 host is available.
- **Rows** (one cold rebuild each: `mqlab box build --all`, then a bootstrap; the box
  builder builds both components itself):
  - `pcmk-ubuntu` @ ubuntu24
  - `nativeha-ubuntu` @ ubuntu24
- **Checks per row:**
  1. Prometheus has `cluster_*` series from every pcmk/san node, or from every
     nativeha node plus the `lab_loglifecycle_*` series, and the cockpits are
     populated (human dashboard review).
  2. `mqlab component status --host <all>` shows each host's installed tree equal to
     `CURRENT`, and runtime `3.14.8+20261003`.
  3. `systemctl status` shows the collector timers and `mq-app-requester` active from
     `/usr/lib/systemd/system`, with no `/etc/systemd/system` copies.
  4. `mq-bench` runs on app-client.
  5. `lab/scripts/dr-run.sh` completes on app-client **and** svc-sim and collects
     both ledgers.
  6. None of the removed paths exist on any guest: `/usr/local/bin/lab-*-state`,
     `/usr/local/lib/lab-rdqm-state`, `~/mqvenv`, `/var/mqm/rvenv` and
     `~/*_probe.py`.
  7. A deliberately broken component (a temporary `raise` at import, on a scratch
     branch, built with `mqlab component build`, then
     `mqlab component install … --host pcmk-a`) fails at selfcheck, and the
     collector keeps running from `venv`. Revert afterwards and record the output.
- **Acceptance:** both rows green; record `Outcome: SUCCESS`.

## Operational V2 (validation): Cold rebuild, RHEL stacks (x86 cloud)

Created with `--kind validation`, `--blocked-by T7`, `--blocked-by T9`.

- **Precondition:** develop contains T3 to T9; the x86 cloud host is available; the
  RHEL DVD is staged.
- **Rows:** `rdqm-rhel` @ rhel9 and `nativeha-rhel-crr` @ rhel9, plus the shared
  `mq-client` box at x86_64 (app-client, svc-sim).
- **Checks:** V1 checks 1 to 6, plus: the RHEL bakes completed **offline**, with no
  PyPI access from any RHEL guest (`journalctl`/bake log shows only
  `--no-index` installs).
- **Acceptance:** all rows green; record `Outcome: SUCCESS`. Then comment on
  `.github#280` that it is unblocked.

---

## Closing bookends (already filed)

- **Documentation review:** `mq-resiliency-lab-for-linux#1348`, blocked by V1 and V2.
  It's a sweep covering `docs/site`, `box-model.md`, `box-bake-manifest.md`, runbooks,
  and the supersession notes listed in spec §10. It spawns per-repo doc tasks as
  needed.
- **Follow-on brainstorm, M3:** `.github#296`.
- **Follow-on brainstorm, own binary wheels:** `.github#298`.
- **Retrospective:** `.github#297` (terminal).
