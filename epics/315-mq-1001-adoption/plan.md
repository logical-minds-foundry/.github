# Plan: Adopt IBM MQ 10.0.1 and the Ubuntu 26 default

- **Epic:** [logical-minds-foundry/.github#315](https://github.com/logical-minds-foundry/.github/issues/315). **Spec:** [spec.md](spec.md).
- **Form:** compact by decision (2026-10-11). Each task's full steps, files and acceptance live in its issue body (or its procedure comment, for the operational tasks). This plan fixes order, gates, routing and done-ness. It does not restate them.
- **Reconciled 2026-10-15** (.github#350) with what shipped:
  - The single bake (D) and matrix (V) tasks were split into per-host validations.
  - The Ubuntu 26 SPCR gate (T9) was replaced by a proactive decision.
  - The tasks filed during the epic were added.

  The original 2026-10-11 rows are kept struck through, so a reader can see what changed.

Repo shorthand: `lab#N` = `logical-minds-foundry/mq-resiliency-lab-for-linux#N`.

## Task table

| ID | Issue | Kind | Run via | Blocked by | Done when |
|---|---|---|---|---|---|
| DOC | .github#316 | docs | this PR | — | This spec and plan merged |
| A | lab#1409 | code | `issue-implement` | 10.0.1 GA and all three Developer tarballs | Re-pin PR merged; kmod findings recorded on lab#1440 |
| B | lab#1440 | code | `issue-implement` | A, plus its own el10/`modver` gate | `rdqm-rhel` offers `rhel:10` (PR merged), **or** closed as not done with the gap in the support matrix §2.4 |
| ~~D~~ | ~~lab#1442~~ | ~~deployment~~ | — | — | **Superseded 2026-10-13** (human decision): split by host into V-mac / V-cloud below |
| ~~V~~ | ~~lab#1443~~ | ~~validation~~ | — | — | **Superseded 2026-10-13** (human decision): split by host into V-mac / V-cloud below |
| RN | lab#1472 | code | `issue-implement` | B (found during it) | RDQM Pacemaker/DRBD resource names derived from the QM (PR merged) |
| V-mac | lab#1474 | validation | `issue-validate`, locally | A, B, RN | SUCCESS: macOS arm64 bake round plus matrix rows on 10.0.1 (cold rebuild) |
| V-cloud | lab#1475 | validation | `issue-validate`, by the cloud agent | A, B, RN | SUCCESS: cloud x86_64 bake round plus matrix rows on 10.0.1, including the first `rdqm-rhel@10` (cold rebuild) |
| RD | lab#1473 | code | `issue-implement` | V-cloud | `rdqm-rhel` default flipped to `rhel:10` (PR merged) |
| E | lab#1410 | code (docs) | `issue-implement` | 10.0.1 docs published (docs part); V-mac / V-cloud (live parts) | Cited `docs/reference/mq-10.0.1-adoption.md` merged; adoption tasks filed; TLS/PQC placement decided (see the row E note) |
| T9 | lab#1285 | code | `issue-implement` | RD (both edit the catalog and support matrix); **no IBM gate** (2026-10-14 decision) | `ubuntu:26` is the default for both Ubuntu stacks and `infra_mq` by recorded lab decision (PR merged) |
| V26-mac | lab#1496 | validation | `issue-validate`, locally | T9 | SUCCESS: `mq-client-ubuntu26` bake plus the full Ubuntu matrix on the new defaults |
| V26-cloud | lab#1497 | validation | `issue-validate`, by the cloud agent | T9 | SUCCESS: `mq-client-ubuntu26` bake plus the full RHEL matrix, including `rdqm-rhel@10` as the default |
| W | lab#1508 | code | `issue-implement` | T9 | Bootstrap warns when `infra_mq` runs MQ on an IBM-unsupported OS (PR merged) |
| F5 | lab#1514 | code | `issue-implement` | E | `mqlab qm reload` (PR merged) |
| F6 | lab#1515 | code | `issue-implement` | E | RDQM HA heartbeat timeout 10 s (PR merged) |
| F6-V | lab#1546 | validation | `issue-validate`, by the cloud agent | F6 | SUCCESS: `rdqm-rhel@10` and `@9` cold with the 10 s timeout. By human decision, evidenced by the cloud agent's in-progress cold rebuild, which includes F6 |
| AD | lab#1557 | code | `issue-implement` | F6-V (found by it) | SVCQM adopts a failed-over sender (`AdoptNewMCA=ALL`, `AdoptNewMCACheck=NAME,QM`) and inter-QM `HBINT(10)` (PR merged, lab#1562) |
| G | lab#1555 | code (docs) | `issue-implement` | DR (found by it) | Site guides, standard and template re-pinned from MQ 9.4 to 10.0.x (PR merged) |
| DR | lab#1441 | docs review | `issue-implement` (sweep) | A, B, V-mac, V-cloud, E, T9, V26-mac, V26-cloud resolved, plus the tasks above | Site, dev and reference docs reflect the epic |
| RETRO | .github#317 | retrospective | `epic-retrospective` | every other child closed | `retrospective.md` merged; the epic closes |

**Defects found in validation and fixed in the epic:**

| Issue | Defect |
|---|---|
| lab#1478 | `CMQXC.MQXPT_TCP` in the ibmmq clients |
| lab#1480 | `MQCNO_CLIENT_BINDING` forced in the clients (a temporary workaround, retired by lab#1545) |
| lab#1484 | crmsh `distutils` on RHEL 10 |
| lab#1486 | RHEL 10 SELinux module for RDQM's DRBD handlers |
| lab#1488 | `rdqmadm -c` lock race |
| lab#1491 | NIC profile SELinux relabel |
| lab#1540 | `nativeha-find-active` fails loud when no Active is elected |
| lab#1511 | authz validation reads `dmpmqaut` evidence from stdout and stderr (MQ 10.0.1 writes it to stderr) |

**Upstream reporting:**

- lab#1483: the ibmmq client-binding bug, filed by the human as [ibm-messaging/mq-mqi-python#29](https://github.com/ibm-messaging/mq-mqi-python/issues/29). Closed.
- .github#337: the RDQM crmsh dependency report. Superseded and queued in the upstream-reporting epic .github#351 (lab#1551–#1553), to be filed at the lab's public-release gate.

**Row E note (lab#1410, closed).** The docs gate opened on 2026-10-14. Dispositions (human, 2026-10-15):

- TLS/PQC hardening → a new epic, .github#346;
- F5 (lab#1514) and F6 (lab#1515) filed in this epic;
- the MQTT 5 demo declined;
- mqweb notes recorded on .github#266;
- outcomes recorded for this epic's retrospective (.github#317), not #280's.

## Order of work (as executed)

1. **DOC.** E's docs part waited for IBM's 10.0.1 "What's new" page, which the human located on 2026-10-14.
2. **At GA (2026-10-13):** **A**, then **B**. B surfaced **RN**.
3. **V-mac** and **V-cloud** (replacing D and V). Defects lab#1478 and #1480 (and, on cloud, #1484, #1486, #1488 and #1491) were fixed in the epic. The affected rows were re-run cold.
4. **RD** after V-cloud's `rdqm-rhel@10` row passed.
5. **T9** (proactive, after RD), then **V26-mac** and **V26-cloud**, then **W**.
6. **E**, then its follow-ons **F5** and **F6** (with **F6-V**).
7. **DR** (which found **G**), then **RETRO**.

## Gate policy

- **10.0.1 media.** Nothing re-pins until GA and all three Developer tarballs exist (spec §2). A records any wait as a comment.
- **T9 (Ubuntu 26 defaults and `infra_mq`), superseded 2026-10-14.** The original rule (flip each change only when its SPCR rows exist, with a move-or-wait choice at DR) was replaced by the human's decision to make `ubuntu:26` the default **proactively**, ahead of IBM's SPCR.
  - The catalog's `lab_default_decision` field (lab#1285) records the decision. `ibm_support` stays truthfully `unsupported`.
  - lab#1508 surfaces the `infra_mq` side on every bootstrap.
  - IBM's Ubuntu 26.04 SPCR status (still unlisted) is recorded at close in the retrospective §5, with the `getTSROsSupportSummary` URL.
- **Cold-rebuild gate.** Validation tasks close only on SUCCESS evidence, never on a merge. Any defect a validation finds is filed as a task under this epic, and the affected rows are re-run cold after the fix merges.
- **Dashboard attestation** (validation step 7) was waived for this epic's validations (human decision, 2026-10-13). Step 7 passed on infrastructure evidence; the dashboards get their own review epic.

## Defect handling

- File each defect with `vrg-issue-create --epic logical-minds-foundry/.github#315`. Implement it in its own worktree, and leave it at report-ready for the human to submit.
- Re-run only the rows the fix affects, cold. Record in the validation result why the other rows didn't need a re-run (the #1284 precedent).
- Upstream defects are drafted for the human to file. From 2026-10-15 they are queued in .github#351 until the lab's public release.

## Notes for implementers

- **Catalog lines stay single-line `{...}`.** `lab/scripts/rhel-catalog-isos.sh` (awk) depends on it.
- **Version tokens appear only in the catalog and in resolved box names** (`tests/test_version_token_guardrail.py`).
- **IBM Docs:** use `python3 tools/ibm_doc_cache.py` (pinned to 10.0.x). Support pages and SPCR JSON can be fetched with curl using a browser User-Agent. Label each claim as data or judgment and give full URLs.
- **Long lab jobs:** run them detached, with stdin from `/dev/null` and a log-mtime stall watch. Capture the **full** e2e output.
- **Validation:** `vrg-container-run -- vrg-validate` is the only gate.
