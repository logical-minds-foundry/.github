# Epic 326 — Observability patch refresh: retrospective

**Epic:** logical-minds-foundry/.github#326 · **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md) · **Follows:** epic #236

## §0 At a glance

**Set out to do:** #236's closing version check (2026-10-12) found newer upstream
releases for six of the nine pinned observability components within four weeks.
The goal was one patching pass to bring them current, validated by cold builds on
arm64 and x86.

**What shipped:** all six bumps, in one PR, with nothing held back: Prometheus
**3.13.4** (still on the 3.13 LTS line), Alloy **1.20.1**, Loki + logcli **3.7.8**,
Grafana **13.2.3** (pin and apt hold kept), and OpenSearch + OpenSearch Dashboards
**3.9.0**. node_exporter, Data Prepper and mq-metric-samples were already latest.
Both platforms cold-built green, and one stale docs link was fixed.

| PR | Merged | What it did |
|---|---|---|
| .github#329 | 2026-10-12 | Spec + plan for this epic |
| #1460 | 2026-10-12 | All six version bumps in one PR (8 files, +18/−18); role defaults and the shared manifest moved together |
| #1467 | 2026-10-12 | Docs review (#1449): OpenSearch date-detection link 3.8 → 3.9 docs |

- **Repos touched:** 2 (`logical-minds-foundry/.github`, `mq-resiliency-lab-for-linux`).
- **PRs:** 3 (spec/plan, code, docs).
- **Children:** 6 — 1 docs, 1 code, 2 validation, 1 docs review, 1 retrospective.
- **Releases cut:** none recorded.
- **Span:** opened 2026-10-12 12:19 UTC; code merged 16:31; both validations closed by 19:34; docs review merged 19:44 the same day.

## §1 How the plan evolved

The plan held. The only real change was in **sequencing**:

- **The implementation started early, by agent overreach.** The maintainer merging
  the docs PR was treated as a go-ahead to implement. The maintainer corrected
  that: starting implementation is their call. The branch was then held at
  report-ready so it could be rebased after a batch of unrelated ad-hoc fixes
  landed. It ended up merging (#1460) before those fixes, so no rebase was needed.
- **The cold builds were batched on purpose.** They were held until four ad-hoc
  fixes had merged after #1460: #1461 Grafana relay port discovery, #1462
  SIGPIPE-safe box pipelines, #1463 following the Native HA Active per attempt,
  and #1464 pcsd readiness. One round of cold builds then validated both this
  epic and those fixes.
- **Validation went wider than the issues asked.** #1447 specified a nativeha
  cold build; the arm64 run added pcmk-ubuntu to exercise #1464. The x86 run did
  nativeha-ubuntu, pcmk-ubuntu, and a full RHEL 10 `nativeha-rhel-crr` (the only
  platform that covers RHEL).

## §2 Lessons learned

- **One PR for same-file bumps works.** #236's five separate PRs needed four rounds
  of hand-resolved merge conflicts in `observability.yaml`. Here, one PR with six
  bumps merged with no conflicts.
- **Small epics move fast when the investigation is already done.** #236's
  research and guardrail meant this epic was the version check plus verification,
  and it went from epic to both validations closed in about seven hours.
- **Prove compatibility claims by running them.** Data Prepper 2.16.0 working with
  OpenSearch 3.9 rested on documentation ("supports OpenSearch 1.0 and greater")
  until a probe log line traced end to end on both platforms (Alloy → Data
  Prepper → OpenSearch, in about 45 s on arm64).
- **Batching validations with unrelated fixes is efficient.** One rebake plus cold
  builds covered this epic and four fixes. The cost: a failure would have needed
  more digging to pin on the right change. That's acceptable when the changes are
  this independent.
- **An agent must not start implementation on its own.** Creating an epic or
  merging its docs PR is not a go-ahead.

## §3 Compromises & tradeoffs

- **Spec pushback and plan alignment were skipped** because the epic was small and
  the design was agreed in chat. The agent self-checked the spec against the plan
  instead.
- **The implementation and validation tasks were filed before the docs PR
  merged.** That way the validations could be linked as blocked by #1446 from the
  start. It's a small departure from the standard sequence.
- **Prometheus stays on the LTS line (3.13.x)**, not the newer 3.15. That's a
  deliberate currency-versus-stability choice carried over from #236.
- **The x86 validation was a person in a cloud session** (by design). The arm64
  run was agent-driven.

## §4 New problems & opportunities

| Surfaced | Where it went |
|---|---|
| `mqlab status` shows the vms phase ✗ after any `--no-dr` bootstrap: `_vms_satisfied` ignores `no_dr` | **#1465** (ad-hoc epic). x86 confirmed it: the full RHEL bootstrap shows vms ✓ |
| Prometheus scrapes every node and exporter in the topology, not just the running stack, so a single-stack lab always shows ~26 down targets | **Logged, not acted on.** The down targets are expected, but they make it harder to spot a real failure |
| Pinned versions go stale within weeks; this epic was one manual pass | Feeds the maintainer's cross-Vergil dependency-management work |

## §5 What's next

- **Dependency management.** Rather than another manual refresh epic, the
  maintainer's upcoming cross-Vergil dependency-management work is the place to
  make version currency a standing, tool-supported process.
- **#1465** (`--no-dr` status bug) is filed under the ad-hoc epic.
- No follow-on epic is planned from this one.
