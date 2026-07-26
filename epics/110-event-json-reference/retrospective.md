# Retrospective — Working with MQ instrumentation events as JSON (epic #110)

Partners [`spec.md`](spec.md) and [`plan.md`](plan.md). Read spec → plan →
retrospective: what we set out to do, how we planned it, and honestly how it went.

## §0 — At a glance

We set out to make IBM MQ's `amqsevt` JSON event feed **usable**, not just
*producible*: IBM publishes no formal schema, so we wrote two reference reports
grounded in **real JSON captured from the live lab** — a **consume-side** schema
reference (what the data is, how to parse it, does it survive syslog) and a
**generate-side** reference (how to force one event of each class) — plus the
captured fixtures behind them.

**Work delivered**

| PR | What it did |
|---|---|
| [#790](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/790) | Report A — the consume-side schema/reference (envelope, PCF→JSON key rule, unordered/conditional-keys caveat, `eventType` taxonomy, annotated captured appendix) |
| [#791](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/791) | Finish: T1 fixtures (13 real captured events + syslog-fidelity table), Report B (generation reference), Report A §8 (the measured syslog-fidelity finding), and the site-guide cross-link |

- **Repos touched:** `mq-resiliency-lab-for-linux` (all reports + fixtures);
  `.github` (this retrospective).
- **Tasks:** 5 delivery tasks — #710 (docs review), #712 (T1 capture),
  #713 (T2 Report A), #714 (T3 Report B), #780 (off-plan Report A) — all closed,
  plus #133 (this retrospective).
- **PRs:** 2. **Releases:** none (documentation epic).
- **Span:** opened ≈2026-07-19 (about a week before close; paused at creation to
  build its prerequisite, #114) → all delivery work closed **2026-07-26**.
- **Deliverables now on `develop`:**
  `docs/reports/2026-07-22-mq-event-json-working-with-the-data.md`,
  `docs/reports/2026-07-26-mq-event-generation-lab-reference.md`,
  `docs/reports/assets/110-mq-event-captures/` (13 fixtures + fidelity table).

## §1 — How the plan evolved

**It began by being paused.** #110 was created before its own foundation existed:
you cannot document *working with* the event feed until the feed is flowing
everywhere. So at kickoff the epic was parked to stand up **#114 — event
monitoring as a de-facto standard on every queue manager**. #110 resumed only once
#114's rollout put `amqsevt -o json_compact` on every QM, which is also what made
"capture a real event of each class from any arm" a trivial precondition rather
than a bespoke setup.

**A self-inflicted duplication.** On resuming, the agent created a fresh Report A
task (#780) **without seeing the epic's existing T1/T2/T3 plan** (#712/#713/#714) —
the parent-issue search returned empty and the tasks were missed. #780 duplicated
T2 (#713). It was then **submitted and merged (#790) prematurely**, against the
agent's own recommendation to hold. Rather than revert, we **worked around it**:
#790 became the Report A base, and #791 completed the plan *on top of it* — the T1
fixtures, the syslog-fidelity retrofit into Report A (§8), Report B, and the docs
cross-link — then #713 was reconciled and closed as the duplicate.

**A toolchain detour that ate a chunk of the middle.** The live-capture bootstrap
kept failing: the `vrg-container-run` image had silently jumped to **Python
3.14**, where `ansible-lint` crashes and `bootstrap` flaked on `ansible-galaxy`.
Several red-herring investigations later, the root cause was a **hardcoded default
container version in vergil-tooling** (`_DEFAULT_VERSIONS["python"] = "3.14"`) that
ignores the repo's declared `[ci].versions = ["3.12"]`. We fixed it in-repo by
**pinning `requires-python` to 3.12** (#782) — uv reads only `pyproject.toml` — and
filed the tooling decoupling upstream. The pin also cleared the bootstrap flake
(#781), proven when the capture bootstrap then came up clean end to end.

**The capture method that stuck.** Because the collector holds the event queues
exclusively (`amqsevt -b` hit `MQRC_OBJECT_IN_USE` 2042), the reliable recipe was:
pause `SERVICE(MQ.EVENT.MONITOR)` → force one event per class → **authoritative
`amqsevt -b` browse** (non-destructive) → resume → capture the **journald** copy →
compare byte-for-byte. That comparison produced the epic's headline empirical
finding.

## §2 — Lessons learned

- **Read the epic's task graph before creating tasks.** The #780/#713 duplication
  was pure avoidable rework — the plan already had T1/T2/T3. When a
  parent/sub-issue query comes back empty, treat it as *tooling uncertainty*, not
  *absence*, and verify before minting a task.
- **Pin the interpreter where the toolchain actually reads it.** Declaring the
  Python version in `vergil.toml` is not enforcement — `uv` only reads
  `pyproject.toml`. `requires-python` is the lever; a floor (`>=3.12`) invites
  silent drift to whatever the container ships.
- **`amqsevt -b` browse (collector paused) is the authoritative-copy technique.**
  It sidesteps the collector's exclusive handles and yields the untruncated event,
  which is what makes a fidelity measurement possible.
- **A hardcoded tool default is a global, unversioned behaviour change.** One
  constant bump in shared tooling flipped every repo's local build with nothing in
  any repo's history to show it — the most dangerous class of surprise this epic hit.

## §3 — Compromises & tradeoffs

- **Report A shipped before it was complete.** The premature merge of #780 (#790)
  put Report A on `develop` without its syslog-fidelity finding or the T1
  fixtures; both were **retrofitted** in #791 instead of arriving in one clean
  deliverable. Honest cost of not holding the branch.
- **Two of eleven event classes were not force-captured live.** **AUTHOREV
  (2035)** could not be forced on the hardened QM in the window (local
  `root`/`mqm` are authorized; the unprivileged path lacked MQ libs) — it is
  documented from the prior capture set (Report A §A.1). **CHADEV** and **SSLEV**
  were deferred (CHAD staging / a TLS cert fault) per the plan's derive-later
  allowance. The 13 captured classes cover all five `eventType` families, so the
  schema coverage is complete even though the class list is not.

## §4 — New problems & opportunities

- **Python-3.14 toolchain instability** → filed for the tooling team:
  `vergil-tooling#2460` (ansible-lint crash), `#2461` (uv hardlink warning), and
  the load-bearing **`#2468`** (container-version decoupling from `[ci].versions`).
  Currently owned by a separate agent.
- **#781** (bootstrap `ansible-galaxy` flake) — **resolved** as a side effect of
  the #782 pin.
- **#768** (rdqm cold-create `crtmqm` mkfs I/O error) — a separate reliability bug
  surfaced during earlier DR verification; **open** under reliability epic
  `.github#104`, not caused by this work.
- **Opportunity:** Report B's per-class force recipes are the seed for
  **mechanising event generation** in the live-lab validation framework
  (`.github#38`).

## §5 — What's next

- The tooling fixes (`#2460`/`#2461`/`#2468`) proceed with the separate agent.
- The reliability tail (`#768`) remains open under `.github#104`.
- Event-generation mechanization (from Report B) is a candidate for `.github#38`.
