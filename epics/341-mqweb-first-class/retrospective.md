# MQ Web (Console + REST) as a first-class, secured lab component — retrospective

- **Epic:** [logical-minds-foundry/.github#341](https://github.com/logical-minds-foundry/.github/issues/341). **Spec:** [spec.md](spec.md). **Plan:** [plan.md](plan.md), amended twice during execution (.github#358, .github#360).
- **Retrospective task:** [.github#343](https://github.com/logical-minds-foundry/.github/issues/343).
- **Opened → closed:** 2026-10-14 → 2026-10-19 (6 days).
- **Outcome:** DELIVERED.
  - mqweb runs on every queue manager node, on every stack and svc-sim, by default. The `mqweb_enabled` gate is gone.
  - Anonymous access is retired. Three sample logins map to IBM's three roles, and a queue-manager-enforced operator tier backs the third.
  - `mqlab console open <stack>` follows the live queue manager: the VIP on RDQM and Pacemaker, a state probe on Native HA.
  - Cold-validated on cloud x86 and macOS/arm64. Accepted live on cloud x86.

Repo shorthand: `lab#N` = `logical-minds-foundry/mq-resiliency-lab-for-linux#N`.

## §0 At a glance

IBM MQ 10.0.1 deprecates MQ Explorer and makes the MQ Console the primary administration experience. The lab had built most of the mqweb groundwork, but it ran mqweb **off by default** (`mqweb_enabled: false`, lab#1171): per-node Liberty cold starts had been infeasible on the 2-vCPU QM nodes. The host-level fixes (lab#1240, lab#1241) had since removed that obstacle. We set out to turn mqweb back on everywhere and secure it, following IBM 10.0.x guidance to the letter, with every departure recorded in a deviations register (spec §8).

Client-mode mqweb was evaluated in the brainstorm and **rejected** on IBM's own documentation. The administrative REST API reaches a remote queue manager only through a gateway queue manager, and the lab adds no extra queue manager (spec §2.1). The epic's working title, "mqweb client mode", outlived that decision; this task's title was corrected at close.

**What shipped:**

- A measurement spike (T1) gated the epic. Per-node mqweb measured cheap on cloud x86: about 21 s first start and about 170 MB steady RSS per node. The spike also found a design flaw before any code was written: the planned endpoint probe would have treated a healthy Native HA replica as a failure.
- An IBM-role registry: `mqadmin` (`MQWebAdmin`), `mqreadonly` (`MQWebAdminRO`) and `mqoperator` (`MQWebUser`). Every password, the keystore password included, is encoded once with one server AES key (IBM's S16 procedure).
- An `mqoperator` OS account on every QM node, with IBM-minimum `SET AUTHREC` grants on every QM. Its writes are refused by the queue manager, with `MQRC 2035` and an authority event in Loki.
- mqweb on by default, keeping the serial start and readiness gate.
- `mqlab console open`, built on the "follow the active" state probe. The unused `mqlab rest` verb, `endpoints.json`, the `pymqrest` dependency and the `CN=pymqrest` PKI entity are removed.
- The docs sweep (lab#1490): per-node mqweb and its security model in the site docs.

**Work delivered (merged PRs, queried from GitHub):**

| PR | Merged | Task | What it did |
|---|---|---|---|
| .github#345 | 10-14 | .github#342 | Spec and plan for the epic |
| lab#1574 | 10-16 | lab#1499 (T1) | Spike findings and gate verdict (`docs/reference/mqweb-spike-findings.md`) |
| .github#359 | 10-16 | .github#358 | Spec and plan amended for the spike: replica-aware discovery; arm64 half deferred to T6/T7 |
| lab#1575 | 10-16 | lab#1569 (B1) | Every `build/` bucket resolves through the `mqlab` authority, so a worktree run no longer forks the state bucket |
| lab#1576 | 10-16 | lab#1570 (B2) | `lab-secret.sh` mints only registered secret names and rejects option-like arguments |
| lab#1580 | 10-16 | lab#1500 (T2) | IBM-role registry, S16 encode-once passwords, anonymous Console retired |
| lab#1586 | 10-16 | lab#1501 (T3) | `mqoperator` account and IBM-minimum grants on every QM |
| lab#1590 | 10-16 | lab#1502 (T4) | mqweb on every QM node by default; `mqweb_enabled` removed |
| lab#1594 | 10-16 | lab#1503 (T5) | `mqlab console open` follows the live mqweb; `mqlab rest` and `pymqrest` removed |
| lab#1599 | 10-16 | lab#1598 | The registry template is the only writer of `mqwebuser.xml` (re-provision no longer restarts mqweb) |
| lab#1607 | 10-17 | lab#1605 | `console open` stops a previous forwarder only when one exists |
| lab#1610 | 10-17 | lab#1608 | The mqweb endpoint verification runbook rewritten for role logins and `console open` |
| .github#363 | 10-17 | .github#360 | macOS/arm64 validation limited to the Ubuntu stacks (RHEL is x86-only) |
| lab#1616 | 10-17 | lab#1614 | Spike findings' macOS/arm64 half filled; the `ready_s` method corrected |
| lab#1618 | 10-17 | lab#1490 | Docs sweep: per-node mqweb and its security model |
| .github#366 | 10-17 | .github#365 | T6 results recorded in the spec (§3.2, §7, §8) |
| lab#1631 | 10-19 | lab#1624 | Event and diagnostics host prep on the DR-site nodes (`rdqm_b`, `pcmk_b`); runbook's RDQM failover step corrected |

**Validation (operational tasks, all `Outcome: SUCCESS`):**

- **lab#1504 (T6, cloud x86):** all four stacks cold, one pass each, with per-node figures for 28 node-runs, and re-provision idempotence. Two intermediate failures each became a fix (lab#1596, lab#1598) and re-ran green.
- **lab#1595 (T6, macOS/arm64):** `pcmk-ubuntu` and `nativeha-ubuntu` cold, one pass each. The REST replica table matched cloud x86 exactly. The human checked all three role logins in a browser.
- **lab#1505 (T7, cloud x86):** live acceptance on `nativeha-rhel-crr`, `rdqm-rhel` and svc-sim, both sites, through in-site failover, DR cutover and fail-back. Reduced scope and one waiver, both by human decision (§3).

**Counts:**

- **Repos:** 2 (the lab and `.github`).
- **Tasks:** 21 children, all closed at this retrospective's merge. .github#344 (the follow-on brainstorm) was moved to the ad-hoc epic as a parked idea.
- **Merged PRs:** 17 (13 lab, 4 `.github`), plus this retrospective.
- **Releases cut:** none (the lab ships from `develop`).
- **Deviations from IBM guidance:** 3 (D1–D3, spec §8), all recorded at design time. Validation added none.

## §1 How the plan evolved

The plan held for the implementation tasks T2–T5, which merged on a single day (10-16). It changed in four places, all on the validation side.

1. **The spike's surprise rewrote discovery before any code depended on it.** The plan's state probe used the per-QM REST query. On a healthy Native HA replica that query answers HTTP 500 (reason 2543, `MQRC_STANDBY_Q_MGR`), so the probe would have reported a login failure. The spike's provisional verdict was PAUSE. The human accepted the fix: the "follow the active" algorithm over the list query, which reports `running`, `replica` or `recoveryGroupLeader` (spec §2.3, .github#358).
2. **The macOS/arm64 half of the spike was deferred** into T6 and T7, because the macOS lab was busy with another effort. The human accepted the small risk that arm64 differs (spec §7). T6 retired that risk.
3. **The arm64 checks were sent to the wrong stacks.** The plan aimed them at `nativeha-rhel-crr`, but IBM ships no RHEL-on-ARM MQ, and `mqlab bootstrap` refuses a RHEL stack on aarch64 (lab#847). lab#1595 found it when it ran; .github#360 moved the arm64 half to `nativeha-ubuntu` (spec §4).
4. **T7 was reduced to the two RHEL stacks, run serially.** The plan's precondition was "every stack is up". The cloud host cannot run more than one stack at a time without slowing everything, and the epic had already cold-bootstrapped every stack with mqweb several times. The human cut T7 to `nativeha-rhel-crr`, `rdqm-rhel` and svc-sim, each bootstrapped, validated and torn down in turn (recorded on lab#1505). The plan text was not reconciled for this; lab#1505 is the record.

Validation found five defects, each fixed inside the epic: lab#1569 and #1570 (found by the spike), lab#1598 and lab#1605 (T6), and lab#1624 (T7). Four more problems surfaced along the way and were fixed outside it (lab#1596, lab#1602, lab#1606, vergil-vm#322; §4).

## §2 Lessons learned

- **Spike first, and let it pause the epic.** The spike took under two days, opened to closed, and caught a probe design that would have failed on every Native HA replica. Because the spec made "any surprise pauses the epic" a decision rather than a hope, the fix went into the spec before T5 was written, not into a rework PR after it.
- **Check a measurement method against how the service starts.** The plan's `ready_s` started the clock after `systemctl restart` returned. mqweb's unit is `Type=forking`, so the restart blocks until Liberty is nearly up, and `ready_s` read 0.07–0.32 s. The macOS run noticed the figure was implausible and recorded `restart_to_ready_s` with the clock started before the restart (3.3–6.7 s). lab#1614 corrected the findings. The cloud spike's warm-restart figure could not be re-checked, because it did not record its exact command. Next time: write the measurement command into the evidence, not only the result.
- **One writer per file, or idempotence fails.** The registry template wrote `mqwebuser.xml`, then `setmqweb properties -k httpHost` re-serialized it. The content was byte-identical, but the template saw a different file on every run, reported `changed`, and restarted mqweb on every re-provision (lab#1598). The fix made the template the only writer. Acceptance criterion 7 (a re-provision changes nothing) caught it; a unit test of the template alone could not have.
- **Validate both sites, not just site A.** The `mqoperator` refusal was correct on site B, but the authority event never reached Loki: `site-rdqm.yml` ran the event collector host prep on `rdqm_a` only, and `pcmk_b` had the same gap (lab#1624). Every site-A check had passed. Only running the full criterion after a DR cutover found it. lab#1624 added a topology-wide guard test.
- **Execute the runbook; reading it is not enough.** The runbook's RDQM failover step (`endmqm -r`) ended and disabled the queue manager instead of failing it over. It had been reviewed when it was rewritten (lab#1608) and was wrong anyway. T7 ran it, found it, and lab#1624 corrected it to `rdqmadm -s` / `-r`.
- **A worktree must not fork shared state.** The spike, run from a worktree, wrote secrets into a private copy of the state bucket, because `lab-secret.sh` derived `build/state` from its own path (lab#1569). Landing that fix before T6, as the plan recommended, kept the cold rebuilds clean.
- **Name the host before blaming the component.** On macOS the guests froze for 3–163 s: Pacemaker fenced nodes and Native HA failed over. An A/B run with mqweb stopped on every QM node still produced 21 stalls in 20 minutes, which cleared mqweb. The cause was host CPU oversubscription (lab#1602), not this epic. The A/B test kept a host problem from becoming a false finding against mqweb.
- **Keep scope decisions where the validation lives.** T7's reduced scope and the Console waiver are recorded on lab#1505, with the human's reasons, before the result. That made the SUCCESS honest about what it covered.

## §3 Compromises & tradeoffs

- **T7 ran on the RHEL stacks only** (human decision, 2026-10-17). `pcmk-ubuntu` and `nativeha-ubuntu` were cold-bootstrapped with mqweb on both platforms (lab#1504, lab#1595), and `nativeha-ubuntu`'s REST replica table and browser logins were checked on macOS. But T7's per-role write refusals, authority events and failover discovery were not run on them. **[judgment]** mqweb's behaviour does not depend on the HA stack beneath it, and the VIP path (`rdqm-rhel`) and the state-probe path (`nativeha-rhel-crr`) were both exercised, so the gap is small. It is a gap, not a pass.
- **The Console UI per-role check was waived on cloud x86** (human decision). The dev VM has no browser, and reaching 9443 from the workstation would have meant rebuilding the dev VM to pick up the vergil-vm#322 firewall fix. The REST checks show the same per-role enforcement, and lab#1595 covered the browser on macOS.
- **`pcmk-drbd-ubuntu` is outside this epic's acceptance.** The stack (epic .github#310) was registered after the spec fixed its scope (spec §4). It runs a per-node mqweb, but no T7-style acceptance has run on it. **[judgment]** Epic #310's own validation is the natural place for it.
- **Password = username for the three logins (D2).** A deliberate lab convention, recorded as a deviation from IBM guidance: an ephemeral, data-less lab, with every mechanism still wired. **Never carry this to a real deployment.**
- **`httpHost=*` (D1).** mqweb answers on every NIC, management included, so it can serve a floating VIP. Plane enforcement is deferred lab-wide (epic #39 §6).
- **The state probe checks the certificate chain but not the hostname.** The lab's `mqweb` certificate is `CN=mqweb` with a single SAN, so hostname verification would fail. lab#1498 (open) issues a name-based certificate and re-enables it.
- **Accepted host constraint on macOS** (human decision). The guest starvation (lab#1602) is a single-workstation limit. The lab moves to a larger host later rather than shrinking its VMs.

## §4 New problems & opportunities

| Surfaced | Where it went |
|---|---|
| The obs box bake dropped SSH during fact gathering on a 2 GB bake guest (T6 cloud) | lab#1596, fixed outside the epic: the bake guest is sized from topology |
| macOS guest vCPU starvation: corosync stalls, fencing, Native HA failovers (T6 macOS) | lab#1602, closed; shipped `mqlab vm starvation` to name the condition |
| The `mqmon` exporter identity floods the active Native HA instance's error log (about 5/s `AMQ8245W`/`AMQ8009W`) | lab#1606, fixed outside the epic |
| The cloud firewall admitted only tcp:3000, so the Console's 9443 was unreachable | vergil-vm#322, fixed; needs a dev VM rebuild to take effect |
| Grafana's anonymous admin conflicts with the new credential convention (out of scope, spec §4) | lab#1493, closed 2026-10-16 |
| The state probe skips hostname verification | lab#1498, open |
| Teardown leaves the Console forwarder running, pointing at a gone address | lab#1619 (ad-hoc), open |
| Teardown destroys the commons implicitly, which races a concurrent bootstrap | lab#1620 (ad-hoc), open: the commons go down only when asked |
| A REST endpoint-discovery helper across the MQ API libraries | .github#344, parked as an idea in the ad-hoc epic; spec §2.3's "follow the active" algorithm is its starting point |

## §5 What's next

- **Epic .github#346, the enterprise TLS floor,** was waiting for this epic. Its round B validations (lab#1536 macOS, lab#1537 cloud) cover mqweb, and its first task re-baselines its spec and plan against what shipped here.
- **lab#1498,** a name-based mqweb certificate, so the state probe can verify hostnames.
- **lab#1619 and lab#1620,** the two teardown fixes found at the end. They touch the same code path and fit one PR or two back-to-back.
- **Pattern worth keeping:** when a validation's scope is cut, record the cut and the reasons on the task before the result, and carry the uncovered part into the retrospective as a named gap (§3). The SUCCESS line then means exactly what it says.
