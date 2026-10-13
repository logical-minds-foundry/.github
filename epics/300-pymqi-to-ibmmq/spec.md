# Migrate the resiliency lab from pymqi to ibmmq — design spec

- **Epic:** `logical-minds-foundry/.github#300`
- **Design task:** `logical-minds-foundry/.github#301`
- **Member repos:** `logical-minds-foundry/mq-resiliency-lab-for-linux` (all code),
  `logical-minds-foundry/mq-resiliency-observability` (one comment)
- **Predecessor:** epic `logical-minds-foundry/.github#294` (guest component
  packaging, M1), which this epic waited on. It moved every MQI client into the
  `mq-resiliency-clients` component, and its follow-on `.github#298` (build our own
  pymqi wheels) was closed as superseded by this epic.
- **Follow-ons (seeded):** `.github#303` (ibmmq modernization and the remaining
  private pymqi consumer)
- **Date:** 2026-10-13, verified against `mq-resiliency-lab-for-linux` `develop`
  @ `96d4e82` and the `ibmmq` 2.1.1 sdist

## 1. Problem and motivation

The lab's MQI clients are built on `pymqi` 1.12.13:

- It ships **only as an sdist**, so every install compiles it against the MQ SDK.
- Its `setup.py` imports `distutils`, which was removed from the stdlib in Python 3.12.
  It only builds on the lab's pinned CPython 3.14 through setuptools' shim, with
  `setuptools` and `wheel` staged by hand because pymqi declares no `[build-system]`.
- Its PyPI description says development is on hold.

