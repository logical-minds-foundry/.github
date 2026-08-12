# Arch-native box building — Retrospective

- **Epic:** `logical-minds-foundry/.github#103`
- **Retrospective task:** `logical-minds-foundry/.github#142`
- **Partners:** `spec.md` · `plan.md` (read spec → plan → retrospective)
- **Span:** opened 2026-07-20 → closed 2026-07-27 (7 days, two working sessions a week apart)
- **Date:** 2026-07-27

## §0 At a glance

We set out to **restore host-arch-native fat-box building** — regressed when the
fat-box baking work (#70/#659) pinned the Ubuntu fat boxes `arch: x86_64`, forcing
slow x86 emulation on Apple Silicon — and to **prove it by baking the two Ubuntu
arms (`mq-nativeha-ubuntu`, `pcmk-ubuntu`) natively on both architectures**. Both
shipped: `platforms.box_build_arch` is now the single build-arch authority, the
orchestrator threads a required `--arch` into `build-fatbox.sh`, the box cache is
uniformly `<box>-<arch>.box`, and the two Ubuntu arms cold-rebuild green booting
their **baked native** boxes on **both** arm64 (Apple Silicon) and x86 (cloud).

### Work delivered

| Repos touched | 2 — `logical-minds-foundry/.github` (spec/plan/retrospective), `mq-resiliency-lab-for-linux` (all code + docs) |
|---|---|
| Sub-issues | 23 total, all closed (7 impl, 4 operational deploy/validate, 7 arch-native fix tasks, 3 docs/brainstorm bookends + docs-review, + this retrospective) |
| PRs merged | 15 across the two repos |
| Releases cut | none (lab repo; not a released artifact) |

### Core PRs

| PR | Task | What it did |
|---|---|---|
| `.github#109` | #106 | Published the spec + plan |
| #711 | T1 #699 | `box_build_arch` + `is_foreign_box_build` — the host-arch build authority |
| #715 | T2 #700 | Orchestrator passes `--arch`; refuses foreign-arch builds |
| #725 | T3 #701 | `build-fatbox.sh` arch-aware guest domain + cache via required `--arch` |
| #716 | T4 #702 | `box.py` arch-aware cache + `build migrate` → `<box>-<arch>.box` |
| #717 | T5 #703 | Un-pin the Ubuntu fat boxes to host-resolved; arch-aware MQ acquisition |
| #726 | T6 #704 | Bake `mq-nativeha-ubuntu` + skip-if-baked; repoint nha-ubuntu nodes |
| #728 | T7 #705 | Bake `pcmk-ubuntu` cluster nodes + skip-if-baked; SAN nodes unchanged |
| #749 | #698 | Docs-review: dev + site docs reflect arch-native box building |

### Fix PRs (all surfaced by driving the real bakes — see §1)

| PR | Task | What it fixed |
|---|---|---|
| #720 | #719 | `test_cache_artifact_names` host-dependent after the un-pin (broke develop on arm64) |
| #730 | #727 | Arch-aware Ubuntu bake-media pre-flight (X64-literal) |
| #733 | #731/#732 | Dry-run must pass `--arch`; build domain guest-arch-aware (arm64 ran TCG, not native KVM) |
| #735 | #734 | `_manifest-hash.sh` box→stem map missing the new Ubuntu arms |
| #738 | #736 | arm64 transient build domain needs UEFI (AAVMF) firmware |
| #748 | #737 | Standalone `mqlab box build` must render the resolved topology first |

### Operational tasks (no PR — proven by running, recorded as comments)

| Task | Host | Outcome |
|---|---|---|
| D-arm #706 | Apple Silicon (arm64) | Both arm64 arms baked/cached/registered |
| V-arm #708 | Apple Silicon (arm64) | Cold rebuild booted the baked arm64 arms (scoped — see §3) |
| D-x86 #707 | cloud x86 | Both x86 arms baked/cached/registered |
| V-x86 #709 | cloud x86 | Cold rebuild of **both** arms fully green (native boot + MQ-install skip + HA formation) |

## §1 How the plan evolved

The plan was **agnostic-code first, operational-proof second** — seven PR-workable
tasks (T1–T7) building the arch authority + cache + builder, then four operational
tasks baking and cold-rebuilding on each host. That sequencing held, and it is the
reason the epic succeeded: **the code tasks all merged clean on 2026-07-20, and
then reality arrived when the bakes actually ran.**

The largest plan↔reality delta was not in the design but in what the unit tests
could not see. T1–T7 shipped at 100% branch coverage, yet driving **D-arm** (the
first real bake, on Apple Silicon) immediately surfaced a cluster of genuine
arch-native defects: the dry-run decision seam never forwarded `--arch` and the
build domain wasn't guest-arch-aware, so arm64 Ubuntu bakes silently ran **TCG
emulation instead of native KVM** (#731/#732); the manifest-hash box→stem map
didn't know the new Ubuntu arms (#734); the arm64 transient build domain needed
**UEFI (AAVMF)** firmware that x86 never required (#736); and standalone
`mqlab box build` didn't render the resolved topology before the builder ran
(#737). A host-dependent test also broke `develop` on arm64 the moment the boxes
were un-pinned (#719). Every one of these was invisible to the suite because the
tests **monkeypatched the very builder seam** the defects lived in. They were
found, fixed, and merged within the epic — the "two native bakes are the
acceptance" clause in the plan did exactly the job it was written for.

The other deviation was time and place. Because the work is intrinsically
**two-platform**, the operational half split across two hosts and two sessions: the
arm64 deploy+validate ran on 2026-07-20 (Apple Silicon), and the x86 deploy+validate
ran a week later on the cloud x86 host (this session, 2026-07-26/27). The
merged-vs-deployed model — deployment tasks gating validations via `Blocked-by` —
sequenced that cleanly without either host blocking the other.

## §2 Lessons learned

- **For cross-arch/infra work, the live bake *is* the test — coverage is not.**
  100% branch coverage passed while arm64 bakes ran fully emulated. The defects
  lived in the exact seam the unit tests stubbed. Planning the two native bakes as
  the acceptance gate (not the unit suite) is what caught them; repeat that pattern
  for any "works on the other architecture/host" claim.
- **Two arch vocabularies must stay separated.** libvirt/qemu speak
  `aarch64`/`x86_64`; Vagrant `box add --architecture` speaks `arm64`/`amd64`.
  Keeping `--arch` canonical and mapping only at the `build-fatbox.sh` seam
  (a Global Constraint in the plan) avoided a whole class of confusion.
- **A fail-loud foreign-build guard beats silent emulation.** Refusing the RHEL
  box on arm64 (`is_foreign_box_build`) turned "burns a core, no networking,
  timeout" into an immediate, legible error.

## §3 Compromises & tradeoffs

- **V-arm was accepted on scoped evidence, not a fully-green one-pass.** A
  non-#103 mqweb/ansible-core drift (#741) ended the arm64 bootstrap in the commons
  provisioning after the arch-native criteria were already proven; it was routed to
  the reliability epic (#104) and V-arm accepted by the epic owner. Vindicated this
  session: with #741 fixed, **V-x86 ran fully green**, so the scoping was a timing
  call, not a gap.
- **RHEL stays x86-only.** RHEL is emulated on arm64 (unused in practice); both
  RHEL arms remain x86-build-only by design (a non-goal, not a shortfall).
- **The `build migrate` build-layout collision** (`build/state/vagrant already
  exists`) is a pre-existing environment wart worked around in both deploys, not
  fixed here — it is unrelated to the arch-suffixed box-cache migration #103 owns.

## §4 New problems & opportunities

- **The arch-native defect cluster (#719, #727, #731, #732, #734, #736, #737)** —
  all found by the bakes and all fixed within the epic.
- **mqweb/ansible-core 2.21 drift (#741)** → reliability epic **#104** (fixed;
  its fix let V-x86 go fully green).
- **`pcmk-rhel` decommission** — the arm was deliberately dropped from Part B
  rather than baked; the closing brainstorm #107 promoted this into its own small
  epic **#138** (spec + plan authored, docs PR pending human submit).
- **SAN-host optimization** — closing brainstorm #108 concluded the SAN targets
  should **stay host-resolved** (baking a `san-ubuntu` box was disproportionate);
  the win is a deb pre-cache of the SAN install-half, filed as task
  `mq-resiliency-lab-for-linux#796`.
- **Credentialed RHEL DVD acquisition** — logged as idea **#105**, not yet acted on.

## §5 What's next

- **Epic #138 — decommission the `pcmk-rhel` arm** (from #107): tasks #794 (code)
  / #795 (docs) under an authored spec + plan; docs PR report-ready.
- **Task #796 — pre-cache the SAN install-half debs** (from #108).
- **Idea #105 — automate credentialed RHEL DVD acquisition** (backlog).

## Appendix A — Operational notes

The bake acceptance is intrinsically **two-host and split**; run order per arch:

1. **arm64 (Apple Silicon):** D-arm #706 — `mqlab build migrate` (no-op if no legacy
   cache), then `mqlab box build mq-nativeha-ubuntu pcmk-ubuntu` (`--arch aarch64`,
   native KVM + UEFI); then V-arm #708 — `mqlab bootstrap nativeha-ubuntu` cold
   rebuild. Ran 2026-07-20.
2. **x86 (cloud):** D-x86 #707 — same with `--arch x86_64` (native KVM); then
   V-x86 #709 — cold rebuild of **both** `nativeha-ubuntu` and `pcmk-ubuntu`. Ran
   2026-07-26/27 (this session), both fully green: native boot of the baked boxes,
   per-run MQ install skipped by the `cmqc.h` guard, Native HA quorum (3/3, INSYNC)
   and the Pacemaker `mq_group` promoted.

Gotchas carried forward: the `build/state/vagrant` migrate collision (§3); RHEL
re-bakes are only needed if the box-cache migration renamed a RHEL box (it did not,
on either host).
