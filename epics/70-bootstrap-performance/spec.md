# Lab bootstrap performance: bake static software into per-role boxes — design spec

- **Epic:** `logical-minds-foundry/.github#70`
- **Design task:** `logical-minds-foundry/.github#71`
- **Origin:** idea `logical-minds-foundry/.github#69` (instrumented ~85 min RDQM build)
- **Analysis:** `mq-resiliency-lab-for-linux#594`
- **Re-checks / may resolve:** `mq-resiliency-lab-for-linux#565` (cold-rebuild validation),
  `.github#68`, `mq-resiliency-lab-for-linux#590` (SSH-under-load)
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-07-13

## 1. Problem & motivation

A full `mqlab bootstrap rdqm-rhel` takes **~85 minutes**, and the instrumented
evidence (idea #69; run 2026-07-12) is unambiguous: the bootstrap is
**I/O-bound, not CPU-bound**.

- Sampler (`tools/sample-host-resources.sh`, 8 vCPU / 31 GB host, 12 nested VMs):
  **iowait avg 38 %, peaks 97 %**, 62 % of samples I/O-heavy, ~6 procs blocked on
  disk on average; CPU **idle avg 43 %**, run-queue avg 1.7; load avg 12 (max 120)
  dominated by **D-state pileup**, not runnable CPU demand. Memory never < ~8 GB
  available (but the host has **no swap**).
- Root cause: **12 nested VMs install the same static software on every node,
  every run**, contending for one shared host disk.

The top time-sinks are all *repeated software installation* (`profile_tasks`):
alloy install/unzip ×12 ≈ **14 min**; MQ install + tar unpack + copy ×12 ≈
**10 min**; the mq-exporter Go build from source ≈ **2.4 min** (+ Go toolchain
0.8 min); an `acl`-package step that stalls **5.8 min** (a documented anomaly);
plus the #569 QM bounce (3.2 min) and DRBD full resyncs.

This does not scale — we run these repeatedly, and we intend to bring the same
treatment to the **pcmk** and **nativeha** stacks. The remedy is not a bigger
machine (43 % CPU idle): it is to **stop doing the I/O work at all** by installing
the static software **once**, into the box, instead of on every node on every run.

**Success is the elimination of each identified inefficiency**, not a target
percentage. We do not yet know the payoff of removing work we have not removed;
after the eliminations land we re-run the (already-instrumented) bootstrap and
**record the observed before/after as closing evidence** — an outcome we report,
never a gate we had to hit.

## 2. Doctrine & principles

- **Eliminate the waste; do not cache it.** The bootstrap's dominant cost is
  installing identical software 12× per run. The correct move is to *not install
  it per run* — bake it into the image — not to build machinery that makes the
  repeated install faster (that would optimize waste that should not happen).
- **Bake once, configure every run.** Static software (packages, binaries, build
  outputs) is image-time state and belongs in the box. Lab-specific, per-run state
  (queue managers, cluster membership, certificates, DNS records) is bootstrap-time
  state and stays in provisioning. The boundary between the two is the epic's
  central artifact.
- **Zero drift by construction.** The box is baked by **the same Ansible roles**
  that install the software today (provision-then-snapshot), so "what the box
  contains" cannot diverge from "what Ansible would have installed."
- **Reuse the proven pipeline.** A host-durable box cache + `mqlab`-driven
  build/reuse already exists (`build-box.sh`, `_LOCAL_BOX_BUILDERS`). Extend it;
  do not introduce parallel tooling.
- **Minimal, generalizable taxonomy.** Each box is the *minimal* software subset
  its role needs — never a fat all-in-one — so the model carries cleanly to the
  other stacks later.

## 3. Grounding: current box & bootstrap architecture

Established from the code:

- **Base-OS box, per-run install.** `lab/Vagrantfile` is a dumb consumer of
  `build/work/lab/topology.resolved.yaml`: each node sets `node.vm.box` to a
  platform box (`lab/topology.yaml` `boxes:` — `cloud-image/ubuntu-24.04`,
  `almalinux/9`, `rhel/9.6-x86_64`) and an optional `box_version` pin from
  `build/work/box-versions.json` (#266). The RDQM nodes
  (`rdqm-a1..3`, `rdqm-b1..3`) use `platform: rhel96-x86_64` with a 10 GiB
  `extra_disk` (the drbdpool PV). Every one boots a **bare OS**; all software
  arrives via Ansible at bootstrap.
- **A box-build pipeline already exists — for one bare-OS RHEL box.**
  `lab/boxes/rhel96/build-box.sh` builds `rhel/9.6-x86_64` once from the DVD ISO +
  an anaconda kickstart (`ks.cfg`) via a transient libvirt domain, then
  `qemu-img convert -c` → `.box`, caches it **host-durably** at
  `build/state/boxes/rhel-9.6-x86_64-libvirt.box` (survives VM rebuilds, #57), and
  `vagrant box add`s it. Re-runs REUSE the cache in minutes; a 30-day age emits a
  non-blocking staleness NOTICE; `--rebuild-box` forces a fresh build.
- **`mqlab` already drives box build/reuse.** `src/mqlab/cli.py`
  `_LOCAL_BOX_BUILDERS = {"rhel/9.6-x86_64": "lab/boxes/rhel96/build-box.sh"}` +
  `_box_build_steps` probe `vagrant box list`, build/REUSE the locally-built boxes,
  and stage the DVD. Ubuntu/Alma are pulled pre-built from Vagrant Cloud, so they
  have **no local builder** today.
- **Provisioning is a monolith of install + configure.** The RDQM bootstrap runs
  `ansible/site-rdqm.yml` (replication TLS, QM create, SSH teardown, #559
  reconcile, #569 diag bounce, app-TLS) plus observability
  (`ansible/observability.yml`, `ansible/host-obs.yml`) and per-node instrument
  plays. The ~50 roles in `ansible/roles/` freely mix *installing software* (e.g.
  `mq-install`, `rdqm-install`, `alloy`, `node-exporter`, `mq-exporter`,
  `grafana`/`prometheus`/`loki`, `bind-dns`) with *configuring lab state* (e.g.
  `mq-qmgr`, `rdqm-ha`, `lab-pki`, `rdqm-app-tls`, `rdqm-active-node`,
  `mq-diag-logging`). Nothing marks which is which.
- **The Go build is per-run.** `mq-exporter` builds `mq_prometheus` from source
  (Go toolchain + compile) on every run — pure image-time state paid at
  bootstrap-time.

The architecture is thus **correct in mechanism** (a cached, `mqlab`-driven box
build already exists) but **applied only to a bare OS**. This epic extends that
exact mechanism to bake the static software, and splits the monolith along the
bake/configure line.

## 4. The correction

### 4.1 The bake/configure split

Carve provisioning into two phases:

- **Bake** (image-time, once per box): install all static software into the box —
  packages, binaries, and build outputs.
- **Configure** (bootstrap-time, every run): only the irreducibly per-run,
  lab-specific state.

First-pass classification of `ansible/roles/` (the authoritative per-role manifest
is produced by the first implementation task; roles that mix concerns are
**split** — install steps hoisted to bake, config steps kept per-run):

| Bake (→ box image) | Configure (→ per-run) |
|---|---|
| `mq-install`, `mq-client` | `mq-qmgr`, `mq-pcmk-qmgr`, `mq-nativeha` (`crtmqm`) |
| `rdqm-install`, `rhel-ha-repo` | `rdqm-ha`, `pcmk-cluster`, `pcmk-stonith` (cluster form) |
| `node-exporter`, `alloy`¹ | `rdqm-active-node`, `rdqm-state`, `cluster-state`, `nativeha-state`, `host-net-state`, `host-resolver`, `net-reach` |
| `mq-exporter`¹ (Go build → **binary baked**) | `lab-pki`, `pki-distribute`, `rdqm-replication-tls`, `rdqm-app-tls`, `rdqm-ssh-access` |
| `grafana`, `prometheus`, `loki`¹ | `mq-inter-qm`, `mq-event-monitor`, `app-requester`, `mq-diag-logging`² |
| `bind-dns`¹ (BIND install) | `bind-dns`¹ (zone data), `drbd-san`, `iscsi-target`, `iscsi-initiator` |

¹ **Mixed role** — split: the software install is baked; the config drop
(alloy/exporter scrape config, obs dashboards, BIND zones, mqweb config) stays
per-run. `mqweb` similarly splits (install baked, per-QM config per-run).
² `mq-diag-logging`: its *default* stanza is baked into the box (§4.4), so the
per-run bounce is eliminated; any genuinely per-lab override remains per-run.

**Selection mechanism.** Each box gets a thin **bake playbook** (e.g.
`ansible/bake-mq-rdqm.yml`, `bake-obs.yml`, `bake-infra.yml`) that runs exactly
the bake-classified roles against the box-build VM. Whether the split is expressed
as dedicated bake playbooks, `bake`/`configure` task tags, or extracted
`*-install` sub-roles is an implementation choice refined in planning; the binding
requirement is that **the bake set and the configure set are disjoint, explicit,
and reuse the existing role logic** (no reimplementation).

**Single-host bakeability (the primary technical risk).** The bake step runs
against **one transient build VM with no lab inventory around it**, whereas the
install roles today run inside the full topology — with host groups (`rdqm_a:rdqm_b`),
`delegate_to: localhost`, `run_once`, and cross-host facts (as in `site-rdqm.yml`
and `pki-distribute`). A role classified "install" can still carry inventory/group/
delegation coupling that breaks or silently no-ops when run standalone. Therefore
the split is not merely a classification: **for each bake role, the first
implementation task must prove it runs green against a lone single-host inventory**,
and where it does not, extract a clean `*-install` sub-role free of group/delegation/
`run_once` coupling. The bake playbooks run against a minimal one-host inventory.
This is an explicit acceptance criterion on Task 1, not a detail left to discover
mid-epic.

### 4.2 Provision-then-snapshot box build

Extend `build-box.sh` (today RHEL-DVD-specific) into a **parameterized,
multi-platform builder** that, per box:

1. starts from the base box — the locally-built bare RHEL (`build-box.sh`'s
   existing path) or a Vagrant-Cloud Ubuntu base;
2. boots a transient build VM;
3. runs the box's **bake playbook** (§4.1) against it;
4. powers off and `qemu-img convert -c` → fat `.box`;
5. caches it host-durably under `build/state/boxes/<box>.box` and `vagrant box
   add`s it — the **exact persistence/distribution mechanism that already
   works**. No external registry (a registry is a generalization-follow-on, §7).

`mqlab`'s `_LOCAL_BOX_BUILDERS` / `_box_build_steps` gain the new boxes so the
build/REUSE/stage logic covers them unchanged in shape.

**Bake package sources.** The bake VM installs from the *same* sources per-node
install uses today, and there is **no new credential surface**: MQ is the
**Developer edition, pulled anonymously**, and everything else (alloy,
node-exporter, Go, the obs stack, BIND) is open source. The only credentialed
prefetch is the **RHEL DVD**, which `build-box.sh` already stages for the OS
install and never commits — the bake step reuses that staging plus network for the
anonymous fetches. (Task-2 acceptance: bake reuses the existing DVD staging +
anonymous fetches; no entitlement artifact enters a committed box or shared cache.)

### 4.3 Box taxonomy (5 boxes; this epic builds 3)

`<role>[-<flavor>]-<os><ver>`; the `<flavor>` qualifier appears only to
disambiguate the two RHEL9 MQ boxes. The qualifier denotes the **RHEL9 build
flavor** — its supporting-software baseline and, critically, its **kernel-pin
condition** — not the specific HADR product bits (installed generically).

| Box | Role stack | Platform / kernel | Built by |
|---|---|---|---|
| `mq-rdqm-rhel9` | MQ + RDQM (DRBD kmod, drbd-utils, pacemaker) | RHEL 9, **kernel pinned** for DRBD kmod | **this epic** |
| `obs-ubuntu2404` | observability (Grafana/Prometheus/Loki/alloy) | Ubuntu 24.04 | **this epic** |
| `infra-ubuntu2404` | DNS + core infra | Ubuntu 24.04 | **this epic** |
| `mq-nativeha-rhel9` | MQ + nativeha supporting baseline | RHEL 9, **leading-edge kernel** | follow-on |
| `mq-ubuntu2404` | MQ (Ubuntu stack) | Ubuntu 24.04 | follow-on |

**The RHEL kernel dilemma is the reason there are two RHEL9 MQ boxes.** RDQM's
DRBD kmod pins the kernel to a supported level; nativeha wants RHEL's leading
edge. They cannot share one image. Observability and infra live only on Ubuntu.
The **RDQM lab** boots `mq-rdqm-rhel9 ×6 + obs-ubuntu2404 + infra-ubuntu2404`, so
those three are what this epic implements.

`infra-ubuntu2404` is a **deliberate node-role in the generic lab model**, not a
BIND-specific box: BIND is merely its first tenant, with core infrastructure
services (e.g. LDAP authentication) intended to follow. It is "overkill for BIND
today" only because BIND is first in a growing list — the box establishes the role
and pipeline for what comes next. (`obs` and `infra` are, by design, *generic lab
infrastructure* reusable across future non-MQ labs — see §8.)

### 4.4 Standalone eliminations

Independent of the box work (each an inefficiency in its own right):

- **`acl` 5.8-min anomaly.** Investigate the dnf/DVD-repo metadata stall at the
  **root** (a tiny package should install in seconds), even though baking may mask
  it — a 5.8-min stall is a defect worth understanding, not just hiding. Note `acl`
  is *not* eliminated: unprivileged `become_user` at configure-time needs `setfacl`,
  so `acl` is **baked (pre-installed)** into the box — what baking removes is the
  per-run *install stall*, not the package.
- **Shrink lab DRBD volumes 3 GiB → ~1 GiB.** Shorter DRBD full resyncs; directly
  shrinks the #591 DR-wait. Verify the lab QMs fit the smaller volume.
- **Bake the qm.ini diagnostic default (#569).** Bake the `DiagnosticMessages`
  default into the box so the RDQM app QM is *born* with it, eliminating the
  post-create `endmqm -w`/`strmqm` bounce (3.2 min).
- **Concurrency / fork tuning — sequenced last.** Revisit fork count and
  provision/observe overlap only **after** baking, because baking removes most of
  the disk I/O that made added parallelism backfire; the I/O picture changes first.

### 4.5 Staleness discipline

A fat box bakes **versioned** software (MQ 9.4.5.0, a pinned alloy, node-exporter,
the compiled exporter, RDQM packages) and freezes its OS packages at bake time.

**OS currency comes from *rebuilding the box*, not from updating it at boot.**
(This reverses an earlier draft of this section, which had a base-OS refresh — a
`dnf`/`apt` update — run at instance-build. Implementation proved that wrong:
pinning and dynamic update are fundamentally in tension. A blanket
`dnf update`/`apt dist-upgrade` fights the pinned baked stacks — on the
kernel-pinned RDQM box it hit a hard, unsatisfiable **depsolve conflict**, trying
to replace the baked **LINBIT** pacemaker/DRBD stack with the DVD's stock
pacemaker. As long as we pin, the pins must be **persistent**, so we sacrifice the
dynamic update. See `mq-resiliency-lab-for-linux#639`.)

Two mechanisms, scaled to *only what must last until the next iteration*:

1. **Manifest-hash rebuild.** The box is stale when its **bake manifest** — the
   baked package/version set — changes; then it is rebuilt, otherwise the cache is
   reused.
2. **Graduated age policy (stricter than today's 30-day NOTICE):** **warn at 7
   days, refuse at 14** (demand a rebuild). Until an automated build exists, force a
   weekly manual rebuild rather than letting boxes rot.

Pins are kept fresh by **periodically revisiting them and rebuilding** — the
box-rebuild cadence *is* the OS/security-update cadence. The longer-term direction —
an out-of-band CI/manifest job that rebuilds (and perhaps publishes) the boxes
daily/weekly like container images, so CVEs and build breaks surface fast — is
**explicitly out of scope here** and seeded to the follow-on (§8).

## 5. Binding decisions (explicit)

1. **Provision-then-snapshot, not kickstart `%post` and not Packer.** Snapshotting
   after running the existing bake roles is the only mechanism that bakes the Go
   build and the RDQM package set without duplicating logic, and it generalizes to
   Ubuntu (which has no kickstart). Packer is deferred to a possible registry
   follow-on (§7); it would add parallel tooling for marginal gain in a pinned lab.
2. **Host-durable `build/state/boxes/` cache; no registry now.** The proven local
   cache + `vagrant box add` is the distribution mechanism for the ephemeral dev
   host.
3. **Kernel is pinned in `mq-rdqm-rhel9`, leading-edge in `mq-nativeha-rhel9`.**
   The two RHEL9 flavors are distinct boxes by construction.
4. **#569 bounce is eliminated by baking the default, not left in place.**
5. **Concurrency tuning is in scope but sequenced after baking.**

## 6. Scope & non-goals

**In scope:** the bake/configure role split; the parameterized
provision-then-snapshot box builder + `mqlab` wiring; the three RDQM-lab boxes
(`mq-rdqm-rhel9`, `obs-ubuntu2404`, `infra-ubuntu2404`); the standalone
eliminations (`acl` anomaly, DRBD 3→1 GiB, #569 bake-the-default, concurrency
last); manifest-hash staleness; and the closing instrumented measurement.

**Non-goals (explicit):**

- **Generalization to the other stacks.** Building `mq-nativeha-rhel9` and
  `mq-ubuntu2404`, and applying the profile→optimize loop to pcmk/nativeha, is the
  **follow-on epic** seeded by the closing brainstorm task (#72). The taxonomy and
  builder are designed to carry over unchanged; this epic proves them on RDQM.
- **A shared/remote box registry and Packer.** Revisited only *if* the
  generalization work wants centrally-published boxes.
- **Hardware rightsizing.** A fast-NVMe reserved instance is a *secondary* lever
  considered only if baking does not relieve the I/O enough (43 % CPU idle means
  more vCPUs are not the answer). Adding swap is a trivial always-do noted for the
  host, not gated here.
- **Engineering the stability-under-load fixes separately.** #565 / #68 / #590 are
  **re-checked as an outcome** of relieving the I/O (§8), not built here.

## 7. Verification & acceptance

- **The inefficiencies are gone**, each verifiable directly: no per-node alloy
  download/unzip, MQ install/unpack, or Go build occurs during a bootstrap (the
  bake roles do not run at bootstrap-time — confirmed in the transcript); the
  `acl` stall no longer appears; the RDQM volumes are ~1 GiB; the RDQM app QM is
  born with its diagnostic stanza (no post-create bounce in the transcript).
- **Boxes build, cache, and reuse** via the extended `build-box.sh` + `mqlab`
  path: a first build produces `build/state/boxes/<box>.box`; a re-run REUSEs it;
  a manifest change triggers a rebuild (proven by unit test of the decision, as
  `build-box.sh --dry-run` already exposes).
- **Cold-rebuild acceptance gate.** Because this touches provisioning, a full VM
  cold rebuild must prove the changes one-pass; lint-green is not "done." This is
  the epic's **validation** operational task, and the same gate `#565` currently
  blocks — it should pass once the I/O is relieved.
- **Closing measurement.** The instrumented bootstrap is re-run and the observed
  before/after (`profile_tasks` + sampler) recorded on the epic as evidence — a
  reported outcome, not a threshold.

## 8. Relationships

- **Origin:** idea `.github#69`; analysis `mq-resiliency-lab-for-linux#594`.
- **May resolve (re-checked, not engineered here):** `#565` (cold-rebuild
  degrades under load), `.github#68` / `#590` (operator SSH drops mid-bootstrap).
  The thesis is these are side effects of the same I/O saturation; if the cold
  rebuild stays green and SSH stops dropping after baking, they close on that
  evidence — otherwise they spin out as a follow-on.
- **Spawns** the generalization epic via closing task `#72` (nativeha + pcmk
  boxes, registry-if-needed, hardware-if-needed).
- **Longer-term architecture (forward context, not this epic).** This box model is
  intended to grow into a **library of lab base images** built out-of-band by a
  CI/manifest job — rebuilt (and possibly published) daily/weekly like container
  images, so CVEs and build breaks surface fast rather than accruing in a
  hand-built cache. Crucially, `obs` and `infra` are **generic lab infrastructure,
  not MQ-specific**: `obs` is Uatu-the-watcher (the observability layer that
  instruments any lab) and `infra` stands up the middleware — so the *same* model
  spins up, e.g., a `db-ubuntu2404` lab for Postgres/database tooling, reusing the
  identical `obs`/`infra` boxes with different technology-specific boxes swapped in.
  The design principle bounding *this* epic: **implement only what must last until
  the next iteration** — spec the base cleanly enough that a future CI job can build
  and publish it, but do not build that job now. Seeded to the follow-on (`#72`).

## 9. Task breakdown

Implementation tasks (land in `mq-resiliency-lab-for-linux`, linked under `#70`):

1. **Bake/configure classification + split** — produce the authoritative per-role
   manifest; split mixed roles (install → bake, config → per-run); establish the
   bake playbooks. **Acceptance: each bake role runs green against a lone
   single-host inventory** (§4.1) — inventory/group/`run_once`/delegation-coupled
   install roles are refactored into clean `*-install` sub-roles. No behavior change
   to a bootstrap that still installs everything (the split is a refactor with the
   seams drawn).
2. **Parameterized provision-then-snapshot builder** — generalize `build-box.sh`
   to multi-platform (RHEL local base + Ubuntu Cloud base), run a box's bake
   playbook, snapshot → cache; add manifest-hash + graduated-age staleness (warn 7d
   / refuse 14d, §4.5); wire `mqlab` `_LOCAL_BOX_BUILDERS` / `_box_build_steps`.
   **Acceptance: bake reuses the existing DVD staging + anonymous MQ-dev/OSS
   fetches; no entitlement artifact enters a committed box or shared cache** (§4.2).
3. **`mq-rdqm-rhel9`** — bake MQ + RDQM (kernel-pinned) + node-exporter + alloy +
   exporter binary + build tools + `acl` + the #569 qm.ini default; boot the RDQM
   nodes from it; strip the now-baked installs from the per-run path. (No base-OS
   refresh at instance-build — §4.5; OS currency is a box-rebuild concern.)
4. **`obs-ubuntu2404`** — bake the observability stack; boot `obs` from it.
5. **`infra-ubuntu2404`** — bake BIND + core infra; boot the infra/DNS nodes.
6. **Standalone eliminations** — `acl` anomaly root-cause; DRBD 3→1 GiB;
   concurrency/fork tuning (sequenced last).

Operational tasks (in `mq-resiliency-lab-for-linux`, linked under `#70`):

- **Deployment** — bake & cache the three boxes so a bootstrap can boot them
  (the "made usable" signal); blocked-by tasks 3–5.
- **Validation** — cold-rebuild-green + record the instrumented before/after;
  blocked-by the deployment. Supersedes the `#565`-blocked validation.

Bookend tasks (already created):

- **`.github#71` Documentation** — this spec + the plan (first task; its PR
  publishes them).
- **`.github#72` Follow-on brainstorm** — seed the generalization epic + a
  what-shipped review (closing).
- **`mq-resiliency-lab-for-linux#601` Documentation review** — verify the site
  docs (`docs/development/build-layout.md` + a new box-taxonomy / bake-vs-configure
  page) reflect the new model (final close gate).
