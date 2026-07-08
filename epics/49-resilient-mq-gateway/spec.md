# Resilient MQ message gateway — replay/reconciliation design study & lab demonstration — design spec

- **Epic:** `logical-minds-foundry/.github#49`
- **Design task:** `logical-minds-foundry/.github#50`
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-07-08

> **Anonymization notice.** This document describes a **generic trading
> back-office message-gateway problem**. All identifying detail has been
> removed and must stay removed: no employer name, no named individuals, no
> internal host/queue labels, no counterparty identities. The requesting
> organization is *"the application team"*; the two remote log machines they
> sketched are *"the alternate log hosts"*; counterparties are *"Counterparty
> A / B / …"*. Generic product/technology terms (MQ, RDQM, streaming queues,
> "the gateway") are fine.

---

## 1. Problem & motivation

An application team plans to insert a **message gateway** between their
applications and IBM MQ. The shape they have described:

- Applications do **not** link the MQ libraries. They speak a **proprietary TCP
  protocol** to the gateway. The gateway — a **C++** process — is the *only*
  component that links the MQ client and holds MQI connections. That, plus being
  a controlled **choke point** (one place for connections, credentials, and the
  MQ network boundary), is the gateway's reason to exist. It is meant to be a
  **reusable concept**: many instances/configurations fronting different apps,
  counterparties, and queue managers.
- **Outbound:** app → gateway → *journal the message* → put to MQ → counterparty.
- **Inbound:** counterparty response → MQ → gateway gets it **under syncpoint** →
  journals it → (possibly forwards to the app — see the open questions).
- **Journal + replay.** The gateway must keep a durable copy of every message
  sent, so that any message they have **no confirmation for** can be identified
  and **replayed/resent**. Explicitly *not* "replay the whole day" — only the
  unconfirmed. Confirmations arrive **asynchronously** as business-level
  response messages; this is **not** synchronous request-reply, it is two
  decoupled streams.

The team's proposed resiliency mechanism is deliberately **not** highly
available: a single process on a standalone host, its logs shipped
**asynchronously** to two alternate hosts (they were explicit: **no quorum**,
"just get the data over there"), and a **human** restarts the process on a host
that has the logs if it dies. They framed this as **"fail-fast / recover-fast
over automated HA clustering,"** accepted **duplicate-on-recovery** as tolerable
for rare outages, and delegated de-duplication to the counterparties.

**Why this is worth a design study.** The proposed design quietly signs up for a
hard distributed-systems problem — atomic dual-write to a log *and* MQ,
consistency across replicated logs, bounded-RPO recovery, idempotent replay —
and answers it with async log copies plus a human. The one question that decides
the whole thing — *how, exactly, are the writes to the alternate hosts done?* —
was never asked in the room; "logging" was left generic, and that is precisely
where the difficulty hides. The purpose of this epic is to be **politely but
brutally honest** about the failure modes, to offer a **ladder of design
options** from their design up to a properly resilient one, and — because this
is what the lab is *for* — to **prove the claims empirically** against a live
3+3 HA/DR topology rather than assert them on a whiteboard.

## 2. Doctrine & principles

- **This deliverable's payload is questions, not a verdict.** Phase 1 is a
  design study whose job is to extract **invariants and requirements** and hand
  back a sharp list of **questions for the application team**. The design ladder
  is presented as **options to iterate**, not a locked recommendation. The
  meeting was one fast, ad-hoc hour; several requirements below are *inferred* or
  *guessed* and are tagged as such — each gap is a question, not an assumption.
- **Augment, don't replace.** The least well-received proposal would be "throw
  your gateway away and put it all in this infrastructure." Every option is
  framed as making *their* gateway simpler and safer, not replacing their
  approach. The team leans **do-it-yourself / anti-vendor**, prefers state "on
  the inside," and does **not** fully trust MQ — so an MQ-leaning design starts
  suspect and must earn its place on merit and honesty.
