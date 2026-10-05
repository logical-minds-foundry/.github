# Guest component packaging (M1) — design spec

- **Epic:** `logical-minds-foundry/.github#294`
- **Design task:** `logical-minds-foundry/.github#295`
- **Origin:** `logical-minds-foundry/.github#293` (triage seed with the full
  evidence)
- **Parallel epic (M2):** seeded from `vergil-project/.github#355`
- **Follow-on (seeded):** `logical-minds-foundry/.github#296`, M3
- **Member repo:** `logical-minds-foundry/mq-resiliency-lab-for-linux`
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-10-05

## 1. Problem and motivation

Guest-side code, meaning the observability collectors and the pymqi clients, is
developed and tested on one runtime, then **copied onto guests as loose source
files and run on another**: whatever system Python each guest OS ships. Nothing
delivers guest code together with the interpreter and dependencies it was
tested against. It is the software equivalent of compiling against IBM MQ 10
and deploying onto a host whose shared libraries are MQ 8.

It broke silently. A ruff pass in #1053 (2026-08-31) rewrote four collector
modules into `except A, B:` form, which is valid only from Python 3.14 (PEP
758). The guests run their system `python3` (3.12 on Ubuntu 24.04; RHEL 9's
default is 3.9). Every pcmk node's `lab-cluster-state.service` has failed with
a `SyntaxError` every 5 s since then. Prometheus holds zero `cluster_*` series,
and the Pacemaker, Native HA, RDQM and log-lifecycle cockpits have had no data
since 2026-08-31. No test checks guest-runtime compatibility, and no cold
rebuild exercised the dashboards. The evidence, with file:line references, is
in `.github#293`.

The OS-axis epic (`.github#280`) multiplies the number of guest runtimes
(Ubuntu 24/26, RHEL 9/10), so the problem only grows. `#280` is paused on this
epic.

### 1.1 Current state (verified against `develop` @ `700fe7e`)

- **No guest code is packaged.** The only `pyproject.toml` builds `src/mqlab`,
  the dev-host CLI. One setting governs everything: `requires-python
  >=3.14,<3.15`, ruff `py314`. No guest Python version is pinned or recorded
  anywhere.
- **Collectors** (`clusterstate`, `nativehastate`, `rdqmstate`, `loglifecycle`)
  are modules inside `src/mqlab`. Ansible `copy`s them as single files into
  `/usr/local/bin`, and the units run them under `/usr/bin/python3`.
  - `loglifecycle` carries an import hack: it tries `from mqlab import
    nativehastate` and falls back to a bare `import`, which needs a second copy
    of `nativehastate.py` alongside it.
  - The RDQM role hand-assembles a cut-down `mqlab` package, because
    `rdqmstate` imports `mqlab.clusterstate`.
- **Clients** (`clients/*.py`) are loose scripts that import each other as
  sibling files.
  - `/home/vagrant/mqvenv` is rebuilt on every provision and runs `pip install
    pymqi` **unpinned from PyPI**.
  - `/var/mqm/rvenv` on svc-sim is baked, but pymqi there is also unpinned.
  - The DR clients (`dr_flow`, `dr_responder`, `dr_mqi`) are **deployed by
    nothing** and import `mqlab.dr.*` and `mqlab.header`, which are in no guest
    venv. `lab/scripts/dr-run.sh` expects an mqvenv on svc-sim that does not
    exist.
- **The host `mqlab` CLI imports none of this.** The collectors, `mqlab.dr` and
  `mqlab.header` are imported only by each other, the client scripts and their
  own tests. They can leave `src/mqlab` cleanly.
- **Prior art: `mq-resiliency-observability` (`mqro`)** was extracted under
  epic `.github#79`, but it was never packaged. Its packaging issues were closed
  as won't-do on 2026-09-28 (portfolio wind-down, `.github#245`), with the
  in-lab copy deliberately kept. Its collectors have diverged substantially from
  the lab's (`cluster.py` vs `clusterstate.py` is close to a rewrite). The lab
  never consumed it.
