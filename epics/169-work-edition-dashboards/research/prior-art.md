# IBM MQ monitoring — prior-art research brief

**Epic:** `logical-minds-foundry/.github#169` (Work-edition Grafana dashboards) · **Design task:** `#170`
**Date:** 2026-08-05 · **Status:** research pass (seeds panel-level co-development, §10 of the design spec)

This brief is a **cited credibility artifact** (design spec §3.1): every non-obvious metric or panel
choice traces to checkable prior art, and **DATA** (what a source literally says, with a URL) is kept
distinct from **JUDGMENT** (our reasoning about what to pull into the three work-edition boards).

## Scope & method note

We consulted four source classes: (1) the **`ibm-messaging/mq-metric-samples`** repository — the
`mq_prometheus` exporter that emits the `ibmmq_*` series our boards will bind to — including its Go
source, its `README`, and all seven of its own reference Grafana dashboards; (2) **IBM MQ 9.4 official
documentation**, fetched through the repo's `tools/ibm_doc_cache.py` (IBM Docs return HTTP 403 to normal
fetching) and cited by their `source_url`; (3) **community best practice** from named authors — **MQGem
Software** (Morag Hughson), the **IBM Middleware User Community (IMWUC)** (Mark Taylor, the exporter's own
author), **MQ Technical Conference** (Rob Parker), and IBM **Redbooks**; and (4) IBM's own **event-message
and monitoring reference** for thresholds. What we could **not** reach: IBM **Support** pages that are PDF
`inline-files` (e.g. the "Configuring IBM MQ Native HA Cross-Region Replication on Linux" PDF) and a couple
of `ibm.com/support` node pages are retrievable by neither WebFetch nor the doc-cache tool and were not
read in full; the exact integer values of the ~15 raw `MQCHS_*` channel-status states and the full
`MQQMSTA_*` set were **not** enumerated from source; and — the most important flag — the **QDEPTHLO 40%
default was not confirmed** against a primary IBM attribute page this session (see D.4). IBM Docs pages are
versioned; we pinned to **9.4 / 9.4.x** throughout.

---

## Method & source reliability

| # | Source | URL | Authority |
|---|--------|-----|-----------|
| S1 | `mq-metric-samples` exporter README | https://github.com/ibm-messaging/mq-metric-samples/blob/master/cmd/mq_prometheus/README.md | **Primary.** The exporter's own authors; defines naming + config. |
| S2 | `mq-metric-samples` top-level README | https://github.com/ibm-messaging/mq-metric-samples/blob/master/README.md | **Primary.** Config keys, monitored-object gating. |
| S3 | Reference dashboards (7 JSON) | https://github.com/ibm-messaging/mq-metric-samples/tree/master/cmd/mq_prometheus | **Primary.** The authors' own metric-selection choices. |
| S4 | `mq-golang` `mqmetric/` (channel.go, qmgr.go) | https://github.com/ibm-messaging/mq-golang/blob/master/mqmetric/channel.go | **Primary.** Defines the metric constants + status-squash encoding. |
| S5 | IBM MQ 9.4 — Real-time monitoring | https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=network-real-time-monitoring | **Primary (vendor).** MONQ/MONCHL doctrine. |
| S6 | IBM MQ 9.4 — DISPLAY CHSTATUS | https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=reference-display-chstatus-display-channel-status | **Primary (vendor).** Channel status + monitoring attributes. |
| S7 | IBM MQ 9.4 — Performance events | https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=monitoring-performance-events | **Primary (vendor).** Perf-event catalog + SYSTEM.ADMIN.PERFM.EVENT. |
| S8 | IBM MQ 9.4 — Queue depth events | https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=events-queue-depth | **Primary (vendor).** Queue Depth High/Low/Full semantics. |
| S9 | IBM MQ 9.4 — ALTER QUEUE (QDEPTHHI/LO) | https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=reference-alter-queues-alter-queue-settings | **Primary (vendor).** Threshold attributes (% of MAXDEPTH). |
| S10 | IBM MQ 9.4 — Event message descriptions | https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=reference-event-message-descriptions | **Primary (vendor).** Full event catalog. |
| S11 | IBM MQ 9.4 — Controlling channel and bridge events | https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=events-controlling-channel-bridge | **Primary (vendor).** CHLEV/SSLEV, SYSTEM.ADMIN.CHANNEL.EVENT. |
| S12 | IBM MQ 9.4 — Native HA | https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=availability-native-ha | **Primary (vendor).** Raft quorum model; CRR = recovery group. |
| S13 | IBM MQ 9.4 — dspmq (display queue managers) | https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=reference-dspmq-display-queue-managers | **Primary (vendor).** `-o nativeha` fields incl. CRR group role. |
| S14 | IBM MQ 9.4 — amqsmon | https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=information-amqsmon-display-formatted-monitoring | **Primary (vendor).** Statistics/accounting queues (distinct from resource pubs). |
| S15 | MQGem — "I want to know when queues are filling" (Jan 2025) | https://mqgem.wordpress.com/2025/01/13/queues-filling/ | **Named expert (Morag Hughson, MQGem / ex-IBM MQ dev).** |
| S16 | IBM MQ 9.4 — Enabling queue depth events | https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=events-enabling-queue-depth | **Primary (vendor).** "Disabled by default"; 80/20 worked example. |
| S17 | MQGem — MaxChannels vs DIS QMSTATUS CONNS (Jun 2016) | https://mqgem.wordpress.com/2016/06/23/maxchannels-vs-dis-qmstatus-conns/ | **Named expert.** Connections ≠ channels; CURSHCNV. |
| S18 | MQGem — Avoiding run-away numbers of channels (Nov 2013) | https://mqgem.wordpress.com/2013/11/12/avoid-run-away-nums-of-chls/ | **Named expert.** MAXINST/MAXINSTC sizing. |
| S19 | MQGem — Dead-letter Queue use by MQ Channels (Sep 2021) | https://mqgem.wordpress.com/2021/09/02/ibm-mq-channels-dlq/ | **Named expert.** DLQ + handler as practice. |
| S20 | MQGem — Example filters for MO71 (Dec 2014) | https://mqgem.wordpress.com/2014/12/17/example-filters-for-mo71/ | **Named expert.** STOPPED(red)/RETRYING(dred) sender filter. |
| S21 | IMWUC — Prometheus + Grafana to monitor MQ (Mark Taylor, IBM, 2016) | https://community.ibm.com/community/user/integration/viewdocument/using-prometheus-and-grafana-to-mon?CommunityKey=183ec850-4947-49c8-9a2e-8e7c7fc46c64 | **Named IBM author (exporter's creator).** |
| S22 | IMWUC — Prometheus to monitor MQ channel status (Mark Taylor, 2018) | https://community.ibm.com/community/user/viewdocument/using-prometheus-to-monitor-mq-chan?CommunityKey=183ec850-4947-49c8-9a2e-8e7c7fc46c64 | **Named IBM author.** status/statusSquash rationale. |
| S23 | MQTC 2016 — Monitoring and Tracking MQ (Rob Parker, IBM) | https://www.slideshare.net/RobertParker54/mqtc-2016-monitoring-and-tracking-mq-and-applications | **Named IBM author (conference).** Per-object watchlist. |
| S24 | IBM Redbook SG24-7839 — HA in WebSphere Messaging Solutions | https://www.redbooks.ibm.com/abstracts/sg247839.html | **IBM Redbook.** HA + monitoring considerations. |
| S25 | IBM manual — Monitoring and Performance for IBM MQ 9.1 (PDF) | https://public.dhe.ibm.com/software/integration/wmq/docs/V9.1/PDFs/mq91.monitor.pdf | **Primary (vendor manual).** Events/stats/real-time consolidated. |