- **Independent parasite of the lab.** This experiment is a **self-contained
  repo**. "Use the lab" means **use the running queue-manager infrastructure**,
  never the lab's tooling — `mqlab` and the HA/DR `validate` framework stay the
  lab's. The experiment adds **only its own** queues to the existing queue
  manager via reproducible Ansible, touches nothing the lab owns, and ships its
  own small CLI and responder app. The lab need not know it exists (see §12/§13;
  consistent with the repo's OSS component-boundary principle).
- **MQ-native and minimalist.** Research and design stay **inside the MQ
  product**: creative, minimal, product-supported mechanisms. No new vendor
  dependencies, no bolt-on products, no "magic" channel exits unless proven
  necessary. Other-product paths (FIX engines, Chronicle/Aeron, MQ↔Kafka
  bridges, commercial capture tools) appear only as a brief *"adjacent paths, if
  already operated"* footnote — no endorsement.
- **Glass-box, human-operable.** Prefer mechanisms an operator can see and reason
  about (explicit app-level puts, browsable archive queues) over opaque ones.
- **No silent loss / no silent failures.** The design's central correctness
  goal (I1 below). Note the lab's standing rule that **browse-mode is not for
  draining** a live queue (it accumulates, it does not consume) — which is
  exactly why reconciliation here browses **archive copies**, never the live
  transmit/reply queues.
- **Honest against ourselves.** Where our own preferred option introduces a cost
  (e.g. data-at-rest, §10), we name it plainly. One-sided advocacy would forfeit
  the credibility the teardown depends on.

## 3. What "replay" means here, and the two invariants

"Replay" here is **not** archival re-processing of a day's traffic. It is:
*identify every message we sent for which no confirmation was received, and
resend exactly those.* The concrete fear, in their terms: an application puts a
message, receives a **success return code and a message ID from MQ**, and logs
it as "sent" — then the queue manager fails and a DR failover occurs. **Was that
message actually transmitted to the counterparty, or not?** Absent a
confirmation, it is a genuine mystery. Resolving that mystery *is* the
reconciliation problem, and it is the legitimate core requirement.

Two invariants follow, and the design lives or dies by them:

- **I1 — No silent loss.** Every message the gateway accepts must end in exactly
  one of two states: **confirmed**, or **durably known to be sent-but-unconfirmed
  (hence replayable)** — and *that fact must survive loss of the gateway host.*
  The fatal third state is **"sent, but no surviving record."** That state is
  not a duplicate; it is an **invisible dropped message**, and it defeats
  replay precisely when replay is needed, because the record of the unconfirmed
  send is the very thing that was lost.
- **I2 — Idempotent replay.** "Resend the unconfirmed" only helps if a replayed
  message is **not processed twice** by the counterparty. This requires a
  **stable correlation key** and an **explicit owner** for de-duplication
  (the counterparty, or the gateway). Without it, replay merely trades loss for
  duplication — and in a trading context a duplicate can be as damaging as a
  drop.

## 4. Requirements ledger

Each requirement is tagged **[confirmed]** (stated clearly by the team),
**[inferred]** (strongly implied), or **[guessed]** (a placeholder to confirm).
Every non-confirmed item has a matching entry in §11.

| # | Requirement | Tag |
|---|---|---|
| **R1** | Only the gateway links the MQ libraries; apps use a proprietary TCP protocol. The gateway is C++. | confirmed |
| **R2** | The gateway is a reusable, parameterized choke point — many instances/configs across apps, counterparties, and queue managers. | confirmed |
| **R3** | Durable journal of every message sent to a counterparty (message data + a correlation key). | confirmed |
| **R4** | Replay = resend only the **unconfirmed**, not the whole day. Requires correlating outbound sends with inbound confirmations. | confirmed |
| **R5** | The team **rejected** "put the state in a database" and "make the whole thing HA so state moves with the process." (Recorded as pre-rejected; still costed in §7.) | confirmed |
| **R6** | Their durability mechanism: single writer → **async** copies to two **alternate log hosts** (no quorum) → **manual** restart on failure. | confirmed |
| **R7** | Not request-reply: two **decoupled** streams; confirmations arrive asynchronously and are matched by **application-level** data. | confirmed |
| **R8** | A confirmation is a **business-level response message**, matched on app data — not the MQ message ID, not any gateway-invented serial. | inferred |
| **R9** | The "internal gateway serial number" is an **implementation artifact**, not a requirement. We are free to choose the correlation key and the store. | confirmed |
| **R10** | Scale: **low volume, bursty, latency-tolerant**; not high-frequency; plausibly **+1–2 orders of magnitude** later. | confirmed (orders of magnitude only) |
| **R11** | **Data-at-rest:** retaining a business day of messages reclassifies data from in-transit to at-rest and may trigger on-disk encryption / retention requirements. | inferred |
| **R12** | Operating model: archive holds **one business day**, cleared at start-of-day after reconciliation; replay horizon is **intraday**. | inferred |

## 5. Rung 0 — the team's design (steelman, then stress-test)

**As they mean it (steelmanned).** A single gateway process on a standalone
host. MQ operations under syncpoint. On restart on the **same** host, it replays
its local log. Its log is shipped asynchronously to two alternate log hosts. On
host loss, a human restarts it on a host that has the logs. Duplicate-on-recovery
is accepted as rare; counterparties are expected to log or reject duplicates.

**Fault suite (what we will actually run in the lab):**

| Scenario | Outcome | Verdict |
|---|---|---|
| Same-host restart | Replays local log | ✅ works |
| Host loss, log already shipped | Re-sends → **duplicates** | ⚠️ their known, accepted case |
| **Host loss, log not yet shipped** | Un-shipped sends are **invisible** on recovery | ❌ **silent loss (I1 violation)** — the case not on their risk list |
| Log divergence across hosts | No defined authoritative log — "which host?" | ❌ recovery cannot be correct |
| Counterparty without dedup | Duplicate processed | ❌ I2 violation |
| Site loss | Ops coordination, unscoped | ❌ not designed |

**The sharpened critique.** The team correctly identified the *tolerable*
failure (duplicates) and missed the *intolerable* one (**silent loss**). Because
copies are async with **no quorum**, a host can die holding log entries that
never reached an alternate host; on recovery those sends cannot be seen, so
replay-the-unconfirmed cannot fire for exactly the messages it exists to
protect. Removing quorum did not remove complexity — it **removed the answer** to
the question recovery must ask ("which log is authoritative?"). And their own
scale numbers (§R10) **neutralize the only reason** to accept this: the work is
low-volume and latency-tolerant, so synchronous replication is affordable — the
"we can't take the latency" objection does not apply here. "Recover-fast, no
quorum" buys nothing while costing correctness.

## 6. Rung 1 — recover-fast, done correctly (minimal, MQ-native)

Keeps the team's stance (standalone process, no cluster manager, manual restart
acceptable) but fixes the algorithm so I1/I2 hold — using nothing but MQ.

