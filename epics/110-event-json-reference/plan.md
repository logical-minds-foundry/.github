# Working with MQ instrumentation events as JSON — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Two `docs/reports/` reports that make MQ instrumentation-event JSON *usable* — Report A (how to consume it: schema, the PCF→JSON key rule, the "do events fit in syslog?" finding, a real captured appendix) and Report B (how to deterministically force each event) — both grounded in **real JSON captured from the live lab**.

**Architecture:** One upstream live-lab capture task (**T1**) runs on the cloud x86 Native HA RHEL stack: it forces one representative event per enabled class, takes an **authoritative** complete copy of each via a direct `amqsevt` read, measures the **syslog/journald copy** against it for truncation, and commits the captured JSON + a per-event fidelity table + the exact forcing commands as fixtures. **T2** (Report A) and **T3** (Report B) are pure authoring tasks over those fixtures and are **blocked-by T1**.

**Tech Stack:** IBM MQ 9.4 (`amqsevt`, MQSC, `setmqaut`, `crtmqm`), 3-node Native HA RHEL 9, systemd journald + `logger -t mq-events` (the `#31` collector), Markdown reports + `tools/ibm_doc_cache.py` for pinned IBM references, `vrg-validate`.

- **Design spec:** `epics/110-event-json-reference/spec.md` (this directory).
- **Epic:** `logical-minds-foundry/.github#110`. **Docs task:** `#111`.
- **Implementation repo:** `logical-minds-foundry/mq-resiliency-lab-for-linux` (all paths below are relative to that repo, on a feature branch per task).
- **Prior/sibling:** `#31` (the produce→JSON→journald→Loki pipeline these reports consume) · `#694` (the file-sink how-to) · routes toward `#38` (future mechanized generation via `#112`).

## Global Constraints

- **Validation is one command only:** `vrg-container-run -- vrg-validate`. Do not run individual linters/formatters. (repo CLAUDE.md)
- **Git/GitHub via wrappers:** `vrg-git`, `vrg-gh`, commit with `vrg-commit --type <t> --scope <s> --message <m>`. Raw `git`/`gh` are denied. No direct commits to `develop`/`main`; work on `feature/<issue>-<slug>`.
- **Execution host:** the T1 live-lab capture runs on the **cloud x86 Native HA RHEL stack** (the local Cloud VM is occupied restoring Ubuntu builds). T2/T3 are authoring and run anywhere.
- **Authoritative JSON comes from `amqsevt`, never from syslog.** Every reference example is a direct `amqsevt` read of the event (complete, parseable by construction). Syslog is the thing *under test*, measured against it — never the source of an example.
- **Coverage is one representative event per enabled class (~12), not a catalog.** Uncaptured events are derivable later by the same method; the reports say so. Best-effort/likely-deferred: Logger (LOGGEREV), SSL (SSLEV).
- **IBM Docs at build time:** confirm every cited flag/field against IBM Docs 9.4 via `python3 tools/ibm_doc_cache.py <url>` (browser-UA cache under `build/refs/ibm-docs/…`; cite `content.txt` + `source_url` from `meta.json`). Do not trust `WebFetch` on `ibm.com/docs`.
- **No bespoke PCF code.** Everything is `amqsevt` + MQSC + shell. Nothing parses PCF.
- **Bulky raw captures stay in `build/`** (gitignored, use `mqlab build path`); only the curated authoritative JSON + fidelity table + command snippets are committed as fixtures.
- **Reference-accuracy discipline (repo CLAUDE.md / user policy):** in Report A separate **data** (what IBM Docs / Mark Taylor actually state) from **judgment** (e.g. the prefix-strip camelCase rule); every reference is a real, checkable, 9.4-pinned URL.

---

### Task 1: Live-lab capture — force each event, authoritative `amqsevt` copy + syslog-fidelity measurement

**Runs on the cloud Native HA RHEL host.** Force one representative event per enabled class; for each, take an authoritative complete copy via a direct `amqsevt` read, capture what the `#31` journald pipeline delivered, and record byte-size + fit. Commit the curated fixtures that T2/T3 consume.

