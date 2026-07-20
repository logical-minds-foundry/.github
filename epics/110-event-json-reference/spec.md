# Working with MQ instrumentation events as JSON — reference + generation design spec

- **Epic:** `logical-minds-foundry/.github#110`
- **Design task:** `logical-minds-foundry/.github#111`
- **Follow-on brainstorm task:** `logical-minds-foundry/.github#112`
- **Documentation-review task:** `logical-minds-foundry/mq-resiliency-lab-for-linux#710`
- **Builds on:** `mq-resiliency-lab-for-linux#694` (the "events to a file as JSON"
  how-to) and the `mq-event-monitoring-guide.md` site guide
- **Sibling / prior epic:** `logical-minds-foundry/.github#31` (event monitoring →
  JSON → Loki — the *produce* side)
- **Routes toward:** `logical-minds-foundry/.github#38` (live-lab validation
  framework — future mechanized event generation)
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-07-20

## 1. Problem & motivation

The lab can now **produce** a JSON instrumentation-event feed: `#694` and the
`mq-event-monitoring-guide.md` site guide both cover enabling event classes and
running `amqsevt -o json` as a managed collector. What neither covers — and what
anyone doing serious work with the feed hits immediately — is **how to work with
the data once it is flowing**.

The obstacle is that **IBM publishes no formal JSON schema for `amqsevt` output.**
From the research done for `#694`:

- The five-key **envelope** — `eventSource`, `eventType`, `eventReason`,
  `eventCreation`, `eventData` — is a convention of the `amqsevt` sample
  (`amqsevta.c`), shown by IBM only as one illustrative example, never as a
  documented contract.
- The `eventData` **field names are derived mechanically from the PCF/MQI constant
  names** (strip the `MQCA_`/`MQIA_`/`MQCACF_` prefix, camelCase). IBM's own
  `amqsevt -d` example proves the mapping (`MQCA_Q_MGR_NAME` → `queueMgrName`,
  `MQCA_BASE_OBJECT_NAME` → `baseObjectName`) but never states the rule.
- The **authoritative field semantics** live one layer down, in the *Event message
  reference → Event message descriptions* — one page per event type, in PCF terms,
  not JSON.
- Critically, IBM states the PCF structures are **not returned in a defined order**
  and some parameters are **optional, present only when relevant**. So the JSON is
  *not* a fixed-shape schema: consumers must key by name and tolerate missing
  fields.

A single illustrative snippet is not enough to build parsing, alerting, or
integration tests against. This epic writes the working knowledge down and — the
essential part — **grounds it in real JSON captured from the live lab**, one
example per event class, produced by **deterministically forcing** each event.

## 2. Doctrine & principles

- **Real captured examples over reconstructed ones.** Every appendix example is
  actual `amqsevt -o json` output from the lab, not hand-written. The value is in
  showing the *actual* key-value pairs and structure an operator will receive.
- **Glass-box.** Report B documents the exact commands used to force each event so
  the result is reproducible by a human without the AI — consistent with the lab's
  "supportable without the AI" gate.
- **Human operates the lab.** The capture runs as a human-driven live-lab session
  on the existing 3-node Native HA lab; the agent prepares the procedure and
  consumes the artifacts.
- **Document now, mechanize later.** Report B captures the *manual* procedure. A
  reusable (Ansible-based, per the repo's Ansible-over-shell convention) generator
  is deliberately **out of scope** here and seeded as the closing follow-on
  brainstorm (`#112`), routed toward `#38`.
- **Deterministic derivation is repeatable.** The report states plainly that any
  event not captured here (or any future MQ version) can be derived in the lab by
  the same method — the schema is *experimentally recoverable*, which is the point.

## 3. Deliverables

Two reports in `mq-resiliency-lab-for-linux/docs/reports/`, both companions to the
`#694` how-to.

### 3.1 Report A — working with the event JSON

`docs/reports/2026-07-20-mq-event-json-working-with-the-data.md`

- **The schema analysis:** the no-formal-schema finding; the sample-defined
  envelope; the PCF→JSON key-derivation rule (with the `amqsevt -d` proof); the
  unordered / conditionally-present caveat and what it means for consumers.
- **References:** the IBM Docs pages (tool page, *Event message format*, *Event
  message descriptions*) pinned to 9.4, plus Mark Taylor's design posts — cited
  with the `source_url` from the cached copies under
  `build/refs/ibm-docs/ibm-mq/9.4.x/…`.
- **Consuming guidance:** how to read `eventType`/`eventReason` name+value pairs,
  how to find a field's meaning (map camelCase key → PCF constant → Event message
  description), and defensive parsing (key by name, tolerate absence).
