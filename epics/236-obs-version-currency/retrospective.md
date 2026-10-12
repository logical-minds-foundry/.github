# Epic 236 — Observability stack version-currency refresh: retrospective

**Epic:** logical-minds-foundry/.github#236 · **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md)

## §0 At a glance

**Set out to do:** after the MQ 10.0 rebaseline and the exporter bump to `v6.0.0`,
bring the observability stack up to current: land the low-risk upgrades
(node_exporter, Alloy, Loki/logcli, a Grafana pin), gate the breaking
Prometheus 2→3 move behind a spike and a go/no-go, reconcile the drift between
manifest and role defaults, and record the OpenSearch trio as current.

**What shipped:** all of it. On develop today: node_exporter **1.12.1**, Alloy
**1.19.2**, Loki + logcli **3.7.7**, Grafana pinned to **13.2.2** with an apt
hold, Prometheus **3.13.3 LTS** (go decision on #236), and `mq_metric_samples_ref`
reconciled from `master` to **v6.0.0**. A guardrail test keeps the two sources of
truth in lockstep. The arm64 cold rebuild then exposed two bake/boot defects
(fixed here) and a larger platform problem: unreliable arm64 bring-up, which was
spun out into its own epic (#249).

| PR | Merged | What it did |
|---|---|---|
| .github#241 | 2026-09-16 | Spec + plan for this epic |
| #1137 | 2026-09-16 | T1: `test_obs_version_sync` guardrail; reconciled `mq_metric_samples_ref` master → v6.0.0 |
| #1138 | 2026-09-16 | T6: Prometheus 2→3 migration spike report + go/no-go recommendation |
| #1139 | 2026-09-17 | T2: node_exporter 1.8.2 → 1.12.1 (both sources of truth) |
| #1140 | 2026-09-17 | T3: Alloy 1.3.1 → 1.19.2 |
| #1141 | 2026-09-17 | T4: Loki + logcli 3.1.1 → 3.7.7 |
| #1142 | 2026-09-17 | T5: Grafana pinned to 13.2.2 + apt hold; guardrail reads the bake-authoritative pin |
| #1143 | 2026-09-17 | T7: Prometheus 2.53.2 → 3.13.3 LTS (version-only; config unchanged) |
| #1145 | 2026-09-17 | #1144: grow the fat-box build disk so the obs bake stops overflowing 8.7 G |
| #1147 | 2026-09-18 | #1146: correct #1145; absolute 18 G under the box's 20 G metadata (fixes guest boot) |

- **Repos touched:** 2 (`logical-minds-foundry/.github`, `mq-resiliency-lab-for-linux`).
- **PRs:** 10 (1 docs, 7 planned code, 2 discovered fixes).
- **Children:** 15 — 1 docs, 7 code, 3 validation, 2 discovered fix tasks, 1 docs review, 1 retrospective.
- **Releases cut:** none recorded (the repo has no GitHub releases).
- **Span:** opened 2026-09-16; all code merged by 2026-09-18; validations and docs review closed 2026-10-12.

## §1 How the plan evolved

*The plan had no "Evolution during execution" log. This narrative is reconstructed from the issue and PR record and the driving session.*

The code went almost exactly as planned. T1 landed first and made the
role-default ↔ manifest lockstep executable: it went RED on the live `master`
exporter drift, then GREEN after the reconcile. The T6 spike showed the lab's
Prometheus config is untouched by every item in the 2→3 breaking set. That made
T7 a one-line version bump, and the maintainer chose 3.13.3 (LTS) over 3.14.0.
Because T6 merged at the same time as T1, T7 ran **in parallel with** the
low-risk wave instead of after its validation.

That reordering, plus all seven code PRs merging within a day, quietly dissolved
the plan's main validation idea: a **separate** cold rebuild for Prometheus 3.x,
so a failure would point clearly at the breaking change. Once everything was on
develop together, no rebuild could isolate it. VAL-A and VAL-C were folded into
one arm64 rebuild.

Merging was serialized by the shared manifest. All five bump PRs edit adjacent
lines of `manifests/_shared/observability.yaml`, so the PRs hit conflicts one
after another and each was resolved by hand after submit.

The cold rebuild is where the plan met reality:

1. **obs box bake overflowed** its 8.7 G build disk (`No space left on device`
   installing MQ web) → #1144, fixed in #1145 with `+12G`.
2. **#1145 introduced a regression.** The resized box's root partition grew past
   the box metadata's `virtual_size: 20`. Vagrant truncated the guest disk at
   20 G, so obs dropped to initramfs (`LABEL=cloudimg-rootfs does not exist`).
   The proof had only shown that obs *bakes*, never that it *boots*. Fixed in
   #1146/#1147 with an absolute 18 G, this time proven by booting a guest.
3. **arm64 bring-up was unreliable** in ways unrelated to the version bumps:
   mqweb was SIGKILLed at its 30-min start timeout, OpenSearch missed its 15-min
   budget, VM boots hit IP-lease timeouts, and the DNS node was unreachable.
   Measurement showed none of this was memory, host CPU, entropy or emulation.
   It was startup contention compounding a nested-virt cost on JVM cold start.
   This was spun out rather than absorbed: #1148 (serialize mqweb startup),
   which cut provision from 82 to 17 min, and epic **#249** (startup
   reliability), which has since closed.

VAL-B (x86 RHEL, cloud) passed on 2026-09-18. VAL-A and VAL-C stayed blocked
until #249 and the lab re-architecture that followed (logsearch consolidated
onto obs, the multi-version OS axis). On 2026-10-12 the maintainer closed both
**on accumulated evidence**: many dozens of arm64 cold rebuilds since, all
running these exact pins with observability healthy. There was no dedicated
attestation run; #1154 sets the precedent. The docs review found nothing to
change, because the human docs are deliberately free of versions.

