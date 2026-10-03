# Epic 288 — Cross-stack bootstrap parity (pcmk-ubuntu, rdqm-rhel, nativeha-rhel-crr)

## Why

Epic #275 made the cold bootstrap of `nativeha-ubuntu --no-dr` reliable and fast on both
platforms. It did this by measuring, naming the bottleneck from data, and fixing one variable
at a time:

- **macOS/arm64:** from a ~23-min failure, and runs that never finished, to **643 / 621 / 587 s**.
  The arm64 nested-virt page-fault cost was fixed with huge-page guest RAM (mq-resiliency-lab-for-linux
  #1240 / #1241).
- **x86 cloud:** **721 / 673 / 684 / 687 / 689 s**, 5/5 clean. Guest I/O was fixed with a `pd-ssd`
  boot disk (#1249), and a series of per-run fixed costs were removed (#1212, #1225–#1230, #1248,
  #1250, #1265).

That evidence covers **one stack**. The lab has four:

| stack | OS (per epic #280's catalog) | runs on | validated by #275 |
|---|---|---|---|
| `nativeha-ubuntu` | Ubuntu | macOS arm64 + x86 cloud | yes (`--no-dr`) |
| `pcmk-ubuntu` | Ubuntu: Pacemaker + SAN/iSCSI + DRBD | macOS arm64 + x86 cloud | no |
| `rdqm-rhel` | RHEL 9 (no RHEL 10 DRBD kmod): RDQM / DRBD / Pacemaker | **x86 only** (#847) | no |
| `nativeha-rhel-crr` | RHEL (#280 default: 10): Native HA + CRR | **x86 only** (#847) | no |

A desk audit (#1264, findings in its comments) confirmed that most of #275 is stack-agnostic or
lives on the shared nodes:
- the perf instrumentation and env auto-detect;
- every obs/infra/mq-commons box fix;
- the pre-flight short-circuit, box reuse, huge pages and the SSD boot disk.

It also found **real, stack-specific gaps** (listed in §3), and that **no non-nativeha stack and
no DR half has ever been measured**.

The maintainer treats parity across stacks as a **milestone before public release**: a lab that
can't be brought up in a timely, reliable way won't be used.

## Dependency: epic #280 (multi-version OS axis)

**This epic is planned and implemented as if logical-minds-foundry/.github#280 is complete.** It
starts immediately after #280 finishes (the maintainer's sequencing). #280 changes several things
this epic builds on:
- it introduces `lab/versions.yaml` (the OS catalog) and role-based boxes with a single box→base
  mapping;
- it moves no-MQ shared nodes, **including the SAN targets**, to the catalog's Ubuntu (26);
- it makes RHEL multi-version, with per-major base boxes and kickstarts (`nativeha-rhel-crr`
  defaults to RHEL 10; `rdqm-rhel` stays on 9).

So every box-touching change here is expressed in #280's model (catalog roles, per-version
kickstarts), and every measurement is pinned to an **explicit OS version**.

## Goal

Bring `pcmk-ubuntu`, `rdqm-rhel` and `nativeha-rhel-crr` to the cold-bootstrap reliability and
performance bar #275 set, by #275's method:

1. **measure first** (the perf report, `prereq:*`/`preflight` phases, sampler and `perf diff`
   already apply to every stack);
2. **fix by data**, one change per issue;
3. prove it with **consecutive-pass streaks**.

## Non-goals

- New stacks or features.
- RHEL on macOS (blocked by #847, x86-only).
- OS-version work itself, which is #280.
- A restructuring of the bring-up pipeline. As in #275, a re-architecture is a possible *outcome*
  of data, not a starting assumption.
- DR **streaks**: DR variants get one measurement each; a streak happens only if the data
  warrants it (§5).

## 1. Acceptance

- **5 consecutive clean cold bootstraps**, within that stack's target (below), each at an
  **explicit OS version** (each stack's #280 default; `rdqm-rhel` on RHEL 9):
  - `pcmk-ubuntu --no-dr` on **macOS** (arm64, huge pages);
  - `pcmk-ubuntu --no-dr` on **x86 cloud** (pd-ssd);
  - `rdqm-rhel --no-dr` on **x86 cloud**;
  - `nativeha-rhel-crr --no-dr` on **x86 cloud**.
- **A "clean" run** means all of the following:
  - exit 0, `MQLAB_ENV` unset (auto-detected), and a perf report produced;
  - **obs healthy**: all obs units active and OpenSearch `_cluster/health` green;
  - **the stack's own queue manager healthy, via the stack's `qm-status` verb**: Native HA
    `dspmq` reports it running; RDQM `rdqmstatus` HA status normal; Pacemaker with `mq_group`
    `Started` (the stricter check, §4 W2).
  
  **The single check is `mqlab status <stack> --check`** (added by this epic). It exits 0 only when
  every phase is satisfied, the stack's `qm-status` verb passes, all obs units are active and
  OpenSearch is green. With `--no-dr` no DR leg is checked. Any failure resets the
  count and is classified, never retried until it passes.
- **Per-stack targets, set from data, at two checkpoints:**
  - **W1 baseline review:** a provisional target only, the hard ceiling **20 min (1200 s)**, plus
    decisions on the W3 items.
  - **Pre-streak checkpoint** (after W2/W3 merge): one post-fix cold run per stack and platform.
    This may be run 1 of that W4 streak. **Target = that run's wall-clock × 1.20, rounded up to
    the minute, capped at 1200 s**, recorded in the W4 ticket before run 2.
- **DR measurement:** one instrumented cold run **with DR** (no `--no-dr`) of each of
  `pcmk-ubuntu`, `rdqm-rhel`, `nativeha-rhel-crr` and `nativeha-ubuntu`, **on x86 cloud** (macOS
  DR runs are optional). Bottlenecks are recorded and filed as follow-ups. No 20-min ceiling
  applies to these one-off DR runs.
- **Every new or changed box** is accepted only by a cold rebuild that bootstraps the affected
  stack in one pass, **on both arches** where the box is built for both. Lint-green isn't enough.

## 2. Measurement (reused from #275)

The perf report (`build/state/runs/perf-<ts>.json` plus a human summary), the `preflight` /
`prereq:*` / `net` / `vms` / `provision` / `observe` phases, host and guest contention samples,
and `mqlab perf diff` all **already apply** to every stack (#1264). No instrumentation work is
needed, with one gap: today's milestones are generic (`boot:<vm>` plus the obs readiness tasks).
Per-stack bring-up steps show up only in the Ansible `profile_tasks` timings in the run log.
Adding milestones for the stack-specific steps is optional (W3 item, §4) and should be done only
if reading `profile_tasks` proves insufficient:
- DRBD connect / initial sync;
- pcs cluster formation (`pcmk-cluster : wait for all nodes online`);
- `rdqm-qm-create`;
- CRR recovery-group formation.

**Measurement hygiene (from #275):** no validation containers, box bakes or other heavy work run
on the Vergil VM during a measured macOS run. Cloud measured runs are detached (`setsid nohup`),
so tool time limits can't kill them.

## 3. Known gaps (seed, from the #1264 audit; file references at develop before #280)

**[V]** = verified in code; **[I]** = inferred, to confirm.

**pcmk-ubuntu**
1. **Cluster packages installed from the network on every run** on 6 nodes: pacemaker, corosync,
   pcs, resource-agents, fence-agents-virsh, with `update_cache: true`. See
   `roles/pcmk-cluster/tasks/install-Debian.yml`, `roles/pcmk-stonith/tasks/install-Debian.yml`;
   `roles/iscsi-initiator/tasks/install-Debian.yml` forces an apt update. [V]
2. **SAN nodes (`san-a`, `san-b`) boot an unbaked Ubuntu cloud image** at develop before #280:
   no #275 box hygiene (apt timers plus the `site-dns.yml` dpkg-lock wait of up to ~10 min, full
   cloud-init, snapd, dynamic MOTD), and node-exporter and alloy are downloaded from GitHub on
   every run [V].
   **Delivered by epic #280 Task T6**: `bake-san.yml`, role `san` → box `san-<infra token>`, with
   #275's hygiene roles, node-exporter and the alloy install half; it supersedes #108. **#288 only
   verifies it** in W1 (SAN boot checks) and extends `fwupd-off` to it (W2).
3. **san-b bypassed the SAN deb cache** (`_pcmk-dr-replication.yml`). **Moot after #280 T6**, which
   retires the SAN deb cache entirely.
4. **Every pcmk boot batch is `--no-parallel`** because nodes share a box (`_batch_shares_box`,
   phases.py). Full pcmk means 14 sequential boots. [I from code logic]
5. **The qm-status probe may give a false positive:** `pcs status resources` likely exits 0 once
   the cluster is up but before `mq_group` exists. [I]

**RHEL (`rdqm-rhel` on 9, `nativeha-rhel-crr` on its #280 default)**

6. **No box hygiene.** Nothing is disabled in the RHEL kickstart or bakes [V]. Likely present:
   `dnf-makecache.timer`, rhsmcertd plus the subscription-manager / product-id dnf plugins,
   insights-client timers / motd hook, and kdump [I]. Confirm with a live probe **per RHEL major**.
7. **1–3 redundant no-op `dnf` calls per RHEL node** (acl, libicu), each loading metadata. [V/I]

**Every stack**

8. **The app-client pymqi venv is built on every cold run** (`roles/mq-client`,
   `site-distributed-shared.yml`). #1227 baked only svc-sim's responder venv. [V]

**Lab-wide**

9. **Ansible `forks` is at the default of 5**, so plays over 8+ hosts run in waves. [V]
10. **`fwupd-refresh.service`** is now obs's slowest boot unit on cloud (3.1 s, from #1267's boot
    checks). The firmware refresher is pointless on throwaway guests. [V]

## 4. Workstreams

**W1, baseline (operational, on post-#280 develop, running in parallel with W2)**
- One instrumented cold `--no-dr` run per stack and platform at a **pinned commit, before any W2
  change**, each at its explicit OS version:
  - `pcmk-ubuntu` on macOS (maintainer's host, check in after the run);
  - `pcmk-ubuntu`, `rdqm-rhel` (RHEL 9) and `nativeha-rhel-crr` (its #280 default) on cloud
    (validation tickets for the cloud agent).
- **Live hygiene probe on each RHEL major that has a streak** (read-only):
  `systemctl list-unit-files --state=enabled`, `systemctl list-timers --all`,
  `rpm -q subscription-manager insights-client cloud-init kexec-tools dnf-automatic`,
  `ls /etc/dnf/plugins/`, `ls /etc/motd.d/`, `systemd-analyze blame | head`.
- **Baseline-review checkpoint (with the maintainer):**
  - read each report's phases and `profile_tasks` top tasks;
  - name each stack's bottleneck;
  - set the provisional target (§1);
  - decide which W3 items proceed.

**W2, known wins (code, independent of W1)**

One issue each, all in #280's model:
- **pcmk cluster packages baked** into the pcmk cluster-node role's box, with
  `update_cache: false` at run time (gap 1; open-iscsi too). The bake leaves pcsd / corosync /
  pacemaker disabled until cluster setup.
- **`mqlab status <stack> --check`:** the scriptable clean-run check (§1).
- **App-client pymqi venv baked** into the **`mq-client` role** box, using #1227's install-half
  pattern (gap 8).
- **`fwupd-refresh` disabled** in the Ubuntu hygiene, for every Ubuntu box **including #280's
  `san` box** (gap 10).
- **pcmk qm-status probe: verify, then fix** (gap 5). Confirm on a live cluster whether
  `pcs status resources` exits 0 before `mq_group` exists. If it does, make the verb check
  `mq_group` `Started`. If it doesn't, close the item as not needed.

**W3, data-gated (each blocked by the W1 review)**
- **`rhel-hygiene` role**, version-agnostic, applied to **every RHEL major's** bake and shaped by
  each major's live probe (gap 6). For example: mask `dnf-makecache.*`, rhsmcertd and the insights
  timers; `enabled=0` for the subscription-manager / product-id dnf plugins; kdump off (in **each
  version's** kickstart or the bake); remove the insights motd hook. Plus a bake-time assertion.
- **Remove the redundant dnf no-ops on baked RHEL boxes** (gap 7).
- **Ansible `forks`:** raise it (e.g. 20), as a measured change (gap 9).
- **pcmk boot-batch parallelism:** only if `vms` dominates pcmk's wall-clock (gap 4).
- **Stack-specific milestones:** only if `profile_tasks` alone isn't enough to read the bottleneck
  (§2).
- **Any new bottleneck the baseline names**, as its own issue.

**W4, acceptance (operational)**
- **Pre-streak checkpoint**, then the four streaks (§1) at the post-W2/W3 develop SHA:
  - cloud ones as validation tickets (#1267's format: SSD precondition, box rebake outside the
    timed run, `MQLAB_ENV` unset, explicit OS version, per-run table with the §1 clean checks,
    comparison against the W1 baseline);
  - macOS checked in after each run.
- The four cloud DR measurement runs.

## 5. Risks and lessons carried from #275

- **Cross-arch box behaviour.** #1265 showed a bake path can differ per arch (snapd purge on x86
  only). Every Ubuntu bake change here (pcmk packages, mq-client venv, `fwupd-off` on every
  Ubuntu box including `san`) is verified on **both** arches before the streaks.
- **Huge pages on macOS for pcmk:** full is 30.25 GiB and `--no-dr` 24.25 GiB, against the Vergil
  VM's ~62 GiB. Check headroom with commons up; fail-loud already exists (#1241).
- **#280 interaction.** Everything here assumes #280's catalog and roles. If #280's final shape
  differs from its spec, the W2 box tasks are re-shaped at the start of this epic, not
  hand-patched.
- **Parallel branches** touch shared files (bake playbooks, the box map, `cli.py`), so trial-merge
  each batch.
- **DR runs are longer and heavier** (DRBD resync). Don't impose the 20-min ceiling on the
  one-off DR measurements; record them.
- **Measurement discipline:** one lever at a time for W3 tuning (forks, batch parallelism), each
  justified by the perf data; measured macOS runs stay free of other host load (§2).

## 6. Relationships

- **Depends on** logical-minds-foundry/.github#280 (multi-version OS axis); it starts after #280
  completes.
- Follow-on to logical-minds-foundry/.github#275 (its retrospective, #277, records this in §5).
- Seeded from mq-resiliency-lab-for-linux#1264.
- The SAN-box decision deferred when #108 was closed (D8) is settled by **#280 Task T6** (the
  baked `san` box, which supersedes #108). #288 verifies it and extends hygiene to it.
- RHEL host constraint: #847.
