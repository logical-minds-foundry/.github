# M3: the lab consumes published component packages — design spec

- **Epic:** `logical-minds-foundry/.github#319`
- **Design task:** `logical-minds-foundry/.github#320`
- **Origin:** follow-on brainstorm `.github#296` (epic `.github#294`, M1)
- **Builds on:** M1 `logical-minds-foundry/.github#294` (in-lab components);
  M2 `vergil-project/.github#356` (binary package publishing)
- **Follow-ons (seeded):** `.github#321`, a config-driven package-org registry;
  `.github#324`, a holistic key and credential rotation strategy
- **Member repos:** `logical-minds-foundry/mq-resiliency-lab-for-linux`,
  `logical-minds-foundry/mq-resiliency-observability`,
  `logical-minds-foundry/packages` (new); cross-org `vergil-project/vergil-tooling`
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-10-11

## 1. Problem and motivation

The lab has two purposes. It is a working environment for IBM MQ HA/DR technology,
and, as the human's primary goal, it **demonstrates an engineering methodology**:
prototype components quickly inside the lab, then turn them into real products.

M1 (`#294`) made every guest component a standalone uv project under `components/`,
built and tested on one pinned runtime and installed from source with standard
tooling. M2 (`vergil-project/.github#356`) gave Vergil-managed repos signed `.deb`/`.rpm`
publishing into per-org package repositories. M3 closes the loop: the lab installs a
**published product it built**, exactly as an outside user would. It keeps a second
component in the lab so the two tiers are visible side by side.

### 1.1 Current state (verified 2026-10-11)