- **The dev container** runs CPython 3.14.7 (a Debian GCC 14.2 build at
  `/usr/local/bin/python3`) and has a uv-managed python-build-standalone 3.14.7.
  `.python-version` is `3.14`, and `[tool.uv] python-preference =
  "only-managed"` already applies to mqlab.
- **Network:** Ubuntu guests reach the internet; RHEL guests are offline and use
  the DVD repo. Large artifacts (MQ tarballs, the exporter) are already staged
  from the host's `build/cache`.

## 2. End state and milestones

**End state.** Every software component a lab needs is a standalone product. It
lives in its own source tree, is built by standard tooling into a signed
`.deb`/`.rpm`, is published to its org's package repository, and is installed
with `apt`/`dnf` exactly as an outside user would install it. Vergil tooling
makes "new component → published package" routine, so future labs (for
example, a PostgreSQL HA/DR lab) start out this way. The purpose is to
demonstrate production-grade engineering, not to serve customers. The lab is
the first consumer.

The path is three milestones, each its own epic:

| Milestone | Scope | Home |
|---|---|---|
| **M1 (this epic)** | Componentize: each component is a standalone standard project inside the lab repo, built and tested on a pinned runtime, and installed from source with standard tooling into the `/opt` layout. Designed as **the build half of M2**. | `logical-minds-foundry/.github#294` |
| **M2** | Vergil tooling builds and publishes signed binary packages into per-org package repositories; the pinned runtime becomes a Vergil package. | from `vergil-project/.github#355` |
| **M3** | The lab installs **published** packages; `mqro` is revived as a real product. | follow-on `.github#296` |

**Design constraint for M1: nothing it builds is throwaway in M2 and M3.**

| Piece | M1 | M2/M3 | Survives |
|---|---|---|---|
| Component source tree, lock and tests | built | same | yes |
| Pinned interpreter | installed by the lab | runtime package | yes |
| `/opt` layout, unit files, `/etc/opt` config | created by the install recipe | created by the package | yes, identical |
| "Venv from source + lock" recipe | runs on the guest | runs in CI, wrapped into a package | yes |
| Ansible `component-install` backend | build from source | `apt`/`dnf install` | one thin backend swapped |

## 3. Goals

1. **Tested runtime = deployed runtime.** Guest code is only ever parsed and
   run by the exact interpreter build it was tested on, identical on every OS
   and arch.
2. **Components are standalone standard projects.** Each component lives in its
   own directory under `components/`. It is a standard uv project that knows
   nothing about Vergil or the lab, so extracting it into its own repo means
   moving the directory and wrapping Vergil around it.
3. **One standard install path.** Every component reaches a guest through one
   recipe: a hash-pinned, offline, from-source install into an isolated venv,
   verified on the box before it goes live. There is no file copying and no
   second path.
4. **Isolation by dependency set.** Each component has its own venv and only its
   own locked dependencies.
5. **Baked, with a fast standard update path.** Cold rebuilds are fully baked
   and one-pass. A running lab picks up new component code in seconds through
   the same recipe.
6. **Restore the dark dashboards**, proven by cold rebuilds on both arches.
7. **Language-neutral contract.** The contract (§5.4) is written so the first
   C++ component follows it without redesign.

## 4. Non-goals

- **Binary packages, package repositories, signing.** That is M2.
- **Consuming published packages, or extracting components into their own
  repos.** That is M3.
- **Full Vergil management of components** (`vergil.toml`, `vrg-validate`, the
  100% coverage gate, container validation). This "vergilizing" happens at
  extraction (§5.1).
- **Changing `vrg-validate`.** No vergil-tooling change is needed (§5.1).
- **The host `mqlab` CLI's own interpreter.** It is dev-host tooling, already a
  real package; its `.python-version` and container stay as they are.
- **Implementing C++.** There is no C/C++ source in the repo today. The contract
  covers C++ in principle only.
- **An interim compatibility patch for `#280`.** It was declined: `#280` waits
  for this epic.
- **Unrelated findings from the same investigation**, tracked elsewhere: the
  mqmon channel-authority decision (`authz_mqmon_grants`),
  `mq-resiliency-lab-for-linux#1343` (requester textfile drop permissions), and
  `#1342` (`e2e-test.sh` keystore stem).
