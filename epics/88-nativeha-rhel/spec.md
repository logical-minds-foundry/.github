# Bake the native-HA RHEL arm to the RDQM bar — design spec

- **Epic:** `logical-minds-foundry/.github#88`
- **Design task:** `logical-minds-foundry/.github#89`
- **Pattern epic:** `logical-minds-foundry/.github#70` (RDQM baking — delivered)
- **Origin:** the #70 stack-baking follow-on brainstorm `logical-minds-foundry/.github#72`
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-07-16

## 1. Problem & motivation

Epic #70 proved a pipeline that bakes the slow, per-run-invariant software install
into per-role boxes: the RDQM arm went from **~85 min → ~33 min** on a one-shot
cold rebuild, reliable enough to loop (build → test → teardown → repeat), and
TLS-validated. It delivered four boxes (`mq-rdqm-rhel9`, `obs-ubuntu2404`,
`infra-ubuntu2404`, and the `mq-ubuntu2404` commons) and every reusable mechanism:
the parameterized `build-fatbox.sh`, per-box bake playbooks, manifest-hash
staleness, phased startup (#642), skip-if-baked guards (#648), and the machine-id
reset (#654).

The **native-HA RHEL arm still pays the full per-run install on every run.** Its
six nodes (`nha-rhel-a1..3`, `nha-rhel-b1..3`) boot the **bare** `rhel/9.6-x86_64`
base and install IBM MQ from the Developer tar at bootstrap-time — the same class
of repeated, identical, I/O-bound work #70 eliminated for RDQM. This epic applies
the **already-proven pipeline** to that arm: the first of the three remaining arms
(the two Ubuntu arms are the follow-on, §6).

**Success is the elimination of the per-run install work for this arm**, not a
target percentage — the same doctrine as #70. After the box lands we run a
native-HA one-shot cold rebuild and **record the observed before/after as closing
evidence** (§7), an outcome we report rather than a threshold we must hit.

**A second, equal goal: prove x86 portability in the cloud.** #70's boxes were
exercised on the cloud x86 host, but the arms as a whole still carried a cloud/
local (Apple-Silicon) split in how they were run. This epic runs the native-HA
RHEL arm **entirely in the cloud on base x86** — killing that split for this arm —
so the lab is demonstrably reproducible for anyone on a Linux/Windows x86 box, not
only on Apple Silicon. This is nearly free here: native-HA RHEL is **already**
`rhel/9.6-x86_64` (x86 is forced for the arm), so the epic is *essentially just the
baking*.

## 2. Doctrine & principles

Inherited from #70 unchanged — this epic reuses, it does not re-invent:

- **Bake once, configure every run.** Static software (the MQ product, the obs
  agents, `acl`) is image-time state and belongs in the box. Per-run, lab-specific
  state (the queue managers, the Native HA group formation, the peer set, CRR
  role, certificates) stays in provisioning.
- **Zero drift by construction.** The box is baked by the **same Ansible tasks**
  that install the software today (provision-then-snapshot), so what the box
  contains cannot diverge from what a bootstrap would have installed.
- **Reuse the proven pipeline.** Extend `build-fatbox.sh` + `mqlab` + the bake
  playbooks + the manifest-hash; add no parallel tooling.
- **Prove it in the cloud on x86.** Run and validate the whole arm on the base
  x86 cloud host — no dependence on the Apple-Silicon dev host.
- **OS currency by rebuild, not by boot-time update.** Per #70's amended §4.5
  (#639/#78): the box freezes its packages; currency comes from rebuilding the
  box, not a `dnf update` at instance-build. Native-HA RHEL keeps that discipline.

## 3. Grounding: the native-HA RHEL arm today

Established from the code:

- **Topology.** `lab/topology.yaml` defines `nha-rhel-a1..3` (site A, HA) and
  `nha-rhel-b1..3` (site B, CRR/DR), all `platform: rhel96-x86_64`, `2 vCPU /
  2 GiB`, **no `extra_disk`** (Native HA is shared-nothing raft-log replication —
  no DRBD, no drbdpool PV). Stack `nativeha-rhel` provisions via
  `ansible/site-nativeha.yml`; CRR/DR cutover via `site-nativeha-switchover.yml`.
- **Install is a bespoke shell adapter, not the shared `mq-install` role.**
  `ansible/roles/mq-nativeha/tasks/install-RedHat.yml` mounts the install DVD as
  the offline BaseOS+AppStream repo, copies + unpacks the
  `9.4.5.0-IBM-MQ-Advanced-for-Developers-LinuxX64.tar.gz` from `build/cache/mq/`,
  accepts the developer licence, `dnf install`s the **base MQ package set (no
  RDQM/DRBD)** — Runtime/Server/GSKit/Java/JRE/Web/SDK/Client/**Samples** — runs
  `setmqinst`, sets mqm ulimits, and adds `libicu`. This is **all image-time
  install work** paid per run today.
- **Formation is per-run.** `mq-nativeha/tasks/main.yml` (shared across OSes)
  seeds JSON diagnostic logging, `crtmqm -lr … -p 1414`, appends the
  `NativeHAInstance` peer set to `qm.ini` (replication on the `net-hb` NIC:9414),
  links the `mqmonitor@` systemd template from `MQSeriesSamples`, sets its
  open-files drop-in (#448), and enables/starts the instance via `mqmonitor@`
  (the site-B recovery group starts `nha_start=false` until `crr.yml` sets its
  GroupRole, #389). `tls.yml` handles replication/app TLS.
- **No kernel constraint.** Unlike RDQM (whose DRBD kmod pins the kernel to the
  baked level), Native HA replicates in MQ's own raft log — **no kernel module, no
  pin.** This is the key structural difference that makes this box the simplest of
  the RHEL MQ boxes.

The pipeline is thus **correct in mechanism and already applied to a sibling RHEL
MQ box** (`mq-rdqm-rhel9`). This epic bakes a **second, simpler** RHEL MQ box that
omits the entire RDQM/DRBD/Pacemaker stack.

## 4. The correction

### 4.1 The box: `mq-nativeha-rhel9`

A RHEL 9.6 fat box baking exactly the native-HA arm's image-time software:

| Baked (→ box image) | Configure (→ per-run) |
|---|---|
| Base IBM MQ product (`install-RedHat.yml`'s package set: Runtime/Server/GSKit/Java/JRE/Web/SDK/Client/Samples) + `setmqinst` + mqm ulimits + `libicu` | `crtmqm`, the `NativeHAInstance` peer set, `mqmonitor@` enable/start, GroupRole/CRR (`main.yml`, `crr.yml`) |
| `node-exporter` (full role — static, left **enabled**, #642 benign exception) | per-instance alloy `config.alloy`, exporter units/CCDT (observe phase) |
| `alloy` install half only (binary + unit, baked **inert** — needs per-run config) | replication/app TLS + keystores (`tls.yml`, `lab-pki` — per-run/secret) |
| `acl` (unprivileged `become` prereq — the #659 acl-stall kill) | the Native HA diagnostic *overrides*, DNS/zone data, net-reach, etc. |
| the MQ diagnostic-logging default (mqs.ini/journald seed, #282) | genuinely per-lab diagnostic overrides |

**No RDQM/DRBD/Pacemaker, no kernel pin, no `extra_disk`.** The box is materially
smaller and simpler than `mq-rdqm-rhel9`.

### 4.2 Reuse the builder

- **`lab/boxes/build-fatbox.sh`** — add one `case` arm:
  `mq-nativeha-rhel9) BASE_KIND=rhel; BASE_BOX="rhel/9.6-x86_64"; BAKE=nativeha-rhel ;;`
  (mirrors the `mq-rdqm-rhel9` arm; the RHEL base, DVD attach, and machine-id
  reset paths are already generic).
- **`ansible/bake-nativeha-rhel.yml`** — a new bake playbook mirroring
  `bake-mq-rdqm.yml`, but baking the **native-HA install *adapter*** — the install
  body only (`include_role: name=mq-nativeha, tasks_from=install-RedHat`, the way
  `bake-mq-rdqm.yml` bakes `rdqm-install` with `tasks_from: install`) — and **never
  the whole `mq-nativeha` role**, whose `main.yml` runs `crtmqm`/formation, a per-run
  concern that must not execute at bake. It also drops the RDQM-specific
  `rdqm.service` benign-startup exception (no RDQM service exists here). It bakes:
  `acl`, the base MQ install adapter, the diagnostic default, `node-exporter`
  (enabled), and `alloy` install-half (inert).
- **`lab/boxes/_manifest-hash.sh`** — register the `mq-nativeha-rhel9 →
  nativeha-rhel` box→bake-stem mapping so staleness digests the right playbook's
  role closure (#649).
- **`src/mqlab/cli.py`** — add `mq-nativeha-rhel9` to the box builders / needed-box
  set so `bootstrap nativeha-rhel` builds/REUSEs it exactly as RDQM does.

### 4.3 The install-adapter skip-if-baked seam (the one genuinely new refactor)

The skip-if-baked pattern is already proven — but **not on this arm's installer.**
Two forms exist today: `mq-install` (the Ubuntu MQ commons) short-circuits its
~700 MiB tar copy + unpack with an explicit `stat` of `/opt/mqm/inc/cmqc.h`
(`when: not mq_installed.stat.exists`) — the #648/#659 fix, added precisely because
keying only on the wiped `/tmp` unpack dir made a baked box **re-copy the media
every run**; `rdqm-install` leans on per-step `creates:` guards.

`mq-nativeha/tasks/install-RedHat.yml` has **neither of them where it counts.** It
carries `creates:` guards on unpack (`/tmp/MQServer`), licence (`/var/mqm`), and the
`dnf install` (`/opt/mqm/bin/crtmqm`) — but the **~2 GiB MQ tar copy step has no
guard at all**, the exact #659 offender: a node booting the baked box would re-copy
the developer tarball on every bootstrap, defeating the bake.

**`mq-install` cannot be reused to fix this** — it installs `ibmmq-*.deb` via `apt`
from the `UbuntuLinux` tarball and has no RHEL path at all; the RHEL install must
stay on `install-RedHat.yml` (rpm/dnf, `LinuxX64` tar). So the fix is singular and
mechanical, not a choice between roles: **apply the proven #659 pattern to
`install-RedHat.yml`** — a `stat /opt/mqm/inc/cmqc.h` short-circuit
(`when: not mq_installed.stat.exists`) guarding the copy + unpack (and, cleanly, the
whole install body). This is a binding Task-1 acceptance criterion (§7), not a
mid-epic discovery. Formation (`main.yml`) is untouched.

### 4.4 Repoint, strip, phased startup

- **Repoint** the six `nha-rhel-*` nodes from `rhel96-x86_64` to a new
  `mq-nativeha-rhel9` `boxes:` entry (mirroring the `mq-rdqm-rhel9` entry: **keeps
  the DVD** attach, since the per-run configure half still uses it as the offline
  BaseOS+AppStream repo for dep resolution; **no** kernel-pin note, **no**
  `extra_disk`).
- **Strip** the now-baked install from the per-run path (via §4.3's guard) so a
  bootstrap runs only formation + config.
- **Phased startup (#642):** services baked inert and started per-run bottom-up,
  with `node_exporter` the sole benign enabled-at-bake exception (no `rdqm.service`
  here). The `mqmonitor@{qm}` units are per-QM and stay per-run (created by
  `main.yml`), correctly unaffected by baking.

### 4.5 Kernel: no pin (refining #70's assumption)

#70's spec (§4.3, §5.3) speculated `mq-nativeha-rhel9` would be a *leading-edge
kernel* box, framed as "the two RHEL9 flavors differ by kernel." **Native HA's raft
replication uses no kernel module**, so there is **no pin to satisfy and no kernel
flavor to choose** — cut 1 simply reuses the stock `rhel/9.6-x86_64` base (the same
base `mq-rdqm-rhel9` bakes from), with no special kernel handling at all. The two
RHEL9 MQ boxes are distinct not by kernel flavor but because native-HA **omits the
RDQM/DRBD/Pacemaker stack entirely.** A deliberately leading-edge kernel remains a
*parked option* (native HA *could* track RHEL's leading edge since nothing pins
it), decided later if there's a reason — not required for cut 1.

## 5. Binding decisions (explicit)

1. **Reuse #70's provision-then-snapshot pipeline verbatim** (`build-fatbox.sh` +
   bake playbook + host-durable `build/state/boxes/` cache + `mqlab` wiring). No
   new tooling.
2. **`mq-nativeha-rhel9` has no kernel pin, no `extra_disk`, no RDQM stack.** Cut 1
   uses the stock `rhel/9.6` base; leading-edge kernel is parked (§4.5).
3. **`install-RedHat.yml` gets the #659 skip-if-baked guard** (a `stat` of
   `cmqc.h`) — the one new refactor; `mq-install` (Ubuntu-only, apt/deb) cannot be
   reused for the RHEL install (§4.3).
4. **Run and validate entirely in the cloud on base x86** — the arm is already
   `rhel/9.6-x86_64`, so this is inherent; the validation is a cloud/x86 run.
5. **OS currency by box rebuild, not boot-time update** (inherited, #639/#78).

## 6. Scope & non-goals

**In scope:** the `mq-nativeha-rhel9` box (bake playbook + `build-fatbox.sh` case +
manifest-hash mapping + `mqlab` wiring); the install-adapter skip-if-baked seam;
repointing the six `nha-rhel-*` nodes + stripping the per-run install + phased
startup; a cloud/x86 one-shot cold-rebuild validation with the closing measurement;
and proving native-HA function (HA switchover + CRR/DR) works from the baked box.

**Non-goals (explicit):**

- **The two Ubuntu arms** (native-HA-Ubuntu + pcmk) and the **x86-on-Ubuntu**
  portability vetting, and **coordination with the parallel Ubuntu
  security-hardening work** (currently entangled with the boot changes) → the
  **follow-on brainstorm bookend** `#90` (its own epic). The split-by-entanglement
  cut put native-HA RHEL first *because* it has none of that entanglement.
- **The pcmk RHEL arm** — also a follow-on (grouped with the Ubuntu/entangled work,
  or its own epic, decided at #90).
- **mqlab box-lifecycle tooling** (selective rebake, cold-boot cadence/staleness) →
  the separate epic `#91`; this epic consumes the pipeline, it does not extend the
  CLI around it.
- **Second-order concurrency / I-O sequencing** (folded from #70's #608): the
  cluster-formation-vs-replication contention is stack-agnostic and native HA's
  raft is **lighter I/O than DRBD full-resync** (no drbdpool sync at all), so this
  arm is expected to provision *faster* than RDQM. Tune holistically as the arms
  are baked — not a gate here.

## 7. Verification & acceptance

- **The inefficiency is gone**, directly verifiable: no MQ tar copy / unpack /
  `dnf install` occurs during a native-HA bootstrap (the baked install is skipped —
  confirmed in the transcript); the box boots with `/opt/mqm/bin/crtmqm` present.
- **Box builds, caches, reuses** via `build-fatbox.sh --box mq-nativeha-rhel9` +
  `mqlab`: first build produces `build/state/boxes/mq-nativeha-rhel9.box`; a re-run
  REUSEs it; a manifest change triggers a rebuild.
- **Cloud/x86 one-shot cold-rebuild gate.** A full cold rebuild of the native-HA
  RHEL stack, **in the cloud on base x86**, comes up green in one pass — lint-green
  is not "done." This is the epic's **validation** operational task (`#665`).
- **The `mqmonitor@` template is baked.** `install-RedHat.yml`'s package set
  includes `MQSeriesSamples`, which ships `/opt/mqm/samp/mqmonitor@.service` — the
  systemd template `main.yml` links per QM (+ the #448 open-files drop-in). Baking
  the Samples deb makes the template present in the box; the per-run link resolves
  against it. (Acceptance: the baked box has `/opt/mqm/samp/mqmonitor@.service`.)
- **Native-HA function works from the baked box.** HA switchover (the raft group
  re-elects a live instance) and CRR/DR cutover (`site-nativeha-switchover.yml`,
  site-B recovery group starting `nha_start=false` until `crr.yml` sets its
  GroupRole, #389) succeed against the baked box, with native HA's own replication +
  TLS/security model — the native-HA analogue of RDQM's #551 proof.
- **Closing measurement.** The instrumented native-HA bootstrap before/after is
  recorded on the epic as evidence — a reported outcome, not a threshold.

## 8. Relationships

- **Pattern:** `logical-minds-foundry/.github#70` (RDQM baking). Proven mechanisms
  reused: `mq-resiliency-lab-for-linux#603` (build-fatbox.sh), `#642` (phased
  startup), `#648` (skip-if-baked), `#654` (machine-id reset), `#649`
  (manifest-hash), `#659` (mq-ubuntu2404 + the strip pattern).
- **Seed:** `.github#72` (the #70 stack-baking follow-on brainstorm).
- **Spawns:** the Ubuntu-arms + x86-vetting + security-coordination epic via the
  closing follow-on brainstorm `.github#90`.
- **Adjacent (not this epic):** `.github#91` (mqlab box-lifecycle CLI + cold-boot
  cadence) consumes the same baked-image layer from the tooling side.

## 9. Task breakdown

Implementation tasks (land in `mq-resiliency-lab-for-linux`, linked under `#88`):

1. **`mq-nativeha-rhel9` box** — author `ansible/bake-nativeha-rhel.yml`; add the
   `build-fatbox.sh` case + `_manifest-hash.sh` mapping + `mqlab` box wiring;
   **make the native-HA install skip-if-baked** (§4.3, the acceptance criterion).
2. **Repoint + strip + phased startup** — add the `mq-nativeha-rhel9` `boxes:`
   entry, repoint the six `nha-rhel-*` nodes, strip the now-baked install from the
   per-run path, apply phased startup (`node_exporter` benign-enabled). (May merge
   with Task 1 if small — decided at plan time.)

Operational tasks (in `mq-resiliency-lab-for-linux`, linked under `#88`):

- **Deployment** — bake & cache `mq-nativeha-rhel9` so a bootstrap can boot it (the
  "made usable" signal); blocked-by Task 1.
- **Validation `#665`** — the cloud/x86 one-shot cold-rebuild-green + closing
  measurement + native-HA switchover/CRR proof; blocked-by the deployment.

Bookend tasks (already created):

- **`.github#89` Documentation** — this spec + the plan (first task; its PR
  publishes them).
- **`.github#90` Follow-on brainstorm** — the Ubuntu-arms + x86-vetting +
  security-coordination successor epic + a what-shipped review (closing).
- **`mq-resiliency-lab-for-linux#664` Documentation review** — verify the site docs
  (box-model / bake-manifest pages) reflect the new box + the no-kernel-pin
  refinement (final close gate).
