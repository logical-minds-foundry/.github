# pacemaker-drbd arm — an RDQM-faithful open-source HA/DR mechanism — design spec

- **Epic:** `logical-minds-foundry/.github#310`
- **Design task:** `logical-minds-foundry/.github#311`
- **Member repo:** `logical-minds-foundry/mq-resiliency-lab-for-linux`
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-10-10

## 1. Problem & motivation

The lab exists in part to answer one question: *IBM recommends RDQM — can the same
HA/DR stack be built from the vanilla open-source parts it bundles (Corosync,
Pacemaker, DRBD), and what exactly does IBM add?* The answer is academic — nobody is
expected to run MQ this way in production, and the architectural recommendation
remains **Native HA** (replication owned by the product, not offloaded to the storage
layer). But the proof is only meaningful if the open-source build actually mirrors
RDQM.

Today it does not. The open-source arm (`pcmk-ubuntu`, mechanism `pacemaker-san`)
was built on the traditional enterprise shared-storage pattern; RDQM is
shared-nothing. Facts from the member repo:

| | RDQM (`rdqm-rhel`) | Current open-source arm (`pcmk-ubuntu`) |
|---|---|---|
| In-site storage | Local disk per node: `drbdpool` VG on `/dev/vdb` (`ansible/roles/rdqm-install/tasks/configure.yml`) | **One** iSCSI LUN served by a single `san-a` VM |
| In-site replication | DRBD across the 3 HA nodes | **None** — the 3 nodes take turns mounting the single copy |
| Cross-site DR | DRBD from the QM's own volume to the DR nodes | DRBD protocol A **between the SAN VMs** (`ansible/roles/drbd-san/tasks/main.yml`) |
| Unit of replication | One logical volume **per queue manager**, mounted at `/var/mqm/vols/<qm>` | The whole LUN |
| In-site single point of failure | None | `san-a` |
| Extra infrastructure | None | `san-a`/`san-b` VMs, `net-san-*`, `iscsi-target`/`iscsi-initiator`, LIO, Booth |

So the existing fault suite compares RDQM against a **structurally weaker** design —
a difference that has nothing to do with IBM's tooling — and the SAN tier adds most of
the arm's complexity. The original design already named shared storage as "the
acknowledged weak link" and RDQM's shared-nothing property as one of its strongest
arguments (`docs/specs/2026-06-03-mq-cluster-lab-design.md` §2.3, §2.5 Q4, §2.7).

**Correction recorded during the brainstorm:** RDQM does not replicate `/var`. It
creates a DRBD resource on a per-queue-manager logical volume, and IBM says "IBM MQ
manages the logical volumes created in drbdpool, and how and where they are mounted"
(IBM Docs, *Requirements for an RDQM DR solution*, MQ 9.4.x — cached at
`build/cache/refs/ibm-docs/ibm-mq/9.4.x/recovery-requirements-rdqm-dr-solution/content.txt`).
The worked example's `crtmqm` output shows `Directory '/var/mqm/vols/qm1/qmgr/qm1'
created`. `/var/mqm` itself (e.g. `mqs.ini`) stays local to each node.

**Success:** a new `pacemaker-drbd` mechanism, built from upstream components to
RDQM's captured configuration, run through the same drill set as `rdqm-rhel`, with a
written **deviation ledger** and **comparison report** stating concretely what IBM
adds — and an evidence-based decision on the future of `pacemaker-san`.

## 2. Goals & non-goals

### Goals

- **G1 — Faithfulness by evidence.** Capture RDQM's live generated configuration and
  build the open-source arm to it; every place it cannot match is recorded, not
  silently approximated.
- **G2 — A runnable `pacemaker-drbd` stack** (`pcmk-drbd-ubuntu`) with HA at site A,
  DR to site B, and the standard stack verbs.
- **G3 — An apples-to-apples comparison**: the same drill set run on `rdqm-rhel` and
  `pcmk-drbd-ubuntu`, results recorded, and a comparison report built on the
  deviation ledger.
- **G4 — Decide the fate of `pacemaker-san`** (retire, demote, or extract its
  reusable parts) as an explicit, recorded decision at the end of the epic.

### Non-goals

- **No change to `pcmk-ubuntu` / `pacemaker-san`** during this epic. It stays built
  and working until the G4 decision; any retirement or extraction is its own
  follow-on epic.
- **No full parity with `pcmk-ubuntu`.** The distributed app↔SVC workload,
  observability, cockpit boards, authz service accounts, and TLS are out of scope,
  except where the stack registry provides them unchanged. Anything needing real work
  goes on a follow-on list (§9).
- **No performance or timing claims.** The RDQM arm runs under x86 emulation; the
  comparison is functional only (consistent with
  `docs/specs/2026-06-15-rdqm-parity-pivot-design.md` §2).
- **No general live-lab validation framework** (`mq-resiliency-lab-for-linux#38`).
  The drills here are scripted procedures whose recorded output is the evidence.