> Local doc-cache copies (canonical text + `meta.json` source_url) live under
> `build/refs/ibm-docs/ibm-mq/9.4.x/<slug>/` for S5–S14 and S16.

---

## § Exporter (`mq-metric-samples`): metric catalog + reference-dashboard choices  — DATA

### E.1 Two distinct metric sources inside one exporter (S1, S2)
The exporter emits **two kinds** of `ibmmq_*` series:
1. **Published resource metrics** — derived from the queue manager's **amqsrua-style** resource
   publications (system topics). Metric names are **generated dynamically** from the publication
   descriptions you see in `amqsrua`, lowercased/underscored, prefixed `ibmmq` (namespace configurable
   via `-namespace`). Verbatim (S1): *"The queue and queue manager metrics … are named after the
   descriptions that you can see when running the amqsrua sample program, but with some minor
   modifications to match the required style."*
2. **Object-status metrics** — collected via `DIS QSTATUS` / `DIS CHSTATUS`-type commands, and
   **gated** behind `useObjectStatus` (YAML) / `-ibmmq.useStatus` (CLI). Verbatim (S1): *"The
   `-ibmmq.useStatus` … or `useObjectStatus` … parameter must be set to `true` to use the DIS QSTATUS
   command."* Attribute names are hard constants in `mq-golang/mqmetric`.

> Note this is **separate** from MQ **statistics/accounting messages** (SYSTEM.ADMIN.STATISTICS.QUEUE /
> SYSTEM.ADMIN.ACCOUNTING.QUEUE, read by `amqsmon` — S14). The exporter consumes the **resource-monitoring
> publications**, not the statistics queues. Do not conflate the two mechanisms.

There is **no flat `metrics.txt` catalog** in the repo — names are dynamic. The exact `ibmmq_*` series
present at any site therefore depend on **exporter version + config + which QM publications are enabled**
(this is exactly why the design spec §3.2 mandates a live-exporter "schema note").

### E.2 `ibmmq_qmgr_*` — names observed in source/dashboards (S3, S4)
Status/object attributes: `ibmmq_qmgr_status`, `ibmmq_qmgr_uptime`, `ibmmq_qmgr_connection_count`,
`ibmmq_qmgr_channel_initiator_status`, `ibmmq_qmgr_command_server_status`, `ibmmq_qmgr_active_listeners`,
`ibmmq_qmgr_max_channels`, `ibmmq_qmgr_max_active_channels` (constants `active_services`, `name` also
defined). Published-resource: `ibmmq_qmgr_system_cpu_time_percentage`, `ibmmq_qmgr_user_cpu_time_percentage`,
`ibmmq_qmgr_log_write_latency_seconds`, `ibmmq_qmgr_queue_manager_file_system_in_use_bytes`,
`ibmmq_qmgr_log_file_system_in_use_bytes`, `ibmmq_qmgr_mq_errors_file_system_free_space_percentage`,
`ibmmq_qmgr_interval_mqput_mqput1_total_count`, `ibmmq_qmgr_interval_destructive_get_total_count`,
`ibmmq_qmgr_published_to_subscribers_message_count`, plus browse/subscription counters.

### E.3 `ibmmq_queue_*` — names observed (S3)
`ibmmq_queue_depth`, `ibmmq_queue_attribute_max_depth`, `ibmmq_queue_oldest_message_age`,
`ibmmq_queue_uncommitted_messages`, `ibmmq_queue_qtime_short`, `ibmmq_queue_qtime_long`,
`ibmmq_queue_time_since_get`, `ibmmq_queue_time_since_put`, `ibmmq_queue_input_handles`,
`ibmmq_queue_output_handles`, `ibmmq_queue_qfile_current_size`, `ibmmq_queue_qfile_max_size`,
`ibmmq_queue_mqget_count`, `ibmmq_queue_mqput_mqput1_count`.