- **Finding — "do events fit in syslog?":** a per-event size/fidelity table (event
  → JSON byte size → binding transport limit → fit/truncated) and, if any event
  overflows, a prominent call-out that the lab's `#31` syslog-based pipeline loses
  data for that event class, with the remediation options (raise rsyslog
  `$MaxMessageSize`, route via journald's larger `LineMax`, or the file sink).
- **Appendix — one section per captured event:** the real (authoritative,
  complete) JSON block, an annotated key-value table (JSON key → PCF constant →
  meaning), the event queue it came from, and the reason code.

### 3.2 Report B — deterministic event-generation lab reference

`docs/reports/2026-07-20-mq-event-generation-lab-reference.md`

Per event: **preconditions**, the **exact command sequence** (`runmqsc` /
`setmqaut` / fill / `START`/`STOP`), the **expected `eventType` + reason code**,
the **event queue** it lands on, and **cleanup** to restore lab state. Raw command
snippets are committed as reference artifacts. A closing section frames this as the
input to `#112`/`#38` mechanization and records any **best-effort / deferred**
events with the reason they were deferred.

## 4. Coverage — one representative event per enabled class

The `#694` gate enables: `AUTHOREV CHADEV CHLEV CONFIGEV INHIBTEV LOCALEV LOGGEREV
PERFMEV REMOTEEV SSLEV STRSTPEV CMDEV`. One representative, easily-forced event per
class:

| Class | Representative event | Forcing sketch | Event queue |
|---|---|---|---|
| AUTHOREV | Not Authorized | `setmqaut` deny → connect/open as unauthorized user | QMGR.EVENT |
| INHIBTEV | Get Inhibited (+ Put Inhibited) | `ALTER QLOCAL GET(DISABLED)` → attempt get | QMGR.EVENT |
| LOCALEV | Unknown Object Name | open a non-existent queue | QMGR.EVENT |
| REMOTEEV | Remote Queue Name Error | misconfigured remote-queue def → put | QMGR.EVENT |
| PERFMEV | Queue Full (+ Queue Depth High) | `MAXDEPTH(n)` + `QDPMAXEV(ENABLED)` (full) / `QDPHIEV(ENABLED)` (high) → fill | PERFM.EVENT |
| CHLEV | Channel Started / Stopped | `START` / `STOP CHANNEL` | CHANNEL.EVENT |
| CONFIGEV | Create / Change object | `DEFINE` / `ALTER` with CONFIGEV on | CONFIG.EVENT |
| CMDEV | Command | any MQSC command with CMDEV on | COMMAND.EVENT |
| STRSTPEV | Queue Mgr Active | queue-manager start (Native HA nuance) | QMGR.EVENT |
| LOGGEREV | Logger | **best-effort, likely deferred** — needs linear logging; Native HA uses its own log-replication model, so the classic Logger event may never fire | LOGGER.EVENT |
| SSLEV | Channel SSL Error | **best-effort** — needs a staged TLS misconfig; may defer | CHANNEL.EVENT |

`CHADEV` (channel auto-definition) is represented within the CHLEV family unless it
forces cleanly on its own. The two **best-effort** rows (Logger, SSL) are the
explicit "derive later if fiddly" cases: if forcing them cleanly on 3-node Native
HA is disproportionate, document *why* and leave them to the follow-on rather than
bloat this epic.

## 5. Approach & data flow

```text
T1 live-lab capture (human-driven)
   force each event ─┬─► AUTHORITATIVE JSON via amqsevt (browse/drain) ──┐
                     ├─► syslog/journald copy ─► compare ─► size & fit?  ─┤
                     └─► EXACT command sequence ────────────────────────── ┤
                                                                           ▼
        committed artifacts: JSON examples + syslog-fidelity table + commands
                 │                        │                       │
                 ▼                        ▼                       ▼
        T2 Report A appendix    Report A "fits in syslog?"   T3 Report B
                                       finding                 procedures
```

T1 is upstream; T2 and T3 are **blocked-by** T1. The capture artifacts are the
single source of truth both reports draw from — no example or command in either
report is written by hand where a captured one exists.

**Capture — authoritative from `amqsevt`; syslog measured against it.** The
reference examples must be **complete and parseable**, so their authoritative
source is a direct `amqsevt` read of each event — a non-destructive
`amqsevt -b -o json` browse, or a foreground drain with the `#31` collector briefly
paused. `amqsevt` formats the full PCF, so this copy is ground truth **by
construction**. T1 does *not* trust the syslog line as the example source, because
syslog is exactly what may corrupt it.

**Syslog fidelity is a first-class, per-event finding.** In the same pass T1
captures what the lab's `#31` journald/syslog→Loki pipeline actually delivered for
each event and **compares it to the authoritative `amqsevt` copy**, recording the
JSON **byte size**, the **binding transport limit on the lab's actual path**, and
whether the event **fit or was truncated**. This answers a question the design
surfaced and that had simply been *assumed*: **do all these events fit in syslog?**

Why it matters beyond the reports: the `#31` event pipeline **is** syslog-based, so
any event that overflows the binding limit is **silently losing data and producing
unparseable JSON in the production-shaped path** — a higher-order architecture
finding, not a capture nuisance, and another entry in `#694`'s file-vs-syslog
trade-off. Binding limits to check against the lab's real path (**verify, do not
assume**): rsyslog `$MaxMessageSize` default **8 KB**; systemd-journald `LineMax`
default **~48 KB**; Loki its own per-line cap. 8 KB is small enough that a fat
config or command event could plausibly exceed it.

