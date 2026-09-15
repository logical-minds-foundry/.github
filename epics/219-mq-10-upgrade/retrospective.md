# Retrospective — MQ 9.4.5 → 10.0 Native HA upgrade + version centralization

_Epic: [logical-minds-foundry/.github#219](https://github.com/logical-minds-foundry/.github/issues/219). Partners [`spec.md`](spec.md) and [`plan.md`](plan.md); read spec → plan → retrospective._

## §0 — At a glance

**Set out to:** produce a documented, tested 9.4.5→10.0 Native HA rolling-upgrade procedure, and collapse the scattered MQ version pins into one source of truth so the upgrade is a one-value change.

**Shipped:** a single authoritative `lab/mq-version` pin that every consumer reads — Python, shell, Ansible, _and_ the box-bake hash; the RHEL Native HA CRR arm cold-rebuilt at 9.4.5, rolling-upgraded to 10.0 and DR-drilled with evidence captured; the lab default rebaselined to 10.0; and — beyond the original plan — a generic, exportable, customer-facing upgrade guide.

| PR | Task | What it did |
|---|---|---|
| #1081 | #1070 | Spike: pinned MQ 10.0 CRR upgrade facts + confirmed/staged media |
| #1082 | #1071 | Single-source `lab/mq-version` pin + Python/shell consumers |
| #1083 | #1072 | Ansible reads the pin; version-aware Native HA floor; dropped stale SUT manifests |
| #1084 | #1073 | First Native HA CRR rolling-upgrade runbook (lab) |
| #1085 | #1075 | Rebaselined the lab default to IBM MQ 10.0 |
| #1088 | #1087 | Box-bake hash keys on the pin (auto-invalidate MQ-bearing boxes) |
| #1090 | #1069 | Docs-review sweep (runbook §5/§3.1 corrections + execution evidence report) |
| #1093 | #1091 | Runbook at-a-glance outline + Sequence-A recommendation |
| #1094 | #1092 | Generic exportable customer upgrade guide |

**Live validations (no PR — outcome recorded on the issue):** #1074 executed the 9.4.5→10.0 rolling upgrade + DR failover/failback (SUCCESS); #1076 cold-rebuilt the arm at 10.0 (SUCCESS on its MQ criteria).

- **Repos touched:** `mq-resiliency-lab-for-linux` (all code/docs); `logical-minds-foundry/.github` (spec / plan / this retrospective).
- **Counts:** 14 tasks (incl. this retrospective) · 9 PRs · 2 live validations · 0 releases.
- **Span:** opened 2026-09-14, closed 2026-09-15 (~2 days).

## §1 — How the plan evolved

Tasks 0–6 plus the bookends ran largely as designed — spike → pin → consumers → runbook → execute → rebaseline → cold-rebuild — and the version-centralization landed one-pass. Three deltas mattered:

1. **Centralization was incomplete at the box layer.** The pin drove the Python, shell, and Ansible consumers, but the baked-box staleness hash ignored it, so bumping the pin silently reused a box baked at the old MQ version. Surfaced at the cold-rebuild gate (#1076) and fixed by an unplanned task (#1087), folding the pin into the box manifest-hash for the MQ-bearing boxes so a version bump now invalidates them.
2. **The runbook deliverable ballooned.** The plan had one runbook task (#1073). The first runbook was accurate but abstract and tied to lab tooling; making it genuinely usable took three more passes — corrections (#1069), an at-a-glance outline plus a firm Sequence-A recommendation (#1091), and finally a from-scratch generic, novice-usable, de-proprietary exportable guide (#1092) — the actual customer deliverable.
3. **Operational reality bit twice.** The 10.0 RPM `%prein` scriptlet refuses to upgrade while the MQ web server is running (worked around live by stopping `mqweb`/`endmqweb` as part of the quiesce), and the observability (`observe`) phase reproducibly OOM-killed a one-pass cold rebuild on the 31 GiB host.

## §2 — Lessons learned

- A "single source of truth" is only true once **every** consumer reads it — including the ones you forget (the box-bake hash). A pin that half-propagates is a silent-downgrade footgun; enumerate consumers explicitly.
- **Correct is not the same as usable.** For an external hand-off, genericity (no in-house tooling names), copy-pasteable commands with expected output, a single recommended path, and an at-a-glance outline are first-class requirements — not polish applied at the end.
- **Ground the risky claims.** The back-out section was rewritten around cited IBM docs once research showed the lab's single-node `tar` is _not_ a sanctioned Native HA backup — the honest answer changed the recommendation (recover forward / rebuild, rather than trust a dubious file rollback).
- **Stage memory-heavy work, but measure it.** Pre-baking boxes separated the box-build memory spike from the `observe` spike; necessary but not sufficient — `observe` still OOM'd, isolating the real bottleneck (the Grafana renderer + full guest set on a 31 GiB host).

## §3 — Compromises & tradeoffs

- **#1076 scored SUCCESS on its MQ criteria while the full one-pass bootstrap did _not_ complete** (the `observe` phase OOM'd). Accepted deliberately: the epic's goal — a validated 10.0 rebaseline and a clean MQ cold rebuild to 10.0 — was proven; the observe/host-memory limit is real but separable, tracked as #1089 rather than allowed to gate the epic.
- Two memory-heavy commons (`obs`, `logsearch`) were parked during the upgrade window — a manual intervention, acceptable for the validation, noted here.
- The generic guide's §5 "restores as a standalone queue manager" point is labelled **inference** from an RDQM/appliance-scoped IBM page (not in the doc cache, not fetchable by the repo tool); its expected-output blocks are representative renderings, not verbatim capture. Both are flagged in-document.
- Commits carry no `Co-Authored-By` attribution — the enforced `vrg-commit` tooling has no path to add it (a tooling gap, not a choice).

## §4 — New problems & opportunities

- **Observe-phase OOM on the 31 GiB host** → **#1089** (ad-hoc): resize the host or trim the Grafana image-renderer so a full one-pass cold rebuild (including `observe`) fits.
- **`mq-nativeha-spike` role floor check rejects 10.0** → **#1086** (ad-hoc): #1072's version-comparison fix was deliberately scoped to the _production_ `mq-nativeha` role; the spike role still uses the old string-membership check.
- **A `vrg-pr-workflow` gap recurred**: `report-ready` recorded ready-state but did not push the branch head, so the first human submit failed until branches were pushed manually; every later task pushed the branch _before_ `report-ready` as a workaround. Worth fixing at the tooling level.

## §5 — What's next

- **Epic: convert the `mq_prometheus` metrics agent from client-mode to an MQ Service** — parked at maintainer request, to be opened via `epic-create`. It is the better architecture (the monitor's lifecycle bound to the queue manager, so it stops/starts with it) and it retires the generic guide's need to special-case the monitor during an upgrade.
- Address **#1089** (host memory / `observe` footprint) before the next full cold-rebuild validation.
- Optionally firm up the generic guide's §5 Native HA standalone-restore inference once an IBM Support-page source can be obtained (outside the repo doc tool's reach today).
