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
- **Status:** design (brainstorm output 2026-07-20), pending pushback + review
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
- **D4 — Uniform arch-suffixed cache with migration.** `build/state/boxes/`
  artifacts become `<box>-<arch>.box` and `<box>-<arch>.manifest-hash` for **all**
  boxes. A one-time migration renames the existing (x86_64) cache entries; ship it
  through `mqlab build migrate` so a returning operator's cache is carried forward,
  not orphaned.
- **D5 — `box.py`/`box status` become arch-aware.** `BoxSpec` gains an `arch`;
  `cache_artifact`/manifest-hash/`box status` key on it. On a given host, `box
  build`/`box status` operate on the host's native arch; the shared cache may also
  hold the *other* arch's artifact (built on the other host) and must not be
  clobbered.
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

- **RHEL native-arm64 / RHEL box building on ARM.** RHEL boxes are x86-build-only;
  emulated-x86 RHEL baking on Apple Silicon is impractical and unsupported for now.
  The uniform `<box>-<arch>` namespace (D4) deliberately preserves the option to add
  an `-arm64` RHEL sibling later with no scheme change — a door left open, not a
  deliverable.
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
  the boot tests. Reachable under 100% branch coverage.
- **arm64 cold rebuild (blocking, this VM).** A one-shot cold rebuild on Apple
  Silicon bakes and boots the **arm64** `mq-nativeha-ubuntu` + `pcmk-ubuntu` cluster
  nodes natively (no emulation).
- **x86 cold rebuild (blocking, cloud).** The same on the x86 cloud host, natively.
- Both green = arch-native building restored (Part A) *and* the two arms baked
  (Part B). Lint-green is necessary but not sufficient.

## 12. Open items for the implementation plan

- The exact topology shape for host-resolved Ubuntu **fat** boxes (a single
  host-arch-resolved entry vs. arch-explicit pair) and how a cluster node references
  it — reconcile with the #276 base-box pattern.
- The `manifest._ARCH_SUFFIX` arch dimension for fat boxes (box-name-keyed today):
  make acquisition pick the right Ubuntu tarball per host without a per-arch
  box-name explosion; audit `scripts/fetch-mq.sh` and `tests/test_manifest.py`.
- The `mqlab build migrate` step for the cache rename (`<box>.box` →
  `<box>-x86_64.box`), including what to do with a mixed old/new cache.
- `box.py` FLEET/`box status` on a host that holds *both* arches in the shared cache
  (host-native default, other-arch visible-but-not-clobbered).
- Whether the cluster-node repoint needs a phased-startup or machine-id-reset
  parallel to #642/#654 for the Ubuntu arms.