**Note on file vs syslog.** `#694` documents a *file* sink because the target work
site declines syslog (hence its file caveats + follow-on burden); the lab uses
syslog/journald. The two are not in tension for capture — but a **confirmed
truncation finding is routed back to `#31` as its own issue** (and strengthens the
file-vs-syslog framing in `#694`). Report A also notes the format difference: the
lab feed is single-line (`amqsevt -o json_compact`, the 9.2.4 single-line format);
`#694`'s file feed is multi-line pretty `-o json`; same keys/values, and the
reports pretty-print for readability.

**Execution environment.** This epic is *planned* in the local macOS-hosted VM,
but *implemented* — specifically the T1 live-lab capture — on the **cloud x86
Native HA RHEL stack**, via a separate `epic-implement` cloud session. (The local
Cloud VM is occupied restoring the Ubuntu builds.) The design is environment-
agnostic — the forcing commands are the same on any 9.4 Native HA RHEL queue
manager — but the plan should assume the cloud stack as the capture host and note
any Native HA nuances (see §10) are exercised there.

## 6. Component boundaries & isolation

- **This epic** documents *consuming* and *deterministically producing* the events.
  It does **not** touch shipping/storing/visualizing the JSON (that stays with the
  producing guides and epic `#31`).
- **Report A** owns the schema/consume knowledge; **Report B** owns the generation
  knowledge; the **capture artifacts** are the shared, independently-inspectable
  interface between them.
- **Tooling** (a mechanized generator) is behind a hard boundary — named as
  follow-on `#112`, not built here.

## 7. Implementation tasks (preview; finalized in the plan)

- **T1 — Live-lab capture** (`mq-resiliency-lab-for-linux`): force the ~12 events,
  capture raw JSON + exact commands as artifacts. Upstream.
- **T2 — Report A** (`mq-resiliency-lab-for-linux`): working-with-the-data report +
  appendix, consuming T1 artifacts. Blocked-by T1.
- **T3 — Report B** (`mq-resiliency-lab-for-linux`): generation lab reference,
  consuming T1 artifacts. Blocked-by T1.
- Bookends already seeded: docs (`#111`), follow-on brainstorm (`#112`),
  documentation-review (`#710`).

## 8. Acceptance criteria

- Both reports exist in `docs/reports/`, pass `vrg-validate`, and are cross-linked
  from `mq-event-monitoring-guide.md` (via `#710`).
- Report A carries the full analysis with **verifiable, pinned (9.4) references**
  separating data (what IBM/Taylor state) from judgment (the derivation rule).
- Report A's appendix has one section per **successfully captured** event, each
  showing **real, authoritative (complete, parseable)** JSON — sourced from the
  direct `amqsevt` read, never from a possibly-truncated syslog line — plus an
  annotated key→PCF→meaning table.
- **Every event is size-checked against syslog:** Report A carries the per-event
  fit/truncated table, and any truncation is called out as a `#31`-pipeline finding
  and filed as its own issue.
- Report B reproduces each captured event deterministically: preconditions,
  commands, expected `eventType`/reason, cleanup — sufficient for a human to
  regenerate it without the AI.
- Any deferred best-effort event (Logger, SSL) is recorded with its reason, and the
  reports state the derive-later method for uncaptured events.

## 9. Out of scope

- A mechanized / Ansible event-generation harness (→ `#112` / `#38`).
- Shipping, storing, or visualizing the JSON (→ `#31` and the producing guides).
- Changing the `#694` producing mechanism or the enabled event-class set.
- z/OS-only events and a comprehensive (~40-event) catalog — deliberately one per
  class; the catalog is derive-on-demand.
- Cold-rebuild validation / deployment bookends — this is docs + a lab capture.

## 10. Open questions (resolve at build time)

- Do Logger (LOGGEREV) and SSL (SSLEV) force cleanly on 3-node Native HA, or are
  they deferred? (Native HA uses its own log model; SSL needs a staged TLS fault.)
- Does `STRSTPEV` "Queue Mgr Active" capture cleanly without disrupting the Native
  HA group, or should it use a scratch standalone QM for the start/stop pair?
- Where do the raw capture artifacts live — under `build/` (gitignored) with the
  relevant blocks inlined into the reports, or committed alongside the reports as
  fixtures? (Leaning: inline into the reports; keep bulky raw captures under
  `build/`.)
- Is the `#31` produce→JSON→journald/syslog→Loki pipeline actually **live on the
  cloud Native HA capture host**? If not, T1 either enables it first or captures
  every event via the foreground `amqsevt -o json` drain path.
- **Syslog fidelity per event** — a required investigation, not just a fallback
  trigger (see §5): measure each event's JSON byte size against the binding
  transport limit on the lab's actual path and record fit/truncated. A confirmed
  overflow is a real `#31`-pipeline data-loss finding → file it as its own issue
  and reflect it in `#694`'s file-vs-syslog trade-off. (The authoritative examples
  come from `amqsevt` regardless, so the reports are never blocked by truncation.)
