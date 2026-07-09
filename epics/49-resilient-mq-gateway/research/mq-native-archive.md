# MQ-native archive, confirmation & replay — research (IBM MQ 9.4, forward to 10)

- **Epic:** `logical-minds-foundry/.github#49` · **Task:** `#54`
- **Scope:** Is there an **IBM-supported way to do the gateway's archive /
  confirmation / replay more abstractly than the application-level two-put** — so
  the copy and the tracking live in MQ, not in bespoke app code? Primary target
  **9.4**; forward-looking to **10** (a continuous-delivery continuation of the
  same codebase — no radical changes expected, but each claim below should be
  re-confirmed against the 10 docs when relevant).
- **Method:** IBM Docs 9.4 fetched via `tools/ibm_doc_cache.py` (browser-UA
  cache under `build/refs/ibm-docs/ibm-mq/9.4.x/`). **Data** (what the docs say)
  is separated from **judgment** (our reasoning). All of it is re-verifiable on
  the live lab, which is about to be available.

> **Bottom line.** The app-level two-put was justified only by "we own it / one
> less MQ dependency." MQ has a feature **purpose-built for exactly this**:
> **streaming queues**. With `STRMQOS(MUSTDUP)` the queue manager writes an
> atomic, MsgId-preserving copy of every message to an archive queue with **zero
> application code** — strictly simpler and more abstract than the two-put. It is
> a genuine "MQ does it better" result and reframes the design: the archive can
> move out of the gateway into the queue manager.

---

## 1. Headline finding — streaming queues are the QM-native archive

**Data.** The streaming-queues feature "allows you to have a duplicate copy of
each message put to a queue, delivered to a second queue," configured per-queue
via two local/model-queue attributes ([configuring streaming queues, 9.4][sq-cfg]):

- **`STREAMQ`** — the queue that receives the duplicate copy.
- **`STRMQOS`** — quality of service for the copy:
  - **`BESTEF`** (default) — best effort; "if there is a problem delivering the
    streamed message, this does not affect delivery of the original message."
  - **`MUSTDUP`** — "the original message is **not delivered** to its queue and
    the application receives `MQCC_FAILED`" if the copy cannot be delivered.

IBM's own worked example is literally an audit archive ([sq-cfg]):

```mqsc
DEFINE QLOCAL(AUDIT.QUEUE)
ALTER  QLOCAL(PAYMENTS.QUEUE) STRMQOS(MUSTDUP) STREAMQ(AUDIT.QUEUE)
```

with the note: "It is important that every message put to the payment queue is
streamed to the audit queue, so the `MUSTDUP` quality of service is used."

**Judgment.** `MUSTDUP` **is** the queue-manager's version of our
two-put-in-one-UOW (spec §6): both the original and the archive copy succeed, or
neither does and the app retries — the same atomicity, the same I1 guarantee,
but enforced by the QM with **no application logic**. Our `gwlab.outbound.send`
and the streaming queue are two implementations of one invariant; the streaming
queue is the more abstract one.

### 1.1 The copy preserves the MsgId (so it is replay-ready)

**Data.** For a direct queue-to-queue stream, "the copy of the message delivered
to the second queue is a duplicate of the original message, **including all of
the message descriptor fields, including the message ID and correlation ID** …
so that they are easier to find and, if necessary, **replay them back into
another IBM MQ system**" (IBM MQ 9.4, "Streamed messages"). Differences: the
streamed copy's expiry is set to `MQEI_UNLIMITED` (or the secondary queue's
`CAPEXPRY`), and certain report options are not carried over.