- **No `drbd-reactor` / non-Pacemaker promoter.** Rejected because RDQM uses
  Pacemaker (§4).

## 3. Key decisions

| # | Decision | Basis |
|---|---|---|
| D1 | **Add `pacemaker-drbd`; defer the `pacemaker-san` decision to the end of the epic.** | The SAN code has reusable value (e.g. a future database-resiliency lab); building the new arm first yields the knowledge needed to decide. |
| D2 | **DRBD 9 from LINBIT's PPA (`drbd-dkms`).** | *Data:* the mainline in-kernel DRBD is 8.4.11, "intended for 2-node failover clusters"; DRBD 9 supports up to 31 peers per volume and adds quorum for 3+ nodes; DRBD 9 in mainline is targeted "possibly as soon as Linux 7.2", not merged ([LINBIT, 2026-04-27](https://linbit.com/blog/working-to-put-drbd-9-in-the-mainline-linux-kernel/)). LINBIT publishes DRBD 9 for Ubuntu LTS at [`ppa:linbit/linbit-drbd9-stack`](https://launchpad.net/~linbit/+archive/ubuntu/linbit-drbd9-stack). RDQM's media ships `Advanced/RDQM/PreReqs/el9/kmod-drbd-9/` (member repo `docs/reference/rdqm-ha-cheatsheet.md`). *Judgment:* in-kernel 8.4 cannot reproduce RDQM's 3-node group; waiting for mainline is not plannable. |
| D3 | **RDQM's live configuration is the blueprint.** | Designing from upstream docs alone risks another "similar in spirit" mismatch — the failure mode the SAN represents. |
| D4 | **Follow RDQM over existing lab habit:** no SAN, **no Booth**; DR is operator-driven. | RDQM DR is promoted/demoted by an operator (`rdqmdr`), not automatically arbitrated. |
| D5 | **Scope: core + comparison** (§2). | Answers "what does IBM add" without paying for full parity; full parity becomes the natural follow-on only if G4 retires `pcmk-ubuntu`. |

## 4. Approaches considered

1. **Copy RDQM's live configuration as the blueprint (chosen).** Capture what RDQM
   generates, rebuild it from upstream parts, ledger every deviation.
2. **Design from LINBIT/ClusterLabs documentation, compare afterwards.** Faster to
   start and needs no live RDQM arm, but invites unmeasured divergence.
3. **DRBD-native promotion with `drbd-reactor`, no Pacemaker.** LINBIT's lighter
   promoter; simpler, but RDQM uses Pacemaker, so the arm would stop mirroring RDQM.

## 5. Architecture

**Invariant:** wherever the blueprint shows how RDQM does something, the open-source
arm does the same thing with upstream parts; wherever it cannot, the deviation is
ledgered (§6.2). The "expected" column below is the brainstorm's working hypothesis;
**the blueprint capture (§6.1) is authoritative and overrides it.**