- **Lab** (`mq-resiliency-lab-for-linux` develop, after #294's validations):
  - `components/mq-resiliency-observability` (import `mqro`, stdlib-only) and
    `components/mq-resiliency-clients` (import `mqrc`, pymqi in the `mqi` extra) are both
    installed by the in-lab `component-install` role. That role uses the `releases/` +
    atomic `venv` symlink layout under `/opt/logical-minds-foundry/<name>/`, with config in
    `/etc/opt/logical-minds-foundry/<name>/`.
  - The runtime is python-build-standalone 3.14.8, staged as a tarball by
    `runtime-install` into `/opt/vergil/cpython-3.14/`, pinned with sha256 in
    `lab/versions.yaml` `runtime.python`.
- **M2 as built**, per the #356 spec and retrospective and vergil-tooling `268458ac8`:
  - **One package per repo, built from the repo root.** `[package]` in `vergil.toml`
    (`builder`, `vendor`, `summary`, `smoke`, optional `targets`/`native`); `config.py:326`,
    `naming.py:24`. Subdirectory or monorepo packaging is not supported.
  - Python apps ship a bundled venv at `/opt/<vendor>/<name>/venv`, built from
    `uv.lock --frozen`. Commands are shimmed into `/usr/bin`. The app depends on the
    runtime package.
  - The runtime package is `vergil-python3.14.8`, installed at
    `/opt/vergil/python/3.14.8/`, one package per patch, side by side. It is never on
    PATH, and it is always published by the **vergil** org.
  - Per-org repositories are on GitHub Pages
    (`https://vergil-project.github.io/packages`). Signing uses an offline primary key and
    a CI subkey; consumers install `<org>-archive-keyring`. Targets are Ubuntu 24.04/26.04
    and RHEL 9/10, each on amd64 and arm64.
  - Package orgs are registered **in code**, in vergil-tooling `lib/package/orgs.py`.
    Only `vergil` is registered, and an unknown vendor is fatal.
  - Packages are produced only for stable `vX.Y.Z` releases from `main`.
- **logical-minds-foundry has no package repository.** M2 left setting one up to this
  org.
- **`mq-resiliency-observability` (repo)** holds the first extraction attempt (#79):
  diverged `mqro` code, a README claiming packages that were never built, and version
  `0.1.0`.

## 2. Decisions (brainstorm 2026-10-11)

| # | Decision |
|---|---|
| D1 | **Extract only `mq-resiliency-observability`.** `mq-resiliency-clients` deliberately stays an in-lab component, so the lab demonstrates both tiers. **No monorepo:** one repo publishes one package, so M2 is not extended for subdirectories. |
| D2 | **The extraction is an import that overwrites the product repo.** The first attempt is treated as a failed iteration. History is not carried; provenance is recorded instead. |
| D3 | **One runtime everywhere:** the `vergil-python3.14.8` package. M1's tarball `runtime-install` and its pin are retired. |
| D4 | **The paths visible from outside are aligned across tiers:** `/opt/logical-minds-foundry/<name>/venv` (what units call) and `/etc/logical-minds-foundry/<name>/` (deployer config). The in-lab tier keeps `releases/` behind the `venv` symlink. Extraction changes only how a component is installed. |
| D5 | **Packages come straight from the published repositories, with exact versions pinned** in `lab/versions.yaml`. The pins feed the box manifest hash. |
| D6 | **No dev override for extracted components:** release-to-test. Post-extraction products are expected to be largely in maintenance mode. The pre-release channel is `vergil-project/.github#370` (deferred). |
| D7 | **Trust anchors are per-user config, never code:** `~/.config/vergil/package-trust.toml` (the `authorized_keys` analogy), authored on the host with vergil-tooling and synced into VMs. The lab commits only which orgs and packages it consumes. A missing anchor fails loudly. |
| D8 | **Registering logical-minds-foundry in vergil-tooling `orgs.py` is interim (tactical).** The follow-on `#321` brainstorms a config-driven registry rooted at `~/.config/vergil/`. |
| D9 | **Versions:** `0.2.x` while the lab dogfoods the product, then `1.0.0` after M3's validations (the extraction roadmap's "1.0 only after the lab dogfoods it" rule). |

## 3. Goals

1. `mq-resiliency-observability` is a fully Vergil-managed product repo that releases signed
   `.deb`/`.rpm` into a logical-minds-foundry package repository.
2. The lab installs it from that repository at the exact pinned version, with keys
   verified against per-user trust anchors, on every observability host.
3. `mq-resiliency-clients` stays in-lab, on the same published runtime and the same
   external paths.
4. One glass-box view (`mqlab component status`) shows both tiers.
5. Native HA stacks pass one-pass cold rebuilds on both arches; then `1.0.0` ships.

## 4. Non-goals

- Extracting `mq-resiliency-clients`, or moving it to `ibmmq` (that is epic `#300`).
- Monorepo or subdirectory packaging in M2.
- A pre-release package channel (`vergil-project/.github#370`).
- An automated extraction tool (a future follow-on).
- The config-driven org registry itself (`#321`). M3 adds only the interim `orgs.py`
  entry.
- Making RDQM or Pacemaker stacks blocking. They are best effort; Native HA is primary
  (repo memory `native-ha-is-primary`).

## 5. Design

### 5.1 Org package infrastructure (logical-minds-foundry)

This mirrors M2's own stand-up (OP1, `vergil-project/vergil-tooling#3073`, and its P1/DEP
tasks):

- **OP1 (human-attested deployment):**
  - an offline primary (certify-only) key and a CI signing subkey for
    logical-minds-foundry;
  - create `logical-minds-foundry/packages`, with Pages served from Actions;
  - a `package-signing` environment, restricted to `main`, in `mq-resiliency-observability`
    and `packages`;
  - an `index-signing` environment, restricted to `develop`, in `packages`;
  - confirm the org's GitHub App can dispatch to `packages`.

  The SUCCESS comment records the 40-hex primary fingerprint and the armored public key.
  **The agent never handles key material.**
- **KR/PUB:** `logical-minds-foundry/packages` runs `publish-index`, and the
  `logical-minds-foundry-archive-keyring` package is built and published exactly as
  vergil's equivalents are. The plan's first step reads how vergil's are built and
  mirrors that.