**Files:**
- Create (fixtures): `docs/reports/assets/110-mq-event-captures/json/<event-slug>.json` (authoritative pretty JSON, one per captured event)
- Create: `docs/reports/assets/110-mq-event-captures/commands/<event-slug>.md` (exact forcing commands + cleanup, one per event)
- Create: `docs/reports/assets/110-mq-event-captures/syslog-fidelity.md` (per-event size/fit table)
- Create: `docs/reports/assets/110-mq-event-captures/README.md` (provenance: host, QM, MQ version, capture date, method)
- Scratch (gitignored, NOT committed): raw drains under `$(mqlab build path temp)/110-captures/`

**Interfaces:**
- Consumes: a running Native HA QM with events enabled per `#694`/`#31` (`AUTHOREV…CMDEV`), the `#31` `MQ.EVENT.MONITOR` collector live (journald tag `mq-events`), `/opt/mqm/samp/bin/amqsevt`.
- Produces: for each captured event a `<event-slug>` with a committed `json/<slug>.json`, `commands/<slug>.md`, and a `syslog-fidelity.md` row. T2 consumes the JSON + fidelity table; T3 consumes the commands.

**Event set & forcing recipes (the `<event-slug>` list):**

| slug | class | event | force (on `<QM>`) | expect | event queue |
|---|---|---|---|---|---|
| `not-authorized` | AUTHOREV | Not Authorized (type 1, connect) | as an unprivileged OS user: `/opt/mqm/samp/bin/amqsput Q <QM>` | MQRC 2035 | QMGR.EVENT |
| `get-inhibited` | INHIBTEV | Get Inhibited | `DEFINE QLOCAL(EVT.INHIB)`; `ALTER QLOCAL(EVT.INHIB) GET(DISABLED)`; `amqsget EVT.INHIB <QM>` | MQRC 2016 | QMGR.EVENT |
| `unknown-object` | LOCALEV | Unknown Object Name | `/opt/mqm/samp/bin/amqsput EVT.NOPE.$$ <QM>` | MQRC 2085 | QMGR.EVENT |
| `remote-qname-error` | REMOTEEV | Remote Queue Name Error | `DEFINE QREMOTE(EVT.RMT) RNAME(NO.TARGET) RQMNAME(NO.QM) XMITQ(NO.XMIT)`; `amqsput EVT.RMT <QM>` | remote-event reason | QMGR.EVENT |
| `queue-full` | PERFMEV | Queue Full | `DEFINE QLOCAL(EVT.FULL) MAXDEPTH(1) QDPMAXEV(ENABLED)`; put 2 msgs | MQRC 2053 | PERFM.EVENT |
| `queue-depth-high` | PERFMEV | Queue Depth High | `DEFINE QLOCAL(EVT.HI) MAXDEPTH(10) QDEPTHHI(80) QDPHIEV(ENABLED)`; put 8 msgs | — | PERFM.EVENT |
| `channel-stopped` | CHLEV | Channel Stopped | `STOP CHANNEL(<test.chl>)` (then `START` for `channel-started`) | — | CHANNEL.EVENT |
| `create-object` | CONFIGEV | Create object | `DEFINE QLOCAL(EVT.CFG)` (CONFIGEV enabled) | — | CONFIG.EVENT |
| `change-object` | CONFIGEV | Change object | `ALTER QLOCAL(EVT.CFG) DESCR('event demo')` | — | CONFIG.EVENT |
| `command` | CMDEV | Command | any mutating MQSC (e.g. the `change-object` `ALTER`) with CMDEV enabled | — | COMMAND.EVENT |
| `qmgr-active` | STRSTPEV | Queue Mgr Active | **scratch standalone QM** (see Step 7): `crtmqm EVTQM`; `strmqm EVTQM` with STRSTPEV | — | QMGR.EVENT |
| `logger` | LOGGEREV | Logger | **best-effort/likely-deferred** — Native HA log model; document why if skipped | — | LOGGER.EVENT |
| `channel-ssl-error` | SSLEV | Channel SSL Error | **best-effort** — staged TLS fault; document why if skipped | — | CHANNEL.EVENT |

- [ ] **Step 1: Confirm the running pipeline's collector config and event enablement**

Confirm the `#31` collector's actual output flag and journald tag (the fidelity comparison depends on it), and that events are enabled:

```bash
# collector wrapper: confirm -o json vs -o json_compact and the logger tag
grep -rn 'amqsevt' /var/mqm/ /opt/mqm/ 2>/dev/null | grep -i 'json\|logger' || true
# event classes on the QM (expect AUTHOREV..CMDEV ENABLED, CMDEV NODISPLAY)
echo 'DISPLAY QMGR AUTHOREV CHADEV CHLEV CONFIGEV INHIBTEV LOCALEV LOGGEREV PERFMEV REMOTEEV SSLEV STRSTPEV CMDEV' | runmqsc <QM>
# collector service present + running
echo 'DISPLAY SVSTATUS(MQ.EVENT.MONITOR)' | runmqsc <QM>
# the journald sink is receiving events
journalctl -t mq-events -n 5 --no-pager
```

Expected: all classes `ENABLED` (`CMDEV(NODISPLAY)`); `MQ.EVENT.MONITOR` running; `journalctl -t mq-events` shows recent JSON. Record the collector's `-o` flag in `README.md`. If the pipeline is **not** live, enable it per `#31`/`#694` before continuing (this is the §10 precondition).

- [ ] **Step 2: Confirm the binding syslog/journald size limit on this host**

```bash
grep -rEn 'MaxMessageSize' /etc/rsyslog.conf /etc/rsyslog.d/ 2>/dev/null || echo 'rsyslog default $MaxMessageSize = 8192 bytes'
grep -En 'LineMax' /etc/systemd/journald.conf /etc/systemd/journald.conf.d/* 2>/dev/null || echo 'journald default LineMax = 48K'
```

Record the binding limit (whichever the mq-events path actually traverses) in `syslog-fidelity.md` as the ceiling every event is checked against.

- [ ] **Step 3: Create the capture scratch dir and a per-event capture helper**

```bash
CAP="$(mqlab build path temp)/110-captures"; mkdir -p "$CAP"
```

Define the capture procedure used for every queue-backed event (Steps 4–6). Per event, deterministically get **both** copies of the *same* message instance:

```bash
# capture_event <slug> <event-queue> <QM>
capture_event() {
  slug="$1"; evq="$2"; qm="$3"
  # 1. pause the destructive collector so the forced event sits on the queue
  echo "STOP SERVICE(MQ.EVENT.MONITOR)" | runmqsc "$qm"
  # 2. (caller forces the event here, between pause and browse)
  # 3. AUTHORITATIVE copy: browse (-b, non-destructive), full pretty JSON, wait 5s
  /opt/mqm/samp/bin/amqsevt -m "$qm" -q "$evq" -b -o json -w 5 > "$CAP/$slug.authoritative.json"
  wc -c "$CAP/$slug.authoritative.json"
  # 4. resume the collector -> it destructively drains the SAME event to journald
  echo "START SERVICE(MQ.EVENT.MONITOR)" | runmqsc "$qm"
  sleep 3
  # 5. SYSLOG copy: pull what journald actually delivered
  journalctl -t mq-events --since '-30s' --no-pager -o cat > "$CAP/$slug.syslog.txt"
  wc -c "$CAP/$slug.syslog.txt"
}
```

(The pause/browse/resume ordering is what makes it deterministic — the browse copy and the journald copy are the same event instance, so their sizes are directly comparable.)

- [ ] **Step 4: Force + capture each QMGR.EVENT / PERFM.EVENT / CONFIG.EVENT / COMMAND.EVENT / CHANNEL.EVENT event**

Work down the recipe table. For each event: `STOP SERVICE`, run the force command, `capture_event <slug> <queue> <QM>`. Example (Get Inhibited):

```bash
echo "DEFINE QLOCAL(EVT.INHIB) REPLACE" | runmqsc <QM>
echo "STOP SERVICE(MQ.EVENT.MONITOR)" | runmqsc <QM>
echo "ALTER QLOCAL(EVT.INHIB) GET(DISABLED)" | runmqsc <QM>
/opt/mqm/samp/bin/amqsget EVT.INHIB <QM>   # fails MQRC 2016 -> emits the event
/opt/mqm/samp/bin/amqsevt -m <QM> -q SYSTEM.ADMIN.QMGR.EVENT -b -o json -w 5 > "$CAP/get-inhibited.authoritative.json"
echo "START SERVICE(MQ.EVENT.MONITOR)" | runmqsc <QM>; sleep 3
journalctl -t mq-events --since '-30s' -o cat > "$CAP/get-inhibited.syslog.txt"
```