### E.4 `ibmmq_channel_*` — names observed (S3, S4)
`ibmmq_channel_status`, `ibmmq_channel_status_squash`, `ibmmq_channel_messages`,
`ibmmq_channel_bytes_sent`, `ibmmq_channel_bytes_rcvd`, `ibmmq_channel_buffers_sent`,
`ibmmq_channel_buffers_rcvd`, `ibmmq_channel_batches`, `ibmmq_channel_time_since_msg`,
`ibmmq_channel_nettime_short/long`, `ibmmq_channel_batchsz_short/long`,
`ibmmq_channel_xmitq_time_short/long`, `ibmmq_channel_cur_inst`,
`ibmmq_channel_attribute_max_inst`, `ibmmq_channel_attribute_max_instc`.
Instance labels: `channel`, `qmgr`, `type` (e.g. `SVRCONN`, `SENDER`), plus connection/job name for
uniqueness.

### E.5 Other families (S1, S3)
- **Native HA:** `ibmmq_nha_*`, e.g. `ibmmq_nha_backlog_average_bytes`, `ibmmq_nha_synchronous_log_sent_bytes`
  (label tag now `nha`, formerly `nhainstance`). **See §Native HA/CRR for the exporter-vs-collector gap.**
- **Cluster:** `ibmmq_cluster_suspend`.
- **Topic:** `ibmmq_topic_*` (a `Topic_Status.json` dashboard exists).

### E.6 Status encodings (S1, S4) — verified vs unverified
- **Channel status squash (VERIFIED, `mqmetric/channel.go`):** `ibmmq_channel_status_squash` has exactly
  three values — **0 = STOPPED-group** (INACTIVE, DISCONNECTED, STOPPED, PAUSED), **1 = TRANSITION**
  (BINDING, STARTING, STOPPING, RETRYING, REQUESTING, INITIALIZING, SWITCHING), **2 = RUNNING**. The
  authors' own dashboards test `== 2` for "running." Purpose (S1): three values are "easier to put
  colours against in Grafana."
- **Raw channel status (PARTIALLY VERIFIED):** `ibmmq_channel_status` carries the standard MQ `MQCHS_*`
  integer; S1 says *"about 15 of these possible values."* The **exact integer→state mapping was NOT
  enumerated from source** — confirm against `cmqc.h`/`cmqcfc` before hardcoding raw integers in a panel.
- **QM status (PARTIALLY VERIFIED):** `ibmmq_qmgr_status` uses `MQQMSTA_*` (source references
  `MQQMSTA_RUNNING`). Full integer set (STARTING/RUNNING/QUIESCING/…) **not enumerated from source.**
- **QM-down sentinel (S1):** with `keepRunning`, when the QM is unreachable the exporter still emits
  `ibmmq_qmgr_status` "indicating that the queue manager is down" — a usable "up?" signal.

### E.7 Reference-dashboard PromQL choices — verbatim (S3)
These are the authors' own expressions — the most directly transferable prior art (identical exporter):
- **Depth %:** `(ibmmq_queue_depth * 100 )/ ibmmq_queue_attribute_max_depth`  *(Queue_Status.json)*
- **Queue-file utilisation %:** `100 * ibmmq_queue_qfile_current_size/ibmmq_queue_qfile_max_size`
- **Oldest message age:** `ibmmq_queue_oldest_message_age`, `max(ibmmq_queue_oldest_message_age)`
- **Put/get rate — MIXED convention (important):** queue-level panels use the counters **raw**
  (`ibmmq_queue_mqget_count`, `ibmmq_queue_mqput_mqput1_count`), while the QM-interval panels wrap with
  `rate()`: `rate(ibmmq_qmgr_interval_mqput_mqput1_total_count[$__rate_interval])` and
  `rate(ibmmq_qmgr_interval_destructive_get_total_count[$__rate_interval])`. **We must verify the `# TYPE`
  (counter vs gauge) on a live exporter before choosing `rate()` vs raw** (matches our
  dashboards-lead-with-time-series memory).
- **Channel status panels:** rendered `+0` to force numeric (`ibmmq_channel_status_squash+0`); running
  count `count(ibmmq_channel_status_squash == 2) by (qmgr)`; channel census
  `count by (channel, qmgr)(ibmmq_channel_status)`. Traffic panels **exclude** SVRCONN:
  `ibmmq_channel_bytes_sent{type!="SVRCONN"}`; xmitq time restricted to senders:
  `ibmmq_channel_xmitq_time_short{type="SENDER"}`.
- **QM band:** `ibmmq_qmgr_status+0`, `ibmmq_qmgr_channel_initiator_status+0`,
  `ibmmq_qmgr_command_server_status+0`, `ibmmq_qmgr_connection_count+0`, `ibmmq_qmgr_active_listeners+0`.

### E.8 Config gating (S1, S2) — which series exist at all
- **Queues:** `monitoredQueues` / `monitoredQueuesFile` (CLI `-ibmmq.monitoredQueues`). Wildcard+exclusion
  syntax, e.g. `"A*,!AB*"`. A queue not in the monitored set (and not publishing) yields **no**
  `ibmmq_queue_*` series.
- **Channels:** `monitoredChannels` / `monitoredChannelFile`; `showInactiveChannels`.
- **Object-status (QSTATUS/CHSTATUS-derived series, incl. oldest-age, handles, uncommitted):** require
  `useObjectStatus`/`-ibmmq.useStatus=true`.
