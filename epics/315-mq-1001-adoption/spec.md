# Spec: Adopt IBM MQ 10.0.1 and the support-gated Ubuntu 26 default

- **Epic:** [logical-minds-foundry/.github#315](https://github.com/logical-minds-foundry/.github/issues/315)
- **Predecessor:** [#280 — multi-version OS axis](https://github.com/logical-minds-foundry/.github/issues/280) ([spec](../280-os-version-axis/spec.md), [plan](../280-os-version-axis/plan.md))
- **Repo:** `logical-minds-foundry/mq-resiliency-lab-for-linux` (the lab)
- **Status:** compact spec by decision (2026-10-11). The tasks were already written up in their own issues during #280; this spec fixes scope, sequencing and gates, and does not re-derive them.

## 1. Why

Epic #280 delivered the OS version axis:

- a catalog (`lab/versions.yaml`), resolver and instance records;
- Ubuntu 24/26 and RHEL 9/10;
- every stack × OS row validated by a cold rebuild on MQ **10.0.0.5**.

Four pieces of work were left, and each waits on something outside the lab. They move here so that #280 can close on what it shipped.

1. **IBM MQ 10.0.1** (Continuous Delivery). IBM announcement AD26-0859, <https://www.ibm.com/docs/en/announcements/mq-1001>, published 2026-10-06, Multiplatforms GA **2026-10-13**.
   - **[data]** The announcement lists, among others:
     - "RDQM is updated to support Red Hat Enterprise Linux 10";
     - "Native HA Quantum Safe Encryption" (post-quantum key exchange for Native HA replication);
     - MQTT 5.0 support;
     - MQ Agent / A2A features;
     - the deprecation of MQ Explorer.

     The quotes come from the text the human pasted into #1405 and #1410, because the page refuses automated fetches. They will be re-verified once the 10.0.1 docs publish.
   - **[direction, human 2026-10-09/10]**
     - The lab shows the **leading edge** of IBM MQ, so each new MQ level is an adoption decision.
     - The lab is not released publicly until it runs on the latest MQ.
     - Security comes first: the most secure TLS everywhere, plus a demonstration of post-quantum key exchange wherever MQ supports it.
2. **IBM support for Ubuntu 26.04.**
   - **[data]** IBM's MQ 10.0 support matrix (SPCR) does not list Ubuntu 26.04: see `docs/reference/os-version-support-matrix.md` in the lab, #1271.
   - **[rule, #280 spec §4.1]** A default may only move to an IBM-supported OS. `ubuntu:26` therefore stays selectable for lab use only, and the Ubuntu stacks keep `default: ubuntu:24` until IBM lists 26.04.
   - **[rule, #280 spec §4.1]** The same gate holds back the MQ-bearing shared nodes (`infra_mq: ubuntu:24`). T9 carries both moves (decided in alignment, 2026-10-11).

## 2. Scope

| # | Task | Kind | Gate |
|---|---|---|---|
| A | [lab#1409](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/1409) — re-pin `lab/mq-version` to 10.0.1, with the Developer media checks and the el10 kmod check | code (PR) | 10.0.1 GA, and **all three** Developer tarballs published (LinuxX64, UbuntuLinuxX64, UbuntuLinuxARM64) |
| B | [lab#1440](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/1440) — enable `rdqm-rhel` on RHEL 10 | code (PR) | A merged; an el10 kmod in the 10.0.1 tarball that IBM's `modver` picks for the RHEL 10.2 DVD kernel `6.12.0-211.7.3.el10_2` |
| D | [lab#1442](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/1442) — bake round on 10.0.1: arm64 (macOS) and cloud x86_64 | deployment | A and B closed (B either merged or closed as not done) |
| V | [lab#1443](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/1443) — validation matrix on 10.0.1: every stack × supported OS, cold rebuild, `qm e2e`, human dashboard review | validation | D |

- **A is only the re-pin PR.** The rebake and revalidation are D and V, which are operational tasks. They close only on a recorded `Outcome: SUCCESS`, never on a merge, so the "lab runs on 10.0.1" bar always rests on bake and validation evidence (decided in pushback, 2026-10-11).
- **One repo-wide MQ pin.** If any of the three Developer tarballs lags, A waits and records the wait. **[judgment, human 2026-10-11]** IBM co-releases the flavours in practice: 10.0.0.0 and 10.0.0.5 each appear as one set in the download directory. Per-arch pins are MQ-axis work for .github#282.
| E | [lab#1410](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/1410) — 10.0.1 adoption evaluation (headline: TLS/PQC) | code (docs PR) | 10.0.1 docs published; live parts after V |
| T9 | [lab#1285](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/1285) — flip the Ubuntu stack defaults **and `infra_mq`** to `ubuntu:26` (including the `mq-client-ubuntu26` bake and re-validation of the MQ-commons rows) | code (PR) | Per stack, on the SPCR x86-64 rows for Ubuntu 26.04: `pcmk-ubuntu` and `infra_mq` need the MQ server row; `nativeha-ubuntu` needs the MQ server **and** Native HA rows. Partial flips are allowed |

**TLS/PQC hardening.** E is an evaluation. Its outcome decides whether the hardening becomes tasks in this epic or its own epic (human decision, 2026-10-11). Smaller adoptions that E identifies are filed as tasks under this epic.

## 3. Decisions carried forward (not reopened)

- **Native HA is primary.** RDQM and Pacemaker/DRBD are completeness counter-options, kept while IBM supports them. See the lab's `docs/reference/rdqm-pacemaker-strategic-position.md` (lab#1436, .github#309).
  - A failure that affects only RDQM or Pacemaker is narrowed or deferred. It never blocks Native HA.
  - B is therefore not on the critical path: if its gate fails, record the gap in the support matrix §2.4 and close B as not done. D (#1442) and V (#1443) go ahead without it.
- **RHEL 9 stays at 9.6** (lab#1396). No minor re-pins in this epic.
- **Lab media is the IBM MQ Advanced for Developers tarball** for the pinned CD level.
  - **[data]** Same program number (5724-H72) and the same package versions as the licensed product.
  - **[judgment]** IBM has not stated that the Developer edition is the same build.
- **Cold-rebuild acceptance gate:** a change to how the lab comes up is accepted only after a full cold rebuild completes in one pass.

## 4. Sequencing

```text
10.0.1 GA ─▶ A (re-pin) ─▶ B (rdqm@10, conditional) ─┐
                    └─────────────────────────────────┴─▶ D (bake, both hosts) ─▶ V (matrix)
10.0.1 docs ─▶ E (evaluation; docs part) ── live parts after V ─▶ adoption tasks / TLS-PQC epic decision
SPCR lists 26.04 ─▶ T9 (independent)
all above ─▶ docs review (lab#1441) ─▶ retrospective (.github#317)
```

- B runs before D, so one bake round and one validation round cover both, including the `mq-rdqm-rhel10` bake and the `rdqm-rhel@10` row.
- If B's gate fails, D starts as soon as A merges.

## 5. Acceptance

1. `lab/mq-version` is 10.0.1. Every catalog box is rebuilt on both hosts (D). Every stack × supported OS row passes a cold full-DR bootstrap in one pass, with `qm status` for both sites, `qm e2e --count 20` and a human dashboard review (V).
2. `rdqm-rhel` offers `rhel:10` and is validated in V. If not, the gap is recorded in the support matrix with IBM's evidence.
3. E's cited evaluation is in the lab's `docs/reference/`, with an adopt / demo / skip table and the per-hop TLS/PQC target configuration. Adoption work is filed, and the TLS/PQC placement decision is recorded.
4. T9 is either done, or **still blocked with the SPCR status recorded**. An open T9 must not hold the epic open indefinitely: at the docs-review stage the human either moves it to the ad-hoc epic or a successor, or keeps it waiting (see plan §Gate policy).
5. The docs review sweeps site, dev and reference docs. The retrospective records the outcomes.

## 6. Out of scope

- A selectable MQ version axis (the follow-on brainstorm .github#282).
- RHEL 9 minor re-pins, and new OS majors.
- Re-validating rows the 10.0.1 change does not touch, beyond the full matrix in V.

## 7. Risks

| Risk | Mitigation |
|---|---|
| The 10.0.1 Developer tarballs lag GA, or the arm64 Ubuntu tarball is missing | A's precondition checks fail loud; nothing re-pins until all three flavours exist, and the wait is recorded on A (§2) |
| The 10.0.1 el10 kmods don't match the lab's RHEL 10.2 DVD kernel (offline guests can't take errata kernels) | B is conditional and off the critical path (§3) |
| Behaviour changes in 10.0.1 break bring-up (deprecations, defaults, removed features) | E §1 reads the "what's changed" list; V's cold rebuild catches what remains, and fixes are filed as defects under this epic |
| The exporter or collectors break against 10.0.1 | The exporter cache is keyed by MQ version and rebuilt against the new SDK in D; V checks the metrics are live |
