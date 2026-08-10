# Parked idea — logsearch front-end & integration follow-ons

**Status: parked (captured, not scheduled).** Origin: the logsearch epic
(`#149`) follow-on brainstorm, lab issue #819, parked 2026-08-10 rather than
brainstormed deeply. v1 of the logsearch tier is live and ingesting the fleet's
logs; none of the work below is needed now. This doc captures the ideas so we can
pick them up later — each area likely spins out its own spec → plan → build epic.

## The through-line

The headline of the follow-on work is **using the OpenSearch Dashboards front-end
itself** — the analysis surface we now have but ship empty. The most valuable next
step is an **integrated operator view that brings event entries and log entries
together in dashboards**, rather than more lab plumbing. Most of what follows is
about *using* the front-end, not extending the pipeline that feeds it.

## Candidate follow-on areas (from #819)

1. **Query / investigation catalog.** Seed the empty tier with saved searches and
   filters for the recurring MQ investigation questions — channel-retry frequency,
   problem recurrence, time-of-day patterns — so the tier is useful out of the box
   instead of a blank Discover page. This is the "make v1 earn its keep" work.

2. **Integrated obs + log operator surface.** Event entries and log entries side by
   side — the combined operator view across the Grafana/obs metrics-and-events
   surface and the OpenSearch Dashboards full-text surface. The likely headline
   epic.

3. **Security-posture review.** A decision, not necessarily build work: does the
   analysis tier warrant any hardening, or is unhardened-on-the-mgmt-plane correct
   per the epic's security-boundary tenet? Resolve the question before assuming
   either answer.

4. **Retention & disk-space management.** Index retention and disk headroom for the
   growing daily `logs-YYYY.MM.DD` corpus, modeled on the lab's baked-image GC
   approach (a per-index drop is already cheap by design).

5. **Synthetic-corpus extraction.** Noted in #819; the purpose needs pinning down
   when picked up (e.g., a portable/synthetic log corpus for tests or demos that
   doesn't require a live lab).

## Disposition

Parked. Revisit when the dashboards / front-end integration work is prioritized —
start with the integrated operator surface (2) and the investigation catalog (1),
which together turn a running tier into a working tool. The tracker
`logical-minds-foundry/.github#184` stays open as the home for that future work;
the closed lab issue #819 points here.
