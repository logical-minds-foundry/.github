# Retrospective — Dual-mechanism Native HA (CRR/IRR, latency-tunable)

- **Epic:** `logical-minds-foundry/.github#227`
- **Retrospective task:** `logical-minds-foundry/.github#229`
- **Spec / Plan:** `epics/227-nativeha-crr-irr-latency/{spec.md, plan.md}` (read spec → plan → this)
- **Date:** 2026-09-16

## §0 At a glance

We set out to extend the RHEL Native HA arm into a **CRR-vs-IRR sync/async comparison lab**
— rename the existing arm to an explicit CRR member, add a strict-sync IRR sibling, add a
tunable WAN-latency knob, and quantify "what does zero-RPO cost" as a curve. What actually
shipped is **smaller and, arguably, more valuable**: the mandatory first task — an IRR-setup
facts spike — discovered that IBM MQ 10.0 **IRR is a 1 + 1 DR-*without*-HA topology**, not a
3 + 3 synchronous sibling of CRR. That invalidated the comparison's core premise, so we
**descoped mid-flight**: we kept the independently valuable pieces (the CRR rename, the
`mqlab netem` knob, the benchmark client, and the documented IRR evaluation) and **cut the
IRR build and the comparison study** — without falling for the sunk-cost fallacy, since the
already-built PRs were still unmerged when the premise collapsed.

**Work delivered — 7 merged PRs (+ this retrospective):**

| PR | Repo | What it did |
|---|---|---|
| `.github#230` | .github | Publish spec + plan (the epic docs) |
| `mq#1116` | mq-resiliency-lab | **IRR facts spike** report — the finding that reshaped the epic |
| `mq#1113` | mq-resiliency-lab | Full atomic **CRR rename** → `nativeha-rhel-crr` / `NHARCAPP` / `nha-rhel-crr-*` |
| `mq#1114` | mq-resiliency-lab | **`mqlab netem`** WAN-latency knob (delay-only, per-tap on `virbr-wan`) |
| `mq#1115` | mq-resiliency-lab | Purpose-built **benchmark client** (`clients/bench_client.py`) |
| `mq#1118` | mq-resiliency-lab | **Doc-review sweep** — 17 docs reconciled to the shipped reality |
| `.github#232` | .github | **Descope** — fold the IRR finding into spec + plan |