- **Dashboard content.** Dashboard-specific findings route to the in-flight
  dashboard follow-up epic.
- **The stdlib-only / non-MQI collector boundary.** It stands. Only the
  "deployed verbatim onto system Python" half of that decision is superseded.

## 5. Design

### 5.1 Two quality tiers and the boundary

| | `mqlab` (the lab's tooling) | Components (`components/<name>/`) |
|---|---|---|
| Bar | Full Vergil: `vergil.toml`, `vrg-validate`, container, 100% branch coverage | A standard Python project: `pyproject.toml`, `src/` layout, unit tests, its own `uv.lock` |
| Driven by | Vergil | Plain `uv` (`uv lock`, `uv run pytest`, `uv build`), with no container and no `vergil.toml` |
| Knows about the other | Yes: `mqlab` and Ansible build, test and install components | **No**: a component knows nothing about Vergil or the lab |
| Fully Vergil-managed | Already | Only at extraction into its own repo ("vergilizing") |

`vrg-validate` already ignores `components/` by construction (verified in
vergil-tooling `lib/languages.py` and `bin/validate_common.py`):

| Check | Scope | Touches `components/`? |
|---|---|---|
| ruff check / format | `src/`, `tests/` explicitly | no |
| mypy | `src/` | no |
| pytest + coverage | `testpaths = ["tests"]`, `--cov=src` | no |
| `uv sync` / `uv lock --check` / pip-audit | the root project and lock | no (components are not workspace members) |
| markdownlint, yamllint, shellcheck | `docs/site`, `README.md`, `epics/`; root YAML, `.github/`, `mkdocs.yml`; `scripts/` | no |
| ansible-lint | repo-wide, minus local `exclude_paths` | **yes, unless excluded** |

Local configuration closes the remaining gaps, with no vergil-tooling change:

- `components/` is added to `.ansible-lint.yml` `exclude_paths`.
- Root `[tool.ruff] extend-exclude = ["components"]`, so a bare root `ruff`
  run or an editor never applies mqlab's rules to component code. Each
  component's own `pyproject.toml` carries its own `[tool.ruff]`, and ruff
  always uses the nearest config.
- **A boundary test in mqlab's suite**, enforced by `vrg-validate`, fails if
  `src/mqlab` imports any component package, if a component imports `mqlab`, or
  if the mqlab wheel includes anything under `components/`.

Components are deliberately **not** uv workspace members. A workspace would
share one lock across components and pull them into the root project's
resolution, coupling them to mqlab's tier.

### 5.2 The pinned runtime

- **What:** a python-build-standalone CPython 3.14 build, the same distribution
  `uv python install` uses. These are generic glibc builds, so one artifact per
  arch serves Ubuntu and RHEL (to be confirmed by the T2 spike, §8).
  - **Data:** release `20261003` (published 2026-10-03) ships
    `cpython-3.14.8+20261003-{x86_64,aarch64}-unknown-linux-gnu-install_only.tar.gz`
    (per the release's `SHA256SUMS`:
    <https://github.com/astral-sh/python-build-standalone/releases/download/20261003/SHA256SUMS>).
- **Where it is pinned:** a new `runtime:` key in `lab/versions.yaml`, the
  lab's single hand-written version catalog (epic `#280`), validated eagerly by
  `src/mqlab/versions.py`:

  ```yaml
  runtime:
    python:
      version: "3.14.8"
      pbs_release: "20261003"
      sha256:
        x86_64: "371b6c281bbb09b29279e9e3a2996bab4ae2ea03cca52bf869f8bd89286b0ae8"
        aarch64: "abc0c8dd54a144909a5e4905737bc33f244cb17ce8afc4a7b0791ee1e006ae23"
  ```

  The hashes are those of the `install_only` artifacts, copied from the
  release's `SHA256SUMS`. A missing or malformed entry fails loudly.
- **Policy: the contract is the minor version; each build pins the patch.**
  Components declare `requires-python = "==3.14.*"`. Moving to a new patch is a
  one-file PR: component builds rerun on the new interpreter, and the next bake
  picks it up. Nothing drifts silently.
- **Where it lives on guests:** `/opt/vergil/cpython-3.14/`. It is a
  Vergil-provided runtime (the provider-name rule is in §5.3), keyed on the
  **minor** version, so a patch bump upgrades in place and existing venvs keep
  working, as a distro package upgrade would. A future 3.15 installs alongside
  it. The directory name `cpython-3.14` is provisional until M2 names the
  runtime's repo; it is recorded in `#355` as an alignment constraint.
- **Ansible's own interpreter is unchanged.**
  `ansible_python_interpreter=/usr/bin/python3` stays for the apt/dnf modules.
  The pinned interpreter is never Ansible's, and the system Python is never a
  component's.

### 5.3 Install layout

Path rule: `/opt/<org>/<repo>/`, where `<org>` is the GitHub org name with any
`-project` suffix removed (`vergil-project` → `vergil`, `logical-minds-foundry`
→ `logical-minds-foundry`, `mq-rest-admin-project` → `mq-rest-admin`). The
component name is the repo name: one component per repo by default, to be
revisited if that stops fitting. This follows the FHS conventions for `/opt`
add-on packages.

```text
/opt/vergil/cpython-3.14/                         pinned interpreter
/opt/logical-minds-foundry/<component>/venv/      component venv + entry points
/opt/logical-minds-foundry/<component>/venv.prev/ previous version (rollback)
/opt/logical-minds-foundry/<component>/INSTALLED.json
/etc/opt/logical-minds-foundry/<component>/       deployer-owned config (*.env)
/var/opt/logical-minds-foundry/<component>/       component state, if any
/usr/lib/systemd/system/<unit>                    component-owned static units
```

Existing operator-facing paths that other systems depend on, such as the node
exporter textfile drop zone, stay where they are (§5.4 rule 6).

### 5.4 The component contract

This is language-neutral in principle. The Python form:

```text
components/<name>/
  pyproject.toml   name = <name>, own version, requires-python = "==3.14.*",
                   hatchling backend, [project.scripts] entry points,
                   own [tool.ruff] / [tool.pytest] config
  uv.lock          this component's lock only
  src/<import_pkg>/
  tests/
  systemd/         static unit files (no templating)
  README.md        purpose, entry points, config keys
```

Rules:

1. **Never imports the lab or Vergil.** No `mqlab` imports, no sibling-file
   imports, no `sys.path` manipulation. The mqlab boundary test enforces this.
2. **Every runnable is a console-script entry point.** Units, playbooks and
   operators call `/opt/logical-minds-foundry/<name>/venv/bin/<entry-point>`.
   Nothing runs `python file.py`.
3. **Units belong to the component; configuration belongs to the deployer.**
   Unit files are static and are installed to `/usr/lib/systemd/system/`.
   Site-specific values (today's Jinja-injected `--role`, `--qm`, and so on)
   come from `EnvironmentFile=/etc/opt/logical-minds-foundry/<name>/<unit>.env`,
   which Ansible renders. This is the split a `.deb`/`.rpm` needs in M2.
4. **Each component ships `<name>-selfcheck`.** It imports every module in the
   package, asserts the running interpreter is CPython 3.14, and prints the
   component version. Install runs it on the box and fails on any error.
5. **The install records what is installed.** `INSTALLED.json` holds the name,
   version, git sha, interpreter version and install time.
6. **Existing operator-facing names stay the same** (`lab-cluster-state`,
   `lab-nativeha-state`, `lab-loglifecycle-state`, `lab-rdqm-state`, `mq-bench`,
   the textfile drop paths). Dashboards, Prometheus and the runbooks see no
   change.

Import names are short (`mqro`, `mqrc`). They are internal leaf packages that
nothing will ever depend on.

### 5.5 The M1 components

| | `mq-resiliency-observability` (import `mqro`) | `mq-resiliency-clients` (import `mqrc`) |
|---|---|---|
| Contents | `clusterstate`, `nativehastate`, `rdqmstate`, `loglifecycle` and their tests | `app_requester`, `bench_client`, `svc_responder`, `authz_probe`, `dlq_probe`, `reconnect_probe`, `dr_flow`, `dr_responder`, `dr_mqi`, `dr_baseline`, `dr_forced`, plus `dr.*` and `header` (from `src/mqlab`) and their tests |
| Entry points | `lab-cluster-state`, `lab-nativeha-state`, `lab-loglifecycle-state`, `lab-rdqm-state`, `mq-resiliency-observability-selfcheck` | `mq-app-requester`, `mq-bench`, `mq-svc-responder`, `mq-authz-probe`, `mq-dlq-probe`, `mq-reconnect-probe`, `mq-dr-flow`, `mq-dr-responder`, `mq-dr-baseline`, `mq-dr-forced`, `mq-resiliency-clients-selfcheck` |
| Third-party deps | none (stdlib) | `pymqi`, pinned and hash-locked, compiled from sdist against the box's MQ SDK |
| Hosts | pcmk, san, nativeha, rdqm nodes | app-client, svc-sim |
| Replaces | the `/usr/local/bin` copies, the duplicate `nativehastate.py`, the `loglifecycle` import fallback, the cut-down `mqlab` on RDQM | `mqvenv` (rebuilt each provision), `rvenv`, loose scripts in `~/` and `/var/mqm`; deploys the DR clients for the first time (fixes `dr-run.sh` on svc-sim) |

The directory name `mq-resiliency-observability` matches the existing retired
repo on purpose: that is where the collectors are meant to end up. In M1 the
lab directory is the source of truth. Reconciling it with the diverged `mqro`
repo code is M3's job.

### 5.6 Build: `mqlab component build <name>`

Runs on the dev VM. It is the **only** producer of installable artifacts.

1. Read `runtime.python` from `lab/versions.yaml`. Ensure the exact pinned
   interpreter is available locally (`uv python install`), and verify the
   interpreter build against the pin.
2. In `components/<name>/`, run `uv lock --check`. A stale lock refuses the
   build.
3. Run `uv run --frozen --python <pinned interpreter> pytest`. Any failure
   refuses the build.
4. Refuse if `components/<name>/` has uncommitted changes, because an artifact
   must correspond to a commit.
5. Stage to `$(mqlab build path cache)/components/<name>/<version>+<sha>/`:
   - `source.tar.gz`: `git archive` of `components/<name>` at HEAD;
   - `requirements.txt`: `uv export --frozen --no-dev --no-emit-project`,
     hash-pinned;
   - `deps/`: every locked dependency artifact, sdists included, so installs
     never reach PyPI and offline RHEL works;
   - `BUILD.json`: name, version, sha, runtime pin and test result.
6. Point `components/<name>/current` in the cache at the new artifact.

`mqlab component status [--host …]` reads `BUILD.json` locally and
`INSTALLED.json` on guests, and reports what is built and what is running where,
so the question can be answered without the AI.

### 5.7 Install: the `component-install` role and `mqlab component install`

One Ansible role, used both by the bake plays and by `mqlab component install
<name> [--host …]`. There is no second install path.

1. **Precondition:** `/opt/vergil/cpython-3.14/` exists and matches the pin. If
   not, fail with a message naming the `mqlab` command that fixes it.
2. Copy the staged artifact (`current`, or an explicit version) to the guest.
3. Build `venv.new` on the pinned interpreter. Install dependencies with
   `--require-hashes --no-index --find-links deps/`, then the project with
   `--no-deps`. `pymqi` compiles here, against the MQ SDK.
4. Run `<name>-selfcheck` from `venv.new`. On failure, **leave the live venv
   untouched**, report, and fail.
5. Swap atomically: the current venv becomes `venv.prev` and `venv.new` becomes
   `venv`. Write `INSTALLED.json`, install the static units, `daemon-reload`,
   and restart the component's units (left inert during a bake, live on a
   running lab).

The `runtime-install` role lays down the interpreter from `build/cache`, verifies
its sha256 against the pin, and is idempotent.

### 5.8 Bake and the dev loop

- **Baked:** the interpreter, each component's dependencies (including the
  pymqi compile) and the component itself, at the `current` artifact. A cold
  provision needs nothing else.
- **Recorded:** each fat box records in its box metadata which component
  versions it baked. The runtime pin and the component artifacts feed the box
  manifest hash (`lab/boxes/_manifest-hash.sh`), so a changed component or pin
  marks the box stale.
- **Dev loop:** `mqlab component install <name> --host …` reruns the same
  recipe against a running lab and picks up new code in seconds. In M3 this
  operation becomes `apt install`/`dnf upgrade` of a newer package version.
- The clients' build toolchain (`gcc`, headers) is needed only in the bake.
  Whether to remove it afterwards is a box-hygiene choice for the plan.

## 6. Testing and guards

| Layer | What runs | Where | Gate |
|---|---|---|---|
| Component tests | the component's own pytest suite (moved tests, plus new client tests including `svc_responder`, which has none today) | `mqlab component build`, on the pinned interpreter | build refuses to stage |
| Interpreter identity | build asserts the test interpreter is the pinned build; `selfcheck` asserts CPython 3.14 on the box | build and every install | build or install fails |
| On-box smoke | `<name>-selfcheck` imports every module with the deployed interpreter, on the actual box, before the swap | `component-install` | live venv untouched; install fails |
| mqlab tests | `component build/install/status`, pin parsing in `versions.py`, the boundary test | `vrg-container-run -- vrg-validate` (100% branch) | PR cannot merge |
| Live acceptance | cold rebuilds on both arches (§9) | validation tasks | epic cannot close |

**Component coverage bar:** moved tests keep the coverage they have. New
component code gets real unit tests, with no mandated percentage; the 100% gate
arrives with vergilizing at extraction.

The `#293` failure would now stop the build (tests run on 3.14) **and** fail the
install, because the box proves it can import everything before going live.
It would no longer fail silently in a service log.

## 7. Error handling

No silent failures and no fallbacks:

- Every refusal names its cause and the fully-qualified `mqlab` command that
  fixes it. Underlying tools (`uv`, `pytest`, `systemctl`) speak for
  themselves.
- No fallback to the system Python, to PyPI, or to an older artifact.
- A failed install leaves the previous version running and says so.
- A failed component install fails the bake.
- A missing or malformed `runtime:` entry in `lab/versions.yaml` fails eagerly
  at load.

## 8. Migration sequence

Every implementation task lands in `mq-resiliency-lab-for-linux`.

| # | Task | Kind | Blocked by |
|---|---|---|---|
| T1 | Spec + plan (`.github#295`) | docs bookend | — |
| T2 | **Spike: runtime feasibility.** python-build-standalone 3.14.8 runs on RHEL 9.6 (x86_64) and Ubuntu 24.04 (both arches); `pymqi` compiles from sdist into a venv on that interpreter against the MQ SDK; an offline, hash-pinned install from staged artifacts works on RHEL. Report in `docs/reports/`. | impl | T1 |
| T3 | Boundary foundations: `components/`, the `runtime:` pin in `lab/versions.yaml`, root ruff `extend-exclude`, `.ansible-lint.yml` `exclude_paths`, the boundary test, and a developer doc for the component contract | impl | T2 |
| T4 | `mqlab component build` + `status` | impl | T3 |
| T5 | `runtime-install` + `component-install` roles + `mqlab component install` | impl | T4 |
| T6 | Carve out `mq-resiliency-observability`; delete the `src/mqlab` copies | impl | T3 |
| T7 | Rewire the collector roles (`cluster-state`, `nativeha-state`, `rdqm-state`) and their bake plays | impl | T5, T6 |
| T8 | Carve out `mq-resiliency-clients` (including `dr.*`, `header`, a `svc_responder` test) | impl | T3 |
| T9 | Rewire the client roles and playbooks (`mq-client`, `app-requester`, `bench-client`, `mq-inter-qm`, the authz/DLQ validate plays, `dr-run.sh`); remove the mqvenv/rvenv creation | impl | T5, T8 |
| V1 | Cold rebuild, Ubuntu stacks (pcmk-ubuntu, nativeha-ubuntu), local arm64 host | validation | T7, T9 |
| V2 | Cold rebuild, RHEL stacks (rdqm, nha-rhel-crr), x86 cloud host | validation | T7, T9 |
| — | Documentation review (`mq-resiliency-lab-for-linux#1348`) | bookend | V1, V2 |
| — | Follow-on brainstorm: M3 (`.github#296`) | bookend | V1, V2 |
| — | Retrospective (`.github#297`) | terminal bookend | all |

After T3 there are two parallel tracks: T4 → T5 (machinery) and T6/T8
(carve-outs).

**Cutover is per component, with no dual running.** Each rewire task replaces
the old delivery path in the same PR and **removes** the old artifacts from
existing guests: the `/usr/local/bin` collectors, `/usr/local/lib/lab-rdqm-state`,
`mqvenv`, `rvenv` and the loose scripts. A re-provisioned lab therefore cannot
keep running stale copies. The dashboards stay dark until V1/V2; that was a
deliberate choice over an interim patch.

## 9. Validation

Both validation tasks are cold rebuilds (the repo's acceptance gate for
bring-up and provisioning changes). Each checks:

- Prometheus holds `cluster_*`, Native HA and RDQM collector series, and the
  cockpits are populated;
- every component unit is active, and `mqlab component status` shows the
  installed versions matching the build on every host;
- the app requester and bench run from the clients component;
- `dr-run.sh` runs on app-client **and** svc-sim;
- none of the removed paths exist on any guest.

V1 covers the Ubuntu stacks on the local arm64 host. V2 covers the RHEL stacks
on the x86 cloud host, since the RHEL boxes are x86_64-only.

## 10. Relationships

- **`.github#280` (OS axis):** paused on this epic; it resumes after V1/V2. Its
  Phase 2/3 OSes inherit the pinned runtime for free, because the interpreter is
  identical across OS versions.
- **`vergil-project/.github#355` (M2):** an alignment comment records the
  decisions M2 consumes: the runtime path `/opt/vergil/cpython-3.14/`, the
  `/opt/<org>/<repo>/` rule, the static-unit + `/etc/opt` config split, and the
  `venv.new` → selfcheck → swap recipe.
- **`.github#293`:** closed as promoted into this epic.
- **Superseded records:** the "deployed verbatim onto system Python" decision in
  `docs/plans/2026-06-14-cluster-cockpit-plan-1a-collector.md`,
  `docs/plans/2026-06-18-nativeha-cockpit-build-pr1-collector.md`,
  `docs/specs/2026-06-18-nativeha-cockpit-design.md`,
  `docs/specs/2026-06-19-rdqm-cockpit-design.md` and
  `docs/reports/2026-07-15-cli-only-mq-metrics-discovery.md`. Supersession notes
  are added in the documentation-review sweep.
- **Extraction roadmap** (`docs/specs/2026-06-27-component-extraction-roadmap-design.md`):
  M1 supplies its missing "how does the lab consume a component" step in
  from-source form. M3 completes it with published packages.

## 11. Risks, data and judgment

| Claim | Status | Resolution |
|---|---|---|
| python-build-standalone 3.14.8 exists for x86_64 and aarch64 linux-gnu | **data** (the release's `SHA256SUMS`, checked 2026-10-05) | — |
| One python-build-standalone glibc build runs on both RHEL 9.6 and Ubuntu 24.04 | **judgment** (how the project builds its artifacts; not verified on our boxes) | T2 spike |
| `pymqi` compiles from sdist against the MQ SDK into a venv on a python-build-standalone interpreter | **judgment** (historically, python-build-standalone sysconfig quirks have affected C-extension builds) | T2 spike |
| Fully offline, hash-pinned installs work on RHEL from staged artifacts | **judgment** | T2 spike |
| `vrg-validate` ignores `components/` except for ansible-lint | **data** (vergil-tooling `lib/languages.py`, `bin/validate_common.py`, as installed) | T3 config + boundary test |
| The mqlab CLI imports none of the modules leaving `src/mqlab` | **data** (grep on `develop` @ `700fe7e`) | T6/T8 boundary test |

If the T2 spike disproves a judgment row, the plan stops and the spec is revised
before T3. There is no workaround by fallback.
