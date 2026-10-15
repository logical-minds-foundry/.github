# Migrate the resiliency lab from pymqi to ibmmq — Retrospective

Epic: [logical-minds-foundry/.github#300](https://github.com/logical-minds-foundry/.github/issues/300) ·
Spec: [`spec.md`](spec.md) · Plan: [`plan.md`](plan.md)

## §0 At a glance

**What we set out to do:** replace the unmaintained `pymqi` binding with IBM's maintained `ibmmq` in one big-bang PR, then prove it live on both lab platforms (ARM64 and x86_64) with a cold rebuild.

**What shipped:** the migration landed as one PR (#1470), as planned. The live proof was much more work. It took two regression fixes for `ibmmq` itself, an upstream bug report, and two to three validation runs per platform. Those runs exposed six more defects, and none of them was in the `ibmmq` binding. Both platforms finished green on 2026-10-15: x86_64 at #1362 run 3, and ARM64 at #1509 run 2. No `pymqi` reference remains outside historical records, and a guard test keeps it that way.

**Span:** 2026-10-05 → 2026-10-15. The epic was paused twice: for epic #294 (component packaging), and for the IBM MQ 10.0.1 upgrade.

**Counts:** 17 child tasks: 12 completed, 4 closed not-planned (#1481, #1548, mq-resiliency-observability#42 and #47), and this retrospective. 12 PRs merged across 2 repos, plus this retrospective's own. Four issues the epic depended on were never linked to it: #1478, #1480, #1483 and #1511. The 12 PRs include the fix PRs for #1478, #1480 and #1511. No releases were cut.

| PR | Repo | Merged | What it did |
|---|---|---|---|
| [#334](https://github.com/logical-minds-foundry/.github/pull/334) | .github | 2026-10-13 | Spec and plan (#301) |
| [#336](https://github.com/logical-minds-foundry/.github/pull/336) | .github | 2026-10-13 | Dropped the on-hold observability repo from scope and deferred #302 (#335) |
| [#1470](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1470) | lab | 2026-10-13 | The big-bang migration: every client from `pymqi` to `ibmmq`, plus the guard test (#1360). +624/−446, 44 files |
| [#1479](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1479) | lab | 2026-10-13 | `MQXPT_TCP` lives in `CMQXC`; adds the selfcheck's check of every `ibmmq` name against the real binding (#1478) |
| [#1482](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1482) | lab | 2026-10-13 | Force `MQCNO_CLIENT_BINDING` on every connect through one helper (#1480) |
| [#1539](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1539) | lab | 2026-10-15 | Authz evidence reads `dmpmqaut` stdout and stderr (MQ 10.0.1 moved its output) (#1511) |
| [#1541](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1541) | lab | 2026-10-15 | mq-bench gets its own authorized `BENCH.COMMIT` queue (#1512) |
| [#1542](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1542) | lab | 2026-10-15 | mq-svc-responder dead-letters unanswerable messages and backs out on error (#1513) |
| [#1558](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1558) | lab | 2026-10-15 | Authz exporter check reads the port from the exporter's YAML config (#1547) |
| [#1559](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1559) | lab | 2026-10-15 | A unique MsgId on every DR and reconnect-probe put (#1549) |
| [#1560](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1560) | lab | 2026-10-15 | mq-dr-flow takes only its own replies (per-run CorrelId), rejects foreign messages, and fails loud (#1550) |
| [#1563](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1563) | lab | 2026-10-15 | Documentation review gaps (#1361) |

**Validation runs:**

| Platform | Task | Run 1 | Run 2 | Run 3 |
|---|---|---|---|---|
| x86_64 (cloud) | #1362 | FAILURE 2026-10-15 00:32 → #1511, #1512, #1513 | FAILURE 17:21 → #1547 | **SUCCESS** 20:55 |
| ARM64 (Lima) | #1509 | FAILURE 16:06 → #1547 (plus the #1549 and #1550 findings) | **SUCCESS** 20:48 | — |

## §1 How the plan evolved

`plan.md` has no "Evolution during execution" log. This narrative is put together from the decisions recorded on the epic and its tasks.

1. **Paused for #294 (2026-10-05).** Pushback on the first spec found that epic #294 was about to rebuild the very code paths this epic would edit, by carving the clients into the `mq-resiliency-clients` component. #294 went first. The first draft spec was then **lost**: it was committed only locally, and the VM was rebuilt. It was rewritten from scratch on 2026-10-13 against the componentized `develop`, and that is the spec published here.
2. **Scope narrowed (2026-10-13, #335).** The `mq-resiliency-observability` repo, on hold under epic #319, was dropped from scope. The aarch64 wheel request (#302) was deferred.
3. **Interleaved with MQ 10.0.1 instead of sequenced.** `develop` already pinned MQ 10.0.1. Rather than reverting the migration, the epic recorded a triage table to attribute each failure to MQ 10.0.1 or to `ibmmq`, plus a revert tripwire for failures in `ibmmq`. Two `ibmmq` regressions did surface (#1478, #1480). Both were **fixed forward the same day** rather than reverted. **[judgment]** That was the right call, because both were small and contained. But the epic record has no explicit decision overriding the tripwire, so the departure from it went undocumented.
4. **Validation was narrowed and split.** The plan had one Operational V1 task covering both platforms. On 2026-10-13 it was split into one issue per platform (#1362 for x86_64, #1509 for ARM64) and limited to `nativeha-ubuntu`. `pcmk-ubuntu` and its authz playbook were skipped, because that stack is to be deprecated. So `mq-dlq-probe` was never exercised live.
5. **Fix-forward grew the epic.** The validation runs filed six fix tasks: #1511, #1512, #1513, #1547, #1549 and #1550. The plan's fix-forward rule said a behaviour-changing fix should be recorded as a modernization candidate rather than folded in. By human decision, #1549 (one MsgId reused across puts) and #1550 (a DR reader that took foreign messages and failed silently) were **fixed inside the epic** instead. The reason: both broke the lab's no-silent-failure rule.
6. **The follow-on brainstorm (#303) was cut short on purpose**, so the epic could close. See §5.

## §2 Lessons learned

- **The binding swap was the easy part.** #1470 needed only two follow-up fixes. Six of the eight defects found while validating were not in `ibmmq`: MQ 10.0.1 behaviour changes, authorization gaps, a stale playbook idiom, and two client bugs carried over unchanged from the `pymqi` era. A live end-to-end validation finds defects that a migration diff never touches. Keep budgeting for it.
- **A fake test module can't catch a name the real binding lacks** (#1478). The fake is written from the same belief as the code. The fix, checking every `ibmmq.*` name the package uses against the real sdist and the compiled binding (#1479), is reusable for any binding migration.
- **One failure can hide another.** #1547 (the authz exporter lookup broken since #947, in August) was hidden on x86_64 run 1, because an earlier assert in the same step failed first (#1511). Re-running a step after fixing the first failure is not optional.
- **Two agents, one checklist.** The cloud x86 agent and the local ARM agent found the same defect independently, and #1548 was filed as a duplicate of #1547. The duplicate was caught and closed, but coordination happened through the human. **Do differently:** before filing a fix task, search the epic's open children for the same symptom.
- **Check prior decisions before filing.** The documentation review filed mq-resiliency-observability#47 for a `pymqi` comment. #42 had already closed that as out of scope, because the repo is on hold. #47 was closed not-planned too.
- **Validation found long-standing bugs.** The single reused MsgId (#1549) and the silent consumer-thread death (#1550) both predate the migration. They surfaced only because the checklist asked for the dead-letter-queue count and the responder journal.

## §3 Compromises & tradeoffs

- **Validation scope:** `nativeha-ubuntu` only. `pcmk-ubuntu` and its authz playbook were skipped, so `mq-dlq-probe` was never run live. Also never run live: `mq-reconnect-probe` (it uses a plaintext channel, which the TLS-only stacks refuse), `mq-dr-baseline` and `mq-dr-forced` (nothing invokes them), and #1550's responder CorrelId echo and dead-letter path (the `DR.PROBE.*` queues don't exist; #1375). All of these are covered by the selfcheck import and unit tests only.
- **A workaround kept in place:** every connect forces `MQCNO_CLIENT_BINDING` to work around an upstream `ibmmq` 2.1.1 bug (#1480; ibm-messaging/mq-mqi-python#29, reported under #1483). #1545 checks at the next MQ upgrade whether it can be removed.
- **Tests that don't gate:** `vrg-validate` does not cover `components/` at all (#1477, open, not under this epic). The component tests for #1549 and #1550 ran only because they were run by hand inside the container. The component's own ruff findings (in the `ibmmq_refs` files) and its 85% coverage, with no minimum enforced, are pre-existing debt.
- **Still building from source on aarch64:** `ibmmq` ships no Linux aarch64 wheel, so the lab compiles the sdist (gcc + the MQ SDK) on every ARM bake. That is accepted indefinitely (see §4).

## §4 New problems & opportunities

| Surfaced | Where it went |
|---|---|
| No Linux aarch64 `ibmmq` wheel | .github#302, **moved to epic #351** (upstream reports, batched for the lab's public release). The lab keeps building its own wheel |
| `ibmmq` client-binding bug | Reported upstream (#1483 → ibm-messaging/mq-mqi-python#29); removal check is #1545 |
| `vrg-validate` doesn't test `components/` | #1477 (open) |
| `CMQSTRC` reason names / `to_string()` | mq-resiliency-lab-for-linux#1564 (ad-hoc epic #29) |
| Dict multi-selector `inquire`, `CNO` connect options | mq-resiliency-lab-for-linux#1565 (ad-hoc epic #29); must keep the #1480 workaround |
| Keystore passwords: `.sth` stash files, `KEYRPWD` under the default `INITKEY`, plaintext passwords on disk (RDQM's leftover `app-tls.mqsc`, mqweb's `mqwebuser.xml`) | Intake idea .github#355, to be expanded via `epic-create` with a spike first. Native HA / CRR keystore and channel security is coordinated with epic #346 |
| The DR harness isn't operational (no `DR.PROBE.*` queues) | #1375 (open); deferred to a later rethink of HA/DR application-behaviour validation |
| Docs still saying MQ appends `.kdb` to the key stem (`operating-the-lab-from-a-dev-session.md` §5); the keystores are `.p12`. The stale `nativeha-crr-tls-guide.md` | Recorded in .github#355. The #1361 doc review **missed** the `.kdb` line |

## §5 What's next

The follow-on brainstorm (#303) closed on 2026-10-15. The full decision record is on #303.

- **Adopted:** the two small `ibmmq` idiom cleanups (#1564, #1565).
- **Own future epic:** keystore-password modernization (#355).
- **Dropped:** OpenTelemetry propagation. On Linux it depends on an exit from IBM's commercial Instana product, which is out of the lab's open-source boundary. Async consume was dropped too: the current synchronous pattern is supported, and switching would be a re-architecture.
- **Deferred:** a working DR harness and reconnect probe, to the later HA/DR application-validation work.
- **The last private `pymqi` consumer:** won't-do for now. That repo's future gets decided after the resiliency lab ships, and only the parts that survive would migrate. Captured as an idea in its own repo, with this epic's lessons attached.

## Appendix A — Operational notes

The sequence that finally went green (ARM64, #1509 run 2, `develop` `f9787f8`):

1. `mqlab box build --all` rebuilds only the boxes whose components changed. On aarch64, `ibmmq` compiles from the sdist inside `component-install`, and both selfchecks (in place, then through `venv/bin`) gate the release flip.
2. Cold rebuild: `mqlab commons up`, then `mqlab bootstrap nativeha-ubuntu`.
3. Binding checks: the selfcheck on app-client and svc-sim, `mqlab component status --host …`, the wheel tag, and the `pymqi` absence check.
4. `mqlab qm e2e`, mq-bench, the authz playbook (`--tags induced,positive,authzpos,evidence,exporter`), then the DR entry points with before and after captures (DLQ depth, `amqsbcg` MQDLH, the responder's `NRestarts`, distinct MsgIds in the responder journal).
5. `mqlab teardown nativeha-ubuntu --commons`.

Gotchas:

- Run ad-hoc `ansible` with stdin from `/dev/null` and output to a file. Agent tool shells have non-blocking stdio, which ansible refuses.
- The clients need `LD_LIBRARY_PATH=/opt/mqm/lib64` when run by hand.
