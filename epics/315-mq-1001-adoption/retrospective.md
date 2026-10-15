# Adopt IBM MQ 10.0.1 and the Ubuntu 26 default — retrospective

- **Epic:** [logical-minds-foundry/.github#315](https://github.com/logical-minds-foundry/.github/issues/315). **Spec:** [spec.md](spec.md). **Plan:** [plan.md](plan.md), reconciled 2026-10-15 (#350).
- **Retrospective task:** [.github#317](https://github.com/logical-minds-foundry/.github/issues/317).
- **Opened → closed:** 2026-10-11 → 2026-10-15 (5 days).
- **Outcome:** DELIVERED.
  - The lab runs IBM MQ 10.0.1.0, cold-validated on both hosts.
  - `rdqm-rhel` runs on RHEL 10 and defaults to it.
  - Ubuntu 26.04 is the Ubuntu default (both stacks and `infra_mq`), by recorded lab decision ahead of IBM.
  - The 10.0.1 adoption review is published, and its TLS/PQC work became its own epic.

Repo shorthand: `lab#N` = `logical-minds-foundry/mq-resiliency-lab-for-linux#N`.

## §0 At a glance

We set out to move the lab to **IBM MQ 10.0.1** on its GA day (2026-10-13, announcement AD26-0859). The plan was to enable **RDQM on RHEL 10** if IBM's el10 kernel modules matched the lab's RHEL 10.2 kernel, to evaluate 10.0.1 for adoption with **TLS / post-quantum** as the headline, and to flip the Ubuntu stacks to **Ubuntu 26** once IBM's support matrix listed it.

**What shipped:**

- IBM released on schedule, and the lab was re-pinned the same day.
- RDQM on RHEL 10 passed cold, including HA failover, DR and SELinux enforcing, and became the `rdqm-rhel` default.
- Ubuntu 26 became the default **proactively**, by a human decision on 2026-10-14. IBM still doesn't list it; the catalog records that truthfully and bootstrap warns about it.
- The adoption review landed. Its TLS/PQC hardening was spun off as epic .github#346.
- Cold validation on both hosts found **eight defects** (#1478, #1480, #1484, #1486, #1488, #1491, #1511, #1557), and implementation found two more (#1472, #1540). All were fixed inside the epic. Two trace to upstream bugs (ibmmq, crmsh); one is filed and the other is queued for reporting.

**Work delivered (merged PRs, queried from GitHub):**

| PR | Merged | Task | What it did |
|---|---|---|---|
| .github#318 | 10-11 | .github#316 | Spec and plan for the epic |
| lab#1469 | 10-13 | lab#1409 | Re-pin to MQ 10.0.1.0: media checks, el10 kmod + `modver` check, docs |
| lab#1471 | 10-13 | lab#1440 | `rdqm-rhel` offers `rhel:10`: el10 vars, `mq-rdqm-rhel10` bake, Pacemaker 3 audit, DVD offline-dependency check |
| lab#1476 | 10-13 | lab#1472 | RDQM Pacemaker/DRBD resource names derived from the QM, not a hard-coded `qmrdqm` |
| lab#1479 | 10-13 | lab#1478 | ibmmq clients use `CMQXC.MQXPT_TCP`; the clients' ibmmq API is checked against the real binding |
| lab#1482 | 10-13 | lab#1480 | ibmmq clients force `MQCNO_CLIENT_BINDING` (workaround for an upstream ibmmq bug) |
| lab#1485 | 10-14 | lab#1484 | `python3-setuptools` on RHEL 10 so IBM's bundled crmsh starts |
| lab#1487 | 10-14 | lab#1486 | SELinux module so DRBD's kernel helpers can run IBM's `rdqmhandler` on RHEL 10 |
| lab#1489 | 10-14 | lab#1488 | `rdqmadm -c` one node at a time per site; fail loud if STONITH setup was skipped |
| lab#1492 | 10-14 | lab#1491 | NIC profile SELinux relabel so `nic-config.sh` hardening saves (found in the epic, filed outside it) |
| lab#1494 | 10-14 | lab#1473 | `rdqm-rhel` defaults to `rhel:10` |
| lab#1495 | 10-14 | lab#1285 | Ubuntu stacks and `infra_mq` default to `ubuntu:26` through a recorded `lab_default_decision` |
| lab#1510 | 10-14 | lab#1508 | Bootstrap warns when `infra_mq` runs MQ on an IBM-unsupported OS |
| lab#1516 | 10-15 | lab#1410 | `docs/reference/mq-10.0.1-adoption.md`: TLS/PQC per hop and every new feature |
| lab#1517 | 10-15 | lab#1514 | `mqlab qm reload` (10.0.1's `systemctl reload mqmonitor@<QM>`) |
| lab#1518 | 10-15 | lab#1515 | RDQM HA heartbeat timeout set to IBM's 10 s VM value |
| lab#1539 | 10-15 | lab#1511 | authz validation reads `dmpmqaut` evidence from stdout and stderr (10.0.1 moved it to stderr) |
| lab#1544 | 10-15 | lab#1540 | `nativeha-find-active` fails loud when no Active is elected |
| lab#1556 | 10-15 | lab#1441 | Docs review: the epic reflected in site, reference and dev docs |
| lab#1561 | 10-15 | lab#1555 | Site guides, standard and template re-pinned from MQ 9.4 to 10.0.x |
| lab#1562 | 10-15 | lab#1557 | SVCQM adopts a failed-over sender's stale receiver (`AdoptNewMCA`); inter-QM `HBINT(10)` |
| .github#354 | 10-15 | .github#350 | The plan reconciled with what shipped |

**Validation (operational tasks, all `Outcome: SUCCESS`):**

- lab#1474 (macOS arm64) and lab#1475 (cloud x86_64): 10.0.1 bake plus full matrix, cold.
- lab#1496 and lab#1497: the Ubuntu 26 defaults plus `infra_mq: ubuntu:26`, cold, on both hosts.
- lab#1546: the RDQM 10 s heartbeat, evidenced by the cloud agent's cold rebuild. Its unplanned-failover step found #1557.

**Counts:**

- **Repos:** 2 (the lab and `.github`).
- **Tasks:** 31 children, all closed.
- **Merged PRs:** 22 (20 lab, 2 `.github`). Two more were spun off into new epics: #346's spec/plan (.github#353) and the upstream-reporting epic .github#351.
- **Releases cut:** none (the lab ships from `develop`).
- **Superseded:** 3 children (lab#1442, lab#1443, .github#337).
- **Upstream reports:** 1 filed (ibm-messaging/mq-mqi-python#29); 3 queued.

## §1 How the plan evolved

The plan was a compact task table (.github#316). It held for the re-pin and RDQM enablement and changed in four places.

1. **Validation split by host.** The plan had one deployment task (bake, D) and one validation task (matrix, V) covering both hosts. On 2026-10-13 the human split them into **per-host validation tasks**, each running its host's bake and then its matrix: macOS run locally, cloud handed to a separate cloud agent. That let the two hosts run in parallel and fail independently, and the pattern was reused for the Ubuntu 26 round (#1496/#1497).
2. **Ubuntu 26 went proactive.** The plan gated T9 on IBM's SPCR listing Ubuntu 26.04, with a move-or-wait choice at the docs review. On 2026-10-14 the human made Ubuntu 26 the default **ahead of IBM**: the lab exists to drive the leading edge and to find and report what breaks. Rather than relabel IBM's status, the catalog policy was changed: an IBM-unsupported default is allowed only through a recorded `lab_default_decision`, and `ibm_support` stays truthful. #1508 then added the `infra_mq` warning so the decision stays visible on every bootstrap.
3. **Validation found defects, and each became a task in the epic.** Two pymqi → ibmmq client regressions (#1478, #1480), four RHEL 10 RDQM defects (#1484, #1486, #1488, #1491), an MQ 10.0.1 behaviour change (#1511), a latent fail-loud bug (#1540), a resource-naming bug (#1472), and an unplanned-failover channel defect (#1557). Each fix merged, and the affected rows re-ran cold.
4. **The adoption review (E) came late and forked.** IBM's 10.0.1 "What's new" page was not reachable on GA day (IBM Docs returned 503). The human found it on 2026-10-14. The review split its follow-ons: two small adoptions stayed (#1514 `qm reload`, #1515 heartbeat); the TLS/PQC hardening became epic #346, brainstormed and planned as a backlog epic; the MQTT 5 demo was declined.

The docs review (#1441) then found the guides still pinned to MQ 9.4 (#1555, done in the epic) and the plan out of date (#350, reconciled before this retrospective).

## §2 Lessons learned

- **Verify GA on the day from primary sources, and expect the docs to lag.** On 2026-10-13 the download page, the Developer tarballs and the RDQM kernel-modules page were all updated, but the SPCR had no 10.0.1 entry, the fix list was stale, and IBM Docs was down. The human's browser check of each page was the reliable confirmation. Next time: check the media plus the kmod page for "go", and treat SPCR and docs as lagging citations, not gates.
- **Test what can be tested offline before the lab gets involved.** IBM's `modver` is a bash `case` on the kernel string, so the el10 kmod choice for the RHEL 10.2 kernel was proven on the dev VM before any bake (#1409). The RHEL 10.2 DVD's offline dependency closure was resolved statically (#1440). Both held up in the cold run.
- **A library migration outside the epic caused two in-epic regressions, and the gate missed both.** The pymqi → ibmmq migration (lab#1470) merged the day validation started. Its tests mocked ibmmq, and `components/` tests aren't run by `vrg-validate` (#1477). The result was a wrong constant module (#1478) and a client-vs-bindings connect mode (#1480), each costing a validation round. Next time: put `components/` tests in the gate (#1477), and validate a client-library swap against the real binding before it lands. #1478 added that check.
- **Validate unplanned failure modes, not just clean ones.** Every earlier failover test reset TCP. #1546 froze a node with `virsh suspend` and found SVCQM refusing the new sender for six minutes (#1557). Validation matrices should include a hang or partition, not only a planned move or a kill.
- **Cold-rebuild discipline found the real defects.** Eight defects surfaced only in cold, single-pass runs on real hosts. No unit gate would have found them.
- **Verify a variable's name against the product docs, not a library's changelog.** The agent tested `MQ_CONNECT_MODE=CLIENT` because ibmmq's CHANGELOG names it, and wrongly reported it "doesn't help". MQ's variable is `MQ_CONNECT_TYPE`, which works. The error was caught in pre-filing research and corrected on #1480 and #1483 before anything went upstream.
- **Capture full evidence, and check your own watchers.** The first matrix kept only the tail of each e2e output, so its 20/20 claims rested on exit codes (fixed for #1496). Two monitor filters missed or over-matched real log lines. A check that reports "NONE" may be a broken pattern, so verify by hand before reporting it.
- **Parallel PRs on the same files: watch for semantic conflicts.** #1473 and #1285 both edited the catalog and docs. Git merged the catalog cleanly, but a test added by #1473 asserted the old Ubuntu default. Only validation of the merged tree caught it.
- **Keep the plan's evolution log as you go.** The plan was reconciled in one pass at the end (#350). A running "Evolution during execution" section would have made §1 a synthesis rather than a reconstruction.

## §3 Compromises & tradeoffs

- **Dashboard attestation waived** (validation step 7) for this epic's validations (human decision, 2026-10-13). The dashboards have known issues and get their own review epic. Step 7 passed on infrastructure evidence (targets up, series live) instead.
- **Ubuntu 26 is the default ahead of IBM support.** This was a deliberate leading-edge choice, kept honest: `ibm_support: unsupported` stays with its SPCR citation, and the stack and `infra_mq` warnings print on every bootstrap. **At close (2026-10-15T22:03Z):** IBM's MQ 10.0 SPCR (`FA94B1F6888342969E7F9B505149F032`, lastModified 2026-09-28) and 10.0.0.5 SPCR (`9CA6A0D6BAD147BEBB45A74CAF2CC1B5`, lastModified 2026-10-01) still list only **Ubuntu 24.04 LTS**, and the deliverable chain has no 10.0.1 entry. Source: <https://www.ibm.com/software/reports/compatibility/clarity-reports/report/json/getTSROsSupportSummary?deliverableId=FA94B1F6888342969E7F9B505149F032>.
- **Temporary client-binding workaround.** `mqrc/mqconn.connect_client` forces `MQCNO_CLIENT_BINDING` until ibmmq fixes upstream #29. It is set to expire: lab#1545 re-checks it at the next MQ upgrade.
- **The RDQM heartbeat uses IBM's VM value (10 s) on the cloud host too,** not IBM's "cloud" value (30 s). The RDQM nodes are VMs either way, and 30 s would triple failure detection and break IBM's under-4× step rule (judgment, recorded in #1515). The cost is a slower failover than with the old 3 s default.
- **`AdoptNewMCACheck` drops `ADDRESS`** (#1557), because a failed-over sender's source address changes. IBM names mutual TLS with SSLPEER as the protection, and these channels have both. `HBINT(10)` on inter-QM channels trades a little heartbeat traffic for roughly 40 s recovery instead of 6 minutes.
- **`mqlab qm reload` refuses on Pacemaker** rather than running `endmqm -r` behind the resource manager (judgment, #1514).
- **#1546's evidence came from the cloud agent's concurrent cold rebuild,** not a dedicated run (human decision). That run included the #1515 change and found #1557.
- **The guides re-pin was docs-only** while the lab was in use. Some claims are marked "to verify on the next bring-up" (`AUTHEVSC` default, `amqsevt -o json_compact`, the 2042 reopen wait, `AMQ5793`, Native HA log-lifecycle details). They are tracked by validation task lab#1566 (ad-hoc epic), run at the next Native HA bring-up.

## §4 New problems & opportunities

| Surfaced | Where it went |
|---|---|
| TLS/PQC: post-quantum key exchange has been available on channels since 10.0, and the lab used it nowhere; Native HA replication was on TLS 1.2 | **Epic .github#346**, "Enterprise TLS floor". Brainstormed, spec and plan merged (.github#353), backlog filed (.github#349, lab#1522–#1538); starts after mqweb (.github#341) |
| Upstream bugs: ibmmq connect binding; crmsh 4.6 `distutils` (ClusterLabs, LINBIT, IBM) | ibm-messaging/mq-mqi-python#29 filed. The rest are queued in **epic .github#351** (lab#1551–#1554), to be filed at the lab's public release |
| Non-customers have no IBM defect channel | Logged. The researched route is upstream owners (ClusterLabs, LINBIT) plus the TechXchange community (#1553) |
| `components/` tests outside the validation gate | lab#1477 (ad-hoc epic), not yet acted on |
| Temporary client-binding workaround | lab#1545 (ad-hoc), triggered at the next MQ upgrade |
| MQ major-version axis (MQ 9/10 selectable) | Seed .github#282, moved to the ad-hoc epic and deferred |
| Guide claims to verify on the next bring-up | lab#1566 (ad-hoc epic), a validation task for the next Native HA bring-up |
| `mq-resiliency-observability` repo collector drift (`qmrdqm`) | Not acted on: the repo is being rebuilt from scratch (human decision) |
| Dashboards (attestation waived) | The planned dashboard-review epic |

## §5 What's next

- **Epic .github#346, the enterprise TLS floor:** the planned follow-on for TLS/PQC hardening. Backlog until the mqweb epic (.github#341) lands; its first task re-baselines its spec and plan.
- **Epic .github#351, upstream and third-party defect reporting:** queued reports, filed at the lab's public-release gate after a reproducibility re-check. It collects new reports as they are found.
- **Ad-hoc follow-ups:** lab#1545 (retire the client-binding workaround at the next MQ upgrade), lab#1477 (`components/` in the gate), .github#282 (MQ version-axis seed).
- **Pattern worth keeping:** a temporary workaround gets an **expiry issue tied to a trigger event** (here, the next MQ re-pin). The next upgrade epic's re-pin task should link lab#1545.