- **Publications/statistics:** `usePublications`, `useStatistics`.
- **Discovery:** `rediscoverInterval` (default 1h), `pollInterval`. Default HTTP port **9157**
  (`httpListenPort`).

---

## § IBM monitoring doctrine — DATA

### D.1 Three separate monitoring mechanisms (do not conflate)
1. **Real-time monitoring** (S5): "determine the current state of queues and channels … accurate at the
   moment the command was issued." Controlled by attributes **MONQ** (queues), **MONCHL** (channels),
   **MONACLS** (auto-defined cluster-sender channels). Verbatim (S5): *"Before you can use some of the
   queue attributes, you must enable them for real-time monitoring."* This is what powers `DIS QSTATUS` /
   `DIS CHSTATUS` monitoring fields — and, transitively, the exporter's status metrics.
2. **Statistics & accounting messages** (S14): PCF records on **SYSTEM.ADMIN.STATISTICS.QUEUE** and
   **SYSTEM.ADMIN.ACCOUNTING.QUEUE**, read by `amqsmon -t statistics|accounting`. Accounting = per-MQI-op
   by application; statistics = system activity. **Separate** from the resource publications the exporter
   uses.
3. **Instrumentation events** (S7, S10, S11): PCF event messages on system event queues — the basis of
   our ES event feed (Wave 1b).

### D.2 Channel status + monitoring attributes (S6, DISPLAY CHSTATUS)
- **STATUS enum (verbatim states, S6):** BINDING, INITIALIZING, PAUSED, REQUESTING, RETRYING, RUNNING,
  STARTING, STOPPING, STOPPED, SWITCHING (plus INACTIVE/DISCONNECTED for inactive instances). RETRYING =
  *"A previous attempt to establish a connection has failed. The MCA will reattempt … after the specified
  time interval."*
- **INDOUBT:** whether the channel is currently in-doubt. S6: for an **inactive** channel, `CURMSGS`,
  `CURSEQNO`, `CURLUWID` are meaningful **only if the channel is INDOUBT** — so in-doubt is the key
  inactive-channel signal.
- **Monitoring fields (require MONCHL set, S6):** `XQTIME` (time on xmitq, short/long indicators),
  `NETTIME`, `MSGS`, `BYTSSENT`/`BYTSRCVD`, `BUFSSENT`/`BUFSRCVD`, `BATCHSZ`, `COMPRATE`/`COMPTIME`,
  `EXITTIME`. `SUBSTATE` shows fine-grained activity (SEND/RECEIVE/RESYNCH/HEARTBEAT/…). `MCASTAT` =
  whether the message channel agent is running. `STOPREQ` = user stop requested.
- Verbatim (S6): monitoring values *"are displayed only when the STATUS of the channel is RUNNING"* and
  *"A value is only displayed for this parameter if MONCHL is set."*

### D.3 Event catalog (S10, S7, S8, S11)
- **Performance events** → **SYSTEM.ADMIN.PERFM.EVENT** (S7): Queue Depth High, Queue Depth Low, Queue
  Full, Queue Service Interval High, Queue Service Interval OK. Scope = the queue.
- **Channel/SSL/bridge events** → **SYSTEM.ADMIN.CHANNEL.EVENT** (S11), controlled by QM attributes
  `CHLEV` (ENABLED / EXCEPTION), `SSLEV`, `BRIDGEEV`, `CHADEV`. Event types (S10) include Channel Started,
  Channel Stopped, Channel Stopped By User, Channel Not Activated, Channel Not Available, Channel Blocked,
  Channel Conversion Error, Channel SSL Error/Warning, Channel Auto-definition OK/Error, Channel Activated.
- **Other catalog entries (S10):** Queue Manager Active / Not Active; Get/Put Inhibited; authority events
  (Not Authorized type 1–6); config/create/delete/change-object events; remote-queue / transmission-queue
  error events; Unknown Object Name; Logger event.

### D.4 Queue-depth thresholds (S8, S9, S16)
- **Queue Depth High** fires when depth rises to the **QDEPTHHI** limit (event gated by **QDPHIEV**);
  **Queue Depth Low** at **QDEPTHLO** (**QDPLOEV**); **Queue Full** at MAXDEPTH (**QDPMAXEV**) (S8, S9).
- Verbatim (S9): QDEPTHHI/QDEPTHLO *"value is expressed as a percentage of the maximum queue depth
  (MAXDEPTH)."* Restriction (S16): *"QDEPTHHI must not be less than QDEPTHLO."*
- **Enabled state:** verbatim (S16) *"By default, all queue depth events are disabled."* — an operator
  must explicitly `ALTER QMGR PERFM(ENABLED)` + set `QDPHIEV(ENABLED)` etc. per queue.
- **Example limits:** IBM's own 9.4 worked example (S16) uses `QDEPTHHI(80)` and **`QDEPTHLO(20)`** — an
  *example*, not a stated product default.
