# Multi-version OS axis — design spec

- **Epic:** `logical-minds-foundry/.github#280`
- **Design task:** `logical-minds-foundry/.github#281`
- **Follow-on (seeded):** `logical-minds-foundry/.github#282`, the MQ major version axis
- **Member repo:** `logical-minds-foundry/mq-resiliency-lab-for-linux`
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-10-02

## 1. Problem and motivation

Ubuntu's newest LTS has moved from 24.04 to 26.04. The lab was built entirely on
Ubuntu 24.04 and RHEL 9.6, and each family is **hard-coded to a single version**.

The lab exists so people can bring up an IBM MQ HA/DR technology and experiment
with it. Real enterprises build both the release they run today and the release
they are moving to. So the lab must build a given stack on a **chosen OS major
version**, keep the previous release easy to build, and grow from one version to
many as releases arrive. RHEL 11 will join RHEL 9 and 10; old majors are removed
deliberately, not by default.

### 1.1 Current state (verified against `develop` @ `8bb5313`)

- **Ubuntu is pinned to 24.04 everywhere and nothing mentions 26.04.** Box names
  are inconsistent:
  - Some carry the version: `infra-ubuntu2404`, `obs-ubuntu2404`, `mq-ubuntu2404`.
  - Some carry only the family: `mq-nativeha-ubuntu`, `pcmk-ubuntu`.

  The default platform is hard-coded at `src/mqlab/platforms.py:60-62`. Box pins
  live in `build/work/box-versions.json`, which has no writer in `src/`.