- **Outbound — atomic dual-write via two puts in one syncpoint.** The gateway
  opens the **remote/transmission queue** *and* an **archive queue**, PUTs to
  both in a single unit of work (`MQPMO_SYNCPOINT`), and commits together. Every
  message in the archive was atomically written to the transmit path. The
  invariant falls out for free: **once a message is no longer on the transmission
  queue, the sender channel has handed it onward.** No bespoke log, no
  log-shipping, no exit.
  - *Three honest levels of "sent":* on the transmission queue = *queued for the
    channel*; gone from it = *transmitted to the counterparty queue manager*; a
    matching inbound confirmation = *processed end-to-end*. Only the third is
    proof of processing; the design keeps all three visible.
- **Inbound archive — the same trick, mirrored, still no exit.** The gateway is
  already doing a destructive GET under syncpoint on the reply queue; it adds a
  PUT of a copy to the **inbound-archive queue** in the **same unit of work**,
  then commits. The inbound archive thereby becomes the authoritative
  **"processed" ledger**: anything in it was definitively taken off the reply
  queue and processed; anything still on the live reply queue has not been.
  (A **channel exit** would only be needed for *transparent* cloning without the
  app doing the copy — and MQ's native **streaming queues** already provide that
  transparent copy if hand-coding is undesirable; see §8.)
