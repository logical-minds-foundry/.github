# MQ portfolio wind-down — ship the lab, publish the SWIFT example, retire the rest — design spec

- **Epic:** `logical-minds-foundry/.github#245`
- **Design task:** `logical-minds-foundry/.github#246`
- **Doc-review bookend:** `logical-minds-foundry/mq-resiliency-lab-for-linux#1149`
- **Retrospective (terminal):** `logical-minds-foundry/.github#247`
- **Validation (outsider cold-run):** `logical-minds-foundry/mq-resiliency-lab-for-linux#1150`
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-09-19

## 1. Problem & motivation

Logical Minds Foundry is making a deliberate strategic refocus. Much of the
in-flight IBM MQ work was queued as follow-on to a specific direction that is now
winding down; most of it no longer has a business rationale to continue. Left
alone, roughly twenty open epics across six repositories — several with partial
code and open child tasks — would read to any visitor as a field of abandoned
work-in-progress.

That outcome is the real risk this epic exists to prevent. The technical work is
sound and, in places, genuinely good (the SWIFT settlement-tracking
proof-of-concept most of all). The goal is to leave the portfolio looking like a
set of **deliberate decisions** — one finished product, one published example,
and a clean, documented retirement of everything else — rather than a pile of
things that were simply walked away from.

This is a **finite, run-once** epic: it manages the direction change itself, then
closes.

## 2. Doctrine & principles

- **Deliberate decisions, not attrition.** Every open epic gets an explicit
  disposition — ship, suspend, or drop — and no epic is left dangling.
- **No dangling WIP.** By the end, there are no open epics or tasks that imply
  work still in flight, except the standing ad-hoc umbrellas for kept repos.
- **Retire with a record.** Every retired epic (suspend or drop) is reviewed,
  given a retrospective, and has any documentation worth keeping extracted and
  published *before* it is closed. Suspension is closure-with-a-parking-note, not
  a permanently open issue.
- **Correctness over cost.** We finish the small amount of ship work properly
  rather than publishing a half-working product with a "fix later" note.
- **Everything technical is shareable.** There is no proprietary code or data;
  publishing is safe. Business/relationship specifics are kept out of public
  wording (see §8).
- **The product is proven by an outsider, not by us.** "Shippable" means a
  stranger can download and cold-run it (validation bookend `#1150`), not that it
  runs on the author's machine.

## 3. Repo roster (final)

| Repo | Visibility now | Fate |
|---|---|---|
| `mq-resiliency-lab-for-linux` | public | **Keep + finish** — the public, downloadable, runnable product |
| `docs` | public | **Keep** — update org narrative/roadmap to the new direction |
| `.github` | public | **Keep** — org config + this epic |
| `mq-resiliency-observability` | public | **Archive** (confirm at its epic review; likely archive read-only) |
| `mq-gateway-replay-lab` | private | **Rename → `mq-swift-settlement-lab`, make public** — SWIFT example lab; retains the request/reply confirmation-tracking example |
| `mq-protocol-gateway` | private | **Delete** — never meaningfully started (empty C++ scaffold) |

## 4. Epic dispositions

State column: DONE / PARTIAL / NOT-STARTED / STANDING (an ongoing umbrella, not a
finite deliverable).

### 4.1 Ship — finish for the public product

| Epic | What | State | Notes |
|---|---|---|---|
| #161 | Public-release site docs (consumer path, no internal tooling) | PARTIAL | Biggest single lift; spec+plan exist, authoring + release-pipeline tasks remain |
| #8 + #169 | Complete, **honest** lab dashboards | PARTIAL | Merge #8's dashboard-honesty bug-fixes (#381 empty cockpit, #383 empty panels, #183 channel metrics, #194 log streaming) with a **fresh rebuild** of the dashboards (#169). Rebuilt from scratch for the lab — the prior work-context versions are inaccessible, which also keeps them unambiguously the Foundry's own |
| #236 | Component version-currency refresh | NEARLY DONE | Research complete; low-risk bumps + manifest reconcile. Prometheus 2→3 breaking change stays gated/deferred. Expect a few further routine bumps as part of finishing |
| #104 | Clean one-shot cold build | STANDING | One focused reliability pass so an outsider gets a working cold rebuild; underpins validation `#1150` |
| #29 / #60 / #128 / #206 | Standing ad-hoc umbrellas (lab / .github / observability / docs) | STANDING | Kept **open** for the kept repos; not deliverables |