- **Repos touched:** `logical-minds-foundry/.github` (epic docs), `logical-minds-foundry/mq-resiliency-lab-for-linux` (code + repo docs).
- **Tasks:** 13 sub-issues — 8 delivered (incl. the `validation` operational task #1098, PASS), 3 deferred (#1107/#1108/#1109 → a future IRR epic), 1 invalidated (#1110), 1 terminal (this).
- **Validation:** #1098 SUCCESS — a genuine cold rebuild on cloud x86, one green pass; `NHARCAPP` healthy CRR pair; netem measured symmetric ≈20 ms at `--delay 10ms`.
- **Releases cut:** none (lab repo).
- **Span:** opened 2026-09-15 17:16Z → retrospective 2026-09-16 (~1 day).

## §1 How the plan evolved

The plan's central risk-control was ordering: **Task 1 was a facts spike that gated the
build** (Tasks 3/4/5). That single decision is why this epic ended well instead of badly.

The delta between plan and reality is large — and deliberately so:

1. **Pushback pre-committed to the right shape.** Before any code, pushback forced two
   decisions that held up: two parallel independent stacks (not co-resident QMs), and
   stating the netem *goal* rather than pinning a `tc` mechanism. Both proved correct — the
   build later confirmed a bridge-root qdisc does nothing (the L2-switching gotcha) and
   per-tap netem is required.
2. **The spike detonated the premise.** #1103 found IRR = two single-instance groups
   (`SyncReplication=Yes` + `SyncConsistency`, `NativeHAInstance` stanzas absent, shared-file
   active-arbitration) — DR without local HA. A CRR-vs-IRR comparison was therefore not
   apples-to-apples: it pits *HADR + async* against *DR-only + sync*.
3. **We verified before redesigning.** Because the finding contradicted the earlier CRR
   spike's wording and was load-bearing, we re-read the cached primary IBM 10.0 pages — which
   *confirmed* the spike rather than overturning it.
4. **Descope, not sunk cost.** With four PRs already built-and-ready (rename, netem, bench,
   report) but unmerged, we kept all four (the rename is a forward-enabling prerequisite for
   a future IRR arm; netem is broadly useful; the report is the durable knowledge) and cut
   the unbuilt IRR chain (#1107/#1108/#1109, deferred) and the comparison (#1110,
   invalidated). The spec and plan were re-banner'd to record the pivot; the original design
   is preserved as the R&D record.

## §2 Lessons learned

- **Gate expensive builds behind a facts spike — it paid for itself.** The spec's Task-1
  spike caught an invalidated vendor-behavior assumption *before* Tasks 3/4/5 were built.
  This is the reusable pattern: when a whole initiative rests on "the vendor's X works like
  Y," prove Y first and make everything downstream depend on it.
- **Verify load-bearing vendor claims against primary docs — both directions.** We had
  *assumed* IRR was "CRR with a sync flag." It wasn't. And when the spike said so, we still
  re-verified before acting. Neither the original assumption nor the correction was taken on
  faith.
- **Refuse the half-measurement.** The strongest call was *not* running the comparison. It
  would have quantified the perf cost of synchronous replication while saying nothing about
  the cost of *losing local HA* — an unquantifiable that dwarfs it. Measuring only the
  quantifiable side, then presenting it as "the tradeoff," is a classic and seductive error.
- **Unmerged work is free to drop; that is what defeats sunk cost.** Because the gate keeps
  PRs at report-ready until a human merges, the premise collapsed while everything was still
  reversible. The keep/cut decision was made on *forward* value, not effort spent.

## §3 Compromises & tradeoffs

- **Kept the CRR rename despite cutting IRR.** `nativeha-rhel-crr` now implies a sibling
  (`-irr`) that does not yet exist — a small naming oddity carried deliberately, as a bet
  that a future IRR arm justifies having done the rename (and the arm-coexistence naming)
  now rather than redoing it later. Reversible if that bet sours.
- **Kept the benchmark client without its motivating consumer.** `bench_client.py` was built
  for the (now-cut) comparison. It is a reusable persistent-commit throughput/latency tool,
  but currently has no epic driving it — retained on the judgment that a generic MQ perf
  probe earns its keep in a resiliency lab.
- **TDD honesty.** Two implementation sub-agents authored tests and code together, so their
  suites passed on first run rather than showing a red step first — noted, not hidden.
- **Attribution trailer gap.** `vrg-commit` (the only sanctioned commit path) exposes no
  trailer flag, so feature-branch commits lack an inline `Co-Authored-By`; it appears on the
  squash-merge commits, consistent with repo history. Flagged for tooling (§4).

## §4 New problems & opportunities

- **A future IRR epic (opportunity).** IRR is a real, if niche, HA/DR technology; the lab is
  meant to be a generic HA/DR-on-Linux testbed. The deferred tasks (#1107/#1108/#1109), the
  facts report (#1103), and the retained rename are the durable seeds. *Logged; not yet acted
  on — becomes its own epic if/when the use case arises.*
- **netem unlocks broader DR-latency testing (opportunity).** The knob was scoped delay-only;
  jitter and packet-loss are the natural next extensions, enabling degraded-link and
  failure-scenario DR testing well beyond this epic. *Logged as forward-axis work (§5).*
- **Stale `.venv` after VM churn (problem).** The validation found the host `.venv` carried a
  broken uv-managed interpreter after a prior VM rebuild; it needed `uv sync` before `mqlab`
  ran. *Logged — candidate for `triage-capture` if it recurs.*
- **`report-ready` does not push the branch (tooling problem).** Twice, a relay
  `vrg-submit-pr` failed with "branch not on origin" because `report-ready` freezes the
  branch and pushes only the metadata ref. The workaround is push-before-report-ready (or
  unfreeze → push → re-ready). *Candidate vergil-tooling issue: `report-ready` (or the
  sub-agent flow) should push the branch first.*
- **Background-watcher exit race (minor).** The validation agent's bootstrap-exit watcher
  missed the process exit (armed as it ended), leaving it parked after the rebuild had
  actually finished; a ground-truth probe from the driver (`virsh list` + `pgrep`) unstuck
  it. *Minor agent-orchestration lesson: verify terminal state directly, don't trust only
  the exit watcher.*

## §5 What's next

- **A dedicated IRR arm epic** — deferred here, seeded by #1103 + #1107/#1108/#1109 + the
  CRR rename. Not scheduled; raised when a concrete IRR use case appears.
- **netem jitter + packet-loss extension** — the delay-only knob's planned follow-on, for
  DR failure-scenario testing.
- **Optional housekeeping** — `triage-capture` for the stale-`.venv` operational issue and
  the `report-ready`-doesn't-push tooling gap.

## Appendix A — Operational notes (validation)

The cold-rebuild validation (#1098) exercised the shipped arm end-to-end on the resized
cloud x86 VM:

- **Cold rebuild, one green pass.** From no guests: all baked boxes showed hash mismatch
  post-#1088 and were **rebaked from scratch** (incl. `mq-nativeha-rhel9` in 445.99 s;
  box stage ~42 min), then the six `nha-rhel-crr-*` guests booted and `site-nativeha.yml`
  provisioned (main provision 222 s). Full-log scan: `failed=` 0, `fatal:` 0, orphaned
  old-name references 0.
- **`NHARCAPP` health (MQ 10.0.0.0).** Live group (site A) `ROLE(Active)` + `QUORUM(3/3)`,
  all `INSYNC`/`HASTATUS(Normal)`; Recovery group (site B) likewise; CRR link
  `GRSTATUS(Normal)`, `CONNGRP(yes)`, backlog actively draining (expected async lag).
- **netem probe.** Baseline WAN RTT ~0.4 ms both directions; with `mqlab netem set
  --delay 10ms` → **A→B 20.514 ms, B→A 20.501 ms** (symmetric ≈2×D, 0 % loss); `clear`
  restored the default qdisc; the heartbeat bridges `virbr-hb-a/-b` stayed `noqueue`
  (untouched) throughout.
- **Gotchas.** A non-fatal storage-pool GC warning during the box stage; the stale-`.venv`
  fix (§4); the bootstrap-exit watcher race (§4).