| Layer | RDQM (expected — capture confirms) | `pacemaker-drbd` |
|---|---|---|
| Nodes | 3 HA nodes at site A + 3 at site B, RHEL 9.6 x86-64 | `pdrbd-a1..3` + `pdrbd-b1..3`, Ubuntu 24.04 (host arch; arm64 on the dev host) |
| Storage | `drbdpool` VG on a local extra disk; one LV per QM (plus a snapshot LV for DR) | Same VG convention on a per-node extra disk; one LV per QM (and the snapshot LV if the blueprint shows RDQM relies on it) |
| Replication | DRBD 9 — synchronous within the site, asynchronous to the DR site | DRBD 9 (`drbd-dkms`), the protocols, peer layout, and quorum/split-brain options copied from the captured `.res` files |
| Mount | `/var/mqm/vols/<qm>` containing `qmgr/` and `log/` data | Same layout, so MQ sees identical paths |
| Cluster manager | Pacemaker/Corosync bundled by IBM; 3-node quorum plus DRBD quorum | Upstream Pacemaker/Corosync: promotable DRBD resource → `ocf:heartbeat:Filesystem` → MQ queue-manager resource → `ocf:heartbeat:IPaddr2` |
| Fencing | As captured (expected: DRBD quorum rather than STONITH) | The same choice; `pcmk-stonith` is reused only if RDQM uses fencing |
| DR control | Operator-driven `rdqmdr` promote/demote | Operator-driven playbook performing the equivalent DRBD and Pacemaker steps; no Booth |
| Lifecycle tooling | `rdqmadm`, `crtmqm -sx`, `rdqmstatus`, `rdqmdr` | Playbooks behind the stack verbs; gaps are ledgered |

### 5.1 Where it lives (member repo)

- **Topology** (`lab/topology.yaml`): a new stack `pcmk-drbd-ubuntu`
  (`mechanism: pacemaker-drbd`, `os_family: ubuntu`, short token `PDRBD` → QM names
  derive per the naming convention), groups `pdrbd_a` / `pdrbd_b`, six nodes with an
  `extra_disk`. Host octets `.71–.73` (site A) and `.81–.83` (site B) on each lab
  network — free today (verified 2026-10-10: `10.50.0.x` uses `.2–.16`, `.31–.43`,
  `.50–.63`, `.91–.96`), so the arm **coexists** with `pcmk-ubuntu` and
  `nativeha-ubuntu` on one host. No `net-san-*` attachment. VIPs chosen from the
  arm's own range in the plan.
- **Bake:** DRBD 9 (LINBIT PPA + `drbd-dkms` + headers, module built) is baked into a
  box, so nodes never compile a kernel module at boot. Whether that is a new
  `pdrbd` box role or an addition to the existing `pcmk` box is a plan decision.
- **Roles:** new `drbd9` (repo, DKMS module, module load, VG on the extra disk); new
  `pdrbd-qm` (per-QM LV, DRBD resource, filesystem, Pacemaker resources). Reused:
  `pcmk-cluster`, the existing MQ queue-manager resource agent wiring
  (`mq-pcmk-qmgr`, refactored only as far as needed to run on a DRBD-backed
  filesystem instead of an iSCSI one). **Untouched:** `drbd-san`, `iscsi-*`, the SAN
  networks.
- **Parity matrix:** a `pcmk-drbd-ubuntu` row in `src/mqlab/parity.py` `MATRIX`.

### 5.2 Stack verbs

| Verb | Implementation |
|---|---|
| `qm-create` / `qm-destroy` | Playbook: create or tear down the per-QM LV, DRBD resource, filesystem, and Pacemaker group |
| `qm-up` / `qm-down` | `pcs resource enable` / `disable` on the QM's group (as `pcmk-ubuntu`) |
| `qm-status` | `pcs status resources` plus `drbdadm status <res>` — the open-source counterpart of `rdqmstatus` |
| `dr-cutover` / `dr-failback` | Playbook mirroring `rdqmdr -s` / `-p`: stop and demote at the active site, promote and start at the other. Operator-run, never automatic |
| `diagnostics` | `runmqras`, as on the other arms |

