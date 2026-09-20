# MQ portfolio wind-down — implementation plan

> **For agentic workers:** this plan is executed through the Vergil epic
> framework, not the generic subagent-driven runner. Each task below maps to a
> GitHub issue filed under epic `#245` (see "Task model"). Code tasks are worked
> with `issue-implement` (TDD applies); retirements with `epic-retrospective`;
> operational tasks (`validation` / repo ops) with `issue-validate` /
> `issue-deploy` or by hand where noted. Checkboxes track task-filing and
> completion.

**Goal:** Bring the MQ portfolio to a coherent, presentable end state — one
finished public product (`mq-resiliency-lab-for-linux`), one published example
(`mq-swift-settlement-lab`), and every other epic and repo retired with a record,
nothing deleted.

**Architecture:** A single coordinating epic (`#245`) whose children are the
wind-down's own operational and coordination tasks. Retirements run through each
target epic's own `epic-retrospective`; kept "ship" epics are driven to
completion under their own umbrellas and tracked here by a coordination task;
repos are archived (read-only), never deleted.

**Tech Stack:** Vergil tooling (`vrg-*`), GitHub issues/PRs, Ansible + shell
(lab), Grafana/Prometheus/Loki (dashboards/observability), Python (SWIFT lab).

**Spec:** `epics/245-portfolio-wind-down/spec.md`

## Global Constraints

- **Archive, never delete.** Every retired repo is archived read-only. No repo is
  deleted. (spec §2, §3)
- **`develop` is read-only; all changes flow through a worktree + feature branch +
  PR, validated with `vrg-container-run -- vrg-validate`.** One worktree per issue,
  named `issue-<N>-<slug>`; sessions start at the project root. (CLAUDE.md)
- **Use `vrg-git` / `vrg-gh`**, never raw `git` / `gh`.
- **Placement law:** a task lives in the repo where its closing PR lands; a PR
  only `Closes` an issue in its own repo. Cross-repo links are `Ref`/comment.
- **Human-gated, never agent-performed:** archiving a repo, flipping a repo
  public, renaming a repo, and cutting any release. The agent prepares and
  attests readiness; a human executes the GitHub/admin action.
- **Non-gating history secret scan** before any repo is made public.

## Task model

Every task below is a child of `#245`, filed with
`vrg-issue-create --epic logical-minds-foundry/.github#245 --repo <home> …`
(step 9 of epic-create, after the docs PR merges). Task kinds:

- **Retirement** — retire a suspended/dropped epic. Filed in the *target epic's
  home repo*. Executed by running `epic-retrospective` on the target (closes the
  target in its own home). The `#245` child is closed by a **completion comment**
  (never a cross-repo `Closes`). Precondition: the target epic's own open children
  are closed as won't-do first (so `epic-retrospective`'s preflight can run).
- **Coordination** — drive a *kept* ("ship") epic to completion. Filed in `#245`'s
  home (`.github`). Closed by comment when the ship epic closes. The detailed work
  lives in that ship epic's own plan; this task only tracks and gates.
- **Code / docs** — ordinary PR-workable task in the named repo (`issue-implement`,
  TDD).
- **Operational** (`validation` / repo-op) — proven by running something and
  recording `Outcome: SUCCESS`, not by a merge.

