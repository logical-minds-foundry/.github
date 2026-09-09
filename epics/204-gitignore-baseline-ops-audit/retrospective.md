# Retrospective — centralized .gitignore baseline + self-policing ops audit (epic #204)

## §0 At a glance

We set out to apply the fleet-wide centralized-`.gitignore`-baseline + self-policing-`ops.yml` rollout — designed upstream in **vergil-project/.github#311** — to this org's three managed repos, bringing each to `vrg-github-repo-config` compliance so that baseline drift becomes a nightly red-X instead of silent rot. This was **operational rollout, not design**: no spec or plan was authored (the design, the baseline, and the audit checks all ship from `vergil-tooling` under #311). We shipped **compliance across all three managed repos** — canonical `.gitignore`, an `ops.yml` audit workflow wired to `ops-github-config.yml@v2.1` on a deterministic staggered cron, and a pinned `marketplace_ref` — plus a residual-compliance pass (CLAUDE.md template markers) that the audit surfaced only after the first rollout wave.

### Work delivered

| PR | Repo | What it did |
|---|---|---|
| `.github#208` | `.github` | gitignore — first sync to the canonical baseline |
| `.github#213` | `.github` | gitignore — re-sync to the vergil-managed block as the canonical spelling settled |
| `.github#217` | `.github` | **`ops.yml` + `marketplace_ref=main` + `ci.yml` repair** (three changes; deterministic cron `23 6`) — closed #216 |
| `#1050` | lab | gitignore — sync the vergil-managed block |
| `#1057` | lab | **`ops.yml`** self-governance workflow — closed #1037 (with #1050) |
| `#1062` | lab | **residual:** wrap CLAUDE.md canonical template in begin/end markers + add `marketplace_ref=main` — closed #1060 |
| `#32` | observability | gitignore — sync the vergil-managed block |
| `#39` | observability | **`ops.yml`** self-governance workflow — closed #27 (with #32) |

