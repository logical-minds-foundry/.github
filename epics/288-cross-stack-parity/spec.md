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

| stack | OS | runs on | validated by #275 |
|---|---|---|---|
| `nativeha-ubuntu` | Ubuntu | macOS arm64 + x86 cloud | yes (`--no-dr`) |
| `pcmk-ubuntu` | Ubuntu: Pacemaker + SAN/iSCSI + DRBD | macOS arm64 + x86 cloud | no |
| `rdqm-rhel` | RHEL 9.6: RDQM / DRBD / Pacemaker | **x86 only** (#847) | no |
| `nativeha-rhel-crr` | RHEL 9.6: Native HA + CRR | **x86 only** (#847) | no |

A desk audit (#1264, findings in its comments) confirmed that most of #275 is stack-agnostic or
lives on the shared nodes:
- the perf instrumentation and env auto-detect;
- every obs/infra/mq-ubuntu box fix;
- the pre-flight short-circuit, box reuse, huge pages and the SSD boot disk.

It also found **real, stack-specific gaps** (listed in §3), and that **no non-nativeha stack and
no DR half has ever been measured**.

The maintainer treats parity across stacks as a **milestone before public release**: a lab that
can't be brought up in a timely, reliable way won't be used.

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
- A restructuring of the bring-up pipeline. As in #275, a re-architecture is a possible *outcome*
  of data, not a starting assumption.
- DR **streaks**: DR variants get one measurement each; a streak happens only if the data
  warrants it (§5).

## 1. Acceptance

- **5 consecutive clean cold bootstraps**, within that stack's target, for each of:
  - `pcmk-ubuntu --no-dr` on **macOS** (arm64, huge pages);
  - `pcmk-ubuntu --no-dr` on **x86 cloud** (pd-ssd);
  - `rdqm-rhel --no-dr` on **x86 cloud**;
  - `nativeha-rhel-crr --no-dr` on **x86 cloud**.

  A "clean" run means exit 0, every service the stack declares is healthy at the end, the env
  auto-detected (`MQLAB_ENV` unset), and a perf report produced. Any failure resets the count and
  is classified, never retried until it passes.
- **Per-stack targets, set from data.** After the baseline (W1) and the known wins (W2), each
  stack's target is its measured floor plus a stated margin, never above a **hard ceiling of
  20 min (1200 s)**. The baseline-review checkpoint (§4, W1) records the target with its evidence.
- **DR measurement:** one instrumented cold run each of `pcmk-ubuntu`, `rdqm-rhel`,
  `nativeha-rhel-crr` and `nativeha-ubuntu` **with DR** (no `--no-dr`), on the platform each runs
  on. Bottlenecks are recorded and filed as follow-ups.
- **Every new or changed box** is accepted only by a cold rebuild that bootstraps the affected
  stack in one pass. Lint-green isn't enough.

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

## 3. Known gaps (seed, from the #1264 audit)

**[V]** = verified in code; **[I]** = inferred, to confirm.

**pcmk-ubuntu**
1. **Cluster packages installed from the network on every run** on 6 nodes: pacemaker, corosync,
   pcs, resource-agents, fence-agents-virsh, with `update_cache: true`. See
   `roles/pcmk-cluster/tasks/install-Debian.yml`, `roles/pcmk-stonith/tasks/install-Debian.yml`;
   `roles/iscsi-initiator/tasks/install-Debian.yml` forces an apt update. [V]
2. **SAN nodes (`san-a`, `san-b`) boot the unbaked `cloud-image/ubuntu-24.04`** (D8, deferred
   when #108 was closed):
   - no #275 box hygiene: apt timers plus the `site-dns.yml` dpkg-lock wait of up to ~10 min,
     full cloud-init, snapd, dynamic MOTD;
   - node-exporter and alloy are **downloaded from GitHub on every run**. [V]
3. **san-b bypasses the SAN deb cache** (`_pcmk-dr-replication.yml`). [V]
4. **Every pcmk boot batch is `--no-parallel`** because nodes share a box (`_batch_shares_box`).
   Full pcmk means 14 sequential boots. [I from code logic]
5. **The qm-status probe may give a false positive:** `pcs status resources` likely exits 0 once
   the cluster is up but before `mq_group` exists. [I]

**RHEL (`rdqm-rhel`, `nativeha-rhel-crr`)**

6. **No box hygiene.** Nothing is disabled in `lab/boxes/rhel96/ks.cfg` or the RHEL bakes [V].
   Likely present: `dnf-makecache.timer`, rhsmcertd plus the subscription-manager / product-id
   dnf plugins, insights-client timers / motd hook, and kdump [I]. Confirm with a live probe first.
7. **1–3 redundant no-op `dnf` calls per RHEL node** (acl, libicu), each loading metadata. [V/I]

**Every stack**

8. **The app-client pymqi venv is built on every cold run** (`roles/mq-client`,
   `site-distributed-shared.yml`). #1227 baked only svc-sim's responder venv. [V]

**Lab-wide**

9. **Ansible `forks` is at the default of 5**, so plays over 8+ hosts run in waves. [V]
10. **`fwupd-refresh.service`** is now obs's slowest boot unit on cloud (3.1 s, from #1267's boot
    checks). The firmware refresher is pointless on throwaway guests. [V]

## 4. Workstreams

**W1, baseline (operational, runs in parallel with W2)**
- One instrumented cold `--no-dr` run per stack and platform at a **pinned pre-fix commit**
  (develop before any W2 change):
  - `pcmk-ubuntu` on macOS (maintainer's host, check in after the run);
  - `pcmk-ubuntu`, `rdqm-rhel` and `nativeha-rhel-crr` on cloud (validation tickets for the cloud
    agent).
- **On the first RHEL run, a live hygiene probe** (read-only):
  `systemctl list-unit-files --state=enabled`, `systemctl list-timers --all`,
  `rpm -q subscription-manager insights-client cloud-init kexec-tools dnf-automatic`,
  `ls /etc/dnf/plugins/`, `ls /etc/motd.d/`, `systemd-analyze blame | head`.
- **Baseline-review checkpoint (with the maintainer):**
  - read each report's phases and `profile_tasks` top tasks;
  - name each stack's bottleneck;
  - set provisional targets;
  - decide which W3 items proceed.

**W2, known wins (code, independent of W1)**

One issue each:
- **pcmk cluster packages baked** into `pcmk-ubuntu`, with `update_cache: false` at run time (gap 1;
  open-iscsi too). The bake leaves pcsd / corosync / pacemaker disabled until cluster setup.
- **san-b uses the SAN deb cache:** reuse the `iscsi-target` install path (gap 3).
- **App-client pymqi venv baked** into `mq-ubuntu2404`, using #1227's install-half pattern (gap 8).
- **New `san-ubuntu2404` box** (gap 2):
  - #275 hygiene roles;
  - node-exporter and the alloy install half;
  - drbd-utils, targetcli-fb, open-iscsi;
  - `linux-modules-extra` handled as kernel-matched (like the RDQM kmod): a bake-time check that
    fails loudly on a kernel mismatch, never at boot;
  - san-a / san-b repointed to it;
  - the `bake-dirs-guard` extended to it.
- **`fwupd-refresh` disabled** in the Ubuntu hygiene, for every Ubuntu box including SAN (gap 10).
- **pcmk qm-status probe checks `mq_group`**, not just cluster liveness (gap 5).

**W3, data-gated (each blocked by the W1 review)**
- **`rhel-hygiene` role** for both RHEL bakes, shaped by the live probe (gap 6). For example: mask
  `dnf-makecache.*`, rhsmcertd and the insights timers; `enabled=0` for the subscription-manager /
  product-id dnf plugins; kdump off (in the kickstart or bake); remove the insights motd hook.
  Plus a bake-time assertion.
- **Remove the redundant dnf no-ops on baked RHEL boxes** (gap 7).
- **Ansible `forks`:** raise it (e.g. 20), as a measured change (gap 9).
- **pcmk boot-batch parallelism:** only if `vms` dominates pcmk's wall-clock (gap 4).
- **Stack-specific milestones:** only if `profile_tasks` alone isn't enough to read the bottleneck
  (§2).
- **Any new bottleneck the baseline names**, as its own issue.

**W4, acceptance (operational)**
- The four streaks (§1) at the post-W2/W3 develop SHA:
  - cloud ones as validation tickets (#1267's format: SSD precondition, box rebake outside the
    timed run, `MQLAB_ENV` unset, per-run table, comparison against the W1 baseline);
  - macOS checked in after each run.
- The four DR measurement runs.

## 5. Risks and lessons carried from #275

- **Cross-arch box behaviour.** #1265 showed a bake path can differ per arch (snapd purge on x86
  only). The SAN box, and any Ubuntu bake change, is verified on **both** arches before the
  streaks.
- **Huge pages on macOS for pcmk:** full is 30.25 GiB and `--no-dr` 24.25 GiB, against the Vergil
  VM's ~62 GiB. Check headroom with commons up; fail-loud already exists (#1241).
- **The kernel-matched module** on the SAN box (linux-modules-extra) must match the guest kernel.
  Pin or assert at bake time.
- **Parallel branches** touch shared files (bake playbooks, `cli.py`), so trial-merge each batch.
- **DR runs are longer and heavier** (DRBD resync). Don't impose the 20-min ceiling on the
  one-off DR measurements; record them.
- **Measurement discipline:** one lever at a time for W3 tuning (forks, batch parallelism), each
  justified by the perf data.

## 6. Relationships

- Follow-on to logical-minds-foundry/.github#275 (its retrospective, #277, records this in §5).
- Seeded from mq-resiliency-lab-for-linux#1264.
- Reopens the SAN-box decision deferred when #108 was closed (D8), on #275's evidence.
- RHEL host constraint: #847.