- **REG (cross-org, vergil-tooling):** an interim `ORGS["logical-minds-foundry"]` entry
  (vendor, `github_org`, `base_url=https://logical-minds-foundry.github.io/packages`,
  fingerprint from OP1, `keyring_package`), plus a vergil-tooling release (human-gated).
  This is needed by `vrg-release`'s package-index stage, which looks up `pkg.vendor`.

### 5.2 Extraction of `mq-resiliency-observability`

- **Import:** one feature-branch PR on the product repo **replaces the entire contents**
  with the lab's `components/mq-resiliency-observability` tree, plus the Vergil
  scaffolding. The commit message and README record provenance: the lab commit, the
  component's git tree hash, and the lineage `#79 → #294 → #319`.
- **Vergil scaffolding:**
  - `vergil.toml`: `[project]`, and `[package] builder="python"
    vendor="logical-minds-foundry"`, with a summary, `smoke`
    (`mq-resiliency-observability-selfcheck`), `[package.python] runtime="3.14.8"`, and
    targets covering Ubuntu 24/26 and RHEL 9/10 on both arches. The component is
    stdlib-only, so there is no native build.
  - **Units ship in the package** through `packaging/nfpm.overlay.yaml` into
    `/usr/lib/systemd/system`, with `EnvironmentFile=/etc/logical-minds-foundry/mq-resiliency-observability/<unit>.env`.
    The package ships **no** `.env` files; the deployer (the lab's Ansible) owns config.
  - The full Vergil quality bar: `vrg-validate`, 100% branch coverage, CI, and docs.
- **Releases:** `0.2.0` is the first, for dogfooding. `1.0.0` follows after the M3
  validations (D9).

### 5.3 Lab: trust, repos and pins

- **`lab/versions.yaml`** gains a `packages:` block:

  ```yaml
  packages:
    repos:                         # org -> repository URL (NO fingerprints: D7)
      vergil:                { url: "https://vergil-project.github.io/packages" }
      logical-minds-foundry: { url: "https://logical-minds-foundry.github.io/packages" }
    pins:
      vergil-python3.14.8:         "3.14.8+20261003-1"
      mq-resiliency-observability: "0.2.0-1"
  ```

  `runtime.python` (the tarball pin) is removed. `versions.py` validates the block eagerly.
- **Trust anchors:** `~/.config/vergil/package-trust.toml`, with one entry per org (its
  primary fingerprint). The file is **authored on the host, owned by vergil-tooling**,
  and synced into VMs by TRUST-SYNC (below). The lab is only one consumer of package
  trust, so the anchors belong to the user's Vergil config, not to the lab.
  - On the host, `vrg-package trust add <org> <fingerprint>` and `vrg-package trust list`
    write and show it (TRUST-SYNC).
  - In the lab, `mqlab package trust list` is **read-only**: it shows the synced anchors
    and which orgs the lab needs.
  - `mqlab` passes the anchors to Ansible. A missing file or entry **fails before anything
    is trusted**, naming the file, the org, the host command (`vrg-package trust add`)
    and the sync step (re-enter with `vrg-vm session`). There is no fallback.
- **The `package-repos` role (new)** uses the **standard third-party repository recipe**
  (the one vendors such as Docker and HashiCorp document), with the fingerprint check
  automated. For each org it:
  1. fetches the org's published key, `<url>/keys/<org>.asc`, over HTTPS (on every run,
     so a re-published key under the same primary is picked up);
  2. **verifies the key's primary fingerprint against the per-user anchor before writing
     any repository source**, and fails loudly on a mismatch;
  3. installs the verified key as the repository's key file and adds the apt source
     (`signed-by`) or yum `.repo` (`gpgkey`, `gpgcheck=1`, `repo_gpgcheck=1`).

  Ansible's native modules do this where they can (the plan confirms which, e.g.
  `rpm_key`'s fingerprint check, `deb822_repository`, `yum_repository`). A re-run on a
  host that is already trusted changes nothing, and a structural test pins "verify before
  any source is written".
  - **The lab does not install `<org>-archive-keyring`.** That package is an optional
    convention for delivering key rotation through package updates, and it is what forces
    M2's consumer bootstrap into a temporary-source dance (the keyring lives inside the
    signed repository). The orgs still publish it for consumers who want it. Rotation is
    deferred to the holistic strategy in `#324`; under D7 a new primary key needs a new
    anchor anyway.