- **RHEL is pinned to 9.6 and RHEL 10 is not supported.** The pin appears in:
  - Base box `rhel/9.6-x86_64`, built by `lab/boxes/rhel96/build-box.sh`.
  - `el9/...` paths in `ansible/roles/rdqm-install/tasks/install.yml`.
  - "RHEL 9.6 DVD" repo literals.
  - The ISO staging scripts.

  The only RHEL 10 material is the research note
  `docs/reference/mq10-rhel9-vs-rhel10-and-rdqm-support.md` (#1095/#1096). It
  concludes that RDQM has **no validated DRBD kernel module for RHEL 10**, and that
  RHEL 10 needs **x86-64-v3** CPUs.
- **The box-to-base mapping is duplicated** in hand-written `case` tables in
  `lab/boxes/build-fatbox.sh:76-83` and `lab/boxes/_manifest-hash.sh:67-74`, and it
  is mirrored again in `src/mqlab/manifest.py` (`_OS_PREFIX`) and
  `src/mqlab/cli.py` (`_LOCAL_BOX_BUILDERS`).
- **The OS is part of stack identity.** `lab/topology.yaml` defines four static
  stacks: `rdqm-rhel`, `nativeha-rhel-crr`, `nativeha-ubuntu` and `pcmk-ubuntu`.
  Each node names a concrete box. The stack's `os:` field is only a label; it is
  used for the aarch64 gate (`src/mqlab/stacks.py:139-161`) and dashboard
  folders.
- **Only the MQ cluster nodes vary by OS.** Every shared node uses Ubuntu 24.04
  boxes, whichever stack runs: `obs`, `infra-svc`, `infra-client`, `svc-sim`,
  `app-client` and `mon-probe`. The `pcmk-ubuntu` SAN targets (`san-a`, `san-b`)
  are also always Ubuntu.
- **Some shared nodes run MQ** (correction, #285). `svc-sim` runs a queue manager
  (SVCQM, the shared counterparty), and `app-client` and `mon-probe` run MQ client
  software (the requester app, the client-mode exporters). They all boot the
  MQ-commons box, which is role `mq-client` in this design. The other shared nodes
  (`infra-svc`, `infra-client`, `obs`, and the SANs) run no MQ product.
- **The MQ version is a single global pin.** `lab/mq-version` (`10.0.0.0`) is read
  by mqlab, Ansible, fetch scripts and the bake manifest hash. It is not
  selectable.

## 2. Goals

1. Build each stack on any **supported OS major version**:

   | Stack | Supported | Default |
   |---|---|---|
   | `nativeha-ubuntu` | ubuntu 24, 26 | ubuntu 24 until IBM lists 26.04, then 26 |
   | `pcmk-ubuntu` | ubuntu 24, 26 | ubuntu 24 until IBM lists 26.04, then 26 |
   | `nativeha-rhel-crr` | rhel 9, 10 | rhel 10 |
   | `rdqm-rhel` | rhel 9 | rhel 9 (no RHEL 10 DRBD kmod) |

   IBM MQ 10.0 lists Ubuntu 24.04 only; 26.04 has no row yet
   (`docs/reference/os-version-support-matrix.md`, #1271). So the §4.1 support gate
   keeps both Ubuntu defaults on 24 for now, and 26 is selectable as lab-only.

2. Shared nodes are split by **whether they run MQ** (correction, #285), and
   neither group offers a version choice:
   - **No MQ** (`infra-svc`, `infra-client`, `obs`, `san-a`, `san-b`) → **Ubuntu 26**
     now.
   - **Runs MQ** (`svc-sim`, `app-client`, `mon-probe`; role `mq-client`) → stays on
     the **IBM-listed Ubuntu** (24), and moves to 26 only when IBM lists 26.04.
3. **Stack identity stays version-free.** A stack name says *what* is built; the
   OS version is configuration of that build.
4. Version tokens appear only in the catalog (`lab/versions.yaml`) and in
   resolved, OS-bearing artifact names (boxes, box caches, base boxes, ISOs), plus
   the per-version vars and kickstart files that exist precisely to differ by
   version. The §6 guardrail enforces this.
5. The design admits the **MQ major version** as a second axis with no redesign
   (§8).

## 3. Non-goals

- Choosing the MQ major version. It is designed for here (§8) and delivered by the
  follow-on epic planned in #282.
- Running two instances of the same stack at once. Builds are sequential: bring
  up, experiment, tear down, rebuild on another version. Node names, IPs, inventory
  groups and QM names do not change.
- Cross-version comparison tooling, and automating the human dashboard review.
- The `-crr` naming question.
- Interim (non-LTS) Ubuntu releases, and RHEL on arm64. The arm64 RHEL refusal is
  unchanged.

## 4. Design

### 4.1 The catalog: `lab/versions.yaml`

A committed file and the **only** place OS version tokens are written by hand. It
absorbs `build/work/box-versions.json`. Illustrative shape:

```yaml
os:
  ubuntu:
    24: { base_box: cloud-image/ubuntu-24.04, box_version: "20260518.0.0" }
    26: { base_box: cloud-image/ubuntu-26.04, box_version: "<pinned by S1>" }
  rhel:
    9:  { point: "9.6", iso: rhel-9.6-x86_64-dvd.iso }
    10: { point: "<pinned by S2>", iso: "rhel-<point>-x86_64-dvd.iso",
          requires: [x86-64-v3] }
infra: ubuntu:26            # shared nodes WITHOUT MQ + pcmk SANs; not selectable
infra_mq: ubuntu:24         # shared nodes WITH MQ (role mq-client); support-gated
stacks:
  nativeha-ubuntu:   { supported: [ubuntu:24, ubuntu:26], default: ubuntu:24 }
  pcmk-ubuntu:       { supported: [ubuntu:24, ubuntu:26], default: ubuntu:24 }
  nativeha-rhel-crr: { supported: [rhel:9, rhel:10],      default: rhel:10 }
  rdqm-rhel:         { supported: [rhel:9],               default: rhel:9 }
```

- **Defaults are declared per stack, not per family.** A family default cannot
  express "RDQM stays on RHEL 9".
- **The support gate.** An entry may carry
  `ibm_support: { status: unsupported, source: <url> }`. Such a version is
  selectable as lab-only, and the resolver prints a warning. **A stack's default
  never points at an IBM-unsupported version.** The rule is to gate the default
  flip, not the support (decided in the brainstorm). **The same gate applies to
  `infra_mq`**: shared nodes that run MQ never default to an IBM-unsupported
  version (correction, #285). `infra` (no MQ) is not gated.
- **What counts as IBM-supported.** A version is supported for the gate only when
  IBM's SPCR (Software Product Compatibility Report) for the pinned MQ 10.0.x
  release has a row for it, e.g. "Ubuntu 26.04 LTS". The checkable method is
  recorded in `docs/reference/os-version-support-matrix.md`. Moving the Ubuntu
  defaults (T9) and moving `infra_mq` to 26 each wait for that row.
- **ARM64 scope note.** IBM ships MQ on Linux ARM64 only as the Developer edition
  ("not suitable for production use … no formal IBM support"), so the SPCR has no
  ARM64 rows. The gate is evaluated against the x86-64 rows. Lab results on arm64
  are representative but unsupported, and findings that depend on production
  support must be confirmed on x86-64.
- The unused `alma9-x86_64` registry entry is dropped in the move.

### 4.2 The build file

Off-default versions are selected **only** through a file:
`mqlab bootstrap <stack> --config f.yaml`.

```yaml
os: rhel:9        # follow-on epic adds:  mq: 9
```

- Keys left out fall back to the stack's default.
- Unknown keys are an error.
- There is no `--os` CLI shorthand: there is one way in, and it extends to the
  MQ axis unchanged.

### 4.3 The resolver: `src/mqlab/versions.py`, a layer in the existing pipeline

The lab **already has a host resolver**:

- `src/mqlab/platforms.py` `resolve` maps each node's `platform:` to a box and
  chooses the provider mechanics (driver, firmware, cpu_mode, machine type).
- `ensure_resolved` renders **every node** into
  `build/work/lab/topology.resolved.yaml`.
- `lab/Vagrantfile` is a dumb consumer of that file (#276).

`versions.py` does **not** replace this pipeline and does not run alongside it.
It is the **version layer in front of it**:

1. `versions.py` owns the catalog, the build file and the instance records. For
   each node it works out the role, the OS major and the concrete box name and pin:
   - From the owning stack's instance record when one exists.
   - Otherwise from the stack's default.
   - For shared nodes, from `infra`; for shared nodes that run MQ (role
     `mq-client`), from `infra_mq` (#285).
2. `platforms.resolve` takes that per-node box selection as input instead of a
   `platform:` key. It keeps sole ownership of the provider mechanics.
3. The rendered file carries the box-version pin per node. `box-versions.json` and
   the Vagrantfile's lookup of it by `platform` are removed. The Vagrantfile stays a
   dumb consumer.

The render covers all stacks at once, so it **combines every stack's record**. Two
stacks may run at once on different OS majors, for example `nativeha-ubuntu` on 24
next to `pcmk-ubuntu` on 26.

Given **stack + optional build file + host arch**, the version layer produces a
**resolved spec** containing:

- Each node's concrete box name. Shared nodes resolve against `infra`, and the
  MQ-bearing `mq-client` role against `infra_mq`.
- The per-version facts the bake and provisioning steps need: base box and
  version pin, RHEL point release and ISO.

The resolver **fails loudly, naming the fully-qualified `mqlab` remedy**, on:

- A version unsupported for the stack.
- A family mismatch (`rhel` asked of an Ubuntu stack).
- A host that cannot run it: RHEL on arm64, or RHEL 10 without x86-64-v3.
- A malformed or unknown build-file key.

It is the single source for every consumer. The `case` tables in
`build-fatbox.sh` and `_manifest-hash.sh` are deleted, and
`manifest.py`/`cli.py` stop carrying their own box maps.

### 4.4 The instance record

Bootstrap writes `build/state/instances/<stack>.json`, containing the resolved
spec plus the build-file inputs. It writes the record **before** rendering the
resolved topology, so Vagrant sees the selected boxes from the first `vagrant up`.

- Every later command on that stack reads it: `bootstrap --from/--only`, `status`,
  `teardown`, box checks and dashboard OS labels.
- A `--config` that disagrees with an existing record is **refused** until
  `mqlab teardown <stack>`, so a running stack cannot silently switch versions
  mid-life.
- **A live stack with no record is refused.** If a stack's domains are running but
  it has no record, every command on it **except `teardown`** refuses with an
  instruction to run `mqlab teardown <stack>` and re-bootstrap. This covers every
  stack running when Phase 1 lands, and any lost record. `teardown` works without a
  record because it destroys domains by name and needs no version. The lab never
  falls back to the stack default for a running instance: after a default flip,
  that would mislabel a 24.04 stack as 26 and bake the wrong boxes.
- Teardown deletes the record.
- It lives in `build/state/` (shared, irreplaceable live-lab facts) and is reached
  through the build-layout API, never a hard-coded path.

### 4.5 Topology

`lab/topology.yaml` stops naming concrete boxes:

- Nodes declare a **box role**: `box: mq-nativeha | pcmk | mq-rdqm | infra | obs |
  mq-client | san`.
- Stacks declare `os_family: ubuntu | rhel`.

No version tokens remain. The existing `MQLAB_ENV` overlay mechanism is unchanged
and orthogonal.

### 4.6 Naming rule

- **Physical (contains a fixed OS) → carries the short major.**
  - Boxes are `<role>-<os><major>`: `infra-ubuntu26`, `obs-ubuntu26`,
    `mq-client-ubuntu26`, `san-ubuntu26`, `mq-nativeha-ubuntu24`, `pcmk-ubuntu26`,
    `mq-rdqm-rhel9`, `mq-nativeha-rhel10`.
  - Caches are `build/state/boxes/<box>-<arch>.box`.
  - RHEL base boxes are `rhel/<major>-x86_64`.
  - Build domains and ISO artifacts follow the same rule.

  The point release is a catalog pin, so 9.6 → 9.7 is a re-pin and re-bake, never
  a rename. Ubuntu is LTS-only, so `26` is unambiguous.
- **Logical → never versioned.** Stacks, nodes, inventory groups, QM names,
  playbooks and dashboards. Playbooks may carry the OS **family**, which is not a
  version token (`bake-nativeha-ubuntu.yml`, never `bake-nativeha-ubuntu24.yml`).
  The per-family playbook split that already exists is kept; merging the playbooks
  is out of scope.

### 4.7 Box baking

- **`lab/boxes/rhel96/` becomes `lab/boxes/rhel/`**, parameterized by major. It
  uses a separate kickstart per major only where the majors actually differ. ISO
  staging (`stage-rhel-iso.sh`, `push-rhel-iso.sh`) takes the major. **RHEL DVDs
  are human-supplied licensed media**, as RHEL 9.6 is today.
- `build-fatbox.sh` takes its role, base and ISO inputs from the resolver.
- **`mqlab box build`** accepts a concrete box name
  (`mqlab box build mq-nativeha-rhel10`) or `--config f.yaml`, which bakes every
  box that build needs. Bootstrap's existing box check bakes or flags what the
  resolved spec needs.
- **The manifest hash** includes the OS major and its point or box pin, so a
  re-pin forces a re-bake.

### 4.7.1 The SAN targets become a baked box (supersedes `.github#108`)

Today `san-a`/`san-b` boot the bare Ubuntu base box. They install `drbd-utils`,
`targetcli-fb` and `linux-modules-extra-*` from a deb pre-cache that
`src/mqlab/sandeb.py` fills by running `apt-get download` **on the controller**.

That downloads the *controller's* release. The controller (the Vergil VM) is
Ubuntu 24.04. Once the SANs move to 26, cache hits would install noble packages on
26.04 SANs and skip the roles' network fallback, with no warning.

The #108 brainstorm rejected a SAN box as disproportionate (two payload-light VMs).
This epic changes that trade-off:

- A box is now a catalog role plus a bake playbook.
- Baking installs on the target OS itself, which removes the controller-release
  coupling.
- Baking also fixes the existing `linux-modules-extra` kernel cache miss, because
  the module is installed against the box's own kernel.

So:

- **New `san` box role**, resolved from `infra` (Ubuntu 26), giving
  `san-ubuntu26-<arch>`.
- **New `bake-san.yml`**, which runs the install half of `drbd-san` and
  `iscsi-target`.
- **Removed:** `sandeb.py`, the SAN deb cache, and the roles' cache-copy and
  network-fallback branches. The roles keep their configure half. Install tasks
  are skipped when the package is already baked, following the existing pattern.
- **Validation:** covered by the `pcmk-ubuntu` validation rows (§6).

The SAN path is a first-class demonstration arm, not a secondary concern.

### 4.8 Ansible indirection

- Bake and site playbooks are version-free. OS-family dispatch stays as is
  (`install-{{ ansible_os_family }}.yml`).
- Version-specific values move to role `vars/<Distribution>-<major>.yml`. They are
  loaded through one shared include that **fails when no file matches**. There is
  no silent fallback to a family default, so an unsupported OS cannot
  half-provision.
- Known first targets:
  - The `rdqm-install` `el9/...` paths and DVD repo literals.
  - The `mq-nativeha` and `mq-nativeha-spike` RedHat repo literals.
  - Ubuntu package assumptions in `pcmk-stonith` (`fence-agents-virsh`),
    `drbd-san` (`linux-modules-extra-*`), `snapd-off`, `motd-off`,
    `grafana-image-renderer` and `data-prepper`, each re-checked against 26.04.
- `host-resolver` comments that assume "RHEL 9 has no systemd-resolved" are
  re-checked for RHEL 10.

### 4.9 One-time migration

- Every existing box name changes, so existing caches go stale.
- `mqlab box gc` learns the retired names and removes them and their libvirt pool
  volumes. The retired names are `*-ubuntu2404`, `mq-ubuntu2404` (now
  `mq-client-ubuntu<N>`), `mq-nativeha-ubuntu`, `pcmk-ubuntu`, and the base box
  `rhel/9.6-x86_64` (now `rhel/9-x86_64`). `mq-rdqm-rhel9` and
  `mq-nativeha-rhel9` already fit the rule and keep their names.
- The first post-merge run re-bakes. That cost is accepted once.
- Tests that hard-code `ubuntu2404` (about 250 hits across `tests/`) move to
  resolver fixtures.

## 5. Phases

| Phase | Content | Proven by |
|---|---|---|
| **0: Spikes** | **S1** `cloud-image/ubuntu-26.04` libvirt boxes exist (amd64 + arm64) and boot under our Vagrant. **S2** IBM's support statement for MQ 10 (server + Native HA) on Ubuntu 26.04 and RHEL 10, fetched via `tools/ibm_doc_cache.py`, cited, with data kept apart from judgment; plus availability of Pacemaker, DRBD and `fence-agents-virsh` on 26.04. Pins the RHEL 10 point release. **S3** lab host KVM exposes x86-64-v3 to guests. | Recorded findings; go/no-go per family |
| **1: Resolver refactor** | Catalog, build file, resolver, instance record, role-based topology, renames, version-free playbooks, vars indirection, `box gc` migration, version-token guardrail, all at **today's versions** (ubuntu24/rhel9). No behavior change, with one deliberate exception: the SAN targets move from cached debs to the baked `san` box (§4.7.1), first as `san-ubuntu24`, so the pcmk-ubuntu regression rebuild proves it before the re-pin to 26. | Cold rebuild of every stack |
| **2: Ubuntu 26** | Catalog entry and role fix-ups; shared nodes **without MQ** and the SANs move to 26 (`infra`); `mq-client` stays on 24 (`infra_mq`); both Ubuntu stacks build on 24 and 26; the default flip and the `infra_mq` move wait for an SPCR 26.04 row (§4.1). | Validation rows below |
| **3: RHEL 10** | RHEL 10 base box, per-major vars; `nativeha-rhel-crr` builds on 9 and 10; default flips to 10 (subject to §4.1 gate). `rdqm-rhel` stays on 9. | Validation rows below |

If a spike returns **no** for a family, that family's phase pauses with the
finding recorded on the epic. The other family proceeds independently. If S2
finds IBM does not support a pairing, the version still ships as lab-only and the
default does not flip (§4.1).

## 6. Validation

Cold-rebuild **validation** tasks (operational, not PR-workable). Each is a full
DR bring-up plus the existing end-to-end check, **with a human dashboard review**:

| Stack | Versions |
|---|---|
| `nativeha-ubuntu` | 24, 26 |
| `pcmk-ubuntu` | 24, 26 |
| `nativeha-rhel-crr` | 9, 10 |
| `rdqm-rhel` | 9 |

The Phase 1 regression rebuild covers the 24 and 9 rows. The 26 and 10 rows run
after Phases 2 and 3, and they also prove that the shared nodes work on Ubuntu 26.

**Mixed-version re-validation.** After the non-MQ shared nodes and SANs move to
26, a default 24 build always runs against 26 shared nodes (the `mq-client` nodes
stay on 24 under `infra_mq`). So each Ubuntu stack is
re-validated at 24 against the 26 shared nodes. `pcmk-ubuntu` at 24 matters most:
its cluster nodes on 24 are iSCSI initiators to DRBD-backed SAN targets on 26.
Additionally:

- The unit tests prove the resolver's failure modes: unsupported version, family
  mismatch, host gate, bad key, and record conflict.
- **Version-token guardrail.** A pytest guardrail, following the logsearch-ref
  guardrail pattern (f0b86c2), so `vrg-validate` runs it.
  - **Scans:** `src/`, `lab/`, `ansible/` and `scripts/` for OS version tokens
    (`ubuntu24`, `2404`, `noble`, `rhel9`, `9.6`, `el9`, and the 26/10 forms).
  - **Exempt:** `lab/versions.yaml`, per-major kickstart files,
    `ansible/**/vars/<Distribution>-<major>.yml`, and one explicit allow-list
    entry for `box gc`'s retired-names table.
  - **Excluded:** `docs/` and `tests/`.

## 7. Error handling

All new failure paths are loud and actionable, following the layered error
vocabulary: mqlab's own messages name fully-qualified `mqlab` commands. Nothing
falls back silently:

- The resolver refuses rather than guessing a default for a bad request.
- The Ansible vars include fails on an unmatched OS.
- A record conflict blocks the command.
- An IBM-unsupported selection warns with its citation.

## 8. Forward compatibility: the MQ axis

The follow-on epic (#282) adds `mq: <major>` to the build file. It adds a
per-major catalog section, replacing the single global `lab/mq-version` with
per-major pins plus a default, and adds an MQ segment to MQ-bearing box names
(e.g. `mq-nativeha-ubuntu26-mq10`). The resolver, record, `--config` interface and
support gate are reused unchanged. This epic does **not** add the MQ segment.
Box names for MQ-bearing roles are generated by the resolver, so appending it
later is a single-point change.