- **Reconciliation — browse the archives, match on correlation.** Assuming the
  reply's `CorrelId` is the original request's `MsgId` (standard MQ request-reply
  correlation — **flagged for verification**, §11), reconciliation is a
  header comparison of the outbound and inbound archive queues: browse both, and
  the outbound entries with no matching inbound entry **are** the unconfirmed
  replay set. Deterministic, transactional, and runnable **at any moment**, not
  just end-of-day. *If `CorrelId`≠`MsgId`, reconciliation falls back to parsing
  the application body — which costs the header-only cheapness of one dashboard
  calculation, not correctness; and since the observability view must show
  **which** messages are unconfirmed (not just how many), it reads the body
  anyway. So this is a **cost** caveat, not a showstopper.*
- **The inbound *delivery-to-app* handoff is the genuinely hard part** — and it
  is independent of archiving. Forwarding the message body to the app over TCP is
  a **non-transactional side effect**: commit-then-crash-before-forward loses it
  to the app; forward-then-crash-before-commit redelivers it (duplicate to the
  app). Whether the gateway even *owns* guaranteed delivery to the app, or only
  the archive, is an open question (§11). This is what was hand-waved in the room.
- **Where state and durability actually live (correcting an easy overreach).**
  The gateway runs *outside* the queue manager, so it keeps its **own** state —
  notably the outgoing replay requirement (R3/R4) and the "did I put that message
  before I crashed?" question. This MQ-native design does **not** make the
  gateway stateless, and claiming so would be wrong. What it provides is a
  **near-free second layer of defense** — the "two levels" the team wants, driven
  by their real fear: *what if the queue manager goes away and we don't know?*
  Two clarifications matter:
  - **Atomicity is not durability.** The two-queue syncpoint put gives
    *atomicity* (both puts commit or neither), but both queues live on the
    **same** queue manager — so the archive's durability across a *queue-manager*
    loss is only as good as **that queue manager's** HA/DR. **I1 holds for this
    design iff the archive's queue manager is HA/DR.** In this architecture it is
    (3+3), so the archive inherits seconds-RTO, zero-loss durability for free —
    which is exactly why leaning on the queue manager beats shipping logs to
    **non-HA** alternate hosts: your archive is only as resilient as the tier it
    sits on, so put it on the resilient one you already run. On a single, non-HA
    queue manager this design would be *worse* than theirs (one copy vs. three).
  - **It reduces, it does not obviate.** Because the gateway keeps its own replay
    state regardless, this layer lets the team *rethink how much replay
    complexity to build into the gateway* — but gateway-level replay stays a
    requirement. Complementary, not a substitute. The cost is small and concrete:
    a simple change to the gateway's MQ API semantics (the atomic two-queue put)
    plus start-of-day / end-of-day archive clearing (standard batch; the app can
    do it dynamically).

**The known tension, named honestly.** "Using a queue as a database" is normally
an anti-pattern. At **this** volume, latency tolerance, and an **intraday**
retention horizon (§R12), it is arguably one of the more elegant, lowest-cost
options — and we will say exactly that, with the caveat, rather than pretend the
tension does not exist.

## 7. Rung 2 — fail-fast, done right (the HA rung; pre-rejected, costed anyway)

The option they ruled out, shown so the trade-off is explicit. Durable state
lives on the **same resilient substrate the queue manager already runs on**:

- **The journal *is* MQ** — persistent queues plus a **streaming-queue** copy as
  the replay journal — so the "log" inherits the queue manager's HA/DR for free
  and there is near-zero bespoke storage; **or**
- a gateway write-ahead store on **RDQM/DRBD-replicated** storage with
  **automated** failover; **and**
- **active/active** gateways *iff* ordering turns out to be order-independent
  (the fork in §11).

Result: bounded seconds-scale RTO, zero-loss RPO, no human in the critical path.
Punchline in the team's own terms: *you already operate this exact reliability
for the queue manager this gateway front-ends; Rung 2 extends it rather than
hand-building a weaker parallel copy — and by your own "minimize complexity,
avoid magic" rule, reusing MQ's durability is the smaller move, not the
log-shipping tier.*

## 8. Cross-cutting research (MQ-native only)

Scope is deliberately confined to **mechanisms inside the MQ product**:

- **Two-queue syncpoint put** (the outbound atomic dual-write, §6).
- **Streaming queues** — the queue manager puts a near-identical copy of every
  message onto a secondary queue; `STRMQOS(MUSTDUP)` makes the copy part of the
  unit of work (no copy → the put fails). The zero-app-code path to an archive. *(Verify before building: whether a
  streaming-queue copy can be taken off a **transmission** queue — flag it like
  the correlation-key assumption; do not build on it unpromised.)*
- **COA / COD report messages** — confirmation-on-arrival / on-delivery, as an
  MQ-level (not business-level) confirmation signal; understand where they help
  and where they do not (they prove MQ delivery, not counterparty processing).
- **Persistent messaging + linear logging** — baseline durability semantics.
- **Reconciliation-by-browse** on archive queues (`MQGMO_BROWSE_*`).
- **Native de-duplication feasibility without exits** (their open question) —
  honestly assess whether MQ can dedup on message/correlation ID, and whether it
  needs an exit (which they want to avoid) or belongs app-side.
- **Adjacent paths, only if already operated (footnote, no endorsement):** if
  Kafka is already strategic, MQ→Kafka source connectors exist; FIX engines
  solve the sequence-store-failover problem in the trading domain; Chronicle
  Queue / Aeron Archive are persisted replay logs. Named for completeness;
  **not** recommended, since introducing an unfamiliar product would be poorly
  received and neither the author nor this study has operated them.

## 9. The observability differentiator

This is expected to be the argument that **sells** the MQ-native option, and it
is the lab's home turf. Because reconciliation and monitoring run against the
**archive** copies and **never** the live queues, the live payload path is
untouched — no browse contention, no in-flight message locking on the hot path.
Browsing the archives every few minutes yields an **authoritative, MQ-level view
of sent-vs-confirmed across the day**, with the unconfirmed set surfaced
continuously. That directly answers the team's own open question — *what
observability is needed to detect/resolve in-flight messages during recovery?* —
with a running artifact rather than a paragraph, and it showcases the lab's
Grafana tier. A mockup of this daily "sent vs. confirmed / unconfirmed set" view
is a Phase-1 deliverable.

## 10. Security & data classification