From #8, the advanced CLI drill runner (#119) is **not** ship scope — suspend it
with #8's other advanced items; ship only the dashboard-honesty fixes.

### 4.2 Suspend — retro + park (not deleted)

| Epic | What | State |
|---|---|---|
| #7 | mqlab CLI maturation (DX, advanced command groups) | PARTIAL |
| #9 | Distributed HA/DR topology & vendor DR | NOT-STARTED |
| #12 | Productization & knowledge transfer | PARTIAL |
| #17 | Concurrent multi-stack lab | PARTIAL |
| #19 | MQ configuration guides | PARTIAL |
| #165 | Discovery-native MQ administration + AUTHREC migration | NOT-STARTED |
| #191 | Native HA log-lifecycle observability (Ubuntu) | NOT-STARTED |

- **#12 before suspension:** cherry-pick any publish-critical documentation
  (e.g. "document the lab as published software", methodology) into the #161 ship
  work first, so nothing publish-blocking is parked by accident.
- **#191 escalation:** promote to ship **only** if the finished dashboards look
  incomplete without the log-lifecycle band; otherwise park.

### 4.3 Drop — retro + close as won't-do

| Epic | What | State | Cleanup cost |
|---|---|---|---|
| #79 | Standalone observability extraction (installable package) | NOT-STARTED | Low |
| #66 | IBM defect reports cache | STANDING | **Archive the dossiers as evidence first**, then close; no filing path remains |
| #268 | Stateless MQ protocol gateway | PARTIAL (heavy) | **~15 open child tasks** — heaviest retirement; drives `mq-protocol-gateway` deletion |
| #228 | MQ message-exit capability | PARTIAL | ~4 open child tasks |
| #59 | Generic request-reply reconciliation | DONE (core) | ~2 wrap-up tasks; effectively just close it |