Repo homes: org epics (#7/#9/#12/#17/#19/#66/#79/#165/#191) home in
`logical-minds-foundry/.github`; the SWIFT-lab epics (#59/#228/#268) home in
`mq-gateway-replay-lab` (→ `mq-swift-settlement-lab` after rename).

## Dependency graph

```text
Phase 1 (drops)        Phase 2 (SWIFT publish)     Phase 3 (ship lab)
 T1 #59  ─┐             T6 rename ─▶ T7 reframe ─┐   T11 dashboards ─┐
 T2 #228 ─┼─▶ (repo     (needs T1,T2,T3 done)    ├─▶ T8 make public  T12 site docs ─┤
 T3 #268 ─┘   clean)                             │   T13 version-cur ─┼─▶ T15 release/
 T4 #79                                          │   T14 cold-build ──┘    download path
 T5 #66 (archive dossiers first)                 │
        └─▶ T9 archive protocol-gw (needs T3)    │
                                                 ▼
Phase 4 (suspends)     Phase 5 (archive obs)   Phase 6/7 (close-out)
 T16..T22 retro+park    T10 archive obs +        T23 doc-review sweep (#1149)
 (T18 #12 after doc      close #128 (needs T4)    T24 validation (#1150)  [gates on T8..T15]
  cherry-pick into T12)                           T25 retrospective (#247) [terminal; all others closed]
```

---

## Phase 1 — Retire the drops

Do first: fastest reduction of the open-WIP surface (spec §6).

### Task 1: Retire epic #59 — generic request-reply reconciliation

**Home:** `mq-gateway-replay-lab` · **Kind:** retirement · **Blocked-by:** none

- [ ] Close #59's open wrap-up child tasks (~2) as won't-do, each with a one-line
      "wind-down #245: not pursued" comment.
- [ ] Run `epic-retrospective` on #59. The core R/R reconciliation shipped; the
      retro records what was delivered (the working reconciliation demo) and that
      only wrap-up polish was dropped. Docs PR lands `retrospective.md` in the repo.
- [ ] Verify #59 closes on rollup.
- [ ] Close this #245 child with a completion comment linking #59's retro.

**Acceptance:** #59 CLOSED with a retrospective; no open #59 children.

### Task 2: Retire epic #228 — MQ message-exit capability

**Home:** `mq-gateway-replay-lab` · **Kind:** retirement · **Blocked-by:** none

- [ ] Close #228's open child tasks (~4) as won't-do with the wind-down note.
- [ ] Run `epic-retrospective` on #228: record that message-exit engineering was a
      niche follow-on, deliberately not pursued; note anything worth knowing for a
      future reader (why exits were considered, why dropped).
- [ ] Verify #228 closes; close this #245 child by comment.

**Acceptance:** #228 CLOSED with a retrospective.

### Task 3: Retire epic #268 — stateless MQ protocol gateway (heaviest)

**Home:** `mq-gateway-replay-lab` · **Kind:** retirement · **Blocked-by:** none

- [ ] Close #268's ~15 open child tasks as won't-do in one deliberate pass, each
      with the wind-down note (the bulk-close is intentional; the retro explains it).
- [ ] Lift keepable material into a surviving home before archival (feeds T9):
      the recorded `imqi.hpp` MQI C++ API-surface findings and the "counter-example"
      writeup → into the SWIFT lab `docs/` or the `docs` repo. (Archival preserves
      everything anyway; this is for discoverability.)
- [ ] Run `epic-retrospective` on #268: record that the stateless C++ gateway was
      the first C++ effort, reached a partial/experimental state, and is retired;
      point to the archived repo and the lifted findings.
- [ ] Verify #268 closes; close this #245 child by comment.

**Acceptance:** #268 CLOSED with a retrospective; keepable material lifted; ready
for T9 archival.

### Task 4: Retire epic #79 — standalone observability extraction

**Home:** `logical-minds-foundry/.github` · **Kind:** retirement · **Blocked-by:** none

- [ ] Close any open #79 children as won't-do.
- [ ] Run `epic-retrospective` on #79: record the decision that observability stays
      in the lab rather than being extracted as a separately-installable package;
      note the existing `mq-resiliency-observability` repo is archived (T10).
- [ ] Verify #79 closes; close this #245 child by comment.

**Acceptance:** #79 CLOSED with a retrospective.

### Task 5: Retire epic #66 — IBM defect reports cache (archive dossiers first)

**Home:** `logical-minds-foundry/.github` · **Kind:** retirement · **Blocked-by:** none

- [ ] **Archive the dossiers as evidence first:** collect the accumulated IBM
      MQ/RDQM defect dossiers into a durable, surviving location (a `docs/` folder
      in the `docs` repo, or an attachment on the retro) so the evidence is not lost
      when the epic closes.
- [ ] Close any open #66 children as won't-do.
- [ ] Run `epic-retrospective` on #66: record that there is no IBM filing path after
      the engagement and the dossiers are archived, not actionable.
- [ ] Verify #66 closes; close this #245 child by comment.

**Acceptance:** #66 CLOSED with a retrospective; dossiers preserved in a live repo.

---

## Phase 2 — Publish the SWIFT example

Runs after the SWIFT-lab drops (T1–T3) so the public repo shows a clean, decided
state, not open WIP.

### Task 6: Rename `mq-gateway-replay-lab` → `mq-swift-settlement-lab`

**Home:** `mq-gateway-replay-lab` · **Kind:** operational (repo-op, human-gated) ·
**Blocked-by:** T1, T2, T3

- [ ] Prepare the rename: inventory internal references to the old name — `README`,
      `vergil.toml`, badge URLs, `pyproject.toml` metadata, docs links, CI config —
      and stage edits (these land via T7's PR, or a small pre-rename PR).
- [ ] **Human-gated:** a human renames the repo in GitHub settings (GitHub keeps
      redirects from the old name).
- [ ] Confirm `vrg-*` tooling and remotes resolve the new name; update any local
      worktree remotes.

**Acceptance:** repo is `mq-swift-settlement-lab`; tooling resolves it; old-name
redirects work.

### Task 7: Reframe the SWIFT lab README + docs (SWIFT-first, POC-complete)

**Home:** `mq-swift-settlement-lab` · **Kind:** docs · **Blocked-by:** T6

- [ ] Rewrite `README.md`: lead with the **SWIFT MT540–548 settlement-tracking**
      proof-of-concept as the headline; describe it as a **completed POC** that
      bolts onto `mq-resiliency-lab-for-linux` (owns no VM); retain the
      **request/reply confirmation-tracking** example as a documented secondary
      example. Set Status to reflect a finished POC, not "early development".
- [ ] Ensure `docs/` entry points (proof-of-concept.md, scale-analysis.md,
      settlement reference) are linked and read as a finished narrative.
- [ ] Add a short "what this is / what this is not" note (an example lab, not a
      supported product).
- [ ] Validate (`vrg-container-run -- vrg-validate`); PR; `vrg-pr-workflow
      report-ready`.

**Acceptance:** README/docs present a coherent finished POC; validation green.

### Task 8: Secret-scan history + make the SWIFT lab public

**Home:** `mq-swift-settlement-lab` · **Kind:** operational (repo-op, human-gated) ·
**Blocked-by:** T7 (and T1–T3 for a clean state)

- [ ] Run a **non-gating** secret scan over full history (`vrg-trivy-scan` /
      available secret tooling) and a quick skim of the SWIFT test fixtures for any
      real-looking data. Record the result in a comment. (Non-gating per owner
      decision; surface anything surprising before proceeding.)
- [ ] **Human-gated:** a human flips the repo to public.
- [ ] Confirm the public repo renders (README, docs, license) and CI/badges work.

**Acceptance:** repo public; scan result recorded; renders cleanly.

---

## Phase 3 — Ship the lab (the public product)

The kept epics are finished under their own umbrellas; each `#245` coordination
task gates on that epic closing. The dashboard rebuild carries concrete scope here
because #169's original ("work-edition") scope is obsolete.

### Task 11: Complete, honest lab dashboards (rebuild + honesty fixes)

**Home:** `mq-resiliency-lab-for-linux` · **Kind:** code (may span sub-PRs) ·
**Drives:** #8 + #169 · **Blocked-by:** none

**Files (indicative):** Grafana dashboard JSON under the lab's dashboards
provisioning path; Prometheus/Loki datasource + panel definitions; any exporter
wiring for missing metrics.

- [ ] **Honesty fixes (#8 children):** eliminate empty/misleading panels — empty
      cockpit (#381), empty network panels (#383), missing channel metrics (#183),
      log streaming (#194). For each: confirm the underlying metric/log source
      exists, wire the panel to real data, and **remove or clearly mark any panel
      whose data source does not exist** (no silent empty panels — spec §2 "honest").
- [ ] **Rebuild (#169):** rebuild the dashboards fresh for the lab (prior
      work-context versions are inaccessible). Cover the queue-manager and cluster
      health story the lab demonstrates (Native HA / RDQM state, channels, queues).
- [ ] **Events & log entries:** surface events and log entries for the queue
      manager and for specific objects, built on the already-shipped log-availability
      tier (#145/#149/#198). *Not* the deep #191 log-lifecycle introspection.
- [ ] **Verify against live metrics:** bring up the lab, load each dashboard, and
      confirm every panel renders real data (no empty/placeholder panels). Capture
      before/after evidence in the PR.
- [ ] Validate; PR(s); `report-ready`.
- [ ] Close #119 (advanced CLI drill runner) and any other non-ship #8 children as
      won't-do/parked with a wind-down note (spec §4.1) — so #8 can retro cleanly.
- [ ] Run `epic-retrospective` on **#8** and on **#169** so each finite epic closes
      with a record of the delivered dashboards. Then close this #245 coordination
      child by comment.

**Acceptance:** every shipped panel shows real data or is intentionally removed;
epics #8 and #169 finished, retro'd, and closed.

### Task 12: Public-release site docs (drive #161)

**Home:** `mq-resiliency-lab-for-linux` · **Kind:** coordination + docs ·
**Drives:** #161 · **Blocked-by:** none (but see T18 #12 cherry-pick)

- [ ] Execute #161's existing plan (consumer-facing site docs, no internal tooling
      assumptions; the "download and run it" path).
- [ ] **Fold in the validation persona (spec §7):** the docs must state the **IBM MQ
      Developer Edition entitlement** and the **host CPU/RAM/disk minimums** a
      downloader needs, and give a cold-clone → running-lab walkthrough that assumes
      no author access.
- [ ] Cherry-pick any publish-critical docs from #12 before it is parked (coordinate
      with T18): "document the lab as published software", methodology.
- [ ] Validate; PR(s); `report-ready`.
- [ ] Run `epic-retrospective` on **#161** so it closes with a record; then close
      this #245 coordination child by comment.

**Acceptance:** public site docs let the validation persona (T24) succeed from a
cold clone; entitlement + host requirements documented; #161 retro'd and closed.

### Task 13: Component version-currency (drive #236)

**Home:** `mq-resiliency-lab-for-linux` · **Kind:** coordination · **Drives:** #236 ·
**Blocked-by:** none

- [ ] Execute #236's existing plan: low-risk component bumps (node_exporter, Alloy,
      Loki/logcli, Grafana pin) + manifest-drift reconciliation + version-sync
      guardrail. Keep the Prometheus 2→3 breaking change **gated/deferred** (spec §4.1).
- [ ] Absorb any further routine bumps that surface while finishing (owner expects a
      few).
- [ ] Run `epic-retrospective` on **#236** so it closes with a record; then close
      this #245 coordination child by comment.

**Acceptance:** obs component versions current (Prometheus 3 excepted, gated); #236
retro'd and closed.

### Task 14: Cold-build reliability pass (drive #104)

**Home:** `mq-resiliency-lab-for-linux` · **Kind:** coordination + code ·
**Drives:** #104 · **Blocked-by:** none

- [ ] One focused reliability pass on cold rebuild / boot / repoint / deploy so a
      stranger gets a working **one-shot cold build** from a clean clone. Fix the
      concrete breakages #104 tracks.
- [ ] This is the code substrate the T24 validation exercises — keep them aligned.
- [ ] Validate; PR(s); `report-ready`.
- [ ] **#104 is a standing ad-hoc umbrella — do the reliability pass but leave #104
      open** (like #29/#60/#206); do **not** retro/close it. Close this #245 child
      by comment when the reliability-pass PR(s) merge.

**Acceptance:** a cold clone builds and boots the lab without author intervention,
and epic #104 remains open as a standing umbrella.

### Task 15: Download / release path (make it obtainable & runnable)

**Home:** `mq-resiliency-lab-for-linux` · **Kind:** code + operational (release
human-gated) · **Blocked-by:** T11, T12, T13, T14

- [ ] Ensure the lab is obtainable and runnable by an outsider: a clean clone +
      documented steps produce a running lab (no private tooling required). Add any
      packaging/release artifacts the download path needs.
- [ ] **Human-gated:** cut any release/tag (`vrg-release`) — attested, never
      agent-performed.
- [ ] Close this #245 child by comment.

**Acceptance:** the product is downloadable and runnable per T12's documented path;
sets up T24.

---

## Phase 4 — Suspend (retro + park)

Each is a retirement task (retro + close), parked not deleted. The retro's §5
records what remained so it is a clean re-entry point (spec §9).

### Task 16: Suspend epic #7 — mqlab CLI maturation

**Home:** `.github` · **Kind:** retirement · **Blocked-by:** none

- [ ] Close open #7 children as won't-do (parked); `epic-retrospective` on #7
      recording what shipped (the working CLI) vs the DX/advanced-command-group work
      parked; close #245 child by comment.

**Acceptance:** #7 CLOSED (parked) with a retrospective.

### Task 17: Suspend epic #9 — distributed HA/DR & vendor DR

**Home:** `.github` · **Kind:** retirement · **Blocked-by:** none

- [ ] `epic-retrospective` on #9 (NOT-STARTED; record the design intent as the
      re-entry point); close children if any; close #245 child by comment.

**Acceptance:** #9 CLOSED (parked) with a retrospective.

### Task 18: Suspend epic #12 — productization & knowledge transfer (cherry-pick first)

**Home:** `.github` · **Kind:** retirement · **Blocked-by:** T12 (doc cherry-pick)

- [ ] **Before parking:** confirm T12 has cherry-picked any publish-critical #12
      docs (lab-as-published-software, methodology) into #161. Do not park until
      that is done (avoids parking publish-blocking docs).
- [ ] Close open #12 children as won't-do; `epic-retrospective` on #12; close #245
      child by comment.

**Acceptance:** #12 CLOSED (parked); no publish-critical doc left behind.

### Task 19: Suspend epic #17 — concurrent multi-stack lab

**Home:** `.github` · **Kind:** retirement · **Blocked-by:** none

- [ ] `epic-retrospective` on #17; close children; close #245 child by comment.

**Acceptance:** #17 CLOSED (parked) with a retrospective.

### Task 20: Suspend epic #19 — MQ configuration guides

**Home:** `.github` · **Kind:** retirement · **Blocked-by:** none

- [ ] `epic-retrospective` on #19; close children; close #245 child by comment.

**Acceptance:** #19 CLOSED (parked) with a retrospective.

### Task 21: Suspend epic #165 — discovery-native MQ administration

**Home:** `.github` · **Kind:** retirement · **Blocked-by:** none

- [ ] `epic-retrospective` on #165 (spec+plan freshly written, #239; NOT-STARTED —
      record it as a ready-to-go re-entry point); close children; close #245 child
      by comment.

**Acceptance:** #165 CLOSED (parked) with a retrospective.

### Task 22: Suspend epic #191 — deep Native HA log-lifecycle introspection

**Home:** `.github` · **Kind:** retirement · **Blocked-by:** none

- [ ] `epic-retrospective` on #191: record it as the *deep* log-lifecycle
      introspection (corner-case log metrics for stability), forward-planning, not
      implemented; distinct from the shipped log-availability tier. Close children;
      close #245 child by comment.

**Acceptance:** #191 CLOSED (parked) with a retrospective.

---

## Phase 5 — Archive repos

### Task 9: Archive `mq-protocol-gateway`

**Home:** `.github` (op tracked as #245 child) · **Kind:** operational (human-gated)
· **Blocked-by:** T3

- [ ] Confirm T3 lifted keepable material and #268's retro is merged.
- [ ] **Human-gated:** a human archives the repo (read-only) in GitHub settings.
- [ ] Close #245 child by comment noting the archived repo URL.

**Acceptance:** repo archived read-only; nothing deleted.

### Task 10: Archive `mq-resiliency-observability` + close umbrella #128

**Home:** `.github` · **Kind:** operational (human-gated) · **Blocked-by:** T4

- [ ] Confirm #79 retired (T4) and observability stays in the lab.
- [ ] **Human-gated:** a human archives the repo (read-only).
- [ ] Close ad-hoc umbrella **#128** with a "repo archived — see #245" note.
- [ ] Close #245 child by comment.

**Acceptance:** obs repo archived; #128 closed; observability continues in the lab.

---

## Phase 6 — Docs sweep + org narrative (bookend #1149)

### Task 23: Documentation-review sweep (already seeded as #1149)

**Home:** `mq-resiliency-lab-for-linux` · **Kind:** doc-review sweep ·
**Blocked-by:** T7, T8, T9, T10, T11, T12, T13, T14, T15

- [ ] Sweep every repo's README/status to state its **real** final state: lab =
      v1 public product; SWIFT lab = published POC; obs + protocol-gw = archived;
      `.github`/`docs` = live.
- [ ] Update the `docs` repo org narrative + roadmap to reflect the new direction
      (spawn a per-repo doc task in `docs` — placement law).
- [ ] Spawn per-repo doc tasks wherever docs need changes outside the lab repo,
      each closed by a same-repo PR.
- [ ] Validate; PRs; `report-ready`.

**Acceptance:** no README/narrative implies in-flight work except the live-repo
umbrellas (#29/#60/#206); each repo's status is honest.

---

## Phase 7 — Close-out

### Task 24: Validation — outsider cold-run of the public lab (already seeded as #1150)

**Home:** `mq-resiliency-lab-for-linux` · **Kind:** validation · **Blocked-by:**
T8, T11, T12, T13, T14, T15

- [ ] Precondition self-check: lab is public and site docs (T12) are merged.
- [ ] As the pinned persona — IBM MQ Developer Edition entitlement + documented
      host minimums — from a **cold clone**, following **only public site docs**,
      with **no author access**: bring the lab up, load the dashboards, confirm real
      data.
- [ ] Record `Outcome: SUCCESS` (or FAILURE with specifics → stays open) as a
      comment.

**Acceptance:** SUCCESS recorded; the product is demonstrably runnable by a stranger.

### Task 25: Retrospective — portfolio-wind-down (terminal, already seeded as #247)

**Home:** `.github` · **Kind:** retrospective · **Blocked-by:** every other #245
child closed

- [ ] Run `epic-retrospective` on #245. Preflight refuses until #245 is the only
      open child. Author `epics/245-portfolio-wind-down/retrospective.md`: what
      shipped (public lab, SWIFT POC), what was retired and why, what was parked as
      re-entry points (§5 forward axis — the suspended epics), and the outcome of
      the archival decisions.
- [ ] Docs PR; `report-ready`. Its merge closes #245.

**Acceptance:** retrospective merged; #245 closed; wind-down complete.

---

## Self-review

**Spec coverage:**

- §3 roster → T6/T8 (SWIFT rename+public), T9 (protocol-gw archive), T10 (obs
  archive), ship phase (lab), T23 (docs/.github status). ✓
- §4.1 ship → T11 (#8+#169 dashboards incl. events/log entries), T12 (#161),
  T13 (#236), T14 (#104), umbrellas kept open (T10 closes only #128). ✓
- §4.2 suspend → T16–T22 (#7/#9/#12/#17/#19/#165/#191). ✓
- §4.3 drop → T1–T5 (#59/#228/#268/#79/#66). ✓
- §5.4 retirement mechanics → Task model + every retirement task closes the #245
  child by comment, retro lands in target home. ✓
- §6 sequencing → phases + dependency graph match (drops → SWIFT → ship →
  suspend → archive obs → docs sweep → validation → retro). ✓
- §7 acceptance → T8/T24 (public + cold-run persona), T9/T10 (archives), T1–T5 &
  T16–T22 (retros), T23 (honest status), T25 (#245 retro). ✓
- §2 "honest dashboards / no silent empty panels" → T11 explicit. ✓

**Placeholder scan:** no TBD/TODO; each task names its home, kind, blocked-by,
deliverable, and acceptance. Dashboard/site-doc file paths are indicative because
those epics carry their own detailed plans (#161, #236) or need a fresh build
(#169) — the concrete scope is stated inline. ✓

**Consistency:** task numbering is grouped by phase (T1–T5 drops, T6–T8 SWIFT,
T9–T10 archive, T11–T15 ship, T16–T22 suspend, T23 sweep, T24 validation, T25
retro); the dependency graph and the per-task Blocked-by agree. Retirement vs
coordination vs code/operational kinds are used consistently per the Task model. ✓

**Resolved in alignment (2026-09-20):**

- Finite ship epics **#8, #169, #161, #236** each get finished *and* retro'd via
  their own `epic-retrospective` (T11–T13) — they are delivered product work and
  deserve a record. **#104** is a standing ad-hoc umbrella: T14 does the
  reliability pass but leaves #104 open (like #29/#60/#206), no retro/close.
- Spec §4.1's carve-out of **#119** (advanced CLI drill runner) is executed in
  T11: closed won't-do/parked before #8's retrospective.