- **TRUST-SYNC (cross-org, vergil-tooling)** has two parts:
  - **`vrg-package trust add|list`** on the host writes and shows
    `~/.config/vergil/package-trust.toml`. It validates the 40-hex fingerprint and refuses
    to silently replace an existing anchor; replacing one is an explicit `--replace`.
  - **`vrg-vm` mirrors the host file into the VM** on create, start, rebuild and restart
    (the credentials stage) and on every `vrg-vm session` entry. If the host has no file,
    the VM copy is removed; the host is authoritative.

  Until TRUST-SYNC is released, the file can be written by hand in the VM; it does not
  survive a dev-VM rebuild.

### 5.4 Lab: consuming the extracted component

- **Install:** an `apt`/`dnf` install of the **exact pinned version** (the pinned runtime
  comes in as a dependency) at **bake**, and again at **provision**. That makes a pin bump
  plus provision a valid upgrade path without a rebake, **from the first packaged install
  onward**.
- **M1 → M3 is a cold rebuild, never an in-place upgrade.** On an M1 host,
  `/opt/logical-minds-foundry/mq-resiliency-observability/venv` is a symlink into
  `releases/<id>/venv`; `dpkg`/`rpm` would follow it and write the package's files into
  the old release directory. So provision **refuses** to install the package over an
  in-lab layout (a `venv` symlink or a `releases/` directory) and names
  `mqlab box build`. Fresh M3 boxes never carry that layout.
- **Roles** `cluster-state`, `nativeha-state`, `rdqm-state`:
  - the "baked?" check becomes **"the installed version equals the pin"**
    (`dpkg-query -W`/`rpm -q`), failing with a message that names `mqlab box build` or the
    pin;
  - env files are rendered to `/etc/logical-minds-foundry/mq-resiliency-observability/`;
  - the package's units are enabled.
- **Bakes:** the boxes that bake observability (pcmk, san, mq-nativeha, mq-rdqm) run
  `package-repos` and then the pinned install. `roles.<role>.components` in
  `lab/versions.yaml` distinguishes packaged components from in-lab ones (form decided
  in the plan).
- **Removal:** `components/mq-resiliency-observability` leaves the lab **in the same PR**
  that switches the roles to the package. The roles do not migrate old in-lab installs in
  place (see the cold-rebuild rule above). `components/README.md` gains an "extracted
  components" note naming the product repo.
- **`mqlab component install <extracted>`** refuses, naming the real path: bump the pin,
  then provision or rebake.

### 5.5 Lab: in-lab tier aligned (clients)

- `component-install` builds the venv on `/opt/vergil/python/3.14.8/bin/python3.14`
  (installed by `package-repos` plus the pinned `vergil-python3.14.8`). The `releases/`
  layout, atomic flip, both selfchecks and pruning are unchanged.
- **`runtime-install` is retired**, along with `/opt/vergil/cpython-3.14/`. Nothing removes
  it in place: M1 → M3 is a cold rebuild (§5.4), so fresh boxes never carry it. Config
  moves to `/etc/logical-minds-foundry/mq-resiliency-clients/`.
- `mqlab component build` tests on the same pinned runtime build. How the dev VM obtains
  it (the `vergil-python` package vs the python-build-standalone tarball, which are the
  same bytes) is decided in the plan, keeping "tested on = deployed on".

