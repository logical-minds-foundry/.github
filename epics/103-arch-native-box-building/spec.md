# Arch-native box building — restore host-resolved fat-box builds (arm64/x86) — design spec

- **Epic:** `logical-minds-foundry/.github#103`
- **Design task:** `logical-minds-foundry/.github#106`
- **Origin:** the #91 closing brainstorm `logical-minds-foundry/.github#93`; surfaced
  validating #91 via a macOS `mqlab bootstrap nativeha-ubuntu` (arm64 build-fatbox
  collision).
- **Regression origin:** `logical-minds-foundry/.github#70` / `#659` (the fat-box
  `arch: x86_64` pins).
- **Builds on (does not duplicate):**
  - the host-arch resolver `src/mqlab/platforms.py` + `src/mqlab/hostfacts.py`
    (#276/#296, `docs/specs/2026-06-18-x86-host-portability-design.md`) — the single
    authority for the host-arch virtualization matrix. This spec extends that
    authority to one more consumer (the fat-box build arch), it does not add a
    second copy.
  - the KVM/TCG box-build decision `platforms.build_domain_virt` (#327,
    `docs/specs/2026-06-23-kvm-aware-box-build-design.md`) — the pattern of
    *mqlab computes, the build script consumes required args*.
  - the fat-box baking pipeline `lab/boxes/build-fatbox.sh` + per-box `bake-*.yml`
    + the manifest-hash + skip-if-baked guards (#70/#659, #667/#668, epic #88).
- **Status:** design (brainstorm output 2026-07-20; pushback-reviewed), pending human review
- **Date:** 2026-07-20

## 1. Problem & motivation

The lab is supported on **two hardware platforms** — Linux/x86 *and* Apple Silicon
(arm64). Before the fat-box baking work, every Ubuntu node booted a
**host-resolved** base box: `cloud-image/ubuntu-24.04` resolves via
`default_platform`/`lab_guests` (#276) to `ubuntu2404-arm64` or `ubuntu2404-x86_64`
per host — arch-native, automatically. Only the IBM MQ **binaries** are x86-locked;
the Ubuntu OS itself ships a native image for each arch, so an Ubuntu box builds
and boots natively on either host.

The fat-box work (#70/#659) then **pinned the Ubuntu fat boxes `arch: x86_64`**
(`obs-ubuntu2404`, `infra-ubuntu2404`, `mq-ubuntu2404` — `lab/topology.yaml`
L58-82) to bake-once-on-cloud-consume-everywhere. That single decision dropped
host-native building: on Apple Silicon these boxes are now forced to slow x86
emulation instead of native arm64.

### 1.1 Where the regression lives (root cause)

Confirmed against the current tree:

- **The pins** — `lab/topology.yaml`: `obs-ubuntu2404` (L60), `infra-ubuntu2404`
  (L70), `mq-ubuntu2404` (L82) each declare `arch: x86_64`, with explicit comments
  (L66-67, L78-79) that the pin is deliberate and *unlike* the host-resolved bare
  Ubuntu nodes. These are the three lines to un-pin.
- **The hardcoded build domain** — `lab/boxes/build-fatbox.sh:223`:
  `<os><type arch='x86_64' machine='q35'>hvm</type></os>` — the transient
  build-domain guest arch is a literal, not derived from host facts. (KVM-vs-TCG
  *is* host-derived via `--domain-type`/`--cpu-mode` from #327; the **guest arch**
  is not.) Symptom on macOS: the base Ubuntu box is added with no `--architecture`
  (L166), so Vagrant Cloud returns the arm64 image → an arm64 disk on an emulated
  x86_64 machine → burns a core, no DHCP lease, timeout.
- **The single-arch cache** — `build-fatbox.sh:95` `CACHE="$CACHE_DIR/${BOX}.box"`
  and `box.py:69` `cache_artifact=f"{name}.box"` key the cache purely on box name.
  Two arches of the same box would collide.
- **The base-image resolution** — `build-fatbox.sh:166` adds the base box with no
  `--architecture`; L173-174's `find … -name box.img` picks the last image with no
  arch discrimination.
- **MQ-media suffix** — `manifest._ARCH_SUFFIX` (`manifest.py` L22-48) maps every
  *fat* box to an x86_64 MQ tarball; there is no arm64 fat-box entry.

RHEL being x86 is correct; **Ubuntu being forced to x86 is the bug.**

## 2. Doctrine & principles

- **One authority, no bash re-derivation.** The per-box build arch is decided by a
  pure function in `platforms.py`, alongside `_provider` and `build_domain_virt`.
  `build-fatbox.sh` receives it as a **required `--arch` argument** and consumes it;
  it never probes the host or re-derives the rule (the #327 doctrine, extended).
- **Native-preferred, arch-explicit.** Guest arch tracks the host for Ubuntu
  (arm64 on Apple Silicon, x86_64 on the cloud); RHEL is x86_64 always. This is the
  #276 rule (`driver = kvm iff guest_arch == host_arch and kvm`) applied to the box
  build. Platforms stay arch-explicit — no logical sentinel (#276 D6).
- **Uniform namespace.** *Every* cached box is `<box>-<arch>.box` — RHEL always
  `-x86_64`. This removes the "is this box arch-forked?" branch from every consumer
  and, deliberately, **leaves the door open** to an `-arm64` RHEL sibling later
  without a scheme change (see §9 non-goals).
- **Reuse the proven baking pipeline.** Part B extends `build-fatbox.sh` + the
  `bake-*.yml` playbooks + the skip-if-baked guard (#667/#668); it adds no parallel
  tooling.
- **Fail loud, no silent fallback.** A missing/invalid `--arch`, an
  arm64-guest-on-x86 request, or a stale cache is an error named at the point of
  failure — never a silent emulated degrade.
- **Prove it on both native hosts.** Acceptance is a cold rebuild on *each* native
  host (arm64 Apple Silicon; x86 cloud), not a single-host claim.

## 3. Decisions (converged 2026-07-20)

- **D1 — `platforms.box_build_arch(box_spec, facts)` is the authority.** Pure,
  display-safe: RHEL fat boxes → `x86_64` always; Ubuntu fat boxes → `facts.arch`.
  Mirrors `build_domain_virt` (#327). The orchestrator computes it; the script
  consumes it.
- **D2 — Orchestrator passes `--arch`.** `cli._box_build_steps` appends
  `--arch <arm64|x86_64>` to the `build-fatbox.sh` argv, next to today's
  `--domain-type`/`--cpu-mode`. `--arch` is **required** in the script (usage-die if
  missing/invalid), one interface for human and harness (#327 D3).
- **D3 — Un-pin the Ubuntu fat boxes.** The three `obs`/`infra`/`mq-ubuntu2404`
  entries become host-arch-resolved (the #276 base-box pattern); the RHEL fat boxes
  (`mq-rdqm-rhel9`, `mq-nativeha-rhel9`) stay `x86_64`.
- **D4 — Cache filenames uniform; platform names follow #276.** Two layers, kept
  distinct:
  - *Cache artifact filenames* (`build/state/boxes/`) become `<box>-<arch>.box` +
    `<box>-<arch>.manifest-hash` for **all** boxes (RHEL always `-x86_64`). A
    one-time migration renames the existing entries via `mqlab build migrate`.
  - *Platform names* follow the #276 base-box pattern: the **Ubuntu fat boxes**
    become arch-explicit host-resolved platforms; the **RHEL fat boxes keep their
    single `mq-*-rhel9` platform name** (only the cache file gains `-x86_64`). RHEL
    node pins and the `test_*_baked_fat_box` assertions stay unchanged.
  - The uniformity is real at the cache layer and intentionally asymmetric at the
    name layer — which is exactly #276, not a new special case.
- **D5 — The cache is per-host, single-arch in practice.** `build/state/boxes/` is
  shared only across git worktrees on **one** host (symlinked to main), *not* across
  the two physical VMs — so a host holds only its own arch's boxes. The arch suffix
  buys code-uniformity (no "is this forked?" branch) and future-proofing (room for an
  `-arm64` RHEL sibling), **not** two arches coexisting in a live cache. Migration is
  therefore per-host and independent — there is no cross-host cache reconciliation.
  `box.py`/`box status` gain an `arch` on `BoxSpec` and key cache/hash/status on it.
- **D6 — MQ-media resolves per arch.** The Ubuntu fat boxes acquire/install the
  arch-matching MQ tarball (`UbuntuLinuxARM64` vs `UbuntuLinuxX64`). At bake time
  the install role already derives the suffix from `ansible_architecture` (#276 §7);
  the acquisition/caching key (`manifest._ARCH_SUFFIX`, box-name-keyed today) gains
  the arch dimension it needs (§8 open item).
- **D7 — Bake only the two Ubuntu arms, cluster nodes only.** Part B bakes
  `mq-nativeha-ubuntu` and `pcmk-ubuntu` (the MQ+Pacemaker cluster nodes) on both
  arches, mirroring #667/#668. These two arms are the load-bearing acceptance test:
  a green native bake + boot on *both* architectures = Part A landed.
- **D8 — SAN nodes stay host-resolved.** `san-a`/`san-b` (the `pcmk-ubuntu` iSCSI
  targets) carry no IBM-MQ payload, so they keep booting the host-resolved base box;
  all SAN treatment is deferred to the SAN-hosts epic (bookend `#108`).
- **D9 — `pcmk-rhel` is decommissioned, not baked.** It is dropped from Part B and
  its removal is the subject of bookend `#107`. `san-a-rhel` is removed *with* it,
  not in the SAN epic.
- **D10 — Arch-aware MQ-tarball acquisition (blocking).** A host-resolved Ubuntu fat
  box has **one name but two arch-variant tarballs**; the box-name-keyed
  `manifest._ARCH_SUFFIX` cannot express that. The acquisition layer
  (`ensure_mq_tarballs` → `tarball_name`) pre-stages a tarball into `build/mq/`
  *before* the bake — so on Apple Silicon it must stage the **host-arch**
  (`UbuntuLinuxARM64`) tarball, or the arm64 bake finds only the x86_64 one and
  fails. Resolve the suffix through the same host-facts/`box_build_arch` path the
  builder uses, so acquisition and bake agree by construction. This is a first-class
  decision, not an open item — it is the crux of the arm64 half — with a test that
  arm64 facts stage `UbuntuLinuxARM64` and x86 facts `UbuntuLinuxX64`.
- **D11 — Refuse emulated RHEL builds on ARM; deprecate, do not remove.** Building a
  RHEL fat box on an arm64 host is full TCG emulation — impractically slow (the
  kernel is emulated; the six OSes of the 3+3 HA/DR arm make it unusable). The build
  **orchestrator refuses loudly** when a RHEL box build is requested on a non-x86
  host, naming the x86 host as where to build it. The emulated-x86-on-arm path
  (`build_domain_virt`'s TCG branch, the builder's emulated route) is **deprecated
  and gated off, not deleted** — a future *standalone, non-HA/DR* RHEL lab (one box,
  not six) is a plausible reason to re-enable it. The pure resolvers stay
  display-safe (never raise); the refusal lives in the orchestrator, so `box status`
  still lists RHEL boxes on ARM (marked x86-only).

## 4. Scope

**In (this epic):**
- **Part A** — restore arch-native fat-box building (D1–D6): the `platforms`
  authority, the `--arch` plumbing, `build-fatbox.sh` arch-awareness, the uniform
  arch-suffixed cache + migration, `box.py`/`box status`, the topology un-pin, and
  the per-arch MQ-media resolution.
- **Part B** — bake `mq-nativeha-ubuntu` + `pcmk-ubuntu` cluster nodes on **both**
  arm64 and x86 (D7): two new `bake-*.yml` playbooks + skip-if-baked guards, the
  node repoint to the baked arch-resolved box, and boot tests.

**Out (deferred to bookends):**
- Decommission `pcmk-rhel` (+ `san-a-rhel`) — brainstorm bookend `#107`.
- SAN-hosts epic covering `san-a` **and** `san-b` — brainstorm bookend `#108`.
- RHEL native-arm64 building (see §9 non-goals).

## 5. The rule & resolution model

The box-build guest arch is a pure function of the box and the host:

| box family | example | build/guest arch | when |
|---|---|---|---|
| Ubuntu fat box | `mq-ubuntu2404`, `mq-nativeha-ubuntu`, `pcmk-ubuntu` | `facts.arch` (arm64 on Apple Silicon, x86_64 on cloud) | host-resolved, native |
| RHEL fat box | `mq-rdqm-rhel9`, `mq-nativeha-rhel9` | `x86_64` always | x86-build-only |

On a non-x86 host a RHEL box build is **refused** (D11), not emulated — the pure
`box_build_arch` still returns `x86_64` (display-safe), but the orchestrator declines
to run the build.

`box_build_arch` composes with the existing `build_domain_virt` (#327): the arch
selects the guest `<type arch=…>`/`qemu-system-<arch>`, and — orthogonally — the
host's KVM/TCG capability selects `--domain-type`/`--cpu-mode`. On the cloud x86
host both agree on x86 (native KVM); on Apple Silicon the Ubuntu box is arm64+KVM
and a RHEL box (if ever built there) would be x86+TCG.

## 6. Components & data flow (Part A)

```
hostfacts.probe() ─► platforms.box_build_arch(box_spec, facts) ─┐
                     platforms.build_domain_virt(facts) ─────────┤
                                                                 ▼
cli._box_build_steps ─► build-fatbox.sh --box <b> --arch <a> --domain-type <t> --cpu-mode <m>
                                          │
                                          ├─ guest domain <type arch='<a>' …>  (was hardcoded x86_64, L223)
                                          ├─ qemu-system-<a> emulator
                                          ├─ vagrant box add --architecture <a>  (was absent, L166)
                                          ├─ base box.img find, arch-scoped        (L173-174)
                                          └─ cache <box>-<a>.box / <box>-<a>.manifest-hash  (was <box>.box, L95)
```

| Unit | Change | File |
|---|---|---|
| `platforms.box_build_arch` | **new** pure authority (RHEL→x86_64; Ubuntu→host) | `src/mqlab/platforms.py` |
| `cli._box_build_steps` | append `--arch`; thread injected `facts` | `src/mqlab/cli.py` (~L1022-1042) |
| `build-fatbox.sh` | parse+validate `--arch`; arch the guest domain, emulator, base-box add, base-img find, cache name | `lab/boxes/build-fatbox.sh` |
| cache namespace | uniform `<box>-<arch>.box` + `.manifest-hash`; migration | `build-fatbox.sh` L92-96; `box.py` L69; `mqlab build migrate` |
| `box.py` / `box status` | `BoxSpec.arch`; arch-keyed cache/hash; arch column | `src/mqlab/box.py` |
| topology | un-pin the 3 Ubuntu fat boxes → host-resolved; RHEL stays x86_64 | `lab/topology.yaml` L58-82 |
| MQ-media | per-arch Ubuntu tarball resolution | `manifest.py` `_ARCH_SUFFIX`; `scripts/fetch-mq.sh` |

## 7. Part B — bake the two Ubuntu arms

Mirror the native-HA-RHEL bake (#667/#668, epic #88):

- **`bake-nativeha-ubuntu.yml` + `bake-pcmk-ubuntu.yml`** — install-body-only
  (base MQ via the install role's `tasks_from`, no crtmqm/formation/secrets),
  node-exporter baked+enabled, alloy install-half only (inert). One playbook per
  arm.
- **Skip-if-baked guard** — the `stat`-marker pattern (`/opt/mqm/inc/cmqc.h`) so a
  baked node skips the install and an un-baked base-box node still installs
  (`roles/mq-install`, `roles/mq-client` already carry the guard).
- **Repoint the cluster nodes** — the `nha-ubuntu` and `pcmk-a*/pcmk-b*` nodes move
  from host-resolved-base-box + runtime-install → the baked, **arch-resolved** fat
  box. SAN nodes (`san-a`/`san-b`) stay on the base box (D8).
- **Boot tests** — `test_<arm>_nodes_boot_the_baked_fat_box`, mirroring
  `tests/test_topology_nativeha.py::test_nativeha_rhel_nodes_boot_the_baked_fat_box`.

Each Ubuntu arm produces **two** cached boxes (`…-arm64.box`, `…-x86_64.box`), each
baked natively on its own host.

## 8. Execution model — the two-platform split

Work is bucketed by the host it must run on; GitHub issues are the platform-neutral
coordination substrate (this session coordinates; the human routes each task to the
agent on the right host — the "human operates the lab" boundary):

- **Agnostic (code)** — all of Part A's Python/shell/topology/playbooks and the
  boot tests. Written and unit-tested on *either* host: facts are injected, so the
  full matrix is reachable under the 100%-branch-coverage gate, and CI runs on x86.
  PR-workable via `issue-implement`.
- **arm64-native (Apple Silicon, this VM)** — bake + boot the **arm64** Ubuntu fat
  boxes; the Apple-Silicon cold-rebuild validation.
- **x86-native (cloud agent)** — bake + boot the **x86** Ubuntu fat boxes; the x86
  cold-rebuild validation; any RHEL-box rebuild needed for the cache migration.

## 9. Non-goals

- **RHEL box building on ARM — actively refused (D11), not merely unsupported.**
  Emulated-x86 RHEL on Apple Silicon is impractically slow, so this lab disables it:
  the builder refuses on a non-x86 host. The supporting code path is **deprecated,
  not removed** — a future *standalone, non-HA/DR* RHEL lab (a single emulated box
  rather than today's six) could re-enable it, and the uniform cache namespace (D4) +
  the deprecated path together keep that door open with no scheme change. RHEL
  native-arm64 (a genuine arm64 RHEL image) is out of scope entirely — RHEL ships x86
  only.
- **The `box.py` CLI verbs** — they stay; only the cache/resolution underneath
  forks by arch.
- **Reliability hardening of the bake/boot path** — that is the perpetual
  reliability epic (`#104`, the #90 sibling). Reliability stumbles hit here get
  worked around and filed there.
- **SAN treatment / `pcmk-rhel`** — deferred to bookends `#108` / `#107` (D8/D9).

## 10. Error handling

Fail loud, matching the repo idiom (`StepFailedError` / usage-die):

- missing/invalid `--arch` in `build-fatbox.sh` → usage message + non-zero exit,
  before any side effect (#327 D3/D5);
- a RHEL box build requested on a non-x86 host → hard refusal in the orchestrator
  (D11), naming the x86 host as where to build it;
- an arm64 guest requested on an x86 host → hard stop in `platforms` (#276 D4);
- a stale/absent arch-suffixed cache when a baked box is required → loud, naming the
  `mqlab box`/`build migrate` fix;
- `box_build_arch` itself never raises (display-safe), mirroring `resolve`.

## 11. Testing & acceptance

Two-tier, mirroring #276/#327 — but here **both tiers are achievable in-epic**
because both native hosts exist:

- **Unit (blocking).** `box_build_arch` across the matrix (RHEL→x86_64 on both
  hosts; Ubuntu→host arch) with injected `HostFacts`; the `_box_build_steps` argv
  carries `--arch`; `box.py` arch-keyed cache/hash/status; the topology un-pin and
  the boot tests. Plus the **D11 refusal** (RHEL box build on injected arm64 facts →
  refused, naming the x86 host; on x86 facts → proceeds) and **D10 acquisition**
  (arm64 facts stage `UbuntuLinuxARM64`, x86 facts `UbuntuLinuxX64`). Reachable under
  100% branch coverage.
- **arm64 cold rebuild (blocking, this VM).** A one-shot cold rebuild on Apple
  Silicon bakes and boots the **arm64** `mq-nativeha-ubuntu` + `pcmk-ubuntu` cluster
  nodes natively (no emulation).
- **x86 cold rebuild (blocking, cloud).** The same on the x86 cloud host, natively.
- Both green = arch-native building restored (Part A) *and* the two arms baked
  (Part B). Lint-green is necessary but not sufficient.

## 12. Open items for the implementation plan

- The precise topology entry for a host-resolved Ubuntu **fat** box (arch-explicit
  pair vs. single host-resolved entry) and the cluster-node reference — inside the
  #276 pattern fixed by D4 (Ubuntu arch-explicit, RHEL single-name).
- The seams to implement **D10** (`ensure_mq_tarballs` / `setup_platforms` /
  `manifest._ARCH_SUFFIX`) without a per-arch box-name explosion, plus the
  `scripts/fetch-mq.sh` and `tests/test_manifest.py` audit.
- The `mqlab build migrate` step for the per-host cache rename (`<box>.box` →
  `<box>-x86_64.box`), including a mixed old/new cache.
- The exact **D11** seam — where in the orchestrator (`_box_build_steps` /
  `_ensure_local_boxes`) the RHEL-on-non-x86 refusal lives, and the deprecation
  marker on the emulated path.
- Whether the Ubuntu cluster-node repoint needs a phased-startup or machine-id-reset
  parallel to #642/#654.