- **⚠ Default caveat (do not bluff this number):** the widely-repeated "QDEPTHHI 80% / QDEPTHLO 40%"
  default is the value carried by `SYSTEM.DEFAULT.LOCAL.QUEUE`. **QDEPTHHI = 80% is well established**, but
  the **QDEPTHLO = 40% default was NOT confirmed against a primary IBM attribute-reference page this
  session** (and IBM's *events* example uses 20%, not 40%). Confirm QDEPTHLO's default directly via
  `DISPLAY QLOCAL(SYSTEM.DEFAULT.LOCAL.QUEUE) QDEPTHHI QDEPTHLO` on a live QM (or the DEFINE QLOCAL
  attribute reference) before stating 40% as fact. Our boards never hardcode a universal value regardless
  (they color at the site's configured QDEPTHHI).

---

## § Community best practice + channel-type scenarios — DATA

### C.1 MQGem Software (Morag Hughson — ex-IBM MQ development, IBM Champion)
- **Queue-fill detection (S15):** advocates **Queue Depth events over polling** (event-driven beats
  waiting for a poll cycle). Worked example on a `MAXDEPTH 5000` queue wanting an alert at 500 msgs (10%):
  `ALTER QLOCAL(...) QDEPTHHI(10) QDEPTHLO(5) QDPHIEV(ENABLED)` — she deliberately sets thresholds **well
  below** 80% because 80% of a large queue is already too late. Prereq `ALTER QMGR PERFM(ENABLED)`; events
  land on SYSTEM.ADMIN.PERFM.EVENT. **Judgment she states:** the right threshold is a per-queue % of
  MAXDEPTH, not a universal number.
- **Connections ≠ channels (S17):** `DIS QMSTATUS CONNS` counts *connections*, which since MQ V7 are
  **not 1:1 with SVRCONN instances** (shared conversations mean one instance carries many MQCONNs). To
  track real channel-instance usage vs `MaxChannels`, count `DISPLAY CHSTATUS(*) CURSHCNV`, not
  connections. Exhaustion symptom: `MQRC_CHANNEL_NOT_AVAILABLE (2537)` + "Maximum number of channels
  reached" in AMQERR.
- **Run-away channels (S18):** cap per-SVRCONN with **MAXINST** and per-client with **MAXINSTC**; sizing
  rule `MaxChannels ≈ sending MCAs + receiving MCAs + Σ SVRCONN MAXINST`. Runaway counts usually mean an
  app reconnecting without disconnecting.
- **DLQ practice (S19):** "Having a Dead-letter Queue employed … is generally a recommended practice" —
  pair it with a **DLQ Handler** to alert operators; messages divert to a DLQ at three failure points
  (app MQPUT, sender MCA e.g. `MSG_TOO_BIG_FOR_CHANNEL`, receiver MQPUT).
- **MO71 channel filter (S20):** ships a filter that colours **sender channels STOPPED = red, RETRYING =
  dark red** — an explicit statement that STOPPED and RETRYING are the two sender states to watch
  (verbatim filter: `qm("*"); bg(status=stopped, red); bg(status=retrying, dred); ... chltype=sender`).

### C.2 IBM Middleware User Community (IMWUC) — named IBM authors
- **Mark Taylor (creator of the exporter), Prometheus+Grafana (S21):** the `mq_prometheus` monitor uses
  MQ V9's **pub/sub resource-monitoring interface** so it doesn't interfere with other tools; default
  scrape port **9157**; queues selected by pattern (`-ibmmq.monitoredQueues="APPA.*,APPB.*"`); **patterns
  expand only at startup** (new queues need a restart — matches E.8). Exposes QM-wide metrics (MQPUT
  rates, CPU, log usage) automatically.
- **Mark Taylor, channel status via Prometheus (S22):** channel metrics `messages`, `status` (numeric,
  e.g. **`3 = MQCHS_RUNNING`**), and **`statusSquash`** collapsing the ~15 states into three buckets;
  recommends Grafana **traffic-light colours on `statusSquash`** to surface not-running channels. Note
  (sourced): Taylor applies the **same** status approach to **all** channel types — the exporter does not
  differentiate per type.

### C.3 MQTC / Redbooks / IBM manual
- **MQTC 2016, Rob Parker (IBM), "Monitoring and Tracking MQ" (S23)** — per-object watchlist:
  - *Queues:* last put/get time; **age of oldest message (seconds)**; depth vs QDEPTHHI/QDEPTHLO/MAXDEPTH;
    queue service interval (QSVCINT/QSVCIEV).
  - *Channels:* type + status; batches/batch size; bytes & buffers sent/received; message sequence number;
    SSL handshake errors.
  - *Connections/apps:* active connections + user IDs; open handles per queue; **flag Units of Work
    running longer than ~4 minutes** (a concrete long-running-UOW signal).
  - *QM:* `display qmstatus all`, ping responsiveness, throughput/API stats.
- **IBM Redbook SG24-7839 (S24):** HA solutions incl. multi-instance QMs, with operational monitoring &
  alerting considerations (title/scope verified via abstract; specifics not line-quoted this session).
- **IBM "Monitoring and Performance" manual, 9.1 PDF (S25):** the authoritative consolidated manual for
  events, resource monitoring, accounting/statistics, and real-time monitoring.

### C.4 Channel-type scenarios — signal per type (sourced vs general)
| Channel type | Primary signal(s) operators watch | Basis |
|---|---|---|
| **SVRCONN** (clients) | Active instance count vs **MAXINST/MAXINSTC** and QM **MaxChannels**; `CURSHCNV`; watch `2537 MQRC_CHANNEL_NOT_AVAILABLE`; unexpected mass client drops. Exclude from throughput panels. | **Sourced** (S17, S18); exclusion from traffic per exporter dashboards (S3). |
| **SENDER / RECEIVER** | **status = RETRYING or STOPPED** (alert states); **in-doubt**; **XMITQ depth backing up**; `XQTIME` (time on xmitq) as leading backpressure indicator; sender-side DLQ diversions. | **Sourced:** STOPPED/RETRYING (S20); XQTIME (S6). |
| **CLUSSDR / CLUSRCVR** | Same status/in-doubt/retry, applied to **`SYSTEM.CLUSTER.TRANSMIT.QUEUE` depth** — rising SCTQ depth = canonical "cluster channel not delivering." | STOPPED/RETRYING **sourced** (S20); SCTQ-depth-as-symptom = **general operator knowledge**, no single-URL threshold. |

**In-doubt** channels need `RESOLVE CHANNEL` — widely-held operator knowledge; `INDOUBT` is a CHSTATUS
field (S6, D.2). The exporter's `statusSquash` gives the not-running signal generically but encodes **no
per-type thresholds** (S22).

---

## § Common alerting rules & thresholds — DATA (sourced) + JUDGMENT (site-specific)

| Signal | Threshold | Sourced? |
|--------|-----------|----------|
| **DLQ depth > 0** | any message on the DLQ = investigate | **Convention** (near-universal). DLQ purpose + "use a DLQ handler" is sourced (S19); the specific ">0 alert" is operator consensus, not a single IBM number. |
| **Queue Depth High** | % of MAXDEPTH; IBM example 80% (S16); MQGem sets far lower per queue, e.g. 10% (S15) | **Semantics sourced** (S8/S9/S16). The *value* is per-queue judgment. **Do not** cite "80/40 default" as fact — see D.4 caveat (QDEPTHLO 40% unverified this session; IBM example uses 20%). |
| **Queue Depth Low** | % of MAXDEPTH (QDEPTHLO); IBM example 20% (S16) | **Semantics sourced;** default figure unverified (D.4). |
| **Queue Full** | depth = MAXDEPTH → Queue Full event / `2053 MQRC_Q_FULL` | **Sourced** (S7/S8). |
| **Channel STOPPED / RETRYING** | status not RUNNING (squash != 2) | **Sourced states** (S6, S20); alerting = convention endorsed by MQGem MO71 filter (S20). |
| **Channel in-doubt** | INDOUBT = YES → needs RESOLVE CHANNEL | **Sourced** state (S6); alert = convention. |
| **SVRCONN instances** | count approaching **MAXINST / MAXINSTC / MaxChannels** | **Sourced** (S17, S18). |
| **Oldest message age** | SLA-driven | **Signal sourced** (S23 "age of oldest message in seconds"); the *number* is site-specific. Metric `ibmmq_queue_oldest_message_age` (S3). |
| **Long-running UOW / uncommitted** | **> ~4 minutes** (rule of thumb) | **Sourced rule of thumb** (S23). `ibmmq_queue_uncommitted_messages` (S3). |
| **Queue service interval** | not serviced within QSVCINT → QSVCIEV | **Mechanism sourced** (S23, S25); value site-specific. |
| **Backout count** | approaching **BOTHRESH** (poison-message loop) | **Mechanism = general knowledge** (BOTHRESH/BOQNAME); no universal alert value; per-queue stock series unconfirmed (see gaps). |
| **Xmitq / SCTQ depth backing up** | site-specific baseline | **Judgment.** Leading indicator via depth + `XQTIME` (S6); SCTQ symptom = general knowledge. |

**Rule:** where a number is not in the "Sourced" column, the board encodes the *shape* of the alert
(threshold coloring, empty=healthy Attention table) and leaves the *value* as a documented per-site
parameter — never invent a universal number.

---

## § Native HA / CRR monitoring — DATA (exporter-vs-collector coverage)

### N.1 What IBM exposes (S12, S13)
- **Native HA model (S12):** three instances, **Raft** consensus; the elected leader is the **active**
  instance; a **quorum** (majority) of log acknowledgements confirms a write; an active instance that
  loses quorum **abdicates**. **CRR (Cross-Region Replication)** is Native HA extended with a **recovery
  group** (S12: *"adds a recovery group, extending the configuration to be a Native HA Cross-Region
  Replication (CRR) configuration"*).
- **`dspmq -o nativeha` fields (S13) — the authoritative status surface:**
  - Base: **ROLE** (Active / Replica / Unknown / Leader / Not configured), **INSTANCE**, **INSYNC**,
    **QUORUM** (`in-sync/configured`, e.g. `3/3`), **GRPLSN**, **GRPNAME**, **GRPROLE**.
  - **GRPROLE = the CRR signal:** Live / Recovery / Pending live / Pending recovery / Unknown / Not
    configured — i.e. whether this group is the **live** (primary region) or **recovery** (DR region) side.
  - `-x` extras (per-instance): **REPLADDR**, **CONNACTV** (connected to active?), **BACKLOG** (**KB
    behind the active** — the replication-lag proxy), **CONNINST**, **ACKLSN**, **SYNCTIME** (last in-sync
    time, ISO-8601), and **HASTATUS** (Normal / Checking / Synchronizing / Rebasing / Disk full /
    Disconnected / Unknown).
  - `-g` (group view): **GRPNAME**, **GRPROLE**, **GRPADDR**.

### N.2 Exporter vs collector — the coverage gap
- **In the stock exporter:** `ibmmq_nha_*` series exist (E.5) — e.g. `ibmmq_nha_backlog_average_bytes`,
  `ibmmq_nha_synchronous_log_sent_bytes` — sourced from the QM's own NHA resource publications. These give
  **byte-level replication/backlog** signals.
- **NOT reliably in the stock exporter:** the **role/quorum/in-sync/HASTATUS** and especially the **CRR
  group role (GRPROLE Live/Recovery)** semantics that operators actually alert on. Those are surfaced by
  **`dspmq -o nativeha [-x|-g]`** (S13), a **CLI** surface — which is exactly why the design spec §3.2 /
  §7.3 gate the Infra board on the **custom collector (#79)** that shells out to `dspmq` and emits
  `cluster_nha_*` metrics, verified against golden sample output.
- **Verification action:** on a live lab exporter, confirm which `ibmmq_nha_*` series actually appear and
  whether any role/quorum field is among them; treat everything role/quorum/GRPROLE-shaped as
  **collector-provided** until proven otherwise. The CRR-on-Linux configuration PDF (IBM Support
  `inline-files`) could **not** be fetched by tooling and should be read manually to finalise the exact
  CRR status commands.

---

## § JUDGMENT — what to pull into each board

*Labelled judgment: our reasoning on top of the DATA above. Every candidate below cites the DATA section
it rests on. Items marked ⚠ need live-exporter `/metrics` verification before commit (design spec §7.2).*

### Board 1 — QM view (pure `ibmmq_*`)
- **Status band:** `ibmmq_qmgr_status` (+ down-sentinel behaviour, E.6); `ibmmq_qmgr_uptime`;
  services = `ibmmq_qmgr_channel_initiator_status` ∧ `ibmmq_qmgr_command_server_status` ∧
  `ibmmq_qmgr_active_listeners` (E.2); `ibmmq_qmgr_connection_count` (with `+0` per S3).
- **Trend band:** message rate via `rate(ibmmq_qmgr_interval_mqput_mqput1_total_count[…])` and
  `…interval_destructive_get_total_count` (E.7) ⚠ verify counter type; connection count over time; if
  available, `ibmmq_qmgr_log_file_system_in_use_bytes` / recovery-log utilisation.
- **Open Q:** does work's exporter enable the CPU/log-latency publications (`system_cpu_time_percentage`,
  `log_write_latency_seconds`)? These are high-value QM-health trends but publication-gated (E.1, E.8).

### Board 2 — Queue/channel view (pure `ibmmq_*`)
- **Depth %:** reuse the authors' own `(ibmmq_queue_depth*100)/ibmmq_queue_attribute_max_depth` (E.7);
  color red at the site's configured QDEPTHHI (IBM example 80%, but per-queue — D.4; MQGem argues for
  lower, S15). ⚠ `attribute_max_depth` requires the queue in `monitoredQueues` / status enabled (E.8).
- **Attention/Inventory (queues):** query-filtered Attention on depth% > threshold and
  `ibmmq_queue_oldest_message_age` > SLA; Inventory = all monitored queues, worst-first.
- **DLQ pill:** `ibmmq_queue_depth{queue="<DLQ>"} > 0`, red (DLQ>0 = investigate; §Common alerting, S19).
  ⚠ DLQ must be in `monitoredQueues`.
- **Put-vs-get on one graph** (the depth derivative, per our memory): `ibmmq_queue_mqput_mqput1_count` vs
  `ibmmq_queue_mqget_count` — ⚠ decide raw vs `rate()` from live `# TYPE` (E.7 mixed convention).
- **Channels:** status via `ibmmq_channel_status_squash` (0/1/2, E.6) for coloring; Attention floats
  squash != 2 and in-doubt; per-type scenarios (§Community): SVRCONN instance-count
  (`cur_inst`/`max_inst`), SENDER/CLUSSDR `xmitq_time`+xmitq depth, in-doubt. ⚠ raw in-doubt exposure via
  exporter status metrics needs live verification (INDOUBT is a CHSTATUS field, D.2).
- **Backout:** `ibmmq_queue_uncommitted_messages` exists (E.3); per-queue **backout count** is **not**
  confirmed as a stock series — ⚠ flag; may need MQSC/event or a computed panel.

### Board 3 — Infra / HA-DR (Native HA + CRR only; `cluster_nha_*` collector, #79)
- **Quorum/replica band:** QUORUM (`in-sync/configured`), per-replica INSYNC, ROLE, HASTATUS (N.1) — from
  the collector, not stock exporter (N.2).
- **CRR role + lag:** **GRPROLE** (Live/Recovery) as the region-role pill; **BACKLOG (KB behind)** and/or
  `ibmmq_nha_backlog_average_bytes` as the **replication-lag trend** (RPO risk over time, per our
  time-series memory); SYNCTIME as "last in-sync." (N.1, N.2)
- **Attention/Inventory (replicas):** float any replica with INSYNC=no / HASTATUS != Normal.
- **Open Qs:** (1) exact `ibmmq_nha_*` series present on the live exporter vs what only `dspmq` gives
  (N.2); (2) the exact CRR status commands — pending the IBM Support CRR PDF we could not fetch; (3) the
  collector's golden-output contract (design spec §3.2).

---

## § Live-exporter verification (2026-08-06) — DATA

Run against a live `nativeha-ubuntu` lab bring-up (the `SVCQM` and, after fixing an authz
bug, the `NHAUAPP` `mq_prometheus` exporters, plus `dspmq -o nativeha -x` on the active
instance). Resolves the ⚠ items flagged in the JUDGMENT section:

- **put/get counter type — RESOLVED → use `rate()`.** `# TYPE` confirms `ibmmq_queue_mqget_count`,
  `ibmmq_queue_mqput_mqput1_count`, `ibmmq_qmgr_interval_mqput_mqput1_total_count`, and
  `ibmmq_qmgr_interval_destructive_get_total_count` are all **`counter`**. So `rate()` everywhere
  — the reference dashboards' raw queue-level usage (E.7) is not what we want.
- **depth% / oldest / uncommitted — CONFIRMED.** `ibmmq_queue_depth`, `ibmmq_queue_attribute_max_depth`,
  `ibmmq_queue_oldest_message_age`, `ibmmq_queue_uncommitted_messages` all present as **gauges**.
- **channel status — CONFIRMED gauges.** `ibmmq_channel_status` and `ibmmq_channel_status_squash`
  are gauges → raw value + value-map (0/1/2 per E.6).
- **backout count — NOT a stock series.** No `ibmmq_*backout*` metric exists → Board 2 backout must
  come from **ES events**, not Prometheus.
- **in-doubt — NOT a stock series.** No stock in-doubt metric → from CHSTATUS / **ES events**, not
  Prometheus.
- **`ibmmq_nha_*` — NONE scraped anywhere.** Confirms Board 3's role/quorum/**GRPROLE(Live/Recovery)**/
  BACKLOG surface is available only via **`dspmq -o nativeha -x`** (the custom collector, #79), not
  the stock exporter. The live `dspmq` output carries exactly the S13 fields (ROLE/QUORUM/INSYNC/
  GRPROLE/BACKLOG/HASTATUS/SYNCTIME) — the golden-output seed for the metric contract.

**Coverage + security findings (motivate the observability-config phase, #173):**

- **We are under-collecting.** The `NHAUAPP` exporter served **~118 `ibmmq_*` series** vs `SVCQM`'s
  **~3223**; `-ibmmq.useStatus`/`useObjectStatus` is **off** (so QSTATUS/CHSTATUS-derived series are
  suppressed); `monitoredQueues`/`monitoredChannels` are `*,SYSTEM.*`; statistics/accounting is not
  consumed. The board data set cannot be trusted complete until the exporter config is reviewed
  top-down.
- **Two security/authz bugs found and fixed** while verifying: `grafana-server` refused to start on
  the default image-renderer token (#935); the `mqmon` least-privilege surface omitted the
  command-queue + `$SYS`-topic authority the exporter needs, so the hardened Native HA exporter
  crash-looped on MQRC 2035 (#936). The exporter's required authorities are undocumented upstream and
  had to be pinned empirically — a full, documented grant model is part of #173.

## § Full source list (checkable)

- S1 https://github.com/ibm-messaging/mq-metric-samples/blob/master/cmd/mq_prometheus/README.md
- S2 https://github.com/ibm-messaging/mq-metric-samples/blob/master/README.md
- S3 https://github.com/ibm-messaging/mq-metric-samples/tree/master/cmd/mq_prometheus (dashboards:
  `Queue_Status.json`, `Channel_Status.json`, `Queue_Manager_Status.json`, `MQ_Prometheus_Overview.json`,
  `Logging.json`, `Topic_Status.json`, `zOS_Status.json`)
- S4 https://github.com/ibm-messaging/mq-golang/blob/master/mqmetric/channel.go (and `qmgr.go`)
- S5 https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=network-real-time-monitoring
- S6 https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=reference-display-chstatus-display-channel-status
- S7 https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=monitoring-performance-events
- S8 https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=events-queue-depth
- S9 https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=reference-alter-queues-alter-queue-settings
- S10 https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=reference-event-message-descriptions
- S11 https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=events-controlling-channel-bridge
- S12 https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=availability-native-ha
- S13 https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=reference-dspmq-display-queue-managers
- S14 https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=information-amqsmon-display-formatted-monitoring
- S15 https://mqgem.wordpress.com/2025/01/13/queues-filling/
- S16 https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=events-enabling-queue-depth
- S17 https://mqgem.wordpress.com/2016/06/23/maxchannels-vs-dis-qmstatus-conns/
- S18 https://mqgem.wordpress.com/2013/11/12/avoid-run-away-nums-of-chls/
- S19 https://mqgem.wordpress.com/2021/09/02/ibm-mq-channels-dlq/
- S20 https://mqgem.wordpress.com/2014/12/17/example-filters-for-mo71/
- S21 https://community.ibm.com/community/user/integration/viewdocument/using-prometheus-and-grafana-to-mon?CommunityKey=183ec850-4947-49c8-9a2e-8e7c7fc46c64
- S22 https://community.ibm.com/community/user/viewdocument/using-prometheus-to-monitor-mq-chan?CommunityKey=183ec850-4947-49c8-9a2e-8e7c7fc46c64
- S23 https://www.slideshare.net/RobertParker54/mqtc-2016-monitoring-and-tracking-mq-and-applications
- S24 https://www.redbooks.ibm.com/abstracts/sg247839.html
- S25 https://public.dhe.ibm.com/software/integration/wmq/docs/V9.1/PDFs/mq91.monitor.pdf

### Explicit gaps (labelled, not bluffed)
- Exact integer→state map of the ~15 raw `MQCHS_*` (`ibmmq_channel_status`) values — **unverified from
  source**; only the 3-value squash is verified (E.6). S22 corroborates `3 = MQCHS_RUNNING`.
- Full `MQQMSTA_*` integer set for `ibmmq_qmgr_status` — **unverified from source** (E.6).
- **QDEPTHLO 40% default — UNVERIFIED this session.** QDEPTHHI 80% is well established; QDEPTHLO's 40%
  (the SYSTEM.DEFAULT.LOCAL.QUEUE value) was not confirmed against a primary IBM attribute page, and IBM's
  events *example* uses 20% (S16). Confirm via `DISPLAY QLOCAL(SYSTEM.DEFAULT.LOCAL.QUEUE) QDEPTHHI
  QDEPTHLO` before publishing "40%". **This is the single most important number for the fact-checker to
  verify.**
- Per-queue **backout count** as a stock exporter series — **not confirmed**; flagged for live check.
- **`ibmmq_nha_*` live coverage** — which NHA/role/quorum series actually appear must be confirmed on a
  live lab exporter; treat role/quorum/GRPROLE as collector-provided until proven (N.2).
- IBM Support **CRR-on-Linux** PDF (`inline-files`) and two IBM Support pages (mq-agent queue-depth
  events) — **not fetchable** by tooling (403/PDF); read manually to finalise exact CRR status commands.
- IBM Redbook SG24-7839 (S24) monitoring specifics were **not line-quoted** — scope verified via abstract
  only.