IBM's MQ Messaging group took the project over and publishes the successor as
**`ibmmq`** ([ibm-messaging/mq-mqi-python](https://github.com/ibm-messaging/mq-mqi-python)).
The lab uses only core MQI: connect, put/get, syncpoint, reconnect, identity
context, and one single-selector inquire. `ibmmq` removed none of these.

### 1.1 Current state (verified against `develop` @ `96d4e82`)

The layout this epic edits did not exist when the epic was first scoped. It comes
from #294:

- **All MQI code is in one package:** `mqrc`, in
  `components/mq-resiliency-clients/src/mqrc/`. Ten modules import pymqi:
  `app_requester`, `bench_client`, `svc_responder`, `authz_probe`, `dlq_probe`,
  `reconnect_probe`, `dr_mqi`, `dr_flow`, `dr_responder` and `selfcheck`. All of them
  do a **lazy, function-local `import pymqi`** and use `pymqi.X` / `pymqi.CMQC.X`.
  Nothing does `from pymqi import`, and nothing uses `CMQXC`. `dr/snapshot.py:6` has
  a docstring naming a pymqi `browse_queue()` that does not exist.
- **`mqro` (observability) has no MQI code.** Its only mention is a "no PyMQI"
  docstring in `loglifecycle.py:6`.
- **Packaging** (`components/mq-resiliency-clients/pyproject.toml`):
  - Extra `mqi = ["pymqi==1.12.13"]`.
  - Group `sdist-build = ["setuptools==84.0.0", "wheel==0.48.0"]`.
  - `uv.lock` records pymqi as sdist-only.
  - `mqlab component build` exports `--all-extras` to `requirements.txt` and the
    `sdist-build` group to `build-requirements.txt`. `stage_deps`
    (`src/mqlab/component.py:227`) stages the sdist **and every wheel a CPython 3.14
    Linux guest can install**, including `abi3` `manylinux_*_x86_64|_aarch64`, all
    hash-verified.
- **Install** (`ansible/roles/component-install/tasks/main.yml:106-153`) runs three
  `pip` steps, all offline (`--no-index --find-links deps --require-hashes`):
  1. Install `build-requirements.txt`.
  2. Export `CC`/`LDSHARED`, derived from the runtime's sysconfig with `clang`
     swapped for `gcc` (#294 errata E4), then install `requirements.txt` with
     `--no-build-isolation`.
  3. Install the component wheel with `--no-deps`.

  `LD_LIBRARY_PATH=/opt/mqm/lib64` is set throughout, and in both systemd units.
- **Where the component runs:** it is baked into **one box only**,
  `mq-client-ubuntu24` (`lab/versions.yaml:44`). Its nodes are `svc-sim`,
  `app-client` and `mon-probe`, the commons shared by all four stacks. The box
  follows the **host architecture**: aarch64 on the macOS/Lima path, x86_64 in the
  cloud. The clients are never installed on RHEL. gcc and the MQ SDK are installed
  at bake (`mq-install`) and kept by `mq-client`.
- **Tests** fake the binding with `types.ModuleType("pymqi")`, installed via
  `monkeypatch.setitem(sys.modules, "pymqi", …)`, in six test files. The fakes put
  `MQCHT_CLNTCONN` in `CMQC`. `test_selfcheck.py` asserts `"ok pymqi 1.12.13"`.
- **Repo guards** that know about pymqi or the compile:
  - `tests/test_component.py`: a fake pymqi sdist fixture, plus the `sdist-build`
    export.
  - `tests/test_install_roles.py`: the `CC`/`LDSHARED` export, build requirements
    first, `--no-build-isolation`.
  - `tests/test_client_roles_rewired.py`: gcc stays on mq-client.
  - `tests/test_clients_component_baked.py`: bake order.
  - `tests/test_version_token_guardrail.py:147`: a `"pymqi 9.6"` benign string.
- **Live entry points:**
  - `mq-app-requester` (app-client) and `mq-svc-responder@` (svc-sim) are systemd
    units started by every stack's bootstrap. `mqlab qm e2e <stack>` drives a
    bounded requester run.
  - `mq-bench` is rendered on **Native HA stacks only** (#294 E5).
  - `mq-authz-probe` and `mq-dlq-probe` are run by the
    `site-*-authz-validate.yml` playbooks. `mq-dlq-probe` is only in the pcmk one.
  - `mq-dr-flow` / `mq-dr-responder` are invoked only by `lab/scripts/dr-run.sh`,
    which is **not operational** (#1375, deferred: no Vagrant env, no DR queues,
    exits 0 on failure). #294 validated them as "complete a TLS MQCONNX; MQRC 2085
    on the undefined DR queues is expected" (#1377).
  - `mq-reconnect-probe`, `mq-dr-baseline` and `mq-dr-forced` have no invoker.
- **Comments and docs** that mention pymqi (the full list is in plan T1):
  - Comment-only Ansible mentions in `mq-install`, `mq-client`, `component-install`,
    `pki-distribute` (the `.sth` rationale), `mq-inter-qm/templates/their-side.mqsc.j2`
    (the "client-only → 2058" rationale), `bake-mq-ubuntu.yml`,
    `site-distributed-shared.yml` and `site-pcmk-authz-validate.yml`.
  - Living docs: `docs/site/docs/operate/guest-components.md`,
    `docs/site/docs/design-and-specs.md`,
    `docs/development/{box-model,box-bake-manifest,operating-the-lab-from-a-dev-session}.md`
    and both component READMEs.
  - `tools/validate-event-monitor-wrapper.sh:6`.
- **The observability repo:** `mq-resiliency-observability/pyproject.toml:7-8` says
  the runtime "must not drag pymqi or any third-party wheel onto a production host".

### 1.2 `ibmmq` 2.1.1 (verified against the sdist, 2026-10-13)

- **[data]** PyPI: 2.1.1 (2026-09-03) is the latest release. It ships
  `cp39-abi3` wheels for `manylinux_2_28_x86_64`, `macosx_11_0_arm64` and `win_amd64`
  plus an sdist. There is **no Linux aarch64 wheel**. `requires_python >=3.9`.
- **[data]** The sdist declares `[build-system] requires = ["setuptools >= 75.0.0"]`
  and `setup.py` no longer uses `distutils`. It links **`libmqm_r`** (libmqic was
  removed in V2), and finds the SDK at `/opt/mqm` or `MQ_FILE_PATH`. The wheel does
  not bundle `libmqm_r.so`.
- **[data]** API, from `docs/MIGRATION_V2.md` and the V2.0.0b1 changelog entry:
  - **Unchanged:** the class and method names the lab uses: `QueueManager`,
    `connect_with_options(name, cd=, sco=, opts=)`, `CD`, `SCO`, `MD`, `OD`, `PMO`,
    `GMO`, `Queue`, `Queue.inquire(int)` (single selector still returns the value),
    `MQMIError` and `PYIFError` with `.reason`, and `CMQC`.
  - **Moved:** `MQCHT_*` are **only in `CMQXC`**.
  - **Inputs:** every `MQCHAR[]` input field accepts `str` or `bytes`. Output
    fields such as `MsgId`, `ReplyToQ` and message bodies are still `bytes`.
  - **Connect:** `connect_with_options` accepts a `str` name and ignores
    `sco=None`. 2.1.0 sets `MQCNO_CLIENT_CONNECT` automatically when an `MQCD` is
    passed.
  - **Version:** `ibmmq.__version__` comes from `importlib.metadata`, and import
    fails on a C/Python version mismatch.
  - **Error text:** messages from `MQMIError`/`PYIFError` changed slightly. The lab
    only reads `.reason`.
- **[data]** OpenTelemetry propagation switches on automatically if OTel packages
  can be imported, unless `MQIPY_NOOTEL` is set. The clients venv has no OTel
  packages, so it stays off. That is a property of the lock, not a code change.

## 2. Goals

1. Every MQI client runs on `ibmmq==2.1.1` and no longer on pymqi, on both
   architectures the mq-client box is built for.
2. **Native idiom, sideways only.** The changes are listed below. Behaviour on the
   wire and in every client's CLI and output stays the same.
   - `import ibmmq` directly, with no `import ibmmq as pymqi` alias.
   - `MQCHT_*` from `CMQXC`.
   - `str` instead of `.encode()`/`b"…"` **for MQCHAR input fields** (names, cipher
     specs, keystore paths, labels, user identifiers).
3. **Packaging follows the binding.** The `mqi` extra pins `ibmmq==2.1.1`. On
   x86_64 the install takes the abi3 wheel. On aarch64 it compiles the sdist. Both
   go through the existing offline, hash-pinned path.
4. **No pymqi references remain** in `mq-resiliency-lab-for-linux` outside
   historical records, enforced by a guard test, nor in `mq-resiliency-observability`.
5. The lab is **rebaked and cold-rebuilt green on ARM64 and x86_64**, with every
   live-invocable client exercised on each.

## 3. Non-goals

These go to the modernization follow-on (§9):

- **Behaviour-changing ibmmq features:**
  - `SCO.KeyRepoPassword` instead of the `.sth` stash.
  - OpenTelemetry propagation.
  - Dict-based multi-selector `inquire`.
  - `CMQSTRC` for reason names in place of `bench_client.reason_names`.
  - Async consume (`cb`/`ctl`), `to_string()` on outputs, `CNO`-based options.
- **Making x86_64 gcc-free.** gcc, the `sdist-build` group and the `CC`/`LDSHARED`
  override stay on **both** architectures. One box definition serves both, and
  aarch64 still compiles. They go once an aarch64 wheel exists, tracked by #302's
  follow-up.
- **Fixing `dr-run.sh` or wiring the DR or reconnect clients into automation**
  (#1375).
- **Payloads stay `bytes`.** Message bodies (`b"REPLY:" + body`, the `DRv1|` wire
  marker, `b"x" * n`) are data, not MQCHAR fields.
- **Historical records are not touched:** `docs/plans/**`, `docs/specs/**`,
  `docs/reports/**`, `CHANGELOG.md`, `releases/**`.

## 4. Design

### 4.1 Name and idiom mapping

| pymqi (today) | ibmmq (target) | Where |
|---|---|---|
| `import pymqi` (function-local) | `import ibmmq` (function-local; keep the lazy import so the modules import on the dev VM, which has no MQ SDK) | every `mqrc` module that touches MQI |
| `pymqi.CMQC.MQCHT_CLNTCONN` | `ibmmq.CMQXC.MQCHT_CLNTCONN` | `authz_probe.py:85`, `dlq_probe.py:95` |
| `pymqi.CMQC.<other>` | `ibmmq.CMQC.<other>` (unchanged names) | everywhere |
| `pymqi.X` for CD/SCO/MD/OD/PMO/GMO/Queue/QueueManager/MQMIError/PYIFError | `ibmmq.X` | everywhere |
| `ChannelName=…encode()`, `ConnectionName=…encode()`, `SSLCipherSpec=b"ANY_TLS13_OR_HIGHER"`, `KeyRepository=…encode()`, `CertificateLabel=…encode()`, `ReplyToQ=…encode()`, `ReplyToQMgr=…encode()`, `UserIdentifier=…encode()`, `b"APP.SVRCONN"` | the same fields as `str` | `app_requester`, `bench_client`, `svc_responder`, `authz_probe`, `dlq_probe`, `reconnect_probe`, `dr_mqi` |
| output bytes: `put_md.MsgId`, `req_md.ReplyToQ`/`ReplyToQMgr` passed back into `MD`/`OD` | **unchanged** (bytes in is still accepted) | `app_requester:258`, `svc_responder:93-94` |
| `reply.decode(errors="replace")`, MQDLH slicing, `dr/wire.py` utf-8 | **unchanged** (outputs are still bytes) | `app_requester:262`, `dlq_probe:38-52`, `dr/wire.py` |
| `except pymqi.MQMIError, pymqi.PYIFError:` (PEP 758) | `except ibmmq.MQMIError, ibmmq.PYIFError:` | `authz_probe:128,133`, `dlq_probe:126,131` |
| `connect_with_options(name, cd=, sco=None, …)` (`svc_responder` plaintext path) | unchanged. ibmmq skips a falsy `sco`. Remove the "None SCO crashes in `sco.pack()`" comments where that was only a pymqi quirk, and keep any guard that is still needed for TLS | `app_requester:204`, `dr_mqi:83` |
| `bench_client.reason_names(pymqi.CMQC)` (walks `dir(CMQC)` for `MQRC_*`) | `reason_names(ibmmq.CMQC)`. ibmmq's `CMQC` is a module re-exporting the platform file, so `dir()` still lists `MQRC_*` | `bench_client:167-175,374` |
| `selfcheck`: `metadata.version("pymqi")` + `import_module("pymqi")`, "ok pymqi 1.12.13" | `metadata.version("ibmmq")` + `import_module("ibmmq")`, "ok ibmmq 2.1.1". Drop the stale-`__version__` note | `selfcheck.py` |
| `dr/snapshot.py:6` docstring "pymqi `browse_queue()`" | reword to describe what the module actually does (no MQI) | `dr/snapshot.py` |

**`str` is an input convenience, not a semantic change.** ibmmq encodes `str`
inputs with its MQI encoding (`ensure_strings_are_bytes`). Every value the lab
passes is ASCII (channel, queue and QM names, cipher names, paths, labels, and
user IDs like `app`/`mqmon`), so the bytes on the wire are identical.

**Client mode is preserved.** pymqi built from sdist linked the client library,
so the co-located `svc-responder` had to connect over `SVC.SVRCONN` (2058 on a
bindings connect). ibmmq links `libmqm_r`, which can do both. Every lab connect
passes an `MQCD`, so ibmmq sets `MQCNO_CLIENT_CONNECT` and the connection is still
a client connect. The `their-side.mqsc.j2` comment is reworded to give the real
reason (the responder deliberately connects client-mode over a TLS SVRCONN),
without the pymqi claim. The channel definition does not change.

### 4.2 Tests (component)

- Every fake becomes `types.ModuleType("ibmmq")`, installed with
  `monkeypatch.setitem(sys.modules, "ibmmq", …)`. The fakes gain a `CMQXC` holder
  for `MQCHT_CLNTCONN`, and `CMQC` loses it, so a test fails if a module reads it
  from the wrong place.
- Fake `PYIFError` messages drop the `"PYMQI Error"` text.
- Where a test asserts the encoded value of an MQCHAR field, the assertion follows
  §4.1: the field is now a `str`.
- `test_selfcheck.py` asserts `"ok ibmmq 2.1.1"` and the skip line names ibmmq.
- Tests still run **without** the `mqi` extra. The dev VM has no MQ SDK, the x86
  wheel would not import without `libmqm_r.so`, and aarch64 has no wheel.

### 4.3 Packaging

- `pyproject.toml`: `mqi = ["ibmmq==2.1.1"]`, and
  `sdist-build = ["setuptools==84.0.0"]`.
  - Comments are rewritten. ibmmq **declares** `setuptools >= 75` as its build
    backend. `--no-build-isolation` (needed because the install is offline and
    hash-pinned) means that backend must be pre-installed.
  - `wheel` is dropped because modern setuptools builds wheels itself. The aarch64
    compile in validation proves this, and if it fails, `wheel` comes back as a
    fix-forward.
- `uv.lock` is regenerated with `uv lock` in the component directory. It now
  records the ibmmq sdist plus three wheels. `stage_deps` stages the sdist and the
  `manylinux_2_28_x86_64` wheel (it filters out macOS and Windows). It also stages
  setuptools.
- **No change to `src/mqlab/component.py` or the `component-install` role logic.**
  On x86_64, pip picks the hash-pinned abi3 wheel. On aarch64 it compiles the sdist
  with gcc against `/opt/mqm`, using the `CC`/`LDSHARED` override. Both pass
  `--require-hashes`, because `uv export` emits the hashes of every artifact.
- `manylinux_2_28` needs glibc ≥ 2.28. The mq-client box is Ubuntu 24.04 with glibc
  2.39, and Ubuntu 26 is newer.

### 4.4 Ansible, tooling and repo tests

- **Comments only.** Each pymqi mention in the roles and playbooks listed in §1.1
  is reworded as "the MQI binding (`ibmmq`)" or similar. Where a comment's
  *rationale* was pymqi-specific, it is restated truthfully:
  - **`pki-distribute`:** the clients use the `.sth` stash. `SCO.KeyRepoPassword`
    now exists but is deferred to modernization.
  - **`their-side.mqsc.j2`:** see §4.1.
  - **`mq-client` / `component-install`:** gcc stays because aarch64 compiles the
    ibmmq sdist.
- **Task names change too,** such as "retire the pre-#294 pymqi venv". The retire
  logic and paths (`/home/vagrant/mqvenv`, `/var/mqm/rvenv`) do not change.
- **No task logic, package lists or bake order change.**
- **`tests/test_component.py`:** the fake sdist fixture becomes an ibmmq sdist,
  plus an abi3 `manylinux_2_28_x86_64` wheel and a macOS wheel. A new case asserts
  that `stage_deps` stages the sdist and the x86_64 wheel but not the macOS wheel:
  this is the mixed lock the real component now has. The hash and unlocked-dep
  examples are renamed.
- `tests/test_version_token_guardrail.py:147`: the benign string becomes
  `"ibmmq 9.6"`.
- `tools/validate-event-monitor-wrapper.sh:6` and `mqro/loglifecycle.py:6`: reword
  to "no MQI binding".
- `tests/test_install_roles.py`, `test_client_roles_rewired.py` and
  `test_clients_component_baked.py` keep their assertions, which are still true.
  Only their pymqi wording changes.

### 4.5 The guard test

`tests/test_no_pymqi_refs.py`, modelled on `test_no_logsearch_node_refs.py`:

- **What it scans:** every **git-tracked** file in the repo, case-insensitive
  `pymqi`, skipping compiled artifacts.
- **Allowlisted historical paths:**
  - `docs/plans/**` and `docs/specs/**`: design history.
  - `docs/reports/**`: dated reports. The guest-runtime spike report and its
    `assets/` record what was proven on pymqi.
  - `CHANGELOG.md` and `releases/**`: generated from commit history, which will
    name pymqi.
  - The guard test itself.
- **The allowlist is fixed.** The module docstring states it, and a hit is always
  fixed at its source, never by widening the list. This is the same rule as the
  version-token guardrail.

### 4.6 Docs

These are living docs, updated in the migration PR:

- `docs/site/docs/operate/guest-components.md`:
  - :36, :120, :161 describe the clients' binding: ibmmq, a wheel on x86_64, sdist
    compile on aarch64.
  - :17 describes the pre-#294 world. Reword it to "ad-hoc client venvs".
- `docs/site/docs/design-and-specs.md:30`: the spike report link's summary says
  "the MQI binding built from sdist". The report keeps its own text.
- `docs/development/box-model.md:37,189` and `box-bake-manifest.md:367,369`.
- `docs/development/operating-the-lab-from-a-dev-session.md:140-206`: the code
  snippet is rewritten in native ibmmq idiom: `import ibmmq`, `from ibmmq import CMQC`,
  `str` names.
- `components/README.md:37,56` and `components/mq-resiliency-clients/README.md:17-21,37`.

The closing documentation-review task (#1361) sweeps for anything this list
misses.

### 4.7 The observability repo

`mq-resiliency-observability#42` rewords `pyproject.toml:7-8`, for example: "must
not drag an MQ client library or any third-party wheel onto a production host".
It is independent of the lab PR.

## 5. Error handling

There are no new error paths. The existing ones carry over:

- **Install time:** `selfcheck` already catches *any* exception from importing the
  binding and fails the install. ibmmq adds two failure modes, and both surface
  there:
  - A C/Python version mismatch (`EnvironmentError`).
  - A missing `libmqm_r.so` (`ImportError`).

  A failed aarch64 compile fails pip, and so the install, as today.
- **Runtime:** every client branches on `exc.reason` only, never on error text, so
  ibmmq's reworded messages change no control flow. Logged messages change text,
  and that is accepted.

## 6. Testing

- Component unit tests: the renamed fakes (§4.2), on the pinned runtime, via
  `mqlab component build`.
- Repo tests:
  - the `stage_deps` mixed-lock case (§4.4);
  - the guard (§4.5);
  - a **lock test** on the clients' `uv.lock`. It requires exactly `ibmmq` 2.1.1,
    with its sdist **and** its `manylinux_2_28_x86_64` wheel, no pymqi, and no
    `opentelemetry-*` package. A later relock can then neither silently force x86
    to compile nor switch on OTel propagation.
- `vrg-container-run -- vrg-validate` is green.
- Live validation is §7. Nothing on the dev VM can import the real binding.

## 7. Validation (`mq-resiliency-lab-for-linux#1362`)

This runs on both platforms, with dedicated lab time, fixing forward. A fix that
would change behaviour is recorded as a modernization candidate (§9) and not folded
in.

**Stacks.** The clients box is the Ubuntu mq-client box on both architectures. The
QMs it connects to are not changed by this epic. So each platform runs the same two
**Ubuntu** stacks, which between them reach every live-invocable client:

- **`nativeha-ubuntu`**: e2e, `mq-bench` (Native HA only), and
  `site-nativeha-ubuntu-authz-validate.yml` (`mq-authz-probe`).
- **`pcmk-ubuntu`**: e2e, and `site-pcmk-authz-validate.yml` (`mq-authz-probe`
  plus `mq-dlq-probe`).

**Preconditions (human-attested):** dedicated use of an ARM64 lab host and an
x86_64 lab host. The x86_64 host can run **both Ubuntu stacks**
(`nativeha-ubuntu`, `pcmk-ubuntu`). #294 used the x86 host only for RHEL stacks, but
Ubuntu boxes resolve to the host's architecture.

**On each platform:**

1. `mqlab component build mq-resiliency-clients`, then `mqlab box rebuild`
   the mq-client box. Rebuild means a forced rebake, so the box carries ibmmq.
2. Cold rebuild: `mqlab commons up`, then `mqlab bootstrap nativeha-ubuntu`.
3. **The binding.** On app-client and svc-sim, run the live venv's
   `mq-resiliency-clients-selfcheck` directly, with `LD_LIBRARY_PATH=/opt/mqm/lib64`.
   It prints `ok ibmmq 2.1.1`. Separately, `mqlab component status
   mq-resiliency-clients --host app-client --host svc-sim` shows the live release
   **matches CURRENT**. That command only reads `INSTALLED.json` and never runs the
   selfcheck. `ibmmq-2.1.1.dist-info/WHEEL` in the live venv shows the right tag for
   the platform. ibmmq builds against the limited API (`py_limited_api="cp39"`), so
   both tags are `cp39-abi3`:
   - **aarch64:** `cp39-abi3-linux_aarch64`. The `linux_` platform tag, not
     `manylinux_`, proves the wheel was built locally from the sdist.
   - **x86_64:** `cp39-abi3-manylinux_2_28_x86_64`, proving the PyPI wheel was used.
4. **The always-on pair.** `mqlab qm e2e nativeha-ubuntu` passes. This exercises
   `mq-app-requester` against the `mq-svc-responder@` unit.
5. **`mq-bench`** runs to completion on app-client, with commits counted.
6. **Authz.** `ansible-playbook site-nativeha-ubuntu-authz-validate.yml` passes.
7. **DR entry points**, with the same bar as #1377. On app-client, `mq-dr-flow`
   completes a TLS MQCONNX to the stack QM. On svc-sim, `mq-dr-responder` completes
   one to `SVCQM`. MQRC 2085 on the undefined DR queues is the expected stop.
8. `mqlab teardown nativeha-ubuntu`, then `mqlab bootstrap pcmk-ubuntu`.
   `mqlab qm e2e pcmk-ubuntu` passes. `site-pcmk-authz-validate.yml` passes when
   run **in full, untagged**. Only a full run reaches `mq-dlq-probe`, which lives in
   the opt-in `n5-induce` play, and the Part A `mq-authz-probe` checks, which are
   untagged. The playbook's documented `--tags n2,n4,n5` run skips both. A full run
   is safe only on a dedicated arm, and this arm is a dedicated cold rebuild.

The exact commands are pinned in the plan.

**Not live-exercised.** These have no invoker today, so they are covered by
selfcheck imports plus unit tests:

- `mq-reconnect-probe` hardcodes a plaintext `APP.SVRCONN`, which the TLS-only
  channels refuse.
- `mq-dr-baseline` and `mq-dr-forced`.

The validation comment states this gap explicitly.

## 8. Migration sequence and tasks

There is one big-bang implementation PR. Splitting it would leave `mqrc` with two
bindings at once, or with tests that disagree with the lock.

| Task | What | Kind | Depends on |
|---|---|---|---|
| `.github#301` | This spec and plan | docs bookend | — |
| `mq-resiliency-lab-for-linux#1360` | Big-bang migration: code, tests, packaging and lock, comments, repo tests, guard, living docs (§4.1–§4.6) | impl | #301 |
| `mq-resiliency-observability#42` | Reword the pyproject comment (§4.7) | impl | — |
| `.github#302` | Ask the ibmmq maintainers for a Linux aarch64 wheel. Closes when sent; its follow-up goes under ad-hoc epic `.github#29` | outreach | — |
| `mq-resiliency-lab-for-linux#1362` | Validation on ARM64 and x86_64 (§7) | validation | #1360, obs#42 |
| `mq-resiliency-lab-for-linux#1361` | Documentation review sweep | bookend | #1362 |
| `.github#303` | Follow-on brainstorm: modernization and the private consumer | bookend | #1362 |
| `.github#304` | Retrospective | terminal bookend | all |

`#1360` and `#1362` were filed against the pre-#294 layout. Their bodies are
rewritten to this spec when it publishes.

## 9. Modernization candidates (input to `.github#303`)

1. `SCO.KeyRepoPassword` in place of the `.sth` stash. This changes
   `pki-distribute` and every client's keystore handling.
2. Drop gcc, the `sdist-build` group and the `CC`/`LDSHARED` override, once
   upstream ships a Linux aarch64 wheel (#302 follow-up).
3. OpenTelemetry propagation as part of the lab's observability architecture.
4. `CMQSTRC` for reason-code names (`bench_client.reason_names`) and `to_string()`
   for output fields.
5. Dict-based multi-selector `inquire`, and `CNO`-based connect options.
6. Async consume for the responders.
7. A working DR harness and reconnect probe (overlaps #1375).

## 10. Risks, data and judgment

| Risk | Data / judgment | Mitigation |
|---|---|---|
| The aarch64 sdist compile fails on 3.14 | **[data]** ibmmq 2.0.4 fixed a 3.14 SEGV, and 2.1.1 says it prepares for free-threading. **[judgment]** pymqi compiled on the same box and toolchain (spike report §2.2), and ibmmq's C layer is the limited API, so this risk is low | Validation step 3 on aarch64; fix forward |
| Dropping `wheel` from `sdist-build` breaks the offline aarch64 build | **[judgment]** setuptools ≥ 70.1 includes `bdist_wheel`, and ibmmq declares only setuptools | Validation step 3; add `wheel==0.48.0` back if needed |
| `libmqm_r.so` missing on a client-only install | **[data]** The baked box installs the full server deb set (`mq-install`), so `libmqm_r` is present | selfcheck at install fails loudly |
| Changing `str` inputs alters bytes on the wire | **[data]** ibmmq encodes str inputs. **[judgment]** every lab value is ASCII | unit tests pin field values; validation e2e |
| Hidden pymqi-specific behaviour (client-only linkage) | **[data]** 2.1.0 sets CLIENT_CONNECT when an MQCD is passed, and every lab connect passes one | validation step 4 (svc-responder co-located with `SVCQM`) |
| OTel switches on unexpectedly | **[data]** it is only enabled if OTel imports succeed. The lock has no OTel | pinned by the repo lock test (§6) in #1360 |
