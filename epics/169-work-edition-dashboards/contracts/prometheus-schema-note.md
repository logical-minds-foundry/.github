# Prometheus schema note — `ibmmq_*` metric/label contract (Wave 1a)

- **Epic:** `logical-minds-foundry/.github#169` · **Task 1:** `#175` · **Wave 1a**
- **Status:** authoritative contract; object-status catalog verified live 2026-08-07;
  publication-driven catalog carried from `phase0-observability-config.md` (2026-08-06) —
  see the [live-capture caveat](#5-live-capture-caveat--the-exporter-is-currently-under-collecting).
- **Purpose (spec §3.2):** the exact `ibmmq_*` metric names, types, and label keys the
  work-edition boards' PromQL binds to, plus the exporter version + config assumed and the
  one-time work-import check. Board queries bind to *this* document; it is what makes the boards
  portable to work's own Prometheus. Nothing hardcodes a datasource UID or a QM name — panels
  reference `$datasource` and scope by `$qmgr` (spec §5).

This note is the completion gate for Tasks 3 (QM board) and 4 (Queue/channel board): every PromQL
they use must map to a row here, and nothing may be bound that is not verified — either live on the
exporter or in the Phase-0 doc.

## 1. Exporter version + assumed config

The boards assume the **`ibm-messaging/mq-metric-samples`** Prometheus exporter (`mq_prometheus`),
run in **client mode** against the QM. This is the same exporter family at the lab and at work; the
per-site difference is only which Prometheus scrapes it (selected via `$datasource` on import).

| Setting | Assumed value | Why the boards need it | Source |
|---------|---------------|------------------------|--------|
| Exporter | `ibm-messaging/mq-metric-samples` (`mq_prometheus`), Go runtime `go1.25.0` (live `go_info`) | the `ibmmq_*` namespace, counter/gauge split | live endpoint 2026-08-07 |
| `useObjectStatus` / `useStatus` | **on** (deprecated → *forced on* by `VerifyConfig`) | QSTATUS/CHSTATUS gauges (status, handles, oldest-age) — the entire object-status catalog in §3 | phase0 §4 (`pkg/config/config.go:383–388`) |
| `usePublications` | **on** | resource-monitoring pubs (CPU/RAM/MQI counts, STATQ put/get, `ibmmq_nha_*`) — the trend-band series in §4 | phase0 §1 (Mech. 1) |
| Queue class | **`EXTENDED`** (plus the default classes) | 9.4.2+ L2/L3 queue diagnostics | phase0 §3 (lever "keep") |
| `monitoredQueues` | `*,SYSTEM.*` (wildcard) | one board serves every queue on `$qmgr` | phase0 §3 / spec §3.2 |
| `monitoredChannels` | `*,SYSTEM.*` (wildcard) | one board serves every channel on `$qmgr` | phase0 §3 |
| `showInactiveChannels` | `true` | defined-but-**stopped** channels must appear (a dashboard-critical breach signal) | phase0 §3 (lever "keep") |
| `rediscoverInterval` | `1m` | newly-created queues/channels appear promptly (default is 1h) | phase0 §3 caveat |
| Grant model | `mqmon` least-privilege identity holds `+dsp +inq` across the monitored-queue namespace | **coverage IS the security model** — the exporter cannot inquire a queue it has no authority on; without the namespace grant idle/backed-up queues are invisible | phase0 §3 key finding, §5 |

There is **no `build_info`/version series** — `mq-metric-samples` does not emit one, and metric names
are generated at runtime from publication descriptions (no static manifest). The catalog is therefore
established by **scrape-and-grep against the live exporter**, which is how §3 was captured.

## 2. One-time work-import check

The whole reusability mechanism hinges on one query resolving (spec §3.2):

1. **Import** the board JSON into work's Grafana; pick work's Prometheus for `$datasource`.
2. **Confirm `$qmgr` populates** — the variable query is `label_values(ibmmq_qmgr_status, qmgr)`.
   If the dropdown lists work's queue managers, the exporter, its config, and the `qmgr` label all
   match this contract.
3. **Done.** If the dropdown is empty, work's exporter differs from §1 — reconcile against this note
   before trusting any panel (check the exporter is `mq-metric-samples`, that `qmgr` is the label
   key, and that `ibmmq_qmgr_status` is emitted).

Before emailing the JSON, sanity-check work's exporter with
`curl -s <work-exporter>:9163/metrics | grep '^# TYPE ibmmq_'` and diff the names against §3/§4.

## 3. Object-status catalog — verified live (2026-08-07)

These series come from `useObjectStatus` (PCF poll of `DISPLAY QSTATUS`/`CHSTATUS`/`QMSTATUS`) and are
present on the live exporter **now**, independent of publications. **Every name, type, and label key
below is copied verbatim from the live `NHAUAPP` exporter at `http://10.50.0.3:9163/metrics`** — 52
`ibmmq_*` series across the qmgr / queue / channel / subscription classes.

Common label keys: `qmgr` (the `$qmgr` variable's key), `platform`, `description`, `hostname` (qmgr
class), `cluster`/`queue`/`usage` (queue class), `channel`/`connname`/`type`/`rqmname`/`jobname`/
`sslciph` (channel class).

Counters take `rate()`; gauges are used raw (spec §7.2, `overrideCType=true` splits them correctly).

### 3.1 `qmgr` class (18 series — all gauges)

Labels: `qmgr, platform, description, hostname` (the two `exporter_*` self-metrics carry only
`qmgr, platform`).

| Metric | Type | Board use |
|--------|------|-----------|
| `ibmmq_qmgr_status` | gauge | QM status band; **`$qmgr` variable source** (`label_values(ibmmq_qmgr_status, qmgr)`) — QM + Flow |
| `ibmmq_qmgr_uptime` | gauge | QM status band (uptime) |
| `ibmmq_qmgr_connection_count` | gauge | QM status band + trend (connections over time) |
| `ibmmq_qmgr_active_listeners` | gauge | QM status band (services ∧) / Attention (which piece is down) |
| `ibmmq_qmgr_active_services` | gauge | QM status band (services ∧) / Attention |
| `ibmmq_qmgr_channel_initiator_status` | gauge | QM status band (services ∧) / Attention |
| `ibmmq_qmgr_command_server_status` | gauge | QM status band (services ∧) / Attention |
| `ibmmq_qmgr_log_extent_current` | gauge | QM trend (recovery-log %) |
| `ibmmq_qmgr_log_extent_restart` | gauge | QM trend (recovery-log %) |
| `ibmmq_qmgr_log_extent_media` | gauge | QM trend (recovery-log %) |
| `ibmmq_qmgr_log_extent_archive` | gauge | QM trend (recovery-log %) |
| `ibmmq_qmgr_log_size_restart` | gauge | QM trend (recovery-log %) |
| `ibmmq_qmgr_log_size_reusable` | gauge | QM trend (recovery-log %) |
| `ibmmq_qmgr_log_size_media` | gauge | QM trend (recovery-log %) |
| `ibmmq_qmgr_log_size_archive` | gauge | QM trend (recovery-log %) |
| `ibmmq_qmgr_log_start_epoch` | gauge | (context; not a lead panel) |
| `ibmmq_qmgr_exporter_publications` | gauge | **health-of-collection meter** — 0 means no resource pubs are flowing (see §5) |
| `ibmmq_qmgr_exporter_collection_time` | gauge | exporter self-metric (scrape cost) |

### 3.2 `queue` class (12 series — all gauges)

Labels: `qmgr, queue, cluster, usage, description, platform`.

| Metric | Type | Board use |
|--------|------|-----------|
| `ibmmq_queue_attribute_max_depth` | gauge | Flow — depth% denominator (MAXDEPTH); DLQ threshold context |
| `ibmmq_queue_attribute_usage` | gauge | Flow — identifies XMITQ/normal usage |
| `ibmmq_queue_oldest_message_age` | gauge | Flow — status band (oldest message age) + Attention/Inventory |
| `ibmmq_queue_uncommitted_messages` | gauge | Flow — Attention (in-flight work) |
| `ibmmq_queue_input_handles` | gauge | Flow — Inventory (open input handles) |
| `ibmmq_queue_output_handles` | gauge | Flow — Inventory (open output handles) |
| `ibmmq_queue_time_since_get` | gauge | Flow — Inventory (idle detection) |
| `ibmmq_queue_time_since_put` | gauge | Flow — Inventory (idle detection) |
| `ibmmq_queue_qtime_short` | gauge | Flow — latency (short-interval on-queue time) |
| `ibmmq_queue_qtime_long` | gauge | Flow — latency (long-interval on-queue time) |
| `ibmmq_queue_qfile_current_size` | gauge | Flow — Inventory (queue-file size) |
| `ibmmq_queue_qfile_max_size` | gauge | Flow — Inventory (queue-file cap) |

> **Current queue depth (`CURDEPTH`) is NOT in this object-status set** — the exporter sources depth
> from STATQ **publications**, so `depth`/`depth over time`/`DLQ depth` (Flow status band + trend
> band, spec §7.2) depend on the publication catalog in §4. See §5.

### 3.3 `channel` class (20 series)

Labels: `qmgr, channel, connname, type, rqmname, jobname, sslciph, description, platform`.

| Metric | Type | Board use |
|--------|------|-----------|
| `ibmmq_channel_status` | gauge | Flow — status band (channels not running) + Attention (Retrying/Stopped) |
| `ibmmq_channel_status_squash` | gauge | Flow — simplified 3-state status (value-mapped) |
| `ibmmq_channel_substate` | gauge | Flow — Attention (what a channel is doing/stuck on) |
| `ibmmq_channel_type` | gauge | Flow — channel-type scenarios (SENDER/RECEIVER/SVRCONN) |
| `ibmmq_channel_instance_type` | gauge | Flow — Inventory (instance type) |
| `ibmmq_channel_messages` | counter | Flow — trend (`rate()` = channel throughput, msgs) |
| `ibmmq_channel_bytes_sent` | counter | Flow — trend (`rate()` = channel throughput, bytes) |
| `ibmmq_channel_bytes_rcvd` | counter | Flow — trend (`rate()` = channel throughput, bytes) |
| `ibmmq_channel_buffers_sent` | counter | Flow — Inventory (buffers) |
| `ibmmq_channel_buffers_rcvd` | counter | Flow — Inventory (buffers) |
| `ibmmq_channel_batches` | counter | Flow — Inventory (batch count) |
| `ibmmq_channel_batchsz_short` | gauge | Flow — Inventory (batch-size, short) |
| `ibmmq_channel_batchsz_long` | gauge | Flow — Inventory (batch-size, long) |
| `ibmmq_channel_nettime_short` | gauge | Flow — Inventory (network round-trip, short) |
| `ibmmq_channel_nettime_long` | gauge | Flow — Inventory (network round-trip, long) |
| `ibmmq_channel_xmitq_time_short` | gauge | Flow — Inventory (time on XMITQ, short) |
| `ibmmq_channel_xmitq_time_long` | gauge | Flow — Inventory (time on XMITQ, long) |
| `ibmmq_channel_time_since_msg` | gauge | Flow — Inventory (idle channel detection) |
| `ibmmq_channel_security_protocol` | gauge | Flow — Inventory (TLS in use) |
| `ibmmq_channel_start_epoch` | gauge | Flow — Inventory (instance start time) |

### 3.4 `subscription` class (2 series)

Labels: `qmgr, platform, subid, subscription, topic, type`.

| Metric | Type | Board use |
|--------|------|-----------|
| `ibmmq_subscription_messsages_received` | counter | (context; not a Wave-1a lead panel — note the exporter's `messsages` spelling) |
| `ibmmq_subscription_type` | gauge | (context) |

## 4. Publication-driven catalog — required by the trend bands

These classes come from **resource-monitoring publications** (Mechanism 1: `$SYS/MQ/INFO/...`,
`useObjectStatus` does **not** produce them). Phase 0 verified them live on 2026-08-06 (~200 total
`ibmmq_*` names incl. **~40 `ibmmq_nha_*`**); they are **absent from the current live scrape** because
the exporter is receiving zero publications (§5). The **exact names must be captured from a live
scrape with publications flowing — they are not enumerated here because guessing them would violate
the "do not invent metric names" rule.**

| Class / signal | Board use (spec §7) | Type (per phase0 §7.2) | Status |
|----------------|---------------------|------------------------|--------|
| `ibmmq_queue_depth` (current CURDEPTH) | Flow — depth% status pill, depth-over-time trend, DLQ depth (red > 0) | gauge | verified live 2026-08-06 (phase0 §7.2 "depth% inputs … present gauges"); **not in current scrape** |
| per-queue put/get counters (STATQ) | Flow — **put-vs-get on one graph** (the leading indicator, `rate()`), backout context | counter (`rate()`) | verified live 2026-08-06 (phase0 §7.2 "put/get … all counters"); **not in current scrape** |
| qmgr MQI-interval counters (STATMQI) | QM — **message rate** trend | counter (`rate()`) | verified live 2026-08-06; **not in current scrape** |
| `ibmmq_nha_*` (~40 series: replica lag, recovery backlog, recovery-group stats) | Wave 2 Infra board *trend* half (lag/throughput/latency); **not a Wave-1a board** | mixed (see note) | phase0 §2 (~40 names); **not in current scrape** |

Notes:

- **`ibmmq_nha_*` is documented here for class-completeness only.** The Wave-1a boards (QM view,
  Queue/channel view) are **pure object-status + trend `ibmmq_*`** and do **not** bind the nha class.
  The Wave-2 Infra board's *discrete status* (ROLE/QUORUM/GRPROLE/HASTATUS) binds the separate
  `cluster_nha_*` custom-collector contract (`nha-crr-metric-contract.md`, Task 7), **not** stock
  `ibmmq_nha_*`. Stock `ibmmq_nha_*` supplies only the Infra board's lag/throughput *trends*
  (phase0 §2 "Board 3 correction: hybrid"). Exact `ibmmq_nha_*` names are out of Wave-1a scope and
  will be pinned when the Infra board (Task 8) is built.
- **Before Tasks 3/4 bind any §4 name, re-run the capture in §5 with publications flowing and record
  the exact names/types/labels here.** Until then, treat §3 (object status) as the verified floor.

## 5. Live-capture caveat — the exporter is currently under-collecting

Captured 2026-08-07 against `http://10.50.0.3:9163/metrics` (QM `NHAUAPP`), stable across three
scrapes over several minutes:

```text
ibmmq_qmgr_exporter_publications{platform="UNIX",qmgr="NHAUAPP"} 0
ibmmq_qmgr_exporter_collection_time{platform="UNIX",qmgr="NHAUAPP"} 0
# 52 distinct ibmmq_* names, all object-status; no ibmmq_nha_*, no per-queue put/get, no CURDEPTH
```

`ibmmq_qmgr_exporter_publications = 0` means the exporter is subscribed to **no** resource-monitoring
publications right now, so the entire §4 publication catalog (CPU/RAM/MQI counts, STATQ put/get,
current depth, `ibmmq_nha_*`) is not being emitted. Phase 0 recorded ~200 names incl. ~40 nha on
2026-08-06, so this is a **regression from the Phase-0-verified state**, not a config choice of this
contract (a prior instance of exactly this was the crash-looping exporter of #936 and the
`SYSTEM.ADMIN.TOPIC` `+sub` grant fix).

**Consequence for the boards:** the §3 object-status floor (status bands, services, channels,
oldest-age, handles, latency, channel throughput) is bindable today. The trend-band signals that need
§4 — QM **message rate**, Flow **depth-over-time** and **put-vs-get** — cannot be bound to
live-verified names until publications are restored and re-captured. This does not block authoring the
status/Attention/Inventory panels; it blocks the trend band's publication-sourced series.

Re-capture procedure once publications flow (`exporter_publications > 0`):

```bash
curl -s http://10.50.0.3:9163/metrics | grep '^# TYPE ibmmq_'        # names + counter/gauge
curl -s http://10.50.0.3:9163/metrics | grep '^ibmmq_' | head        # sample lines with labels
```

Then fill the exact names/types/labels into §4 and drop this caveat.

## 6. NOT in Prometheus — sourced from the ES event feed (Wave 1b), never PromQL

Two Queue/channel-board signals are **not stock `ibmmq_*` series** and must **never** be written as
PromQL. They come from the **Elasticsearch event feed** (Wave 1b, `es-doc-field-contract.md`),
re-confirmed live in Phase 0 and in spec §7.2:

- **Per-queue backout** (backout count / backout-threshold breaches) — **not** a stock series. The
  Flow board's "backout rate" trend is an **ES-sourced** panel, not a PromQL `rate()`. (There is no
  `ibmmq_queue_backout*` metric; do not invent one.)
- **In-doubt** (channels/UOW in doubt) — **not** a stock series. Channel Attention floats in-doubt
  from the **ES event feed**, not from `ibmmq_channel_*`.

Any board panel needing backout or in-doubt binds to `$logs` (Elasticsearch) via the Wave-1b ES
contract, not to `$datasource` (Prometheus). This is the hard boundary between the metric boards
(Wave 1a) and the event feed (Wave 1b).

## 7. Sources

- **Live exporter:** `http://10.50.0.3:9163/metrics` (QM `NHAUAPP`), scraped 2026-08-07 — the exact
  names/types/labels in §3 and the `exporter_publications = 0` evidence in §5.
- **`phase0-observability-config.md`** (this epic, 2026-08-06) — §1 mechanisms, §2 available-vs-
  collected (~200 names incl. ~40 `ibmmq_nha_*`), §3 config levers + the coverage-is-security grant
  finding, §4 exporter collection model (`useStatus` forced on, runtime-generated names,
  `overrideCType`), §5 the `mqmon` least-privilege grant model.
- **`spec.md`** §3.2 (schema-note contract), §5 (template variables), §7.1/§7.2 (board structure +
  the 2026-08-06 counter/gauge verification and the backout/in-doubt → ES boundary).
- **Exporter source:** `ibm-messaging/mq-metric-samples` (`pkg/config/config.go`) and
  `ibm-messaging/mq-golang/mqmetric/` — names generated at runtime, no static manifest (phase0 §4).