## 6. The RDQM blueprint and the deviation ledger

### 6.1 Blueprint capture

- A **read-only** Ansible playbook, `capture-rdqm-blueprint.yml`, run by the human
  operator against `rdqm_a` and `rdqm_b` with an HA/DR queue manager running.
  It changes nothing on the nodes.
- **DRBD:** generated `/etc/drbd.d/*.res`, `drbdadm dump`, and the installed DRBD
  version (`kmod-drbd`, `drbdadm --version`) — this also verifies the judgment that
  RDQM's DRBD is LINBIT's DRBD 9.
- **Pacemaker/Corosync:** cluster configuration as **text** (`pcs config` /
  `crm configure show` — not the raw CIB XML), cluster properties
  (`stonith-enabled`, `no-quorum-policy`, …), `corosync.conf`, and the resource
  agents in use, flagging which are IBM-proprietary.
- **Storage:** `vgs`/`lvs` (QM LV and any snapshot LV), mounts under
  `/var/mqm/vols`, filesystem type and mount options.
- **RDQM:** `rdqm.ini`, `rdqmstatus`, installed systemd units, and the package
  versions of drbd, pacemaker, and corosync.
- **Both sites**, so the cross-site relationship is captured.
- **Output:** raw output under the lab's `build/state/` bucket; a cleaned copy
  committed as `docs/reference/rdqm-blueprint/` in the member repo. Keystores,
  passwords, and any secret material are **excluded at capture time** (never
  committed and then scrubbed).
- **Failure:** any capture command that fails fails the playbook loudly — an
  incomplete blueprint is worse than none.

### 6.2 Deviation ledger

`docs/reference/rdqm-vs-vanilla-deviations.md`, seeded from the blueprint and
maintained throughout the build. One row per RDQM element: what RDQM does, the
open-source equivalent, the status (**exact** / **partial** / **none**), and the
consequence. Expected categories: cluster and storage configuration (likely
matchable), IBM-proprietary resource agents (substituted), lifecycle tooling
(`rdqmadm`/`rdqmdr`/`rdqmstatus` → playbooks), replication TLS, and
support/installation. Each row cites the blueprint file it derives from.

### 6.3 Comparison report

Built near the end of the epic: the drill results of both arms side by side,
alongside the ledger, ending in the "what IBM adds" conclusion. Functional
comparison only (§2). Captured facts and drill results are kept clearly separate
from interpretation (data vs. judgment), with checkable citations.

## 7. Error handling

- **Fail loud, no masking.** No `|| true`; the command that does the work is the
  one that fails. (The existing `drbd-san` role uses `drbdadm up … || true` with a
  separate assert; the new role must not repeat that shape.)
- **DRBD operational lessons pre-applied** from the member repo's
  `docs/reference/drbd-operations.md`: `create-md` with `</dev/null` and a
  `timeout`; live-state probes via `drbdadm status` (never `dump-md` on an in-use
  device); attach asserted via `dstate`; resync buffers sized for the virtual WAN.
- **Split-brain and quorum behavior come from the blueprint** (`after-sb-*`,
  `quorum`, `on-no-quorum`), not invented.
- **DR playbooks prove preconditions before acting:** before promotion, assert the
  target's disk state (UpToDate, or Outdated on an explicit forced cutover). A
  cutover that cannot prove its precondition stops with a clear message.
- **Long operations are watched** (initial sync, DKMS build): progress is observed
  and a stall is treated as a failure, not waited on indefinitely.

## 8. Testing & acceptance

- **Unit tests** (within `vrg-validate`): the stack registry entry, the parity matrix
  row, and any Python helpers, at the repo's 100% branch-coverage gate.
- **DKMS early proof:** the bake task must show the DRBD 9 module **loading** on the
  lab's Ubuntu 24.04 kernel on the dev host's architecture before HA work starts.
