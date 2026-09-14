# MQ 9.4.5 → 10.0 Native HA rolling-upgrade runbook + version centralization — design spec

- **Epic:** `logical-minds-foundry/.github#219`
- **Design task:** `logical-minds-foundry/.github#220`
- **Doc-review bookend:** `logical-minds-foundry/mq-resiliency-lab-for-linux#1069`
- **Retrospective (terminal):** `logical-minds-foundry/.github#221`
- **Design seed:** `docs/specs/2026-06-03-mq-cluster-lab-design.md` Appendix C
  (9.4 baseline / 9→10 gap analysis; C.5 calls for a "documented, tested
  9.4→10.0 upgrade runbook")
- **Prior art (centralization):** the #266 version-manifest design (SUT half
  dropped in #350)
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-09-14

## 1. Problem & motivation

We need a **documented, tested procedure** for upgrading IBM MQ from 9.4.5 to
10.0 on a Native HA cluster — the artefact Appendix C.5 of the lab design spec
already called for. The headline output is an **operational runbook** suitable
to hand to work, plus an **evidence report** demonstrating the procedure was
executed against the lab and works.

Two facts make this both possible and timely (data, verified):

- IBM MQ 10.0 LTS went GA on **2026-06-16**, and **MQ Advanced for Developers
  10.0** is a free download **with Native HA included** — so the lab can install
  and test it. (Sources: [Introducing IBM MQ v10.0][ann],
  [Downloading IBM MQ 10.0 LTS][dl].)
- MQ 10.0 supports **RHEL 9 or 10** ([System Requirements for IBM MQ 10.0][sr]);
  the lab runs RHEL 9.6, so the MQ upgrade needs **no OS move**.

Alongside the runbook, a smaller enabling problem: the MQ version is scattered
across **three hand-synced literals** (`scripts/fetch-mq.sh:9`,
`src/mqlab/manifest.py:70`, `ansible/group_vars/all/versions.yml:5`), **two
stale** `manifests/*/default.yaml` files, and a **hardcoded floor assertion** in
the `mq-nativeha` role. "What MQ version are we on?" has no single answer, and
the upgrade would otherwise mean editing the same number in several places. We
fix this **leanly** so the upgrade *is* changing one authoritative pin.

## 2. Doctrine & principles

- **This is a maintenance-window upgrade of the primary service, not a
  zero-downtime exercise.** A major-version upgrade of MQ itself is performed
  with the application quiesced during an agreed outage window. (OS *patching*
  of a dependent component can be done rolling-while-live; upgrading the primary
  service is a different discipline.) The runbook does not pretend to be
  continuous-availability.
- **Rolling the upgrade is for staged risk containment, not availability.**
  Two levels of staging give two revert positions: **across sites**, the Live
  group is untouched and still serving while the Recovery group upgrades; and
  **within a group**, replica instances upgrade before the active, so basic
  software-installation problems are found and fixed before the new version is
  exposed to the queue-manager data and before the active instance is touched.
  Both hold only up to the point of no return.
- **Quiesce, then *assert* the quiesce.** Reaching a quiesced state is easy to
  claim and hard to prove. The runbook makes it concrete: stop the listener(s),
  drain in-flight work, and verify — no active application connections, channels
  inactive, transmit-queue depths zero — before any node is touched.
- **Lean centralization.** One authoritative MQ pin; every consumer reads it;
  the stale sources die. Observability/toolchain version drift is out of scope
  (noted as follow-on).
- **Cold rebuild is the acceptance gate** for the provisioning change (per the
  repo's cold-rebuild acceptance doctrine): lint-green ≠ done.
- **Generic and anonymized.** The runbook carries no client-identifiable
  material; it reads as a generic Native HA CRR upgrade procedure.
- **Operational correctness is proven by running the operational procedures.**
  There is no automated full-functionality checkout (the target environments are
  manually operated), so the way we prove the upgrade is operationally sound is
  to execute the manual DR runbook after it: a failover and a failback. The
  upgrade is "done" only once those succeed on 10.0 and the roles are back to
  their starting assignment. (This manual, runbook-driven validation is the
  honest reflection of how the environment operates; mechanizing it is the
  larger ambition, out of scope here.)
- **No upgrade/downgrade automation.** The procedure is manual (AI-executed).
  Automating it is explicitly out of scope.

## 3. Scope

### In scope

- A single authoritative MQ version pin, threaded to all consumers (§4).
- A generic Native HA **CRR** rolling-upgrade runbook, 9.4.5 → 10.0 (§5) — the
  cross-region (Live + Recovery) configuration, because that is the real
  deployment shape; a standalone single-group upgrade is not representative.
- A tested execution against the **full RHEL Native HA CRR topology** (site-A
  Live + site-B Recovery, DR enabled), including the post-upgrade DR
  failover/failback validation, with an evidence report (§6).

### Out of scope

- Upgrade/downgrade automation.
- OS upgrade (MQ 10.0 runs on the lab's RHEL 9.6).
- Broad version-manifest consolidation (obs/toolchain) — follow-on.
- Separately re-running the procedure on the Ubuntu Native HA peer — documented
  as "same procedure."
- A DR switchover *as part of the version upgrade itself* — the upgrade does not
  switch to the Recovery site to carry traffic; the maintenance window has the
  application quiesced (see §2, §5.4). DR failover is exercised only as
  post-upgrade validation (§5.6), and the roles are returned to their starting
  assignment.

## 4. Workstream A — Version centralization (lean)

### 4.1 Target

A single authoritative MQ version value (the pin), read by every consumer.
`src/mqlab/manifest.py` is the natural home — it already centralizes the
observability versions and the arch/tarball mapping, and already holds
`DEFAULT_MQ_VERSION`. The shell fetcher, which cannot import Python cheaply,
reads the pin from a single small file the Python module also reads, so there is
**one** source and no hand-syncing.

### 4.2 Consumers to re-thread

| Consumer | Today | After |
|---|---|---|
| `scripts/fetch-mq.sh:9` | `VER="9.4.5.0"` literal | reads the pin |
| `src/mqlab/manifest.py:70` | `DEFAULT_MQ_VERSION` literal | the pin (or reads the file) |
| `ansible/group_vars/all/versions.yml:5` | `mq_version:` literal | sourced from the pin |
| `mq-nativeha` floor assertion | hardcoded `9.4.4/9.4.5/9.4.6` string-match | version-aware floor incl. 10.0 |
| `manifests/distributed-*/default.yaml` | stale `mq.version` | **removed** |

### 4.3 Floor assertion

The Native HA role's off-container floor is a brittle string match. It is
widened to accept 10.0 (and expressed as a real version comparison, not a
substring test, so future bumps don't silently fail the gate).

### 4.4 Acceptance

Cold rebuild of the RHEL Native HA arm from the centralized pin **at 9.4.5**
succeeds one-pass (proves the re-threading is faithful before any version
change). The pin flip to 10.0 is exercised in §6.

## 5. Workstream B — The Native HA CRR rolling-upgrade runbook

A generic operational runbook for a **cross-region (Live + Recovery) Native HA**
queue manager. Two ordering rules govern it, both from IBM
([Upgrading Native HA IRR configurations][irr]):

- **Across sites: Recovery group first.** The Recovery group is upgraded before
  the Live group, and the Recovery group must always be at a version **≥** the
  Live group (you cannot fail/switch over to a lower version). Getting this
  ordering backwards is the primary way a real CRR upgrade goes wrong — so it is
  stated as a headline rule of the runbook.
- **Within a group: replicas first, active last.** Per IBM's rolling-update
  guidance for a Native HA queue manager ([considerations][ru]): upgrade one
  instance at a time, a **replica** first, gating each step on the instance
  returning to a healthy **REPLICA** status (`dspmq -o nativeha` / `-o role`);
  the **active instance is upgraded last**.

The upgrade does **not** perform a DR switchover to carry traffic (see §2, §3):
the maintenance window has the application quiesced, and switching the live
role — which in reality can require coordination with the counterparty — is not
part of a routine version upgrade. The end state's role assignment is identical
to the start.

### 5.1 Pre-flight & back-out

- Pre-checks: current version on every instance in **both** groups, group
  health/quorum, free space, media staged, entitlement; record the starting
  Live/Recovery role assignment.
- **Back-out plan with the point of no return.** While the Recovery group is
  being upgraded, the Live group is untouched and still serving — a strong
  revert position. Because a major-version queue-manager data migration is
  **one-way**, the runbook states explicitly the step after which roll-back is
  no longer possible and only roll-forward remains, and what "back-out" then
  concretely means. (Exact boundary and mechanism pinned against IBM docs in the
  plan; see §8.)

### 5.2 Quiesce & assert (maintenance window opens)

1. Stop the listener(s) — no new inbound connections.
2. Allow in-flight messages to deliver; transmit queues to drain; channels to
   shut down cleanly.
3. **Assert quiesced-with-respect-to-the-application:** no active application
   connections (`DISPLAY CONN`), channels inactive (`DISPLAY CHSTATUS`),
   transmit-queue depths zero. The queue manager keeps running; only application
   flow has stopped.

### 5.3 Upgrade the Recovery group (site-B) first

Roll the Recovery group to 10.0 using the within-group rule (replicas first,
its active last), gating each instance on healthy REPLICA status. On completion
the Recovery group is fully on 10.0 while the Live group is still on 9.4.5 —
the Recovery ≥ Live invariant holds throughout.

### 5.4 Upgrade the Live group (site-A)

Roll the Live group to 10.0 using the within-group rule (replicas first, active
last). No DR switchover is performed; the queue manager is simply unavailable
during the Live-active step, which is acceptable inside the maintenance window.

### 5.5 Post-upgrade sanity gate

All instances in both groups report 10.0 (`dspmqver`); both groups
healthy/quorate; objects intact; a **test message round-trips**. This is the
"validate it's sane" checkpoint before any operational validation.

### 5.6 Post-upgrade operational validation — DR failover / failback

Because there is no automated functional checkout, the operational runbook is
exercised to prove the upgrade is sound:

1. **Fail over Live → Recovery** using the manual DR runbook; verify
   functionality on the (now-live) Recovery site.
2. **Fail back Recovery → Live**; verify functionality; confirm the Live/Recovery
   role assignment is **identical to the pre-upgrade starting state**.

Both drills run on 10.0 and both must succeed. This doubles as a validation of
the DR runbook itself under the new version.

### 5.7 Go-live

Re-enable the listener; application flow resumes. Maintenance window closes.

## 6. Workstream C — Test & evidence

- **Target:** the **full RHEL Native HA CRR topology** — site-A Live
  (`nha-rhel-a1/a2/a3`) + site-B Recovery (`nha-rhel-b1/b2/b3`), **DR enabled**
  (no `--no-dr`) — brought up on a **cold-rebuilt** lab at **9.4.5**.
- **Execution:** drive the §5 runbook **manually (AI-executed)** — quiesce and
  assert, upgrade the Recovery group to 10.0, upgrade the Live group to 10.0,
  sanity-gate, then the failover/failback DR validation, then go-live.
- **Evidence captured into an operational report** (`docs/reports/`):
  - the starting Live/Recovery role assignment;
  - clean quiesce (no active connections, channels inactive, xmitq depths zero);
  - per-instance `dspmqver` 9.4.5 → 10.0 across **both** groups;
  - group health/quorum (`dspmq -o nativeha`) at each step, and the
    **Recovery ≥ Live** invariant holding throughout;
  - the two post-upgrade DR drills (Live→Recovery, Recovery→Live) succeeding on
    10.0, with functionality verified after each;
  - final state: all instances on 10.0, data/objects intact, test message
    round-trips, and the role assignment **identical to the start**;
  - the point-of-no-return as actually observed.
- **Ubuntu peer:** documented as "same procedure," not separately re-run.
- **After the test:** the centralized pin's default is moved to 10.0 so the
  lab's cold-rebuild default is the upgraded version (a closing cold-rebuild
  validation confirms a from-zero build at 10.0).

## 7. Deliverables & documentation placement

- **Runbook** (generic, anonymized): `docs/reference/` in the lab repo
  (operational runbook convention, alongside `drbd-operations.md`,
  `rdqm-ha-cheatsheet.md`, `nativeha-crr-setup-guide.md`), e.g.
  `docs/reference/nativeha-mq-upgrade-runbook.md`.
- **Evidence report:** dated `docs/reports/YYYY-MM-DD-nativeha-mq-10-upgrade.md`.
- **Version-centralization** code/config changes across `src/mqlab/`,
  `scripts/`, `ansible/`, `manifests/`.
- **Site docs:** the doc-review bookend (#1069) sweeps `docs/site/` and
  `docs/reference/` for reflection of the new default version and the runbook.
- **spec.md / plan.md:** this epic's design docs, in `.github`.

## 8. Risks & open questions

- **Exact Native HA rolling rules & point of no return** — §5's ordering is
  grounded in IBM docs at design level; the plan pins the precise per-step
  status gates and the data-migration point of no return, caching the IBM pages
  via `tools/ibm_doc_cache.py` under `build/refs/ibm-docs/`.
- **MQ media availability for a two-version test** — the §6 test needs **both**
  the 9.4.5 and the 10.0 developer tarballs staged. IBM rotates older developer
  editions off the public download page when a new LTS ships, so the 9.4.5
  tarball may no longer be freshly fetchable. Mitigation: "both tarballs staged"
  is an explicit, verified **precondition** of the validation task (the 9.4.5
  media is expected to be present in the shared `build/cache/mq` from current lab
  use); the plan confirms the 10.0 developer tarball filename/URL for
  `fetch-mq.sh` in its first spike.
- **Real back-out mechanism for a major-version upgrade** — a major-version
  queue-manager migration is one-way, so "back-out" after the point of no return
  is not an instance roll-back but a restore from a pre-upgrade backup. The plan
  pins the exact NHA/CRR point of no return and states the true back-out
  mechanism plainly, even if that mechanism is "restore from backup."
- **Manual DR failover runbook dependency** — §5.6 exercises the lab's existing
  CRR switchover path (`ansible/site-nativeha-switchover.yml`, the `dr-cutover`
  verb, `docs/reference/nativeha-crr-setup-guide.md`). The upgrade runbook
  references that procedure rather than restating it.
- **Cold-rebuild cost** — the RHEL NHA arm is baked (#88); a full 6-node CRR
  rebuild from 9.4.5 plus the in-place upgrade of both groups plus two DR drills
  is the long pole. Judgement: acceptable but non-trivial; the arm is
  loop-friendly post-baking, and the human operates the lab.

## 9. References

- Appendix C, `docs/specs/2026-06-03-mq-cluster-lab-design.md` — 9→10 gap
  analysis and the C.5 call for this runbook.
- [Introducing IBM MQ v10.0][ann] · [Downloading IBM MQ 10.0 LTS][dl] ·
  [System Requirements for IBM MQ 10.0][sr]
- [Considerations for a rolling update of a Native HA queue manager][ru]
- [Upgrading Native HA IRR configurations (10.0)][irr]
- #266 version-manifest design (prior art; SUT half dropped in #350).

[ann]: https://www.ibm.com/new/announcements/introducing-ibm-mq-v10-0
[dl]: https://www.ibm.com/support/pages/downloading-ibm-mq-100-lts
[sr]: https://www.ibm.com/support/pages/system-requirements-ibm-mq-100
[ru]: https://www.ibm.com/docs/en/ibm-mq/9.2.x?topic=byomcdc-considerations-performing-your-own-rolling-update-native-ha-queue-manager
[irr]: https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=irr-upgrading-native-ha-configurations
