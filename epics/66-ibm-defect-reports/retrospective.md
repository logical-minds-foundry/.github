# IBM defect reports — collect & cache — retrospective

- **Epic:** `logical-minds-foundry/.github#66`
- **Retrospective task:** `logical-minds-foundry/.github#271`
- **Retired under:** MQ portfolio wind-down — `logical-minds-foundry/.github#245`
  (task #254)
- **Outcome:** DROPPED — no filing path post-engagement; dossiers preserved.
- **Date:** 2026-09-28

## §0 At a glance

This was a **standing collection epic — a cache, not a build.** As the labs ran,
we kept hitting defects that looked like IBM MQ / RDQM product bugs rather than
lab bugs, but had no way to file them (no IBM defect-reporting access). Rather
than lose the evidence, each reportable defect was captured as a self-contained,
reproducible dossier, parked until reporting access arrived. With the strategic
refocus, that access is not coming, so the epic is dropped — but the dossiers are
kept.

**Dossiers collected (both preserved as closed issues in kept repos):**

| Dossier | Where it lives now |
|---|---|
| RDQM cold coordinated-create leaves site-B HA group in DRBD split-brain (never UpToDate; DR never establishes) | `logical-minds-foundry/.github#67` (closed, preserved) |
| Manual (no-SSH) DR/HA RDQM create fails to reach UpToDate | `logical-minds-foundry/mq-resiliency-lab-for-linux#563` (closed, preserved) |

- **Repos touched:** none for code — a cache of diagnosed defects only.
- **Implementation:** n/a (collection epic).
- **Span:** opened 2026-07-12, retired 2026-09-28.

## §1 How the plan evolved

There was no plan to execute — the epic's whole shape was "accumulate dossiers
until we can file them." Two dossiers accrued, both on the RDQM / DRBD vendor path
(the RHEL platform, not the kept Ubuntu product). The blocking precondition
(IBM defect-reporting access) never materialised, and the refocus removed the
reason to keep waiting for it, so the epic is closed rather than left standing.

## §2 Lessons learned

- **Capturing suspected product bugs as reproducible dossiers at the moment you
  hit them is cheap and worth doing** — the diagnosis is freshest then, and the
  evidence survives even when the filing path never opens.
- **A "cache" epic with an external precondition should have an explicit expiry
  check.** This one sat open on a precondition (IBM access) that was always
  uncertain; a periodic "is this still worth holding?" would have surfaced the
  drop decision sooner.

## §3 Compromises & tradeoffs

The defects go **unfiled** — that is the accepted outcome, not hidden debt. Both
are on the RDQM/DRBD vendor platform, which is not the kept Ubuntu product, so the
unfiled reports do not affect what ships. The dossiers remain available verbatim
(closed issues #67 and #563) should a filing path ever open.

## §4 New problems & opportunities

- The two dossiers are genuine, reproducible RDQM/DRBD failure modes; if IBM
  defect-reporting access is ever obtained, they are filable more-or-less
  verbatim from the preserved issues — no re-diagnosis needed.

## §5 What's next

- Dropped under wind-down #245; **no follow-on is scheduled.**
- Evidence preserved as closed issues #67 (`.github`) and #563
  (`mq-resiliency-lab-for-linux`) — both in kept, un-archived repos.