**Judgment.** This is exactly what reconciliation needs — the archive copy
carries the **same MsgId** the counterparty echoes back as `CorrelId`. IBM
states replay as the *intended* use. This resolves, in our favour, the
MsgId-preservation concern the spec raised for the two-put (§6): the QM-native
path preserves it by design. *(To confirm on the box: the exact MQMD fields
retained on a direct stream, since this specific line came from the "Streamed
messages" topic, which the cache tool could not resolve by slug.)*

### 1.2 The restriction that shapes the topology

**Data.** Not supported ([streaming-queue restrictions, 9.4][sq-restrict]):
**"Defining `STREAMQ` on a queue configured with `USAGE(XMITQ)`."** Also:
`STREAMQ` cannot be *set on* a remote or alias queue definition (but it may
*point to* one); no chains (`Q1→Q2→Q3`) or loops; and most `SYSTEM.*` queues are
excluded.

**Judgment.** This **resolves the spec §8 verify item**: you **cannot** take a
streamed copy off a *transmission* queue. So for a message bound to a remote
counterparty (which resolves to an xmitq), you stream off the **application's
local put queue** — the copy is taken at PUT time, before transmission — not off
the xmitq. In our self-contained demo (a local `GW.CPB.SEND`), streaming applies
directly. For a real remote counterparty, the gateway PUTs to a local queue that
carries `STREAMQ→archive`, and a separate step moves the original onward.

### 1.3 Fan-out variant (pub/sub) — and why it is *not* for MsgId replay

**Data.** If more than one copy is needed, `STREAMQ` can name an **alias queue
whose target is a topic**; each PUT is then published to that topic and every
subscription gets a copy. But such messages "follow the same rules as other
publish/subscribe messages … each message has a **new message identifier** and
the context fields of the MQMD are different" ([sq-cfg]).

**Judgment.** The topic route is good for **fan-out** (N independent audit /
analytics consumers) but **breaks MsgId-keyed replay** (new MsgId per copy). For
our archive-and-reconcile use case, the **direct queue-to-queue** stream (§1.1)
is the right one; the topic route is a note for future multi-consumer needs.

---

## 2. Confirmation side — COA / COD report messages

**Data.** MQ generates **report messages** natively ([types of message,
9.4][types-msg]):

- **COA** (confirmation of arrival) — "the message has reached its target queue …
  generated by the queue manager."
- **COD** (confirmation of delivery) — "the message has been retrieved by a
  receiving application … generated by the queue manager." If the getter retrieves
  under a unit of work, the COD is generated **within that UOW** — not available
  until commit, and **not sent if the UOW is backed out** (WebSearch summary of
  the 9.4 MQMD/report material).

**Judgment.** COA/COD are a **native confirmation channel that needs no
cooperation from the counterparty application** — the gateway can request COD on
each outbound message and track the returning COD reports to know what was
delivered. But note the semantic ceiling: **COA = arrived on the queue**, **COD =
retrieved by the counterparty's app** — *neither means the counterparty
business-processed it*. A business-level reply (our current model, R8) is the
only true "processed" signal. So COA/COD are best as a **complementary, native
liveness/delivery signal**, not a replacement for business confirmations where
"processed" matters.

---

## 3. Reconciliation key — confirmed as the documented default

**Data.** For request/reply, the MQMD **report field** controls correlation: "You
can request that either the `MsgId` or the `CorrelId` of the original message is
to be copied into the `CorrelId` field of the reply message (**the default action
is to copy `MsgId`**)" ([types of message, 9.4][types-msg]).

**Judgment.** This **resolves spec §11 Q7**: our reconciliation key
(`reply.CorrelId == request.MsgId`) is not merely plausible — it is IBM's
**documented default** for request/reply. The header-only reconciliation path is
therefore the expected case, not the optimistic one. *(Still confirm the
counterparty actually follows the default; some apps override the report field.)*

---

## 4. Options matrix

| Mechanism | Archives a copy? | Atomic w/ send? | MsgId preserved? | App code? | Best for |
|---|---|---|---|---|---|
| **App-level two-put** (our C1) | yes | yes (one UOW) | yes (gateway-assigned) | **yes** | full control, glass-box, no MQ feature dependency |
| **Streaming queue `MUSTDUP`** | yes | **yes (QM-enforced)** | **yes** | **none** | the abstract, minimal archive — QM owns it |
| Streaming queue `BESTEF` | yes | no (copy may drop) | yes | none | analytics where a lost copy is tolerable |
| Streaming → topic (alias) | yes, fan-out | best-effort | **no (new MsgId)** | none | N audit/analytics subscribers |
| **COA / COD reports** | no (confirms, not copies) | n/a | n/a (report refs original) | request the option | native delivery confirmation w/o counterparty cooperation |

---

## 5. Recommendation

1. **Lead with the streaming-queue (`MUSTDUP`) archive as the primary MQ-native
   option** in the design ladder — it is the "MQ does it better, simpler, and
   more abstract from the app" answer, and IBM ships it for exactly this purpose.
   It reframes Rung 1: the outbound archive need not live in the gateway at all.
2. **Keep the app-level two-put as the glass-box alternative** — same invariant,
   fully in the app, one fewer MQ feature to depend on. The two are a clean
   "abstract vs. explicit" pair to present to the app team (honouring their
   DIY leaning while showing the product can carry the weight).
3. **Offer COA/COD as a native confirmation layer** — with the honest caveat that
   it proves MQ delivery, not business processing.
4. The reconciliation design is unchanged and now better-grounded: **MsgId ↔
   CorrelId is the documented default**, and the archive (however produced)
   preserves the MsgId.

## 6. Forward look to IBM MQ 10

MQ 10 is the continuous-delivery continuation of the 9.x codebase; streaming
queues (9.2.3+) and report messages are stable, long-standing core features, so
no breaking change is expected. **Action:** when the 10 docs are the reference,
re-confirm the streaming-queue restrictions list and the "Streamed messages"
retained-fields list — those are the two places a quiet change would matter.

## 7. Verify on the live lab (about to be available)

- `STRMQOS(MUSTDUP)` atomicity: a copy failure (e.g. full archive queue) fails
  the original PUT with `MQCC_FAILED`.
- A **direct** queue-to-queue stream preserves the MsgId (and which other MQMD
  fields); expiry is reset to unlimited.
- `STREAMQ` on a `USAGE(XMITQ)` is rejected.
- COA/COD generation and the COD-within-the-getter's-UOW timing.

## Sources

- Configuring streaming queues (9.4) — [`ibm.com/docs/…?topic=configuring-streaming-queues`][sq-cfg]
- Streaming queue restrictions (9.4) — [`…?topic=queues-streaming-queue-restrictions`][sq-restrict]
- Types of message / report messages (9.4) — [`…?topic=messages-types-message`][types-msg]
- Streaming queues overview (9.4) — [`…?topic=scenarios-streaming-queues`][sq-overview]

All cached under `build/refs/ibm-docs/ibm-mq/9.4.x/` (cite `content.txt`, `source_url` in `meta.json`).

[sq-cfg]: https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=configuring-streaming-queues
[sq-restrict]: https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=queues-streaming-queue-restrictions
[types-msg]: https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=messages-types-message
[sq-overview]: https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=scenarios-streaming-queues