The SWIFT epics in the replay-lab repo (#65 settlement-tracking, #105 COA/COD
telemetry, #175 traffic realism, #235 least-privilege) are already **closed** —
consistent with "the SWIFT work is done; rename and publish the repo."

**Rename conflict to honour:** dropped epic #268 contains a task (#272) to rename
`mq-protocol-gateway`. That is moot — the protocol-gateway repo is deleted, and
the SWIFT rename (`mq-gateway-replay-lab → mq-swift-settlement-lab`) is the one
that stands.

## 5. Scope & work breakdown

Grouped by workstream. Precise task graph, repo placement, and dependency
reflinks are the plan's job (§6 gives the intended order); the retirement
**mechanics** (how each retired epic maps to a child task here vs. its own
`epic-retrospective` run) are the key structural question for the plan +
alignment stage — see §5.4.

### 5.1 Ship the lab (constructive)

- Dashboards: honest bug-fixes + fresh rebuild (#8 + #169).
- Public-release site docs (#161), including the release/download path.
- Version-currency finish + routine bumps (#236).
- Cold-build reliability pass (#104).
- Publishing/release: make it downloadable and executable by an outsider. Cutting
  any release is **human-gated** (attested, not agent-performed).

### 5.2 Publish the SWIFT example (constructive)

- Rename `mq-gateway-replay-lab → mq-swift-settlement-lab`.
- README/docs reframe: SWIFT-first, marked as a completed proof-of-concept that
  bolts onto the resiliency lab; retain the request/reply example as a secondary
  example.
- Flip the repo to public (human-gated).

### 5.3 Delete protocol-gateway (constructive/destructive)

- Extract anything worth keeping (e.g. the recorded `imqi.hpp` MQI C++ API-surface
  findings) into the SWIFT lab or docs first.
- Delete the `mq-protocol-gateway` repo. **Irreversible — human-gated** (§8).

### 5.4 Retire the remaining epics (records)

Each retirement = review → extract/publish valuable docs → **retrospective** →
close, with any open child tasks first closed as won't-do so
`epic-retrospective`'s preflight can run. Targets, by home repo:

- **Org (`.github`) epics:** #7, #9, #12, #17, #19, #165, #191 (suspend); #79, #66
  (drop).
- **SWIFT-lab repo epics:** #228, #59 (drop). #268 (drop) is retired alongside the
  protocol-gateway deletion.

**Open structural question for the plan/alignment stage:** whether each retirement
is a formal child task of #245 (filed in the target epic's home repo, closed when
that epic's retrospective lands) or is tracked here as a checklist and executed as
each epic's own terminal `epic-retrospective` run. The placement law (a task lives
where its closing PR lands; a PR only `Closes` an issue in its own repo) constrains
the answer and must be respected either way.

### 5.5 Docs sweep + org narrative (bookend #1149)

- Site docs (in the lab repo) and every kept repo's README state their **real**
  final status (v1 product / published POC / archived / parked).
- The `docs` repo org narrative and roadmap reflect the new direction.
- Multi-repo sweep: spawn per-repo doc tasks where docs live outside the lab repo.

## 6. Sequencing, validation & closure

1. **Drops first** — retire #59, #228, #268 (+ delete protocol-gateway), #79, #66.
   Fastest reduction of the open-WIP surface; makes the portfolio legible early.
2. **SWIFT publish** — rename, reframe, make public.
3. **Ship work** — dashboards, site docs, version-currency, cold-build pass; then
   the download/release path.
4. **Suspends** — retro + park the remaining org epics (after #12 doc cherry-pick).
5. **Archive** `mq-resiliency-observability` (confirmed at its review).
6. **Docs sweep** (#1149) across all repos.
7. **Validation** (#1150) — an outsider downloads and cold-runs the lab; closes on
   a SUCCESS comment.
8. **Retrospective** (#247) — terminal; its merge closes epic #245.

## 7. Acceptance criteria

- `mq-resiliency-lab-for-linux` is public and an outsider can **download and
  cold-run it** (validation #1150 = SUCCESS), with honest, complete dashboards and
  public-release site docs.
- `mq-swift-settlement-lab` exists (renamed), is **public**, and its README
  presents a completed SWIFT proof-of-concept that bolts onto the lab.
- `mq-protocol-gateway` is deleted; anything worth keeping was extracted first.
- Every dropped epic (#79, #66, #268, #228, #59) is **closed** with a
  retrospective; #66's dossiers are archived.
- Every suspended epic (#7, #9, #12, #17, #19, #165, #191) is **closed as parked**
  with a retrospective; no publish-critical #12 docs were lost.
- `mq-resiliency-observability` disposition is executed (archived or as decided).
- No open epics/tasks imply in-flight work except the standing ad-hoc umbrellas.
- Every kept/renamed/archived repo README and the `docs` org narrative state the
  true final status.
- Epic #245's own retrospective (#247) is authored and merged.

## 8. Risks & mitigations

- **Dashboards rebuilt from memory (#169).** The prior versions are inaccessible.
  *Mitigation:* rebuild against the lab's own live metrics; the outsider cold-run
  validation catches empty/broken panels. Building fresh also removes any question
  of external provenance.
- **Repo deletion is irreversible (`mq-protocol-gateway`).** *Mitigation:*
  extract-worth-keeping step first; deletion is **human-gated**, never
  agent-performed. Same gate for flipping the SWIFT repo public and cutting any
  release.
- **Public wording of a business change.** The technical work is shareable; the
  fact of a specific engagement ending is not broadcast. *Mitigation:* public
  issue/README/narrative wording frames a **strategic refocus**, not the
  relationship detail. Confirm with the owner if any wording should be more
  explicit.
- **#268's heavy cleanup (~15 tasks).** *Mitigation:* close its children as
  won't-do in one deliberate pass; the retrospective records why, so the bulk
  close reads as intentional.
- **Ship scope creep.** The temptation to finish "just one more" suspended
  feature. *Mitigation:* §4.1 is the closed ship list; #191 is the only
  conditional promotion, on a stated trigger.

## 9. Follow-on

Recorded here so the retrospective (#247) can close the forward axis honestly:

- The **suspended** epics (#7, #9, #12, #17, #19, #165, #191) are the natural
  re-entry points if MQ work resumes; each carries a parking note explaining what
  was done and what remained.
- The **SWIFT lab** and the **resiliency lab** could each grow follow-on examples
  or features, but none are chained as enabling work for this epic — so no
  follow-on brainstorm bookend is seeded.
