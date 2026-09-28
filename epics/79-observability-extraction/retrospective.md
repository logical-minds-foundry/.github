# Extract mq-resiliency-observability — retrospective

- **Epic:** `logical-minds-foundry/.github#79`
- **Retrospective task:** `logical-minds-foundry/.github#273`
- **Retired under:** MQ portfolio wind-down — `logical-minds-foundry/.github#245`
  (task #253)
- **Outcome:** DROPPED before productization — the built component is preserved
  (repo archived), observability stays in the lab.
- **Date:** 2026-09-28

## §0 At a glance

We set out to take the lab's ad-hoc, non-MQI observability code and extract it into
a **first-class, standalone, installable product** (`mq-resiliency-observability`,
packaged `.rpm`/`.deb`) — the first replay of a "harden-and-extract" pattern for
the component-extraction roadmap (`mq-resiliency-lab-for-linux#368`), after which
the lab would dogfood the published artifact. Unlike the other drops, **a large
part of this was actually built.** What did *not* happen is the productization
tail — packaging, CI-green, publish 0.1.0, and dogfood. Under the strategic
refocus we drop that tail: observability **stays in the lab**, the extracted
component is preserved but not published, and its repo is archived.

**Delivered (merged) — the extraction itself:**

| Area | Work landed |
|---|---|
| Repo + design | Repo bootstrap (#82), spec + plan (#80), packaging sub-brainstorm (#84), CLI/filesystem-state discovery (#646) |
| Collectors | Collector base + Native HA collector (obs#2), RDQM collector (obs#17), Pacemaker/DRBD collector (obs#16), shared DRBD parser (obs#15) |
| Framework | Detection framework — precedence + exclusion (obs#3), metrics-contract module + consistency test (obs#5) |
| Dashboards | render-dashboards generator + portable boards (obs#18), Native HA + RDQM cockpit boards (obs#22) |

**Dropped (closed won't-do) — the productization tail:**

- `mq-resiliency-observability`: nfpm packaging (obs#1), package the Native HA
  collector (obs#4), CI green (obs#6), publish 0.1.0 (obs#7)
- `mq-resiliency-lab-for-linux`: docs-review sweep (#643), .rpm/.deb install
  validations (#644/#645), full-lab cold-rebuild gate (#647), dogfood (#652),
  **delete the in-lab collector copy (#653)** — deliberately *not* done
- `.github`: follow-on brainstorm (#81)

- **Repos touched:** `mq-resiliency-observability` (the build),
  `mq-resiliency-lab-for-linux` (integration tasks), `.github` (epic docs).
- **Releases:** none (0.1.0 never published).
- **Span:** opened 2026-07-15, retired 2026-09-28.

## §1 How the plan evolved

The plan ran a long way before it stopped. The extraction proper — collectors for
all three HA/DR mechanisms, the detection framework, the metrics contract, and the
dashboard generator — landed as merged, tested code in the standalone repo. The
work halted at the **packaging/publish boundary**: nfpm packaging (obs#1/#4), CI
(obs#6), and the human-gated 0.1.0 release (obs#7) were the next steps and were
never taken. The refocus behind wind-down #245 then converted "not yet published"
into "won't publish": there is no longer a reason to stand up and maintain a
separately-installable observability product when the lab that consumes it is the
one deliverable we are finishing.

## §2 Lessons learned

- **The extract-and-harden pattern worked technically** — ad-hoc lab code did
  reach a clean, tested, standalone shape. The idea is sound; it was the
  *productization overhead* (packaging, release, dogfool loop) that proved to be
  the expensive, and ultimately unjustified, part for a single component.
- **Publish-and-dogfood is a real cost, not a formality.** Most of the *remaining*
  effort at drop time was packaging/release/validation, not code. Extracting code
  is cheap next to shipping and supporting it as a product — worth weighing before
  committing to extract future components.

## §3 Compromises & tradeoffs

- **No technical debt in the lab.** Because the "delete the in-lab collector copy"
  task (#653) was deliberately closed won't-do, the lab keeps its own working
  observability. Dropping the extraction costs the lab nothing operationally.
- **Sunk effort, honestly.** Real engineering went into a standalone component that
  will not be published. That work is preserved (archived repo), not deleted, so it
  is revivable — but as things stand it is shelved, and that is the trade we are
  accepting for a tighter wind-down.
- The **component-extraction roadmap** (`mq-resiliency-lab-for-linux#368`) that
  #79 was the first replay of is, in effect, parked with it.

## §4 New problems & opportunities

- The extracted, tested collectors (Native HA / RDQM / Pacemaker-DRBD), the DRBD
  parser, the detection framework, the metrics contract, and the dashboard
  generator all survive in the archived `mq-resiliency-observability` repo — a
  ready-made starting point if a standalone observability product is ever wanted.
- The metrics-contract + consistency-test approach (obs#5) is a reusable idea worth
  remembering independent of this component.

## §5 What's next

- Dropped under wind-down #245; **no follow-on is scheduled.**
- The built component is preserved in `mq-resiliency-observability`, which is
  **archived** read-only under wind-down task #262 (and its ad-hoc umbrella #128
  closed at the same time). Observability continues to live in
  `mq-resiliency-lab-for-linux`.