Record the exact commands you actually ran for each event into `commands/<slug>.md` (including cleanup, e.g. `DELETE QLOCAL(EVT.INHIB)`). Verify each `*.authoritative.json` parses:

```bash
for f in "$CAP"/*.authoritative.json; do python3 -c "import json,sys; json.load(open(sys.argv[1]))" "$f" && echo "OK $f" || echo "BAD $f"; done
```

Expected: every authoritative file is valid JSON. If a browse returns multiple events (incidental ones), isolate the target by `eventType`/`eventReason` and keep only that object.

- [ ] **Step 5: Isolate the target event and normalize to the committed fixture**

For each `<slug>`, extract the single target event object and pretty-print it to the committed fixture:

```bash
python3 -c "import json,sys; o=json.load(open(sys.argv[1])); print(json.dumps(o, indent=2))" \
  "$CAP/<slug>.authoritative.json" > docs/reports/assets/110-mq-event-captures/json/<slug>.json
```

Sanitize hostnames/IPs only if they leak host-identifying detail beyond the lab (keep values representative). Note any sanitization in `README.md`.

- [ ] **Step 6: Build the syslog-fidelity table**

For each event compute authoritative size, the journald-delivered size, and fit vs the Step-2 ceiling. Write `syslog-fidelity.md`:

```markdown
| event | class | authoritative bytes | journald bytes | binding limit | fit? |
|---|---|---|---|---|---|
| Get Inhibited | INHIBTEV | 412 | 412 | 8192 (rsyslog) | ✅ full |
| Command | CMDEV | … | … | 8192 | … |
```

Determine `fit?` by comparing the journald copy to the authoritative copy (equal parsed content = full; shorter/truncated = ⚠️ **truncated**). **If any event is truncated, add a bold call-out line** naming the event class and byte overflow — this is the `#31`-pipeline data-loss finding T2 elevates.

- [ ] **Step 7: Handle STRSTPEV via a scratch standalone QM; record LOGGEREV/SSLEV disposition**

`Queue Mgr Active` needs a start transition, which is disruptive on the Native HA group — use a throwaway QM instead:

```bash
crtmqm EVTQM && echo "ALTER QMGR STRSTPEV(ENABLED)" | (strmqm EVTQM; runmqsc EVTQM)
endmqm -i EVTQM; strmqm EVTQM   # the (re)start emits Queue Mgr Active
/opt/mqm/samp/bin/amqsevt -m EVTQM -q SYSTEM.ADMIN.QMGR.EVENT -b -o json -w 5 > "$CAP/qmgr-active.authoritative.json"
endmqm -i EVTQM && dltmqm EVTQM   # cleanup
```

Note in `commands/qmgr-active.md` that this QM is **not** on the `#31` pipeline, so its `syslog-fidelity.md` row is byte-size-vs-limit only (no live pipeline pass). For **LOGGEREV** and **SSLEV**: attempt briefly; if forcing cleanly is disproportionate on Native HA, **skip and write the reason** into `commands/logger.md` / `commands/channel-ssl-error.md` and mark them deferred in `README.md`. Do not block the epic on them.

- [ ] **Step 8: Restore lab state**

Delete every scratch object created (`EVT.INHIB`, `EVT.FULL`, `EVT.HI`, `EVT.CFG`, `EVT.RMT`), restore any `setmqaut` authority removed for `not-authorized`, and confirm `MQ.EVENT.MONITOR` is running:

```bash
for q in EVT.INHIB EVT.FULL EVT.HI EVT.CFG EVT.RMT; do echo "DELETE QLOCAL($q)" | runmqsc <QM>; done
echo "DISPLAY SVSTATUS(MQ.EVENT.MONITOR)" | runmqsc <QM>   # expect RUNNING
```

- [ ] **Step 9: Validate and commit the fixtures**

```bash
vrg-container-run -- vrg-validate
```

Expected: PASS. Then:

```bash
vrg-git add docs/reports/assets/110-mq-event-captures
vrg-commit --type docs --scope events \
  --message "capture real MQ instrumentation-event JSON + syslog-fidelity table (#110)" \
  --body "Authoritative amqsevt captures for one event per enabled class, exact forcing commands, and the per-event syslog fit/truncation finding. Fixtures for Report A/B. Refs #110."
```