- **Cold-rebuild acceptance:** a full VM cold rebuild brings the new arm up in one
  pass (the repo's standing acceptance gate for bring-up/provisioning changes).
- **Drill set** — identical procedures run against **both** `rdqm-rhel` and
  `pcmk-drbd-ubuntu`, results recorded:
  1. Controlled `qm down` / `qm up`, with `mqlab qm e2e` before and after.
  2. Hard kill of the active node (`virsh destroy`).
  3. Partition of the HA replication network (`lab/scripts/net-down.sh net-hb-a`).
  4. Loss of 2 of 3 HA nodes — quorum-loss behavior must match RDQM's.
  5. DR cutover and failback, RPO measured with the existing `mqlab dr` accounting.
  6. Forced DR with a degraded replication link (`lab/scripts/drbd-degrade.sh`).

## 9. Epic structure

Epic home: `logical-minds-foundry/.github` (the member repo is public).

| # | Task | Kind | Repo | Depends on |
|---|---|---|---|---|
| D | Documentation: this spec + plan (`#311`) | docs | `.github` | — |
| 1 | `capture-rdqm-blueprint.yml` read-only capture playbook | impl | member | D |
| 2 | Run the capture on the live RDQM arm; commit `docs/reference/rdqm-blueprint/`; seed the deviation ledger | impl (human runs capture) | member | 1 |
| 3 | DRBD 9 bake: LINBIT PPA + `drbd-dkms` in a baked box; module proven to load | impl | member | D |
| 4 | Topology: `pcmk-drbd-ubuntu` stack, `pdrbd_*` nodes/groups, extra disk, parity row | impl | member | 2 |
| 5 | Site-A HA formation from the blueprint: VG, per-QM LV/DRBD/filesystem, Pacemaker group, `qm-*` verbs | impl | member | 2, 3, 4 |
| 6 | DR: site B joins; `dr-cutover` / `dr-failback` playbooks | impl | member | 5 |
| 7 | Cold rebuild brings the new arm up in one pass | validation | member | 6 |
| 8 | Drill set on `rdqm-rhel` and `pcmk-drbd-ubuntu`, results recorded | validation | member | 7 |
| 9 | Comparison report + finalized deviation ledger | impl (docs) | member | 8 |
| 10 | `pacemaker-san` decision (retire / demote / extract), recorded as a decision doc; retirement or extraction becomes its own follow-on epic | impl (decision doc) | member | 9 |
| R1 | Documentation review (`mq-resiliency-lab-for-linux#1419`) | docs bookend | member | 10 |
| R2 | Retrospective (`#312`) — terminal | retrospective | `.github` | all |

Tasks 1–2 and 3 run in parallel, so the DKMS risk surfaces before HA work. No
follow-on brainstorm task is seeded: task 10 is the forward decision, and full
parity with `pcmk-ubuntu` becomes a follow-on only if task 10 retires that arm.

**Follow-on candidates (not in scope):** full `pcmk-drbd-ubuntu` parity
(observability, cockpit, distributed workload, authz, TLS); `pacemaker-san`
retirement or extraction per task 10; rerunning the comparison on an x86 host for
timing.

## 10. Risks

| Risk | Mitigation |
|---|---|
| `drbd-dkms` fails to build or load on the lab's Ubuntu kernel/arch | Task 3 runs first, in parallel with the capture; a failure is surfaced before any HA work and re-plans the epic. |
| RDQM depends on IBM-only resource agents with no upstream equivalent | That is a finding, not a blocker: substitute the nearest upstream agent and ledger the gap. |
| The blueprint shows RDQM behavior the hypothesis in §5 got wrong (e.g. it does use fencing, or a different DR peer layout) | §5 is explicitly overridden by the capture; the plan is reconciled after task 2. |
| Host capacity: running the RDQM arm (x86 emulation) and the new arm together | Arms run one at a time for drills (pivot spec §3.4); coexistence is only for address/namespace isolation. |
| Pacemaker and DRBD 9 version skew between RHEL (IBM bundle) and Ubuntu (distro + LINBIT) | Record both versions in the blueprint and ledger; treat behavioral differences as findings. |