- **Repos touched:** 3 — `logical-minds-foundry/.github`, `…/mq-resiliency-lab-for-linux`, `…/mq-resiliency-observability`
- **Sub-issues:** 5 — three per-repo rollout tasks (#216 · #1037 · #27), one residual-compliance task (#1060), the retrospective (#205)
- **PRs merged:** 8
- **Releases cut:** none — integration to `develop`; `ops.yml` tracks the rolling `v2.1` tag rather than a cut release
- **Span:** opened **2026-08-13**, closing **2026-09-09** (~4 weeks). Active work was **2026-08-26 → 09-09**; the first ~2 weeks were the stated precondition — waiting on `vergil-tooling` to ship the baseline + checks under #311 and cut the `v2.1` the org's repos track.

## §1 How the plan evolved

There was no `plan.md` to evolve — by design. The epic's charter was "no new design," so the only baseline to measure against is the per-repo checklist in the epic body: for each managed repo, (1) audit and fix pre-existing non-compliance, (2) reconcile `.gitignore` to a superset of the baseline, (3) add `ops.yml`, (4) confirm a clean audit and merge — **"one same-repo PR each."**

The load-bearing deviation is that **"one same-repo PR each" held for no repo.** Every repo split into at least two PRs, and the split was consistent: the `.gitignore` reconciliation and the `ops.yml`/compliance changes are logically distinct and landed independently (gitignore across late August, `ops.yml` at end of August). `.github` itself took three passes — two gitignore syncs a day apart (#208 then #213, as the canonical block spelling settled) plus the compliance PR (#217). Only `.github` folded `ops.yml` + `marketplace_ref` + `ci.yml` into a single compliance PR; the lab and observability kept gitignore and `ops.yml` separate.

The second deviation was a **residual-compliance layer the checklist didn't anticipate.** After the lab's gitignore + `ops.yml` wave (#1037) closed, `vrg-github-repo-config audit` still reported two findings outside that task's scope: the `CLAUDE.md` canonical-template sections were present but interleaved with lab-specific content and **not wrapped in the `vergil:template:claude-md` begin/end markers** the audit requires, and the `vergil-marketplace` source was missing `ref=main`. Rather than reopen the already-landed #1037 (frozen-branch rule), this became its own task (#1060) and PR (#1062). Notably, the same `marketplace_ref=None` drift also hit `.github` (fixed inside #217), making it a fleet pattern rather than a one-repo slip.

## §2 Lessons learned

- **Repo-config compliance is multi-dimensional; "one PR per repo" under-scopes it.** A single repo's compliance spans `.gitignore`, `ops.yml`, `marketplace_ref`, and the `CLAUDE.md` template markers — and each dimension tends to want its own PR. Plan the rollout as "bring each dimension to compliance," not "one sweep per repo."
- **The audit tool is the source of truth, and it earns its keep.** Running `vrg-github-repo-config audit` per repo caught the CLAUDE.md-marker and `marketplace_ref` residuals a human reading the checklist missed. Automated compliance-checking, not a manual checklist, is what closes the gap.
- **Drift that recurs across repos is a sweep, not a repeat.** `marketplace_ref=None` appeared independently on `.github` and the lab. When the same finding shows up in the second repo, that is the signal to stop fixing it repo-by-repo and consider a one-shot fleet sweep.
- **The self-policing loop is the actual deliverable.** The rollout's value is not the eight PRs; it is that `ops.yml` now runs nightly and turns any future baseline drift into a red-X email. The PRs were the cost of admission to that standing mechanism.

## §3 Compromises & tradeoffs

- **No spec/plan, so no plan-vs-actual delta to measure.** Deliberate — the design lives upstream in vergil #311 — but it means this retrospective's only baseline is the epic body's checklist. For a pure operational-rollout arm of a cross-org design, that is the right call; it is worth naming so a later reader doesn't hunt for a missing plan.
- **Residual compliance fragmented "bring the lab to compliance" across two tasks.** Splitting #1060 out of the closed #1037 was correct per the frozen-branch rule (never reopen a landed task), but it means the lab's compliance story is spread over #1037 + #1060 / #1050 + #1057 + #1062 rather than one clean unit.
- **A duplicate abandoned worktree/branch for #1060 had to be cleaned up before submit.** A crashed/restarted agent left a byte-identical `feature/1060-repo-config-residual` branch (worktree + relay ref) alongside the finished one; confirmed identical via an empty tree-diff, then torn down. No work was lost, but it is process friction worth noting.
- **The remote half of `vrg-github-repo-config audit` returns HTTP 403 under the user identity's token** (`actions/permissions` — a token-scope limitation, not a compliance failure). Local audit is authoritative today; remote-settings compliance leans on the nightly `ops.yml` run rather than on-demand local verification.

## §4 New problems & opportunities

- **`marketplace_ref=None` drift is a fleet pattern** (seen on `.github` #216 and lab #1060). Candidate one-shot sweep to pin `ref=main` across every remaining managed repo in the fleet, rather than catching it per-repo as each is touched. *(Noted in #1060; not yet filed as its own task.)*
- **Remote-audit token-scope 403.** `vrg-github-repo-config audit` cannot read `actions/permissions` under the user identity's token, so its remote checks 403. Worth resolving so the audit can verify remote settings on demand, not only via the nightly workflow. *(Logged here; not yet acted on.)*
- **The nightly `ops.yml` audit is now live across the org.** Opportunity — and the point of the whole epic — is to confirm the loop actually fires: the first real drift should produce a GitHub-failure email, feeding the operational-discipline habit.

## §5 What's next

- **Outcomes roll up to vergil-project/.github#311's retrospective §5** — this epic is the per-org arm of that fleet-wide driver; the cross-org synthesis belongs there.
- **Candidate one-shot `marketplace_ref` sweep** across the remaining fleet repos, if the drift keeps recurring as more repos are onboarded.
- **Resolve the remote-audit token-scope 403** so `vrg-github-repo-config audit` verifies remote settings, not just local.
- **No further rollout tasks for this org** unless a new managed repo is added — at which point the same four-dimension compliance checklist (now with the CLAUDE.md-marker and `marketplace_ref` dimensions made explicit) applies to it.