## §2 Lessons learned

- **Two sources of truth need an executable guard.** T1's guardrail paid off
  right away. It caught the real `master` drift, and in T5 it exposed that
  Grafana's bake-authoritative pin lives in `group_vars`, not the role default.
  Do this first on any dual-source pin.
- **Prove by boot, not just by bake.** #1145 passed every check it was given and
  still broke guest boot. For box and disk changes, acceptance must include
  booting a guest from the new box.
- **Measure before adding resources.** The intuitive fix (more vCPU, #1152) was
  wrong: 6 vCPU still left OpenSearch at ~225 s. The tell was that the cloud runs
  cleanly on *fewer* resources. Treat "more resources" as a hypothesis to test.
- **Validation isolation has to be enforced at merge time.** If a breaking change
  needs its own rebuild, hold its merge until the low-risk wave is validated.
  Planning a separate rebuild isn't enough.
- **Same-file bumps conflict at merge, however independent they are logically.**
  Five adjacent-line edits to one manifest meant four rounds of resolution.
  Either sequence them deliberately or put the manifest changes in one PR.
- **Don't trust exit codes; check phase state.** `mqlab box build` exited 0
  while obs had failed, and `mqlab bootstrap` exited 0 with observe ✗. Every
  verdict here came from reading box status and phase status directly.
- **Spin out instead of growing scope.** The bring-up problem was bigger than
  this epic. Moving it to #249 let this epic stay a version-currency effort and
  close cleanly.

## §3 Compromises & tradeoffs

- **VAL-A and VAL-C were closed on accumulated evidence, not attested runs.**
  VAL-A's specific checks (a known log line reaching OpenSearch through Data
  Prepper, a node panel rendering) are covered by weeks of continuous use and
  #1242's sink metrics, not by an explicit probe. This was accepted because
  dozens of real cold rebuilds are stronger evidence than one scripted run.
