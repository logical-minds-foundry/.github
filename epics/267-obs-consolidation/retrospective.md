# Retrospective — Consolidate the logsearch tier onto the obs node (epic #267)

Partners [`spec.md`](spec.md) and [`plan.md`](plan.md). Read spec → plan →
retrospective: what we set out to do, how we planned it, and honestly how it went.

## §0 — At a glance

VAL-A (epic #249) showed that the 2-vCPU `logsearch` node running three JVMs
(OpenSearch, Dashboards, Data Prepper) was the recurring cold-bootstrap bottleneck on
macOS/arm64. Dashboards also deadlocked its saved-objects migration on a nested-virt
request timeout. The epic set out to **fold the log tier onto the `obs` node**, so that
one adequately sized instrumentation machine runs the whole observability platform
(metrics and logs). It also aimed to cap the JVM heaps so the services coexist, and to
raise the Dashboards request timeout.

**What shipped:** the `logsearch` node, `logsearch_box` group, `logsearch-ubuntu2404` box
and `bake-logsearch.yml` are retired. A single merged obs box (7.12 GiB, well under the
20 G guest-disk ceiling) bakes the full platform, and the inter-service links are now
localhost. The obs node ended at **12 vCPU / 10240 MB**, not the planned 4 / 10240. Getting
from "consolidated" to "observe green on a cold arm64 bootstrap" took five further fixes,
each one found by the cold-rebuild gate. The consolidated node has been in daily use since
early October.

### Work delivered

| PR | What it did |
|---|---|
| [.github#270](https://github.com/logical-minds-foundry/.github/pull/270) | Spec + plan for the epic (#268) |
| [#1181](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1181) | T1: box-fit gate measured; bake the log tier into the obs box (additive) (#1178) |
| [#1183](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1183) | T2: consolidate onto obs (topology, observe plays, phase code, DNS/renders/`mqlab` retarget); retire the logsearch node/box/bake; add the ref guardrail (#1179) |
| [#1182](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1182) | T3: cap OpenSearch 2g / Data Prepper 512m / Dashboards Node 1024m; Dashboards `requestTimeout` 120 s + `migrations.scrollDuration` 30m (#1180) |
| [#1185](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1185) | Guardrail fix: the logsearch-ref scan read stale `.pyc` files; exclude compiled artifacts (#1184) |
| [#1187](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1187) | obs 4 → 8 vCPU: Grafana plugin backends were CPU-starved on 4 (#1186) |
| [#1189](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1189) | Gate the `mq-qmgr` mqweb include as well; svc-sim still ran a per-node mqweb (#1188) |
| [#1191](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1191) | Bake a haveged entropy daemon into every box: Java SecureRandom blocked JVM cold-start (#1190) |
| [#1193](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1193) | Entropy role refreshes the apt cache at bake so haveged installs (#1192) |
| [#1195](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1195) | Serialize observe startup (log tier before metrics); obs 8 → 12 vCPU (#1194) |
| [#1198](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1198) | Widen the log-tier readiness budgets from 15 to 40 min for host-oversubscribed cold bootstrap (#1197) |
| [#1388](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1388) | Docs review: logsearch runbook retargeted to obs; observe-order and wording fixes (#1176) |

- **Repos touched:** `mq-resiliency-lab-for-linux` (box, topology, Ansible, `mqlab`,
  tests, docs); `.github` (spec, plan, this retrospective).
- **Tasks:** 14 children: spec/plan (#268), T1–T3 (#1178–#1180), seven fix tasks
  (#1184, #1186, #1188, #1190, #1192, #1194, #1197), VAL (#1177), docs review (#1176),
  and the retrospective (#269).
- **PRs:** 12 merged, plus this retrospective. **Releases:** none cut by this epic.
- **Span:** opened **2026-09-24** → closing **2026-10-07**. The code was effectively done
  by 2026-09-30, and the first clean arm64 cold bootstrap on record is 2026-10-01.

## §1 — How the plan evolved

`plan.md` has no "Evolution during execution" log, so this section is reconstructed from
the task and PR record.

**The planned work landed as planned.** T1's box-fit gate came first, and the merged box
measured 7.12 GiB actual, about 11 GiB under the ceiling. That removed the riskiest
unknown in the spec, the "can't bump the disk past 20 G" risk. T1 was deliberately
additive, so `develop` stayed green, and T2 then did the coordinated retire in one PR so
that topology, box and bake stayed consistent. T3 ran in parallel with T2 as planned. One
plan detail was wrong: the "migration retry budget" assumed a `migrations.retryAttempts`
key, which OpenSearch Dashboards does not have (it fatally rejects unknown keys). The
implementer found this against the real product and used `migrations.scrollDuration`
instead.

**The plan's model of the bottleneck was incomplete.** The spec diagnosed the problem as
"two starved 2-vCPU nodes" and sized the merged node as the sum, 4 vCPU / 10 GB, to be
raised "only if a cold-rebuild shows pressure". It showed pressure three times, and
**memory was never the constraint** (about 9 GB stayed free throughout):

1. **4 vCPU wedged observe before the JVMs even started.** Prometheus, Grafana, Loki and
   the Ansible run saturated the node (load ~15), and Grafana's plugin backends were
   killed. Fixed by raising to 8 vCPU (#1186).
2. **Co-location created a new failure mode: startup contention.** At 8 vCPU the metrics
   services' simultaneous cold-start burst (load ~27) starved OpenSearch so it never went
   green. Before the epic, OpenSearch had its own node and never competed. The fix was to
   **reorder, not throttle**: the log-tier roles already had readiness gates, so starting
   them first serializes the heavy cold-starts at no extra cost. obs also went to 12 vCPU
   for headroom (#1194).
3. **The host, not the node, became the ceiling.** OpenSearch went green in ~15 s
   uncontended but took 20–33 min during a full bootstrap, because the 24-core host was
   oversubscribed across every guest. The 15-min budget was failing runs that would have
   succeeded, so it was widened to 40 min (#1197).

Two causes had nothing to do with sizing: **entropy starvation** on headless nested-virt
blocked Java `SecureRandom` (#1190, then #1192 when the first bake could not find the
package), and a **missed mqweb gate** on svc-sim left over from #1171 (#1188).

**The epic spun off its successor.** Finding #3 showed that the remaining cold-boot cost
was host oversubscription, which this epic could not fix by resizing one node. That became
**epic #275** (measurement-driven bootstrap staging), opened 2026-09-30. #275 added
per-bootstrap perf reports and staged VM bring-up. Those perf reports are the evidence
that closed this epic's VAL. Under staging, the later runs record OpenSearch green in
~11 s.

**VAL closure lagged.** #1177 got an interim FAILURE comment on 09-28 and then no final
outcome, even though the lab was bootstrapping clean from 10-01. It was closed on 10-07 as
SUCCESS on the strength of the stored run reports (see §3).

## §2 — Lessons learned

- **The cold-rebuild gate earned its keep again.** Every change after T3 was a defect that
  `vrg-validate` could not see: CPU starvation, startup contention, entropy, a premature
  budget, a stray mqweb. Lint-green ≠ done held without exception.
- **Co-location changes the failure modes, not just the resource totals.** Summing two
  nodes' vCPU/RAM assumes the workloads don't interact. They do: cold-start bursts collide.
  Next time, when merging services onto one machine, plan the **startup order** as part of
  the design, not only the capacity.
- **Prefer ordering on existing readiness gates over throttling.** The #1194 fix added no
  limits or sleeps; it put the heavy services first and let the existing health gates
  serialize them.
- **Measure before you tune.** The vCPU bumps were reasonable guesses, but the real cause
  (host oversubscription) only became clear once someone measured OpenSearch uncontended
  vs contended. That measurement-first stance is what #275 then made systematic.
- **Verify plan-specified config keys against the product.** A plan can name a knob that
  doesn't exist (`migrations.retryAttempts`). The implementer catching it before merge is
  the right outcome. A wrong key in a strict-config product is a startup failure, not a
  no-op.
- **Additive-then-retire keeps `develop` green across a structural change.** T1 added the
  log tier to the obs box without removing anything; T2 retired the old node, box and
  bake in one coordinated PR.
- **Tree-scanning guardrail tests must ignore build artifacts.** The logsearch-ref
  guardrail read a stale `__pycache__/*.pyc` and failed on a clean source tree (#1184).
- **Record the VAL outcome when it turns green, not when the epic closes.** The week
  between the first clean run and the #1177 record kept the epic open while it was
  already in daily use.

## §3 — Compromises & tradeoffs

- **VAL #1177 was closed on routine run reports, not a dedicated re-run.** The evidence is
  `build/state/runs/perf-20261001T163125Z.json` (arm64 macOS, `nativeha-ubuntu --no-dr`,
  0 failed steps, OpenSearch/Data Prepper/Dashboards ready, no logsearch node), backed by
  six later clean runs. Those reports show fresh VM boots and a green observe, but they do
  not prove the obs box was re-baked immediately before. The operator accepted this
  explicitly in lieu of another cold rebuild.
- **12 vCPU is an over-allocation.** obs now claims half the host's cores and relies on
  hypervisor time-slicing; it is not a measured right-size. Memory (10240 MB) was never
  revisited, because it was never the constraint.
- **The 40-min readiness budgets are a generous ceiling.** They still fail loud, but under
  #275's staging OpenSearch greens in seconds, so a regression back to "20 minutes but
  passing" would not trip anything. The perf reports are now the place to notice that.
- **The heap caps are engineering estimates.** OpenSearch 2g, Data Prepper 512m and
  Dashboards 1024m come from `docs/reports/2026-09-28-obs-coexistence-jvm-budget.md`, and
  their only proof is that the node runs. They have not been load-tested.
- **The svc-sim mqweb was disabled, not removed (#1188),** at the owner's request, so it can
  come back with the mqweb client-mode work.
- **The `mqlab logsearch` CLI namespace kept its name** even though the node is gone. It now
  targets obs, and the docs say so. It is a naming wart, not a defect.

## §4 — New problems & opportunities

- **Host oversubscription is the cold-boot limit** → spun off and completed as epic
  [#275](https://github.com/logical-minds-foundry/.github/issues/275) (bootstrap staging +
  perf instrumentation; closed).
- **Dashboards' post-green API calls hit the uri module's default 30 s timeout on a cold
  start** → found in #275's baseline; fixed by #1214 / PR #1218 (timeout + bounded idempotent
  retries).
- **VAL-A is unblocked** → [#1154](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/1154)
  (arm64 5/5 reproducibility gate, epic #249) can resume; it is still open.
- **Log-tier hardening that followed the consolidation** (Data Prepper JSON gating, PR #1222;
  OpenSearch date-detection off, PR #1236; Data Prepper sink metrics/alerts, PR #1242;
  sink DLQ, PR #1243) was tracked outside this epic.
- **Instrumentation-plane rework, the mqweb half** → the sibling follow-on in
  [.github#266](https://github.com/logical-minds-foundry/.github/issues/266) (per-node mqweb →
  a single client-mode instance) is still open. It is also where the disabled svc-sim mqweb
  comes back.

## §5 — What's next

- [.github#266](https://github.com/logical-minds-foundry/.github/issues/266): the mqweb
  client-mode brainstorm, the other half of the instrumentation-plane rearchitecture.
- [#1154](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/1154):
  resume VAL-A (epic #249) now that the observability tier is no longer the bottleneck.
- No new follow-on brainstorm was raised by this epic itself; its forward work went to
  epic #275, which has already closed.
