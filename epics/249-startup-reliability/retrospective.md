# Retrospective — Epic 249: macOS/arm64 startup reliability & bring-up hardening

## 0. At a glance

**Set out to do:** make a cold lab bring-up on macOS/arm64 (nested virtualization)
reliable and reproducible: serialize JVM cold-starts (A), make readiness budgets
bounded and fail-loud with clean phase resume (B), and harden the VM boot and DNS layer
(G). Proof would be 5 consecutive clean cold `nativeha-ubuntu --no-dr` bootstraps on
arm64 (VAL-A) and no regression on the x86 cloud (VAL-B).

**What shipped:** all of A, B and G landed on day one (2026-09-20). They did not make
bring-up reliable on their own. VAL-A kept failing, and each failure was traced and
fixed, until the lab's actual bottleneck was found. Finding it took two child epics
(#267, obs consolidation; #275, measurement-driven bootstrap staging). The real cause
was arm64 nested-virt second-level page-fault cost, not the "JVM cold-start tax" this
epic's spec named. Huge-page guest RAM fixed it (#1241, under #275). A cold
`nativeha-ubuntu --no-dr` bootstrap now takes about 9–13 minutes on both platforms.
Before, it failed after about 23 minutes or never finished. VAL-A and VAL-B closed on
the #275 evidence (#1200 on arm64, #1267 on x86).

| PR | Task | Merged | What it did |
|---|---|---|---|
| .github#264 | #250 | 2026-09-20 | Spec + plan for the epic |
| #1165 | #1161 (A) | 2026-09-20 | Serialized the remaining concurrent JVM cold-starts using the #1151 mqweb pattern |
| #1166 | #1162 (B) | 2026-09-20 | Bounded, fail-loud readiness budgets with a fatality rule and a guardrail test |
| #1167 | #1163 (B) | 2026-09-20 | `--from <phase>` resume carries `--no-dr` so a resumed run re-enters cleanly |
| #1168 | #1164 (G) | 2026-09-20 | Infra/DNS-first boot order and a bounded 3-attempt boot retry for DHCP-lease timeouts |
| #1170 | #1169 | 2026-09-22 | mqweb's TLS restart only runs on an already-running mqweb, ending a start/restart race |
| #1172 | #1171 | 2026-09-22 | Per-node mqweb gated off by default (`mqweb_enabled: false`) in all 8 stack/DR plays |
| #1175 | #1173 | 2026-09-22 | Provision masks apt timers and waits out the dpkg lock early |
| #1389 | #1153 | 2026-10-07 | Docs sweep: mqweb off by default, bring-up failure guidance, later-finding notes |

- **Repos touched:** `mq-resiliency-lab-for-linux`, `.github`.
- **Tasks:** 15 children. 9 delivered by PR, 2 validations closed on evidence, 3 closed
  as superseded or overtaken (#1174, #1196, #252), and this retrospective.
- **Releases cut:** none.
- **Span:** opened 2026-09-20, closed 2026-10-07 (17 days).

## 1. How the plan evolved

`plan.md` has no "Evolution during execution" log. This narrative is reconstructed from
the task and validation record.

The plan assumed the problem was understood. The spec said the macOS slowdown was a
roughly 7× nested-virt tax on JVM cold-starts, that adding CPU or RAM would not help,
and that the fix was to stop starting JVMs concurrently and to size budgets for the
tax. All four implementation tasks were written against that model and merged the same
day.

VAL-A then became the epic's real driver, and each run exposed a new failure:

1. **mqweb start/restart race** (run 1): the role restarted mqweb for TLS while Liberty
   was still cold-starting, and systemd reaped the JVM. Fixed by #1169.
2. **Per-node mqweb never started** on the 2-vCPU QM nodes, even with the race fixed.
   Rather than keep fighting it, the maintainer retired per-node mqweb (#1171; #1188
   later closed the svc-sim path under #267). #266 was raised from a brainstorm to "the
   chosen direction" (single client-mode mqweb), then moved to the ad-hoc backlog on
   2026-10-07 as not a priority.
3. **apt auto-updates held the dpkg lock** mid-provision. Fixed by #1173, with a
   permanent bake-time fix later (#1225, which superseded #1174).
4. **The 2-vCPU logsearch node** could not start three JVMs, and OpenSearch Dashboards
   deadlocked its migration. That spun off epic **#267** (fold logsearch into obs), and
   VAL-A paused on #267's validation.
5. **#267's validation** surfaced several more layers: OpenSearch's launcher blocking
   on `/dev/random` (#1190, #1192), the metrics stack's startup burst starving
   OpenSearch (#1194), a readiness budget shorter than a throttled cold-start (#1197),
   and app-client's network failing to come up during the concurrent boot. These all
   looked like host oversubscription, but the exact mechanism could not be pinned down
   without measurement.
6. That spun off epic **#275**: instrument the bootstrap first, then tune from the
   data. Its perf reports named the real arm64 cause, second-level page-fault cost
   (#1240), and huge-page guest RAM (#1241) took macOS from 40+ minute non-finishing
   runs to about 10 minutes. The same instrumentation found the cloud's bottleneck
   (standard persistent-disk I/O, fixed with a pd-ssd boot disk).

So the plan's structure (A/B/G, then VAL-A and VAL-B) held, but its root-cause premise
did not. The epic grew two nested child epics, and the validation gate was satisfied by
the third layer of work, not by the work this epic planned.

## 2. Lessons learned

- **Measure before naming the root cause.** The spec's "JVM nested tax" was inferred
  from symptoms (slow JVMs, idle host, low iowait) without profiling, and deep
  profiling (F) was explicitly deferred. #275's instrumentation found the cause within
  days of existing. Next time an epic is built on a performance hypothesis, put the
  instrumentation in scope and run it first.
- **"Adding CPU doesn't help" was true but misleading.** It ruled out the obvious
  lever, yet was read as confirming the JVM theory. Ruling out a cause is not evidence
  for the remaining guess.
- **The validation gate did the real diagnostic work.** Each failed VAL-A run named a
  concrete defect. The cold-rebuild acceptance gate was worth its cost.
- **Lint-green and merged are not done.** All four planned tasks merged on day one, and
  reliability arrived 12 days later.
- **Recursive epics are hard to follow.** #249 → #267 → #275 each paused the one above.
  The relationships were tracked in issue comments but are easy to lose. A short
  "blocked on child epic" note on the parent epic issue would have helped.

## 3. Compromises & tradeoffs

- **VAL-A was accepted on 3 runs, not 5.** The maintainer accepted macOS on three
  huge-page cold runs (#1200), one of which failed at an unrelated step that #1261 later
  fixed.
- **VAL-A and VAL-B were closed on older evidence.** They were closed on 2026-10-07
  using evidence from about 2026-10-02 (#1200, #1267), more than 50 commits earlier.
  Later routine macOS runs were about 30% slower (526–585 s on 10-01/02, 751–758 s on
  10-04/06, mostly in `vms up [3/3]` and provision). They are still under the 900 s
  target, so this was accepted as normal run-to-run variance rather than investigated.
- **Per-node mqweb was retired, not fixed.** The REST/Console plane is off by default.
  The decision predates the page-fault fix, so it may no longer be necessary.
- **Readiness budgets were widened to 40 minutes** (#1197) to survive the throttled
  cold-starts. With the page-fault fix, the log tier now greens in seconds, so a real
  failure now takes up to 40 minutes to surface.
- **obs is still at 12 vCPU.** It was raised 4 → 8 → 12 (#1186, #1194) to fight
  contention. #275's own discipline said to reduce it only on evidence, and that
  measurement was never made.

## 4. New problems & opportunities

| Surfaced | Where it went |
|---|---|
| mqweb's future (per-node or client-mode) | `.github#266`, ad-hoc backlog (epic #29). Re-measure per-node mqweb on the huge-page platform first. |
| logsearch node too small for three JVMs | Epic `.github#267` (obs consolidation). Its validation, docs review and retrospective (PR .github#307) are done. |
| Bring-up bottleneck unmeasured | Epic `.github#275` (closed): perf reports, `MQLAB_ENV` profiles, `mqlab perf diff`. |
| Do the #275 levers apply to the other stacks? | `#1264` follow-on brainstorm (closed). |
| Perf reports don't record exit status or commit, so a run's cleanliness can't be confirmed from the report alone | Logged, not yet acted on. |
| macOS wall-clock rose ~30% between 10-02 and 10-04/06 | Logged, not yet acted on. Still under target. |
| obs 12 vCPU never re-measured | Logged, not yet acted on. |
| Non-fatal readiness gates can mask a dead service as a green bootstrap (seen with mqweb in VAL-A run 1) | Logged, not yet acted on. Moot while mqweb is off. |

## 5. What's next

The seeded follow-on brainstorm (#252) was closed on 2026-10-07 as overtaken:
consolidation (E) was delivered by #267; per-start cost and profiling (C, F) were
overtaken by #275, which found and fixed the real cause; selective right-sizing (D) is
handled data-first through #275's `MQLAB_ENV` profiles. No new epic is seeded from
this one. The open thread is the mqweb decision in #266.