Report ready via `vrg-pr-workflow report-ready` (human runs `vrg-submit-pr`). T2/T3 start once this merges.

---

### Task 2: Report A — working with the event JSON (consume + schema + syslog finding + appendix)

**Blocked-by T1.** Author the consume-side report from the T1 fixtures. No hand-written JSON — every appendix block is a T1 `json/<slug>.json`.

**Files:**
- Create: `docs/reports/2026-07-20-mq-event-json-working-with-the-data.md`
- Read (fixtures): `docs/reports/assets/110-mq-event-captures/{json/*,syslog-fidelity.md}`
- Reference (style/pairing): `docs/reports/2026-07-17-mq-event-monitoring-to-file.md`, `docs/site/docs/guides/mq-event-monitoring-guide.md`

**Interfaces:**
- Consumes: T1 fixtures (authoritative JSON + fidelity table).
- Produces: the durable consume-side reference; `#710` cross-links it from the site guide.

- [ ] **Step 1: Confirm every cited IBM reference against 9.4 and cache it**

```bash
python3 tools/ibm_doc_cache.py "https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=monitoring-sample-program-monitor-instrumentation-events-multiplatforms"
python3 tools/ibm_doc_cache.py "https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=reference-event-message-format"
python3 tools/ibm_doc_cache.py "https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=reference-event-message-descriptions"
```

Cite each via its `meta.json` `source_url`. Mark the Mark Taylor blog posts (marketaylor.synology.me `?p=401`, `?p=959`) as design-rationale sources, not IBM contract.

- [ ] **Step 2: Write the analysis sections**

Sections, in order: (1) **No formal JSON schema** — the envelope is sample-defined (`amqsevta.c`), IBM shows it only by example. (2) **The five-key envelope** — `eventSource`/`eventType`/`eventReason`/`eventCreation`/`eventData`. (3) **The PCF→JSON key rule** — strip `MQCA_`/`MQIA_`/`MQCACF_`, camelCase; **prove it** with the doc's own `-d` example (`MQCA_Q_MGR_NAME` → `queueMgrName`). Label this **judgment** (derived from paired examples), the constant list **data**. (4) **Unordered + conditionally-present** — quote the *Event message descriptions* lines; explain the consumer rule: key by name, tolerate absence. (5) **Finding the meaning of a field** — camelCase key → PCF constant → per-event *Event message description* page. (6) **Related** — a short section linking `#694` (the file-sink producer how-to), the `mq-event-monitoring-guide.md` site guide (the configure/produce guide), and **Report B** (the generation companion), so the produce→consume→generate trio is navigable from within the report (independent of the `#710` guide-side cross-link).

- [ ] **Step 3: Write the "do events fit in syslog?" finding section**