A messaging path in flight is *data in transit*. The moment a business day of
messages is **retained** — in archive queues (this study's option) **or** in the
replicated logs on the alternate hosts (their design) — those messages become
**data at rest**, which likely pulls in **on-disk encryption and
retention-compliance** requirements a pure transit component could sidestep. We
name this against **both** designs, not just theirs. The honest upside of the
MQ-managed option: the data-at-rest answer is then a **known, product-supported**
one (queue-file/disk encryption of the queue manager's storage, TLS already
covering in-transit, message-level encryption-at-rest via Advanced Message
Security if ever required — an interceptor-based capability, not free, but a
known product feature rather than a bespoke scheme bolted onto hand-rolled log
files. Options
are presented factually, inside the MQ-native boundary — no over-proposing.

## 11. Open questions for the application team (the deliverable's payload)

**Inbound path (the vaguest area):**
1. When a confirmation lands on the reply queue and is logged — **then what**?
   Stored only, or forwarded on?
2. **How does an app receive an inbound message through the gateway** — push,
   poll, or a connection to a waiting daemon? Who owns the socket lifecycle?
3. If forwarded: get-loop → log → forward → commit — **in what order**? (The
   commit-ordering trap of §6.)
4. Predetermined destination per stream, or dynamic delivery (which instance,
   discovered how)?
5. Does the gateway own **guaranteed delivery to the app**, or only the archive?

**Architecture-forking:**
6. **Ordering** — strict FIFO per counterparty, or independent self-contained
   trades? *Provisionally FIFO, unconfirmed.* This forks the resilient design:
   FIFO → a single failover-able writer (the FIX-shaped problem); independent →
   active/active becomes viable and much cheaper.

**Correctness-critical:**
7. **Confirmation semantics** — is the reply's `CorrelId` the original request's
   `MsgId`? What exactly counts as a confirmation (business response vs. MQ
   COA/COD)? *This sets the **cost** of reconciliation (header-only vs. body-parse), not its feasibility — see §6.*
8. **Replication mechanism** — how, exactly, are the writes to the alternate log
   hosts done? (Never specified; the hidden complexity.)

**Scoping:**
9. Mean message size and size distribution? (Decides whether "queue-as-database"
   is elegant or abusive.)
10. Replay-delay tolerance (RTO)? Retention / regulatory horizon? DR / site-loss
    scope?
11. On-disk encryption requirements once a business day is retained (§10)?
12. Per-counterparty duplicate handling — who dedups, and how (I2)? Is
    MQ-native dedup without exits acceptable?
13. Is delegating durability to MQ categorically off the table ("on the inside"),
    or negotiable if shown safe?

## 12. Lab demonstration / proof plan

The empirical half of the epic. It is delivered by an **independent repo** that
**bolts onto the queue-manager infrastructure the lab already runs** — it does
not reuse the lab's tooling (see §2, the parasite model), and it is
**arm-agnostic**: it attaches to whichever running queue manager and counterparty
the lab exposes.

- **Setup (reproducible Ansible, parasite-style):** add the experiment's **own**
  round-trip queues (new names) to the existing queue manager and counterparty
  app. Nothing the lab owns is modified. Tear the lab down and the setup is lost
  — re-applied from Ansible; that trade-off is accepted.
- **Its own small CLI** drives the demo: **start-of-day** (clear the archive
  queues) → **run the simulation in batches** → **end-of-day** (clear the queues,
  shown live clearing on the dashboard). This is *not* `mqlab` and *not* the
  HA/DR `validate` framework — both stay the lab's.
- **A small responder / dummy app** plays the client, consuming replies — and can
  **deliberately drop replies** at a controlled rate (e.g. toward end of day) to
  inject **missing confirmations**, manufacturing exactly the condition the
  observability view exists to surface.
- **What the demo shows:** batches flowing; the responder's drop-rate opening a
  visible gap; the §9 dashboard surfacing the growing **unconfirmed set**
  (sent-vs-confirmed) live; reconciliation-by-browse run against the **archives**
  with zero impact on the live path; and the SOD/EOD clearing demonstrated on the
  board. The "quantification" is the **visible gap**, produced by a known
  drop-rate — a live demonstration, not a pass/fail assertion.
- **Optional deeper arm (stretch):** stand up a minimal Rung-0-style async-log
  gateway to reproduce the **host-loss silent-loss** scenario directly and
  contrast it with the MQ-native option holding. The primary demonstration is the
  MQ-native archive + observability story above; this is additive if pursued.

## 13. Scope, phasing & the parallel scaffolding track

1. **Phase 1 (this task, #50):** the design-study spec + its plan. Payload = the
   §11 questions for the app team.
2. **Scaffolding (parallel, design-independent — start now):** stand up the
   **new, independent repo** for the gateway experiment. Its cloud sessions run
   inside the existing lab VM via the repo's **cross-VM dependency** — a proven,
   trivial configuration feature (already wired for other repos), not a risk;
   standing it up is just applying that config. The repo's setup Ansible adds
   **its own** round-trip queues to the running queue manager and counterparty
   (§12); it ships its **own** small CLI and responder app; it reuses **none** of
   the lab's tooling. This does not gate on the design being nailed.
3. **Lab demonstration:** §12.
4. **Iterate** the design ladder with app-team feedback; expect their §11 answers
   to reshape the options — this is a conversation, not a one-shot verdict.

Out of scope for Phase 1: choosing a final rung (the app team's call, informed by
the demonstration). **Verify-before-build items** to resolve early (neither
blocks, but do not build on either unpromised): the `CorrelId`=`MsgId`
correlation assumption (§6/§11), and whether a streaming-queue copy can be taken
off a transmission queue (§8).

## 14. Success criteria

- The app team can read the study and **see their own design fairly represented**
  (steelmanned), then see the failure modes — including silent loss — laid out
  without hand-waving.
- The **§11 questions** are sharp enough to run the next meeting from.
- The lab **demonstrates** the silent-loss scenario and the MQ-native option's
  correctness on a real 3+3, with the sent-vs-confirmed view rendered.
- Nothing in any artifact is client-identifiable (§ anonymization notice).