### 5.6 Staleness and the glass box

- **Box manifest hash:** each box folds in the **pinned versions of the packages it
  installs** plus the **tree hashes of the in-lab components it builds**. A pin bump
  rebakes exactly the boxes that use it.
- **`mqlab component status`:**
  - packaged rows show `installed <ver>` vs `pin <ver>` (`dpkg-query`/`rpm -q`);
  - in-lab rows keep `INSTALLED.json` vs `CURRENT`;
  - every row is labeled with its tier (`packaged`/`in-lab`).

## 6. Error handling

- Missing trust anchors, a fingerprint mismatch, an unpinned package, or an installed
  version that differs from the pin each **fail loudly** with the fix named (an `mqlab`
  command, the config file, or the pin). There are no fallbacks: never to an unsigned
  repo, an unpinned version, or the retired tarball runtime.
- An in-lab layout where the package would install refuses the install and names
  `mqlab box build` (§5.4).
- A packaged install that fails at bake fails the bake. At provision it fails the play,
  and the previous package version keeps running (package-manager atomicity).

## 7. Sequencing

| # | Task | Kind | Repo | Waits for |
|---|---|---|---|---|
| D | Spec + plan | docs bookend | LMF `.github` | — |
| OP1 | Key ceremony and trust roots | deployment (human-attested) | LMF `.github` | D |
| KR/PUB | Packages repo (`publish-index`) and keyring package | code | `logical-minds-foundry/packages` | OP1 |
| REG ⟂ | Interim `orgs.py` entry + release | code + human-gated release | vergil-tooling | OP1 |
| TRUST-SYNC ⟂ | `vrg-package trust add\|list` on the host; `vrg-vm` mirrors `package-trust.toml` into VMs; release | code + human-gated release | vergil-tooling | D |
| X1 | Import (overwrite) + Vergil scaffolding | code | `mq-resiliency-observability` | D |
| X2 | Release `0.2.0`, confirmed in the LMF repo | deployment (release human-gated) | `mq-resiliency-observability` | X1, REG, KR/PUB |
| L1 | Trust and repos (`mqlab package trust list`, `packages:`, `package-repos`) | code | lab | D |
| L2 | In-lab tier on `vergil-python3.14.8`; retire `runtime-install`; `/etc/logical-minds-foundry` | code | lab | L1 |
| L3 | Observability from the package; remove the in-lab copy (same PR) | code | lab | L2, X2 |
| V1 | Cold rebuild, macOS arm64 (`nativeha-ubuntu` on `ubuntu:24` must pass; `pcmk-ubuntu`, whose SAN nodes run the `ubuntu:26` infra OS, best effort) | validation | lab | L2, L3, TRUST-SYNC |
| V2 | Cold rebuild, x86 cloud (`nativeha-rhel-crr` on `rhel:10` must pass; `rdqm-rhel` on `rhel:9` best effort) | validation | lab | L2, L3, TRUST-SYNC |
| X3 | Release `1.0.0`; bump the lab pin; reprovision check | deployment + code | product + lab | V1, V2 |
| — | Documentation review (multi-repo sweep) | bookend | lab (`#1444`) | X3 |
| — | Follow-on brainstorm: config-driven org registry | bookend | LMF `.github` (`#321`) | X3 |
| — | Follow-on brainstorm: holistic key and credential rotation | bookend | LMF `.github` (`#324`) | X3 |
| — | Retrospective | terminal | LMF `.github` (`#322`) | all |

⟂ marks a cross-org task, filed in vergil-project/vergil-tooling. It is not a GitHub
sub-issue of `#319`: the epic body lists it, and dependent tasks carry dependency notes.