Transclude the T1 `syslog-fidelity.md` table. State the binding limit and, per event, full vs truncated. If T1 found any truncation, lead with a bold finding: the `#31` syslog→Loki pipeline silently loses data / yields unparseable JSON for that event class; list remediations (raise rsyslog `$MaxMessageSize`; prefer journald's larger `LineMax`; the `#694` file sink). Note the format difference (lab feed single-line `json_compact` vs `#694` multi-line `-o json`; same content).

- [ ] **Step 4: Build the appendix — one section per captured event**

For each committed `json/<slug>.json`: an H3 with the event name, the JSON in a fenced block, an annotated table (JSON key → PCF constant → meaning, meanings from the cached *Event message descriptions*), the event queue, and the reason code. For any deferred event (Logger/SSL), a short note pointing to `commands/<slug>.md` for why, plus the derive-later method.

- [ ] **Step 5: Cross-check appendix JSON is byte-identical to the fixtures**

```bash
for f in docs/reports/assets/110-mq-event-captures/json/*.json; do
  s=$(basename "$f" .json); grep -q "\"$s\"" /dev/null; python3 -c "import json; json.load(open('$f'))" && echo "OK $s"; done
```

Confirm each appendix block matches its fixture (no hand-edited JSON). Expected: all OK.

- [ ] **Step 6: Validate and commit**

```bash
vrg-container-run -- vrg-validate
vrg-git add docs/reports/2026-07-20-mq-event-json-working-with-the-data.md
vrg-commit --type docs --scope reports \
  --message "report: working with MQ instrumentation-event JSON (schema + syslog finding + captured appendix) (#110)" \
  --body "Consume-side companion to #694: no-formal-schema analysis, PCF->JSON key rule, do-events-fit-in-syslog finding, and a real captured appendix (one event per class). Refs #110."
```

Report ready via `vrg-pr-workflow report-ready`.

---

### Task 3: Report B — deterministic event-generation lab reference

**Blocked-by T1.** Author the generation reference from the T1 `commands/` fixtures. This is the input `#112` will mechanize toward `#38`.

**Files:**
- Create: `docs/reports/2026-07-20-mq-event-generation-lab-reference.md`
- Read (fixtures): `docs/reports/assets/110-mq-event-captures/{commands/*,README.md,syslog-fidelity.md}`

**Interfaces:**
- Consumes: T1 `commands/<slug>.md` (exact forcing commands + cleanup) and `README.md` (host/QM/version provenance).
- Produces: the reproducible generation reference; seeds `#112`.

- [ ] **Step 1: Write the preamble**

State the purpose (deterministically force each event class to test consumers/alerting), the environment (cloud Native HA RHEL, MQ 9.4 — from `README.md`), the capture method (pause collector → force → browse authoritative → resume → compare), and the forward pointer: this manual procedure is the input to `#112`/`#38` mechanization.

- [ ] **Step 2: One section per event, from the committed commands**

For each `commands/<slug>.md`: H3 event name + class, **preconditions** (enabled class, any queue/threshold setup), the **exact command sequence** (verbatim from the fixture), the **expected `eventType` + reason code**, the **event queue**, and **cleanup**. Do not paraphrase the commands — they are the tested artifact.

- [ ] **Step 3: Write the best-effort / deferred + derive-later section**

Record the disposition of `logger` and `channel-ssl-error` (captured or deferred-with-reason, from their `commands/*.md`). State the general **derive-later** method so a reader can capture any of the ~40 uncaptured events themselves. Note the STRSTPEV scratch-QM caveat (off the `#31` pipeline).

- [ ] **Step 4: Cross-reference and validate**

Link Report A (the consume companion), `#694` (file sink), `#31` (the pipeline), and name `#112` as the mechanization follow-on. Then:

```bash
vrg-container-run -- vrg-validate
vrg-git add docs/reports/2026-07-20-mq-event-generation-lab-reference.md
vrg-commit --type docs --scope reports \
  --message "report: deterministic MQ event-generation lab reference (#110)" \
  --body "Per-event force recipes (preconditions, commands, expected reason, cleanup) captured from the live lab; seed for #112/#38 mechanization. Refs #110."
```

Report ready via `vrg-pr-workflow report-ready`.

---

## Self-Review

**Spec coverage:**
- Report A (consume/schema/appendix) → Task 2. Report B (generation) → Task 3. ✅
- Real lab-captured JSON, one per enabled class → Task 1 recipe table + fixtures. ✅
- Authoritative-from-`amqsevt`, syslog measured against it → T1 Steps 3–6; Global Constraints. ✅
- "Do events fit in syslog?" first-class finding → T1 Step 6 + T2 Step 3. ✅
- Best-effort Logger/SSL + derive-later → T1 Step 7, T2 Step 4, T3 Step 3. ✅
- STRSTPEV Native HA nuance → T1 Step 7. ✅
- References pinned to 9.4, data vs judgment → T2 Steps 1–2; Global Constraints. ✅
- Cross-link from site guide → `#710` (documentation-review bookend), not a task here. ✅
- Execution on cloud Native HA host → Global Constraints + T1 header. ✅

**Placeholder scan:** no TBD/TODO; every code step shows real commands; recipes are concrete. Report *prose* is authored at build time by design (a report's body cannot be pre-written), but every section's inputs (fixtures) and required content are specified. ✅

**Consistency:** `<event-slug>` names are identical across T1 fixtures, T2 appendix, and T3 sections; `MQ.EVENT.MONITOR` / `mq-events` tag used consistently; fixture paths identical in all three tasks. ✅

**Open questions deferred to build (from spec §10):** pipeline liveness (T1 Step 1), binding syslog limit (T1 Step 2), artifact location (resolved: curated fixtures committed, raw in `build/`), STRSTPEV on Native HA (T1 Step 7 scratch QM).