- **Prometheus 3.x was never validated in isolation**, for the reasons in §1.
  Accepted once all changes had merged together.
- **The Grafana guardrail is asymmetric.** It compares the `group_vars` pin
  (what the bake actually uses) against the manifest, not a role default.
  That is correct, but it is a special case a future reader may trip over.
- **Merge conflicts were resolved after submit, by hand,** instead of being
  designed out (see §2).
- **The rebuild sessions relied on non-persistent workarounds** for host-side
  snags: a `GIT_CONFIG_*` `safe.directory` env, and repairing a worktree's
  `build/state` that was a real directory instead of a symlink. They weren't
  fixed at the source here.
- **The OpenSearch trio was held at 3.8.0.** That was correct on 2026-09-16, and
  3.9.0 has since shipped (§5).

## §4 New problems & opportunities

| Surfaced | Where it went |
|---|---|
| obs fat-box bake overflowed 8.7 G | #1144 → #1145 (fixed) |
| #1145 resize broke obs guest boot (box exceeded `virtual_size: 20`) | #1146 → #1147 (fixed) |
| arm64 bring-up unreliable: mqweb timeout, OpenSearch budget, boot IP-lease, DNS | epic #249 (closed); #1148 mqweb serialization; mqweb later gated off by default (#1171) |
| Right-sizing logsearch/obs vCPUs (#1152) | Parked, then closed **won't do** 2026-10-12. CPU wasn't the bottleneck, and allocations have since been reworked |
| `mqlab box build` / `bootstrap` returning 0 on component failure | **Logged, not verified.** Whether #249 or later work fixed it isn't checked here |
| git "dubious ownership" on the repo during bootstrap; worktree `build/state` created as a real directory | **Logged, not acted on** (worked around in session) |
| Version currency decays fast. Four weeks later, 6 of 9 components have newer releases (§5) | Feeds the maintainer's cross-Vergil dependency-management work |

## §5 What's next

**Version check, 2026-10-12** (data: GitHub Releases API):

| Component | Pinned | Latest upstream |
|---|---|---|
| Prometheus (LTS) | 3.13.3 | 3.13.4 LTS patch (2026-10-02); 3.15.0 non-LTS |
| node_exporter | 1.12.1 | 1.12.1 (current) |
| Alloy | 1.19.2 | 1.20.1 |
| Loki + logcli | 3.7.7 | 3.7.8 |
| Grafana | 13.2.2 | 13.2.3 |
| OpenSearch | 3.8.0 | 3.9.0 (2026-09-29) |
| OpenSearch Dashboards | 3.8.0 | 3.9.0 (2026-10-12) |
| Data Prepper | 2.16.0 | 2.16.0 (current) |
| mq-metric-samples | v6.0.0 | v6.0.0 (current) |

These deltas are **not** acted on in this epic. They feed the maintainer's
upcoming dependency-management work across Vergil and its consumers, where
keeping pins current becomes a standing, tool-supported process instead of a
periodic epic.

The bring-up reliability thread continues under epic #249, which is closed, and
its follow-on brainstorm #252.

## Appendix A — Operational notes

- **Drive mqlab with `uv run mqlab`**, never `.venv/bin/mqlab`. The bake shells
  out to `ansible-playbook` by bare name and needs the venv on `PATH`.
- **Driving the lab from a worktree:** make sure `build/state` and `build/cache`
  are symlinks to the main checkout (`rm -rf build/state && uv run mqlab build
  ensure`), or `_build_ensure` refuses to run.
- **The box content hash doesn't cover disk geometry.** After changing the
  build-disk size, force-rebake the affected boxes (`mqlab box rebuild <box>`).
  `box build` will otherwise reuse the stale image.
- **Submitting adjacent-line manifest bumps:** expect one conflict per PR after
  the first. Resolve with the union of the develop values plus the branch's own
  line, run `vrg-validate`, then `vrg-commit` and push to the submitted branch.