L3 switches observability to the package in a single PR, so develop is never left
without a working observability install (unlike `#294`'s T6–T9 chain).

Live validation proves the must-pass OS for each arm (`ubuntu:24` arm64, `rhel:10`
x86_64). Package-level coverage of every target (Ubuntu 24/26 and RHEL 9/10, both arches)
comes from M2's per-target install test in the product's release pipeline: install,
`systemd-analyze verify` on the shipped units, the smoke command, purge, and a residue
check (vergil-tooling `install_test.py`).

## 8. Validation

Each cold rebuild checks the following:

- On every observability host, the pinned `mq-resiliency-observability` version is
  installed **from the signed LMF repository** (`apt-cache policy`/`dnf info` show the
  origin). Its units load from `/usr/lib`, and its collector series are in Prometheus.
- The clients run in-lab on `vergil-python3.14.8`. `/opt/vergil/cpython-3.14` and
  `/etc/opt/logical-minds-foundry` are absent everywhere.
- `mqlab component status` shows both tiers correctly.
- With a trust anchor removed, `mqlab` fails **before** trusting any repository, and names
  the fix. With a wrong anchor, `package-repos` fails before writing any repository source.
- The bootstrap succeeds in one pass. Native HA stacks must pass; RDQM and Pacemaker are
  best effort.

## 9. Relationships

- **`#294` (M1):** its retrospective (`#297`) records M3 as its follow-on.
- **`vergil-project/.github#356` (M2):** M3 is its first external-org consumer.
- **`vergil-project/.github#370`:** the pre-release channel (deferred).
- **`#321`:** the config-driven registry, which removes REG's interim code entry.
- **`#324`:** the holistic key and credential rotation strategy, which decides whether
  the lab should adopt keyring packages (or an equivalent) after all.
- **`#300` (pymqi → `ibmmq`):** independent. Clients stay in-lab either way.

## 10. Risks, data and judgment

| Claim | Status | Resolution |
|---|---|---|
| M2 packages one package per repo from the repo root | **data** (vergil-tooling `config.py:326`, `naming.py:24`) | D1 |
| The runtime package is `vergil-python3.14.8` at `/opt/vergil/python/3.14.8/` | **data** (#356 spec, `python_builder.py:63-71`) | D3 |
| `orgs.py` registers only `vergil`; an unknown vendor is fatal | **data** (`orgs.py:27-44`) | REG |
| M2's python builder can ship systemd units through the nfpm overlay and run a smoke command for a library-style product repo | **judgment** | the plan's first step verifies it against vergil-tooling and the #356 docs |
| vergil's keyring package and packages repo can be mirrored for a second org without vergil-tooling changes beyond REG | **judgment** | the plan's first step reads OP1, P1 and DEP1 from #356 |
| M2's consumer bootstrap (`repo_setup.bootstrap`) is Python in vergil-tooling, takes its fingerprint only from `orgs.py`, and exists to bootstrap the keyring package | **data** (`repo_setup.py:231-308`) | L1 uses the standard recipe instead (§5.3) |
| The standard recipe, without the keyring package, costs the lab little: same-primary key changes are re-fetched each run, and a new primary needs a new anchor under D7 anyway | **judgment** | `#324` revisits it holistically |
| M2 install-tests every target cell (install, unit verify, smoke, purge, residue) | **data** (vergil-tooling `install_test.py:1-21`) | §7 coverage note |
| `vrg-vm` can be taught to sync one more `~/.config/vergil` file | **data**: the credentials stage (`vrg_vm.py:629-631`, in the create/start/rebuild stage lists and the cloud create list), `_cmd_restart` (`:1192`) and `_cmd_session` (`:2570`, which already copies Claude config on every entry) all have a guest transport | TRUST-SYNC |
| The anchors must be authored on the host: `mqlab` runs only inside the disposable dev VM, while `vrg-vm` reads `~/.config/vergil` on the host (as it does `identities.toml`) | **data** (found while planning, 2026-10-12; human chose host-side authoring) | D7, TRUST-SYNC |
