# Migrate the resiliency lab from pymqi to ibmmq — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Every MQI client in `mq-resiliency-lab-for-linux` runs on `ibmmq==2.1.1`
in native idiom instead of `pymqi`. No pymqi reference survives outside historical
records, enforced by a guard. The lab is rebaked and cold-rebuilt green on ARM64 and
x86_64.

**Architecture:**

- The work is a **sideways binding swap** inside one package: `mqrc`, under
  `components/mq-resiliency-clients/`.
  - `import pymqi` becomes `import ibmmq`.
  - `MQCHT_*` move to `CMQXC`.
  - MQCHAR input fields take `str`. Payloads and output fields stay `bytes`.
- The component's `mqi` extra re-pins to `ibmmq==2.1.1`. The existing `mqlab
  component build` / `component-install` path already stages the sdist plus the abi3
  x86_64 wheel, so it needs **no logic change**:
  - x86_64 installs the wheel.
  - aarch64 compiles the sdist with the gcc / `CC`/`LDSHARED` /
    `--no-build-isolation` path that #294 built for pymqi.
- Everything else is rewording (Ansible comments, docs), plus two repo tests: a
  mixed-lock staging case and the no-pymqi guard.

**Tech Stack:** Python 3.14 (component, pinned CPython 3.14.8) and mqlab (pytest at
100% branch coverage), uv, hatchling, pip, Ansible, `ibmmq` 2.1.1 (IBM MQ 10.0
client SDK).

**Spec:** `epics/300-pymqi-to-ibmmq/spec.md` (this directory). Executors read both.

**Repos:**

- Implementation, guard and validation: `logical-minds-foundry/mq-resiliency-lab-for-linux`.
- One comment: `logical-minds-foundry/mq-resiliency-observability`.
- This plan, the spec, the outreach and the retrospective: `logical-minds-foundry/.github`.

## Global Constraints

- **Binding pin:** `ibmmq==2.1.1`, exactly, in `[project.optional-dependencies] mqi`
  of `components/mq-resiliency-clients/pyproject.toml`.
- **Build group:** `sdist-build = ["setuptools==84.0.0"]`. `wheel` is dropped (spec
  §4.3). Re-add `wheel==0.48.0` only as a fix-forward if the aarch64 compile in V1
  fails for lack of it.
- **Native idiom:**
  - `import ibmmq` (never `import ibmmq as pymqi`), kept **function-local / lazy**.
  - `ibmmq.CMQXC.MQCHT_CLNTCONN`; every other constant from `ibmmq.CMQC`.
  - `str` for MQCHAR input fields: names, conn names, cipher spec, keystore stem,
    cert label, `UserIdentifier`, `ReplyToQ`/`ReplyToQMgr`.
- **Payloads and output fields stay `bytes`:** message bodies, `b"DRv1|"`,
  `b"x" * n`, `b"REPLY:"`, `MsgId`/`CorrelId`, and MQDLH parsing.
- **No behaviour change:** CLI flags, outputs, exit codes, units, file paths, the
  `.sth` stash and the TLS/plaintext branching all stay as they are. Error *text*
  from ibmmq may differ. Control flow reads only `.reason`.
- **No logic change** to `src/mqlab/component.py` or any Ansible task. Only
  comments and task names change.
- **Guard allowlist (fixed):** `docs/plans/**`, `docs/specs/**`, `docs/reports/**`,
  `CHANGELOG.md`, `releases/**`, plus the guard test file itself.
- **Component tests** run without the `mqi` extra, with fake `ibmmq` modules. The
  dev VM has no MQ SDK.
- **Validation:** `vrg-container-run -- vrg-validate` covers mqlab and `tests/`.
  Component tests are gated by `mqlab component build mq-resiliency-clients`, which
  refuses a dirty tree, so **commit before you run it**. The fast loop
  (not a gate) is:
  `cd components/mq-resiliency-clients && UV_PROJECT_ENVIRONMENT="$(mqlab build path work)/scratch-mqrc-venv" uv run --frozen pytest -q`.
- **Commits:** `vrg-commit --type <t> --scope <s> --message <m>`. Git operations
  use `vrg-git`.

## Review Focus

1. **A constant read from the wrong namespace on the real binding.**
   `ibmmq.CMQC.MQCHT_CLNTCONN` does not exist, so `authz_probe`/`dlq_probe` would
   raise `AttributeError` on the guest even though a lenient fake passes. The fakes
   must put `MQCHT_CLNTCONN` **only** in `CMQXC`, and a test must assert the value
   lands in `cd.ChannelType`. Pinned by
   `test_probe_sets_clntconn_channel_type_from_cmqxc` in `test_authz_probe.py` and
   `test_dlq_probe.py` (Task T2, step 3).
2. **A leftover `pymqi` on a path no unit test drives,** such as
   `reconnect_probe.probe()` or a docstring. That would mean `ModuleNotFoundError`
   on the guest, or a broken definition of done. Pinned by the repo guard
   `test_no_pymqi_reference_outside_history` (Task T2, step 9), which scans every
   tracked file.
3. **The regenerated lock drifts:** a different ibmmq version, a missing x86_64
   wheel (so x86 silently compiles), or OpenTelemetry packages pulled in (which
   would switch on ibmmq's OTel propagation, a behaviour change). Pinned by
   `test_clients_lock_pins_ibmmq_wheel_and_sdist_without_otel` (Task T2, step 8).
4. **A binding that imports but fails in a new way.** ibmmq raises
   `EnvironmentError` (an `OSError`) on a C/Python version mismatch, not
   `ImportError`. selfcheck must report `FAIL ibmmq` and exit 1, never crash or
   pass. Pinned by `test_selfcheck_fails_on_a_binding_version_mismatch` (Task T2,
   step 6).
5. **MQCHAR fields set after construction** (`cd.SSLCipherSpec = …`,
   `sco.KeyRepository = …`, `md.UserIdentifier = …`) must carry `str`, not a stale
   `.encode()`. The updated assertions pin each one. ibmmq accepts both, so only the
   tests catch idiom drift (Task T2, steps 2–5).

## Task dependency graph

```text
T1  docs: spec + plan (.github#301)                        [—]
T2  big-bang migration (lab#1360)                          [T1]
T3  obs pyproject comment (obs#42)                         [—]
T4  outreach: aarch64 wheel (.github#302)                  [—]
V1  validation: rebake + cold rebuild, ARM64 + x86_64
    (lab#1362)                                             [T2, T3]
Closing  docs review (lab#1361) [V1] · follow-on brainstorm (.github#303) [V1]
         · retrospective (.github#304) [all]
```

T2, T3 and T4 can run in parallel once T1 merges. T3 and T4 do not need T1.

---

## Task T1: Spec + plan (`.github#301`)

This document and `spec.md`, published in one docs PR to `logical-minds-foundry/.github`.
On publish:

- Rewrite the bodies of `lab#1360` and `lab#1362` to point at this plan. They were
  filed against the pre-#294 layout.
- Refresh the epic body (`.github#300`) for the component layout.
- Comment on the epic that it has resumed.

---

## Task T2: Big-bang migration (`mq-resiliency-lab-for-linux#1360`)

**Files:**

- Modify (component source, `components/mq-resiliency-clients/src/mqrc/`):
  `app_requester.py`, `bench_client.py`, `svc_responder.py`, `authz_probe.py`,
  `dlq_probe.py`, `reconnect_probe.py`, `dr_mqi.py`, `dr_flow.py`,
  `dr_responder.py`, `selfcheck.py`, `dr/snapshot.py`.
- Modify (component tests, `components/mq-resiliency-clients/tests/`):
  `test_app_requester.py`, `test_bench_client.py`, `test_svc_responder.py`,
  `test_authz_probe.py`, `test_dlq_probe.py`, `test_dr_entry_points.py`,
  `test_selfcheck.py`.
- Modify (component packaging): `components/mq-resiliency-clients/pyproject.toml`,
  `components/mq-resiliency-clients/uv.lock` (regenerated),
  `components/mq-resiliency-clients/README.md`, `components/README.md`.
- Modify (observability component): `components/mq-resiliency-observability/src/mqro/loglifecycle.py:6`.
- Modify (Ansible, comments and task names only):
  - `ansible/bake-mq-ubuntu.yml:51,62`
  - `ansible/site-distributed-shared.yml:40`
  - `ansible/site-pcmk-authz-validate.yml:42`
  - `ansible/roles/component-install/tasks/main.yml:106`
  - `ansible/roles/mq-client/tasks/main.yml:65-68,97-99`
  - `ansible/roles/mq-install/tasks/main.yml:51`
  - `ansible/roles/mq-inter-qm/templates/their-side.mqsc.j2:7-8`
  - `ansible/roles/pki-distribute/defaults/main.yml:9-10`
  - `ansible/roles/pki-distribute/tasks/main.yml:40-42`
- Modify (tools): `tools/validate-event-monitor-wrapper.sh:6`.
- Modify (repo tests): `tests/test_component.py`,
  `tests/test_version_token_guardrail.py:147`, plus wording only in
  `tests/test_client_roles_rewired.py`, `tests/test_clients_component_baked.py` and
  `tests/test_install_roles.py`.
- Create (repo tests): `tests/test_no_pymqi_refs.py`, `tests/test_clients_lock.py`.
- Modify (living docs):
  - `docs/site/docs/operate/guest-components.md:17,36,120,161`
  - `docs/site/docs/design-and-specs.md:30`
  - `docs/development/box-model.md:37,189`
  - `docs/development/box-bake-manifest.md:367,369`
  - `docs/development/operating-the-lab-from-a-dev-session.md:140-206`

**Interfaces:**

- Consumes: nothing new. `mqlab component build` / `stage_deps` /
  `wheel_installable` are used unchanged.
- Produces, for V1:
  - The component's `mqi` extra resolves to `ibmmq==2.1.1`.
  - `mq-resiliency-clients-selfcheck` prints `ok ibmmq <version>` on a guest with
    the extra.
  - No entry point names, flags or units change.

The steps below are ordered tests-first per area. Commit at each **Commit** step. The
component gate (`mqlab component build`) needs a clean tree.

- [ ] **Step 1: Worktree.** From the project root:
  `vrg-git worktree add .worktrees/issue-1360-pymqi-to-ibmmq -b feature/1360-pymqi-to-ibmmq origin/develop`.
  Then `cd` into it. All paths below are relative to that worktree.

- [ ] **Step 2: Rename every fake and flip the MQCHAR assertions to `str` (red).**
  In each of the seven component test files:
  - `types.ModuleType("pymqi")` → `types.ModuleType("ibmmq")`.
  - `monkeypatch.setitem(sys.modules, "pymqi", …)` → `… "ibmmq", …`.
  - Rename the helper `_fake_pymqi` → `_fake_ibmmq`.
  - Docstrings saying "fake pymqi" → "fake ibmmq".
  - In `test_authz_probe.py:64`, change `PYIFError("PYMQI Error: not connected")`
    to `PYIFError("not connected")`. Change the `"pymqi's own error"` docstrings
    (`test_authz_probe.py:30`, `test_dlq_probe.py:59`) to `"the binding's own error"`.

  Then change these **input-field** assertions from bytes to `str`:

  ```python
  # tests/test_dr_entry_points.py
  assert qmgr.cd.ConnectionName == "vip(1414)"                  # :99
  assert qmgr.cd.SSLCipherSpec == "ANY_TLS13_OR_HIGHER"         # :132, :141, :159
  assert qmgr.sco.KeyRepository == "/home/vagrant/ssl/key"      # :133
  assert qmgr.sco.CertificateLabel == "app-client"              # :134
  assert qmgr.cd.ChannelName == "APP.SVRCONN"                   # :135
  assert qmgr.sco.KeyRepository == "/k/key"                     # :142, :160
  assert qmgr.sco.CertificateLabel == "lbl"                     # :161

  # tests/test_svc_responder.py
  assert (name, cd.ChannelName, cd.ConnectionName) == (
      "SVCQM",
      "SVC.SVRCONN",
      "localhost(1414)",
  )                                                             # :140-145
  assert cd.SSLCipherSpec == "ANY_TLS13_OR_HIGHER"              # :166
  assert sco.KeyRepository == "/var/mqm/ssl/k"                  # :167
  assert sco.CertificateLabel == "svc"                          # :168

  # tests/test_authz_probe.py
  assert state["put_uid"] == "mqm"                              # :157
  assert state["put_uid"] == "someone"                          # :167
  ```

  **Leave these as bytes; they are outputs or payloads:**
  - `test_svc_responder.py:123-129`: request tuple, `od.ObjectName`/`ObjectQMgrName`
    (passed straight from the request MD), `md.CorrelId`, `md.Format`, and the
    `b"REPLY:hello"` body.
  - Every `MsgId`, the `MQFMT_STRING = b"MQSTR   "` fake constants, and all MQDLH
    bytes in `test_dlq_probe.py`.

  Run the fast loop. Expected: **FAIL**, with `ModuleNotFoundError: No module named
  'pymqi'` from every lazy import, plus the bytes/str assertion mismatches.

- [ ] **Step 3: Move `MQCHT_CLNTCONN` to `CMQXC` in the probe fakes and pin it (red).**
  In `test_authz_probe.py` and `test_dlq_probe.py`, delete `MQCHT_CLNTCONN = 6` from
  the fake `class CMQC`. Add a sibling class and register it:

  ```python
  class CMQXC:
      MQCHT_CLNTCONN = 6
  ```

  Add `"CMQXC": CMQXC,` to the `setattr` dict in `test_authz_probe.py:90-101`. In
  `test_dlq_probe.py`, add it the same way, next to where its fake registers `CMQC`.
  Neither fake's `connect_with_options` (`test_authz_probe.py:54`,
  `test_dlq_probe.py:78`) records its arguments yet. In each, add
  `state["connect"] = (_name, cd, sco)` as the first line.

  Then add one test to each file. Here is the authz version. In `test_dlq_probe.py`,
  use the state shape and the `_args(reason, destq)` helper (`:112`) the same way its
  cases at `:134-157` do. Call that file's probe entry point the way those cases do:

  ```python
  def test_probe_sets_clntconn_channel_type_from_cmqxc(monkeypatch):
      state: dict = {}
      fake = _fake_ibmmq(state)
      monkeypatch.setitem(sys.modules, "ibmmq", fake)
      ap.probe(_args([]))
      assert state["connect"][1].ChannelType == fake.CMQXC.MQCHT_CLNTCONN
      assert not hasattr(fake.CMQC, "MQCHT_CLNTCONN")
  ```

  Run the fast loop. Expected: **FAIL**, because the modules still read
  `CMQC.MQCHT_CLNTCONN`.

- [ ] **Step 4: Migrate the client modules (green).** Apply this mapping to every
  listed line. Keep each `import` exactly where it is, inside its function.

  | File:line(s) | From | To |
  |---|---|---|
  | all `import pymqi` (`app_requester:197`, `bench_client:270`, `svc_responder:49,107`, `authz_probe:80`, `dlq_probe:90`, `reconnect_probe:23`, `dr_mqi:76,108`, `dr_flow:53,133`, `dr_responder:38,96`) | `import pymqi` | `import ibmmq` |
  | every other `pymqi.` attribute access in those files | `pymqi.X` | `ibmmq.X` |
  | `authz_probe:85`, `dlq_probe:95` | `pymqi.CMQC.MQCHT_CLNTCONN` | `ibmmq.CMQXC.MQCHT_CLNTCONN` |
  | `app_requester:200-201`, `bench_client:273-274`, `svc_responder:53-54`, `authz_probe:83-84`, `dlq_probe:93-94`, `dr_mqi:79-80` | `ChannelName=<x>.encode(), ConnectionName=<y>.encode()` | `ChannelName=<x>, ConnectionName=<y>` |
  | `reconnect_probe:26-27` | `ChannelName=b"APP.SVRCONN", ConnectionName=conn.encode()` | `ChannelName="APP.SVRCONN", ConnectionName=conn` |
  | `app_requester:208`, `bench_client:279`, `svc_responder:59`, `dr_mqi:87` | `cd.SSLCipherSpec = b"ANY_TLS13_OR_HIGHER"` | `cd.SSLCipherSpec = "ANY_TLS13_OR_HIGHER"` |
  | `authz_probe:88`, `dlq_probe:98` | `cd.SSLCipherSpec = args.cipher.encode()` | `cd.SSLCipherSpec = args.cipher` |
  | `app_requester:209`, `bench_client:280`, `dr_mqi:88` | `sco.KeyRepository = <k>.encode()` | `sco.KeyRepository = <k>` |
  | `svc_responder:60`, `authz_probe:89`, `dlq_probe:99` | `SCO(KeyRepository=<k>.encode())` | `SCO(KeyRepository=<k>)` |
  | `app_requester:211`, `bench_client:282`, `svc_responder:62`, `authz_probe:91`, `dlq_probe:101`, `dr_mqi:90` | `sco.CertificateLabel = <c>.encode()` | `sco.CertificateLabel = <c>` |
  | `authz_probe:105,109` | `md.UserIdentifier = args.as_user.encode()` | `md.UserIdentifier = args.as_user` |
  | `app_requester:251-252` | `ReplyToQ=args.reply_queue.encode(), ReplyToQMgr=args.qm.encode()` | `ReplyToQ=args.reply_queue, ReplyToQMgr=args.qm` |
  | `authz_probe:128,133`, `dlq_probe:126,131` | `except pymqi.MQMIError, pymqi.PYIFError:` | `except ibmmq.MQMIError, ibmmq.PYIFError:` |

  **Do not touch** these payload/output lines:
  - `dr_flow:48,77`, `dr_responder:62-70`, `svc_responder:93-94,99`
  - `app_requester:238,258,262`, `bench_client:284`, `authz_probe:111`
  - `reconnect_probe:38`, `dlq_probe:35-52`, `dr/wire.py`

  Then reword the comments and docstrings, keeping their meaning:
  - `app_requester:204-205` and `dr_mqi:83-84` ("a None sco crashes"): replace with
    `# One code path for TLS and plaintext: an empty SCO on the plaintext path, TLS just sets its fields (#442).`
    Keep `sco = ibmmq.SCO()`.
  - `app_requester:187-188`, `dr_flow:233`, `dr_responder:107` and `dr_mqi:72`:
    `"(pymqi's SCO has none)"` → `"(the clients use the .sth stash; SCO.KeyRepoPassword adoption is deferred, epic .github#300 §9)"`.
  - `svc_responder:41-43`: `"pip-installed pymqi links the client library only, so a true bindings connect fails 2058 (#180)"` →
    `"the responder deliberately connects client-mode over the TLS SVC.SVRCONN (#180); passing an MQCD makes ibmmq connect as a client"`.
  - Every remaining docstring or comment mention (`app_requester:47,195`,
    `bench_client:168-169,268`, `authz_probe:11-13,26-28,122`, `dlq_probe:11-15,121`,
    `dr_flow:26`, `dr_mqi:41-43`, `dr_responder:18`, `reconnect_probe:1,9`,
    `svc_responder:17`): `pymqi` → `ibmmq`. Where the text says "pymqi's own
    error", write "the binding's own error".
  - `dr/snapshot.py:6`: read the module. Replace the sentence naming
    `browse_queue()` with one describing what the module actually does: it parses
    bodies only and has no MQI.

  Run the fast loop. Expected: **PASS** for steps 2–3.

- [ ] **Step 5: Bench reason names.** `bench_client:374` becomes
  `reason_names(ibmmq.CMQC)`. In the `reason_names` docstring (`:168-169`), say it
  reads an `ibmmq.CMQC` namespace. No logic change: ibmmq's `CMQC` star-imports the
  platform constants, so `dir()` still lists `MQRC_*`. The fast loop passes.

- [ ] **Step 6: selfcheck (test, then code).** Replace the pymqi parts of
  `tests/test_selfcheck.py`:

  ```python
  def _ibmmq_dist(installed: bool):
      """A stand-in for importlib.metadata.version that reports (or denies) an ibmmq
      distribution, passing every other name through."""
      real = importlib.metadata.version

      def version(name: str) -> str:
          if name == "ibmmq":
              if installed:
                  return "2.1.1"
              raise importlib.metadata.PackageNotFoundError(name)
          return real(name)

      return version


  def test_selfcheck_skips_ibmmq_without_the_mqi_extra(monkeypatch, capsys):
      monkeypatch.setattr(importlib.metadata, "version", _ibmmq_dist(installed=False))
      assert selfcheck.main([]) == 0
      assert "ibmmq not installed (no mqi extra): real-binding check skipped" in capsys.readouterr().out


  def test_selfcheck_requires_ibmmq_when_installed_with_mqi(monkeypatch, capsys):
      monkeypatch.setattr(importlib.metadata, "version", _ibmmq_dist(installed=True))
      monkeypatch.setitem(sys.modules, "ibmmq", None)  # distribution present, binding broken
      assert selfcheck.main([]) == 1
      assert "FAIL ibmmq" in capsys.readouterr().err


  def test_selfcheck_fails_on_a_binding_version_mismatch(monkeypatch, capsys):
      monkeypatch.setattr(importlib.metadata, "version", _ibmmq_dist(installed=True))
      monkeypatch.delitem(sys.modules, "ibmmq", raising=False)
      real_import = importlib.import_module

      def import_module(name, *a, **kw):
          if name == "ibmmq":
              raise OSError("Mismatch between C '2.1.0' and Python '2.1.1' package versions")
          return real_import(name, *a, **kw)

      monkeypatch.setattr(importlib, "import_module", import_module)
      assert selfcheck.main([]) == 1
      assert "FAIL ibmmq: OSError" in capsys.readouterr().err


  def test_selfcheck_imports_the_real_ibmmq_when_installed(monkeypatch, capsys):
      monkeypatch.setattr(importlib.metadata, "version", _ibmmq_dist(installed=True))
      monkeypatch.setitem(sys.modules, "ibmmq", types.ModuleType("ibmmq"))
      assert selfcheck.main([]) == 0
      assert "ok ibmmq 2.1.1" in capsys.readouterr().out
  ```

  Keep the file's other tests and its imports. Before replacing the existing
  `_pymqi_dist` (`:32-39`), read it and keep its exact pass-through shape. Where the
  existing tests call `main` with a different signature, follow it. The fast loop
  is red.

  Then, in `src/mqrc/selfcheck.py`:
  - Add a module constant `BINDING = "ibmmq"`.
  - Replace the three `"pymqi"` literals (`:30,54,55,57,60`) with `BINDING` /
    f-strings: `f"ok {BINDING} {importlib.metadata.version(BINDING)}"`,
    `f"FAIL {BINDING}: {exc!r}"`, and
    `f"{BINDING} not installed (no mqi extra): real-binding check skipped"`.
  - Rewrite the module docstring's last two sentences as:
    `When the mqi extra is installed (the ibmmq distribution is present, as on every guest), also import the real compiled binding, so a broken build or a missing libmqm_r.so fails the install instead of the first client run. Versions come from importlib.metadata.`

  The existing `except Exception` already catches `OSError`. The fast loop is green.

- [ ] **Step 7: Packaging.** In `components/mq-resiliency-clients/pyproject.toml`:

  ```toml
  [project.optional-dependencies]
  # IBM's maintained MQI binding. Installed on the guest: the abi3 wheel on x86_64,
  # compiled from sdist against the box's MQ SDK on aarch64 (no upstream wheel yet,
  # .github#302). NEVER on the dev VM (no MQ SDK): tests use a fake ibmmq module.
  mqi = ["ibmmq==2.1.1"]
  ```

  ```toml
  # Build requirements of our sdist deps, pre-installed because guest installs are
  # offline with --no-build-isolation: ibmmq's sdist declares setuptools>=75 as its
  # build backend (needed on aarch64, where there is no wheel). Exported by
  # `mqlab component build` to build-requirements.txt.
  sdist-build = ["setuptools==84.0.0"]
  ```

  Then, in `components/mq-resiliency-clients`, run `uv lock`. Inspect the diff:
  - `pymqi`, `wheel` and its `packaging` dependency are gone.
  - `ibmmq 2.1.1` has one `sdist` and three `wheels` (`manylinux_2_28_x86_64`,
    `macosx_11_0_arm64`, `win_amd64`).
  - No `opentelemetry-*` package appears.

  `uv lock --check` passes.

- [ ] **Step 8: Lock guard (repo test).** Create `tests/test_clients_lock.py`:

  ```python
  """The clients component's lock pins the MQI binding exactly (epic .github#300 spec §4.3).

  x86_64 guests must install ibmmq's abi3 wheel and aarch64 guests compile its sdist,
  so the lock must carry BOTH artifacts. No OpenTelemetry package may enter the lock:
  ibmmq switches on OTel propagation automatically when OTel imports succeed, which is
  a behaviour change this epic defers (spec §1.2, §3).
  """

  from __future__ import annotations

  import tomllib

  from mqlab.paths import repo_root

  LOCK = repo_root() / "components" / "mq-resiliency-clients" / "uv.lock"


  def _packages() -> dict[str, dict]:
      return {p["name"]: p for p in tomllib.loads(LOCK.read_text())["package"]}


  def test_clients_lock_pins_ibmmq_wheel_and_sdist_without_otel():
      pkgs = _packages()
      assert "pymqi" not in pkgs
      ibmmq = pkgs["ibmmq"]
      assert ibmmq["version"] == "2.1.1"
      assert ibmmq["sdist"]["url"].endswith("/ibmmq-2.1.1.tar.gz")
      wheels = [w["url"].rsplit("/", 1)[1] for w in ibmmq.get("wheels", [])]
      assert "ibmmq-2.1.1-cp39-abi3-manylinux_2_28_x86_64.whl" in wheels
      assert not [name for name in pkgs if name.startswith("opentelemetry")]
  ```

  `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 9: No-pymqi guard (repo test).** Create `tests/test_no_pymqi_refs.py`:

  ```python
  """No pymqi reference outside historical records (epic .github#300 definition of done).

  The lab's MQI binding is ibmmq. Every git-tracked file is scanned, case-insensitively,
  for "pymqi". The allowlist is FIXED: dated, historical records only.

  - docs/plans/**, docs/specs/**: design history;
  - docs/reports/**: dated reports (e.g. the guest-runtime spike proved pymqi);
  - CHANGELOG.md, releases/**: generated from commit history;
  - this file.

  A hit is fixed at its source (reword it). Never widen the allowlist to silence one.
  """

  from __future__ import annotations

  import re
  import subprocess
  from pathlib import Path

  from mqlab.paths import repo_root

  NEEDLE = re.compile(r"pymqi", re.IGNORECASE)
  HISTORICAL_PREFIXES = ("docs/plans/", "docs/specs/", "docs/reports/", "releases/")
  HISTORICAL_FILES = frozenset({"CHANGELOG.md", "tests/test_no_pymqi_refs.py"})


  def _is_historical(rel: str) -> bool:
      return rel in HISTORICAL_FILES or rel.startswith(HISTORICAL_PREFIXES)


  def _tracked(repo: Path) -> list[str]:
      result = subprocess.run(  # noqa: S603 - fixed argv, no shell
          ["git", "-C", str(repo), "ls-files", "-z"],  # noqa: S607
          capture_output=True,
          text=True,
          check=False,
      )
      if result.returncode != 0:
          msg = f"git ls-files failed in {repo} (exit {result.returncode}): {result.stderr.strip()}"
          raise RuntimeError(msg)
      return sorted(filter(None, result.stdout.split("\0")))


  def hits(repo: Path) -> list[str]:
      found = []
      for rel in _tracked(repo):
          path = repo / rel
          if _is_historical(rel) or not path.is_file() or "__pycache__" in path.parts:
              continue
          try:
              text = path.read_text(encoding="utf-8")
          except UnicodeDecodeError:
              continue  # binary asset (images, archives): not prose or code
          found.extend(
              f"{rel}:{n}" for n, line in enumerate(text.splitlines(), 1) if NEEDLE.search(line)
          )
      return found


  def test_allowlist_is_history_only():
      assert _is_historical("docs/specs/x.md")
      assert _is_historical("docs/reports/assets/guest-runtime-spike/spike.yml")
      assert _is_historical("CHANGELOG.md")
      assert not _is_historical("docs/site/docs/operate/guest-components.md")
      assert not _is_historical("docs/development/box-model.md")
      assert not _is_historical("components/mq-resiliency-clients/src/mqrc/selfcheck.py")


  def test_scan_finds_a_planted_reference(tmp_path):
      subprocess.run(["git", "init", "-q", str(tmp_path)], check=True)  # noqa: S603,S607
      (tmp_path / "docs" / "specs").mkdir(parents=True)
      (tmp_path / "a.py").write_text("import os\nimport PyMQI\n")
      (tmp_path / "docs" / "specs" / "old.md").write_text("pymqi was used\n")
      subprocess.run(["git", "-C", str(tmp_path), "add", "-A"], check=True)  # noqa: S603,S607
      assert hits(tmp_path) == ["a.py:2"]


  def test_scan_reports_a_git_failure(tmp_path):
      import pytest

      with pytest.raises(RuntimeError, match="git ls-files failed"):
          hits(tmp_path)  # not a git repo


  def test_no_pymqi_reference_outside_history():
      found = hits(repo_root())
      assert not found, "pymqi references outside history (reword at source):\n" + "\n".join(found)
  ```

  Move `import pytest` to the top-level imports if ruff flags it. Run
  `vrg-container-run -- vrg-validate`. Expected: **FAIL** on
  `test_no_pymqi_reference_outside_history`, listing every remaining hit. Steps 10–12
  clear them.

- [ ] **Step 10: Repo tests and tools.**
  - **`tests/test_component.py`:**
    - Rename `PYMQI_SDIST`/`PYMQI_HASH` → `IBMMQ_SDIST`/`IBMMQ_HASH` (value
      `b"ibmmq sdist bytes"`).
    - Add `IBMMQ_WHL_X86 = b"ibmmq x86 wheel bytes"` and its `IBMMQ_WHL_X86_HASH`.
    - In `URLS`, replace the pymqi entry with
      `"https://f.ex/ibmmq-2.1.1.tar.gz": IBMMQ_SDIST` and
      `"https://f.ex/ibmmq-2.1.1-cp39-abi3-manylinux_2_28_x86_64.whl": IBMMQ_WHL_X86`.
    - In `UV_LOCK`, the ibmmq package gets that `sdist`, plus `wheels = [` the x86
      wheel with its hash, and
      `{{ url = "https://f.ex/ibmmq-2.1.1-cp39-abi3-macosx_11_0_arm64.whl", hash = "sha256:00" }} ]`.
    - In `PYPROJECT`, use `mqi = ["ibmmq==2.1.1"]`.
    - The staged-`deps/` assertion (`:265-268`) becomes:

      ```python
      assert sorted(p.name for p in (final / "deps").iterdir()) == [
          "ibmmq-2.1.1-cp39-abi3-manylinux_2_28_x86_64.whl",  # macOS wheel not staged
          "ibmmq-2.1.1.tar.gz",  # sdist: aarch64 compiles it
          "setuptools-84.0.0-py3-none-any.whl",  # the win32 wheel is not staged
      ]
      ```

    - In `test_stage_deps_exported_but_not_locked`,
      `test_stage_deps_hash_mismatch_deletes_the_file` and
      `test_stage_deps_refuses_a_non_sha256_hash` (`:418-463`), replace
      `pymqi`/`1.12.13` with `ibmmq`/`2.1.1`, including the `match=` strings and
      file names.
    - Add `("ibmmq-2.1.1-cp39-abi3-manylinux_2_28_x86_64.whl", True)` and
      `("ibmmq-2.1.1-cp39-abi3-macosx_11_0_arm64.whl", False)` to the
      `test_wheel_installable` table.
  - **`tests/test_version_token_guardrail.py:147`:** `"pymqi 9.6"` → `"ibmmq 9.6"`.
  - **`tests/test_client_roles_rewired.py`, `test_clients_component_baked.py`,
    `test_install_roles.py`:** reword any pymqi mention in docstrings, comments and
    assertion messages to "the MQI binding (ibmmq)" or "the ibmmq sdist compile
    (aarch64)". Keep every assertion: gcc still stays on mq-client, and the bake
    order and compile flags are unchanged.
  - **`tools/validate-event-monitor-wrapper.sh:6`:** `(no ansible/pymqi)` →
    `(no ansible/MQI binding)`.
  - **`components/mq-resiliency-observability/src/mqro/loglifecycle.py:6`:**
    `(no MQI, no PyMQI, no compiled deps)` → `(no MQI, no MQI binding, no compiled deps)`.

- [ ] **Step 11: Ansible comments and task names.** No task logic, package list or
  order changes.
  - **`bake-mq-ubuntu.yml:51`:** `SDK for pymqi` → `SDK for the ibmmq binding`.
    **`:62`:** `pymqi compiles from sdist against the MQ SDK (and gcc)` →
    `ibmmq compiles from sdist against the MQ SDK (and gcc) on aarch64, where upstream ships no wheel`.
  - **`site-distributed-shared.yml:40`:** `MQ client libs + pymqi` → `MQ client libs + the ibmmq binding`.
  - **`site-pcmk-authz-validate.yml:42`:** `(pymqi + app keystore live there)` → `(the MQI clients + app keystore live there)`.
  - **`roles/component-install/tasks/main.yml:106`:** `sdist dependencies (pymqi) compile here` →
    `sdist dependencies (ibmmq on aarch64; x86_64 takes its wheel) compile here`.
  - **`roles/mq-client/tasks/main.yml`:**
    - `:65-67`: `recompiles pymqi against the MQ SDK on the guest` →
      `compiles the ibmmq sdist against the MQ SDK on aarch64 guests (no upstream aarch64 wheel, .github#302)`.
    - `:68`: task name `install client + sdk debs and the pymqi compiler` →
      `install client + sdk debs and the MQI binding compiler`.
    - `:97`: `the pre-#294 per-provision pymqi venv` → `the pre-#294 per-provision client venv`.
    - `:99`: task name → `retire the pre-#294 client venv and loose client scripts`.
  - **`roles/mq-install/tasks/main.yml:51`:** `sdk (pymqi bindings need cmqc.h)` →
    `sdk (the ibmmq binding compiles against cmqc.h on aarch64)`.
  - **`roles/mq-inter-qm/templates/their-side.mqsc.j2:7-8`:**

    ```text
    * The co-located service responder connects client-mode via this SVRCONN (#180):
    * it deliberately uses a TLS client connection, not bindings. MCAUSER(mqm)
    * gives it admin authority; CHLAUTH is already disabled on this QM.
    ```

  - **`roles/pki-distribute/defaults/main.yml:9-10`:** `(pymqi app/responder, the exporter) read a PKCS#12 keystore's password from a sibling .sth stash, since pymqi's SCO has no password field.` →
    `(the ibmmq app/responder, the exporter) read a PKCS#12 keystore's password from a sibling .sth stash (SCO.KeyRepoPassword adoption is deferred, epic .github#300 §9).`
  - **`roles/pki-distribute/tasks/main.yml:40-41`:** `so pymqi (no SCO password field) can open` →
    `so the MQI clients (which use the .sth stash, not SCO.KeyRepoPassword) can open`.

  If any test in `tests/` asserts an exact task name changed here, update that
  string in the same commit. Use
  `grep -rn "pymqi compiler\|pymqi venv" tests/` to find them.

- [ ] **Step 12: Living docs.**
  - **`docs/site/docs/operate/guest-components.md`:**
    - `:17`: `ad-hoc pymqi venvs` → `ad-hoc client venvs`.
    - `:36`: describe the binding as "`ibmmq` (IBM's MQI binding): x86_64 guests
      install its wheel; aarch64 guests compile its sdist against the MQ SDK".
    - `:120`: `sdist dependencies such as pymqi` → `sdist dependencies such as ibmmq on aarch64`.
    - `:161`: `(pymqi compiles here with gcc against /opt/mqm)` →
      `(on aarch64, ibmmq compiles here with gcc against /opt/mqm; x86_64 installs its wheel)`.
  - **`docs/site/docs/design-and-specs.md:30`:** `pymqi built from sdist` → `the MQI binding built from sdist`.
  - **`docs/development/box-model.md`:**
    - `:37`: `(pymqi compiled from sdist at bake; …)` → `(ibmmq: wheel on x86_64, compiled from sdist at bake on aarch64; …)`.
    - `:189`: `SDK for pymqi` → `SDK for the ibmmq binding`.
  - **`docs/development/box-bake-manifest.md`:**
    - `:367`: `SDK for pymqi` → `SDK for the ibmmq binding`.
    - `:369`: `pymqi compiles from its hash-pinned sdist against the MQ SDK with gcc` →
      `on aarch64, ibmmq compiles from its hash-pinned sdist against the MQ SDK with gcc (x86_64 installs the hash-pinned wheel)`.
  - **`docs/development/operating-the-lab-from-a-dev-session.md:140-206`:** rewrite
    the snippet at `:164-183` in native idiom. `from pymqi import CMQC as C` becomes
    `import ibmmq` plus `from ibmmq import CMQC as C`. Every `pymqi.X` becomes
    `ibmmq.X`. MQCHAR names (`b"APP.SVRCONN"` and similar) become `str`. Keep the
    bodies as bytes. Any `MQCHT_*` comes from `ibmmq.CMQXC`. Update the prose and the
    table row at `:206` to say ibmmq, and describe `/home/vagrant/mqvenv` as the
    retired pre-#294 venv: the component venv is the place to run it.
  - **`components/README.md:37`:** `e.g. setuptools + wheel for pymqi` → `e.g. setuptools for ibmmq's sdist`.
    **`:56`:** rephrase the stale-`__version__` aside so it no longer names pymqi,
    for example "never a module's `__version__`, which can be stale".
  - **`components/mq-resiliency-clients/README.md:17-21,37`:** describe the `mqi`
    extra as `ibmmq==2.1.1` (wheel on x86_64, sdist compile on aarch64,
    `setuptools` in `sdist-build`), lazy `import ibmmq`, and a fake `ibmmq` module
    in tests. The selfcheck row says "prove the real ibmmq binding when the `mqi`
    extra is installed".

- [ ] **Step 13: Gate.**
  1. `vrg-commit --type refactor --scope clients --message "migrate the MQI clients from pymqi to ibmmq (native idiom)"`, with a body citing epic `.github#300` spec §4.
  2. `mqlab component build mq-resiliency-clients` passes. Tests run on the pinned
     runtime, with no extra.
  3. `vrg-container-run -- vrg-validate` passes, including
     `test_no_pymqi_reference_outside_history` and
     `test_clients_lock_pins_ibmmq_wheel_and_sdist_without_otel`.
  4. Verify the deliverable: `grep -rniI pymqi --exclude-dir=.git . | grep -vE '^\./(docs/(plans|specs|reports)/|CHANGELOG\.md|releases/|tests/test_no_pymqi_refs\.py)'`
     prints nothing.
  5. Hand off with `vrg-pr-workflow report-ready`.

---

## Task T3: Observability repo comment (`mq-resiliency-observability#42`)

**Files:** Modify `pyproject.toml:7-8` in `logical-minds-foundry/mq-resiliency-observability`.

- [ ] **Step 1:** Create the worktree `.worktrees/issue-42-drop-pymqi-mention` on
  `feature/42-drop-pymqi-mention`, in that repo's checkout.
- [ ] **Step 2:** Replace the two comment lines with:

  ```toml
  # Runtime is stdlib-only by design: the collectors deploy verbatim onto MQ nodes
  # and must not drag an MQ client library or any third-party wheel onto a production host.
  ```

- [ ] **Step 3:** `grep -rni pymqi .` in that repo prints nothing outside its own
  historical docs, if it has any. `vrg-container-run -- vrg-validate` passes.
- [ ] **Step 4:** `vrg-commit --type docs --scope pyproject --message "drop the pymqi mention from the stdlib-only runtime comment"`, then `vrg-pr-workflow report-ready`.

---

## Task T4: Outreach, a Linux aarch64 wheel (`.github#302`)

This task has no code. Per the issue body:

- [ ] **Step 1:** Draft two messages for the human.
  - **(a) A public issue for `ibm-messaging/mq-mqi-python`.** Title: "Publish a
    Linux aarch64 (manylinux_2_28_aarch64) wheel". Body:
    - IBM ships MQ for Linux on ARM64.
    - `ibmmq` 2.1.1 publishes `cp39-abi3` wheels for `manylinux_2_28_x86_64`,
      `macosx_11_0_arm64` and `win_amd64`. The CHANGELOG notes that the Linux x64
      wheel was enabled on 2026-07-06.
    - Without an aarch64 wheel, every Linux ARM64 install compiles the sdist, which
      needs gcc and the MQ SDK on the target.
    - The limited-API build should make one abi3 aarch64 wheel enough.
    - Offer to test it on Ubuntu 24.04 aarch64.
  - **(b) A short private note to the human's contacts on the IBM Messaging team,**
    pointing at (a).
- [ ] **Step 2:** The human sends both. The agent never posts on the human's behalf.
- [ ] **Step 3:** Record a comment on `.github#302` with what was sent, to whom,
  when, and the public issue URL. Then close it.
- [ ] **Step 4:** File the tracking issue under the ad-hoc epic `.github#29`:
  `vrg-issue-create --epic logical-minds-foundry/.github#29 --repo logical-minds-foundry/.github --title "Check upstream for an ibmmq Linux aarch64 wheel"`.
  Its body says: when one ships, feed spec §9 item 2 (drop gcc, `sdist-build` and
  `CC`/`LDSHARED`) into the modernization epic.

---

## Operational V1 (validation): rebake + cold rebuild, ARM64 + x86_64 (`mq-resiliency-lab-for-linux#1362`)

This issue already exists. Its body is rewritten to this section when T1 publishes.
It is blocked by `lab#1360` and `obs#42`.

**Preconditions, a self-check run first:**

- **Probe:** `lab#1360` and `obs#42` are closed (merged to develop).
- **Human-attested:** dedicated use of an ARM64 lab host and an x86_64 lab host. The
  x86_64 host can run **both Ubuntu stacks** (`nativeha-ubuntu`, `pcmk-ubuntu`).

If any precondition is unmet, comment `blocked: preconditions not met — <which>` and
stop.

**Per platform, on a develop checkout that contains T2.** Commands run from the
project root. `INV=$(mqlab build path work)/inventory.ini`, and
`CLIENTS=/opt/logical-minds-foundry/mq-resiliency-clients/venv/bin`.

1. **Rebake.** Run `mqlab component build mq-resiliency-clients`, then
   `mqlab box rebuild mq-client-ubuntu24`.
2. **Cold rebuild.**
   - Tear down anything running: `mqlab teardown <stack>` for each live stack.
   - `mqlab commons up`
   - `mqlab bootstrap nativeha-ubuntu`
3. **The binding.**
   - `ansible app-client,svc-sim -i "$INV" -b -m ansible.builtin.shell -a "LD_LIBRARY_PATH=/opt/mqm/lib64 $CLIENTS/mq-resiliency-clients-selfcheck"`
     shows `ok ibmmq 2.1.1` and `mq-resiliency-clients 0.1.0 on 3.14.8` on both
     hosts, with rc 0.
   - `mqlab component status mq-resiliency-clients --host app-client --host svc-sim`
     shows both hosts **matches CURRENT**.
   - `ansible app-client -i "$INV" -b -m ansible.builtin.shell -a "grep ^Tag: /opt/logical-minds-foundry/mq-resiliency-clients/venv/lib/python3.14/site-packages/ibmmq-2.1.1.dist-info/WHEEL"`
     prints:
     - **aarch64:** `Tag: cp39-abi3-linux_aarch64`, which means it was built
       locally from the sdist.
     - **x86_64:** `Tag: cp39-abi3-manylinux_2_28_x86_64`, which means the PyPI
       wheel was used.
4. **The always-on pair.** `mqlab qm e2e nativeha-ubuntu` passes, exercising
   `mq-app-requester` against `mq-svc-responder@`.
5. **mq-bench.** On app-client, as vagrant, run
   `/home/vagrant/mq-bench --warmup-seconds 5 --measure-seconds 10 --results /tmp/bench.jsonl`.
   The wrapper supplies the stack's QM, CONN and TLS. It exits 0 and reports commits
   greater than 0.
6. **Authz.** In `ansible/`, run
   `ansible-playbook -i "$INV" site-nativeha-ubuntu-authz-validate.yml --tags induced,positive,authzpos,evidence,exporter -e qm_app=NHAUAPP`.
   It passes. This is the pre-failover pass from the playbook's RUN header.
7. **DR entry points** (the #1377 bar). Read `QM` and `CONN` from
   `/etc/opt/logical-minds-foundry/mq-resiliency-clients/mq-app-requester.env` on
   app-client.
   - **app-client, as vagrant:**
     `LD_LIBRARY_PATH=/opt/mqm/lib64 $CLIENTS/mq-dr-flow --qm "$QM" --conn "$CONN" --channel APP.SVRCONN --keyrepo /home/vagrant/ssl/key --certlabel app-client --seconds 3 --ledger /tmp/dr-probe-app.jsonl`
     exits 0, and its "flow done: N sent" line has N greater than 0.
   - **svc-sim, as mqm:**
     `sudo -u mqm env LD_LIBRARY_PATH=/opt/mqm/lib64 $CLIENTS/mq-dr-responder --qm SVCQM --conn 'localhost(1414)' --channel SVC.SVRCONN --keyrepo /var/mqm/ssl/svc-responder/key --certlabel svc-responder --in-queue DR.PROBE.IN --out-queue DR.PROBE.OUT --seconds 3 --ledger /tmp/dr-probe-svc.jsonl`
     connects, then stops with MQRC **2085** (expected, #1375).
   - Neither may show 2393 or 2381.
8. **The pcmk stack.**
   - `mqlab teardown nativeha-ubuntu`, then `mqlab bootstrap pcmk-ubuntu`.
   - `mqlab qm e2e pcmk-ubuntu` passes.
   - `ansible-playbook -i "$INV" site-pcmk-authz-validate.yml`, run as its RUN
     header directs for a full pass, passes. This includes the `mq-dlq-probe`
     checks.
9. **Leave the lab clean:** `mqlab teardown pcmk-ubuntu`.

**Explicitly not exercised.** State this in the result comment:

- `mq-reconnect-probe` hardcodes a plaintext `APP.SVRCONN`, which the TLS-only
  channels refuse.
- `mq-dr-baseline` and `mq-dr-forced` have no invoker.

These three are covered by the selfcheck imports and unit tests.

**Fix-forward rule.** A failure caused by a name or idiom is fixed in a new task
linked under the epic, then re-run. A fix that would change behaviour is recorded as
a modernization candidate in the spec's §9 list, through the retrospective, and is
not folded in.

**Acceptance:** steps 1–8 are green on **both** platforms. Record
`Outcome: SUCCESS` with a per-step table for each platform, and close.

---

## Closing bookends (already filed)

- **Documentation review:** `mq-resiliency-lab-for-linux#1361`, blocked by V1. It
  sweeps `docs/site`, `docs/development`, `docs/reference`, the component READMEs and
  runbooks for any MQI-binding description T2 missed. It spawns per-repo doc tasks if
  another repo needs edits.
- **Follow-on brainstorm:** `.github#303`, blocked by V1. It covers the modernization
  epic (spec §9) and the private-consumer migration.
- **Retrospective:** `.github#304` (terminal), via `epic-retrospective`.
