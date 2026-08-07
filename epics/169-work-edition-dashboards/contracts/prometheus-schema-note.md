# Prometheus schema note — `ibmmq_*` metric/label contract (Wave 1a)

- **Epic:** `logical-minds-foundry/.github#169` · **Task 1:** `#175` · **Wave 1a**
- **Status:** authoritative contract; **full 238-name catalog verified live 2026-08-07** against the
  `NHAUAPP` exporter (3167 series, `ibmmq_qmgr_exporter_publications` climbing). Cross-checked against
  `phase0-observability-config.md` (2026-08-06).
- **Purpose (spec §3.2):** the exact `ibmmq_*` metric names, types, and label keys the work-edition
  boards' PromQL binds to, plus the exporter version + config assumed and the one-time work-import
  check. Board queries bind to *this* document; it is what makes the boards portable to work's own
  Prometheus. Nothing hardcodes a datasource UID or a QM name — panels reference `$datasource` and
  scope by `$qmgr` (spec §5).

This note is the completion gate for Tasks 3 (QM board) and 4 (Queue/channel board): every PromQL
they use must map to a row here, and nothing may be bound that is not verified live.

Catalog shape (live 2026-08-07): **238 distinct `ibmmq_*` names / 3167 series** —
**115** `qmgr` · **61** `queue` · **40** `nha` · **20** `channel` · **2** `subscription`.

Two sources feed the catalog, and the distinction is load-bearing for the boards (§5):

- **Object status** (§3) — PCF poll of `DISPLAY QSTATUS`/`CHSTATUS`/`QMSTATUS` over the command
  queue (`useObjectStatus`). **Failover-resilient:** present continuously, including during a Native
  HA failover.
- **Resource-monitoring publications** (§4) — the `$SYS/MQ/INFO/...` pub/sub feed (`usePublications`,
  the `amqsrua` data). Richer (CPU/RAM/MQI counts, per-queue depth + put/get, `ibmmq_nha_*`) but
  **briefly interrupted by a failover** while the client-mode exporter re-subscribes (§5).

## 1. Exporter version + assumed config

The boards assume the **`ibm-messaging/mq-metric-samples`** Prometheus exporter (`mq_prometheus`),
run in **client mode** against the QM. This is the same exporter family at the lab and at work; the
per-site difference is only which Prometheus scrapes it (selected via `$datasource` on import).

| Setting | Assumed value | Why the boards need it | Source |
|---------|---------------|------------------------|--------|
| Exporter | `ibm-messaging/mq-metric-samples` (`mq_prometheus`), Go runtime `go1.25.0` (live `go_info`) | the `ibmmq_*` namespace, counter/gauge split | live 2026-08-07 |
| `useObjectStatus` / `useStatus` | **on** (deprecated → *forced on* by `VerifyConfig`) | QSTATUS/CHSTATUS gauges — the object-status catalog (§3) | phase0 §4 (`pkg/config/config.go:383–388`) |
| `usePublications` | **on** | resource-monitoring pubs — the §4 catalog (CPU/RAM/MQI counts, per-queue depth + put/get, `ibmmq_nha_*`) | phase0 §1 (Mech. 1); live 2026-08-07 |
| Queue class | **`EXTENDED`** (plus the default classes) | 9.4.2+ L2/L3 queue diagnostics (`msg_search`/`msg_examine`/`avoided_*`/`lock_contention_percentage`) | phase0 §3 (lever "keep"); confirmed live |
| `monitoredQueues` | `*,SYSTEM.*` (wildcard) | one board serves every queue on `$qmgr` | phase0 §3 / spec §3.2 |
| `monitoredChannels` | `*,SYSTEM.*` (wildcard) | one board serves every channel on `$qmgr` | phase0 §3 |
| `showInactiveChannels` | `true` | defined-but-**stopped** channels must appear (a dashboard-critical breach signal) | phase0 §3 (lever "keep") |
| `rediscoverInterval` | `1m` | newly-created queues/channels appear promptly (default is 1h) | phase0 §3 caveat |
| Grant model | `mqmon` least-privilege identity holds `+dsp +inq` across the monitored-queue namespace | **coverage IS the security model** — the exporter cannot inquire a queue it has no authority on; without the namespace grant idle/backed-up queues are invisible | phase0 §3 key finding, §5 |

There is **no `build_info`/version series** — `mq-metric-samples` does not emit one, and metric names
are generated at runtime from publication descriptions (no static manifest). The catalog is therefore
established by **scrape-and-grep against the live exporter**, which is how §3/§4 were captured.

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

## 3. Object-status catalog (failover-resilient floor) — verified live

These 52 series come from `useObjectStatus` and are present continuously, **including during a Native
HA failover** (§5). Names, types, and label keys are copied verbatim from the live `NHAUAPP` exporter.

Common label keys: `qmgr` (the `$qmgr` variable's key), `platform`, `description`, `hostname` (qmgr
class), `cluster`/`queue`/`usage` (queue class), `channel`/`connname`/`type`/`rqmname`/`jobname`/
`sslciph` (channel class). Counters take `rate()`; gauges are used raw.

### 3.1 `qmgr` object-status (18 — all gauges)

Labels: `qmgr, platform, description, hostname` (the two `exporter_*` self-metrics carry only
`qmgr, platform`).

| Metric | Type | Board use |
|--------|------|-----------|
| `ibmmq_qmgr_status` | gauge | QM status band; **`$qmgr` variable source** (`label_values(ibmmq_qmgr_status, qmgr)`) — QM + Flow |
| `ibmmq_qmgr_uptime` | gauge | QM status band (uptime) |
| `ibmmq_qmgr_connection_count` | gauge | QM status band + trend (connections over time) |
| `ibmmq_qmgr_active_listeners` | gauge | QM status band (services ∧) / Attention |
| `ibmmq_qmgr_active_services` | gauge | QM status band (services ∧) / Attention |
| `ibmmq_qmgr_channel_initiator_status` | gauge | QM status band (services ∧) / Attention |
| `ibmmq_qmgr_command_server_status` | gauge | QM status band (services ∧) / Attention |
| `ibmmq_qmgr_log_extent_current` / `_restart` / `_media` / `_archive` | gauge | QM trend — recovery-log extents (failover-resilient recovery-log signal) |
| `ibmmq_qmgr_log_size_restart` / `_reusable` / `_media` / `_archive` | gauge | QM trend — recovery-log sizes (failover-resilient) |
| `ibmmq_qmgr_log_start_epoch` | gauge | context |
| `ibmmq_qmgr_exporter_publications` | gauge | **health-of-collection meter** — see §5 (a failover briefly drops this toward 0) |
| `ibmmq_qmgr_exporter_collection_time` | gauge | exporter self-metric (scrape cost) |

### 3.2 `queue` object-status (12 — all gauges)

Labels: `qmgr, queue, cluster, usage, description, platform`.

| Metric | Type | Board use |
|--------|------|-----------|
| `ibmmq_queue_attribute_max_depth` | gauge | Flow — depth% denominator (MAXDEPTH) |
| `ibmmq_queue_attribute_usage` | gauge | Flow — identifies XMITQ/normal usage |
| `ibmmq_queue_oldest_message_age` | gauge | Flow — status band (oldest message age) + Attention |
| `ibmmq_queue_uncommitted_messages` | gauge | Flow — Attention (in-flight work) |
| `ibmmq_queue_input_handles` / `ibmmq_queue_output_handles` | gauge | Flow — Inventory (open handles) |
| `ibmmq_queue_time_since_get` / `ibmmq_queue_time_since_put` | gauge | Flow — Inventory (idle detection) |
| `ibmmq_queue_qtime_short` / `ibmmq_queue_qtime_long` | gauge | Flow — latency (on-queue time) |
| `ibmmq_queue_qfile_current_size` / `ibmmq_queue_qfile_max_size` | gauge | Flow — Inventory (queue-file size/cap) |

> **Current depth** (`ibmmq_queue_depth`) is publication-driven — see §4.2. `depth%` = `depth` (§4.2)
> ÷ `attribute_max_depth` (here). The `attribute_max_depth` denominator is failover-resilient; the
> `depth` numerator is not (§5).

### 3.3 `channel` class (20 — object-status)

Labels: `qmgr, channel, connname, type, rqmname, jobname, sslciph, description, platform`.

| Metric | Type | Board use |
|--------|------|-----------|
| `ibmmq_channel_status` | gauge | Flow — status band (channels not running) + Attention (Retrying/Stopped) |
| `ibmmq_channel_status_squash` | gauge | Flow — simplified 3-state status (value-mapped) |
| `ibmmq_channel_substate` | gauge | Flow — Attention (what a channel is doing/stuck on) |
| `ibmmq_channel_type` | gauge | Flow — channel-type scenarios (SENDER/RECEIVER/SVRCONN) |
| `ibmmq_channel_instance_type` | gauge | Flow — Inventory (instance type) |
| `ibmmq_channel_messages` | counter | Flow — trend (`rate()` = channel throughput, msgs) |
| `ibmmq_channel_bytes_sent` / `ibmmq_channel_bytes_rcvd` | counter | Flow — trend (`rate()` = channel throughput, bytes) |
| `ibmmq_channel_buffers_sent` / `ibmmq_channel_buffers_rcvd` | counter | Flow — Inventory (buffers) |
| `ibmmq_channel_batches` | counter | Flow — Inventory (batch count) |
| `ibmmq_channel_batchsz_short` / `ibmmq_channel_batchsz_long` | gauge | Flow — Inventory (batch-size) |
| `ibmmq_channel_nettime_short` / `ibmmq_channel_nettime_long` | gauge | Flow — Inventory (network round-trip) |
| `ibmmq_channel_xmitq_time_short` / `ibmmq_channel_xmitq_time_long` | gauge | Flow — Inventory (time on XMITQ) |
| `ibmmq_channel_time_since_msg` | gauge | Flow — Inventory (idle channel detection) |
| `ibmmq_channel_security_protocol` | gauge | Flow — Inventory (TLS in use) |
| `ibmmq_channel_start_epoch` | gauge | Flow — Inventory (instance start time) |

### 3.4 `subscription` class (2)

Labels: `qmgr, platform, subid, subscription, topic, type`.

| Metric | Type | Board use |
|--------|------|-----------|
| `ibmmq_subscription_messsages_received` | counter | context (note the exporter's `messsages` spelling) |
| `ibmmq_subscription_type` | gauge | context |

## 4. Publication-driven catalog — verified live 2026-08-07

These classes come from resource-monitoring publications (`usePublications`). All names/types/labels
below are copied verbatim from the live exporter. Counters take `rate()`; gauges raw.

### 4.1 `qmgr` publication-driven (97 series)

Labels: `qmgr, platform, description, hostname`.

**Board-bound (QM view, spec §7.1):**

| Metric | Type | Board use |
|--------|------|-----------|
| `ibmmq_qmgr_interval_mqput_mqput1_total_count` | counter | QM trend — **message rate** in (`rate()`) |
| `ibmmq_qmgr_interval_destructive_get_total_count` | counter | QM trend — **message rate** out (`rate()`) |
| `ibmmq_qmgr_interval_mqput_mqput1_total_bytes` | counter | QM trend — byte throughput in (`rate()`) |
| `ibmmq_qmgr_interval_destructive_get_total_bytes` | counter | QM trend — byte throughput out (`rate()`) |
| `ibmmq_qmgr_log_current_primary_space_in_use_percentage` | gauge | QM trend — **recovery-log %** (richer than §3.1 extents; disappears during failover) |
| `ibmmq_qmgr_log_workload_primary_space_utilization_percentage` | gauge | QM trend — recovery-log workload % |
| `ibmmq_qmgr_cpu_load_one_minute_average_percentage` | gauge | QM trend/Attention — CPU pressure (also `_five_` / `_fifteen_minute_`) |
| `ibmmq_qmgr_ram_free_percentage` | gauge | QM trend/Attention — memory pressure |
| `ibmmq_qmgr_concurrent_connections_high_water_mark` | gauge | QM — connection HWM context |

**Full `qmgr` publication set** (all bindable under the fire-hose config; board iteration may add
them per spec §3.3):

- **CPU (7, gauge):** `cpu_load_one_minute_average_percentage`,
  `cpu_load_five_minute_average_percentage`, `cpu_load_fifteen_minute_average_percentage`,
  `system_cpu_time_percentage`, `user_cpu_time_percentage`,
  `system_cpu_time_estimate_for_queue_manager_percentage`,
  `user_cpu_time_estimate_for_queue_manager_percentage`.
- **RAM / filesystem (10, gauge):** `ram_free_percentage`, `ram_total_bytes`,
  `ram_total_estimate_for_queue_manager_bytes`, `queue_manager_file_system_free_space_percentage`,
  `queue_manager_file_system_in_use_bytes`, `mq_errors_file_system_free_space_percentage`,
  `mq_errors_file_system_in_use_bytes`, `mq_trace_file_system_free_space_percentage`,
  `mq_trace_file_system_in_use_bytes`, `mq_fdc_file_count`.
- **Extended log — gauges:** `log_current_primary_space_in_use_percentage`,
  `log_workload_primary_space_utilization_percentage`, `log_in_use_bytes`, `log_max_bytes`,
  `log_occupied_by_reusable_extents_bytes`, `log_required_for_media_recovery_bytes`,
  `log_write_latency_seconds`, `log_write_size_bytes`, `log_slowest_write_since_restart`,
  `log_timestamp_of_slowest_write`, `log_disk_written_log_sequence_number`,
  `log_quorum_log_sequence_number`, `log_file_system_free_space_bytes`,
  `log_file_system_in_use_bytes`, `log_file_system_max_bytes`.
- **Extended log — counters (`rate()`):** `log_logical_written_bytes`, `log_physical_written_bytes`.
- **MQI call counts (counter, `rate()`):** `mqcb_count`, `mqclose_count`, `mqconn_mqconnx_count`,
  `mqctl_count`, `mqdisc_count`, `mqinq_count`, `mqopen_count`, `mqset_count`, `mqstat_count`,
  `mqsubrq_count`; failed variants `failed_mqcb_count`, `failed_mqclose_count`,
  `failed_mqconn_mqconnx_count`, `failed_mqget_count`, `failed_mqinq_count`, `failed_mqopen_count`,
  `failed_mqput_count`, `failed_mqput1_count`, `failed_mqset_count`, `failed_mqsubrq_count`,
  `failed_browse_count`, `failed_topic_mqput_mqput1_count`.
- **Message throughput / counts (counter, `rate()`):** `interval_mqput_mqput1_total_count`,
  `interval_mqput_mqput1_total_bytes`, `interval_destructive_get_total_count`,
  `interval_destructive_get_total_bytes`, `interval_topic_put_total`,
  `topic_mqput_mqput1_interval_total`, `put_persistent_messages_bytes`,
  `put_non_persistent_messages_bytes`, `got_persistent_messages_bytes`,
  `got_non_persistent_messages_bytes`, `persistent_message_mqput_count`,
  `non_persistent_message_mqput_count`, `persistent_message_mqput1_count`,
  `non_persistent_message_mqput1_count`, `persistent_message_destructive_get_count`,
  `non_persistent_message_destructive_get_count`, `persistent_message_browse_count`,
  `non_persistent_message_browse_count`, `persistent_message_browse_bytes`,
  `non_persistent_message_browse_bytes`, `persistent_topic_mqput_mqput1_count`,
  `non_persistent_topic_mqput_mqput1_count`, `published_to_subscribers_message_count`,
  `published_to_subscribers_bytes`, `expired_message_count`, `purged_queue_count`, `commit_count`,
  `rollback_count`.
- **Subscriptions (mixed):** counters `alter_durable_subscription_count`,
  `create_durable_subscription_count`, `create_non_durable_subscription_count`,
  `delete_durable_subscription_count`, `delete_non_durable_subscription_count`,
  `resume_durable_subscription_count`, `failed_create_alter_resume_subscription_count`,
  `subscription_delete_failure_count`; gauges `durable_subscriber_high_water_mark`,
  `durable_subscriber_low_water_mark`, `non_durable_subscriber_high_water_mark`,
  `non_durable_subscriber_low_water_mark`.
- **Connections (gauge):** `concurrent_connections_high_water_mark`.

### 4.2 `queue` publication-driven (49 series)

Labels: `qmgr, queue, cluster, usage, description, platform`.

**Board-bound (Queue/channel view, spec §7.2):**

| Metric | Type | Board use |
|--------|------|-----------|
| `ibmmq_queue_depth` | gauge | Flow — **depth% status pill, depth-over-time trend, DLQ depth (red > 0)** |
| `ibmmq_queue_mqput_mqput1_count` | counter | Flow — **put** side of put-vs-get (`rate()`) |
| `ibmmq_queue_mqget_count` | counter | Flow — **get** side of put-vs-get (`rate()`) |
| `ibmmq_queue_mqput_bytes` / `ibmmq_queue_mqget_bytes` | counter | Flow — byte throughput (`rate()`) |
| `ibmmq_queue_rolled_back_mqget_count` | counter | Flow — **rollback (backout) rate** trend, get side (`rate()`) — see §6 note |
| `ibmmq_queue_rolled_back_mqput_count` | counter | Flow — rollback rate trend, put side (`rate()`) |
| `ibmmq_queue_average_queue_time_seconds` | gauge | Flow — latency companion to `qtime_*` |
| `ibmmq_queue_expired_messages` | counter | Flow — Inventory (expiry) |
| `ibmmq_queue_purged_count` | counter | Flow — Inventory (purges) |

**Full `queue` publication set:**

- **Gauges:** `depth`, `average_queue_time_seconds`, `avoided_percentage`,
  `avoided_puts_percentage`, `browse_handles`, `publish_handles`, `lock_contention_percentage`.
- **Put/get/browse counters (`rate()`):** `mqput_mqput1_count`, `mqput_bytes`,
  `mqput_persistent_message_count`, `mqput_non_persistent_message_count`,
  `mqput1_persistent_message_count`, `mqput1_non_persistent_message_count`, `mqget_count`,
  `mqget_bytes`, `destructive_mqget_persistent_message_count`,
  `destructive_mqget_non_persistent_message_count`, `destructive_mqget_persistent_bytes`,
  `destructive_mqget_non_persistent_bytes`, `mqget_browse_persistent_message_count`,
  `mqget_browse_non_persistent_message_count`, `mqget_browse_persistent_bytes`,
  `mqget_browse_non_persistent_bytes`, `persistent_bytes`, `non_persistent_bytes`.
- **Failure / rollback counters (`rate()`):** `destructive_mqget_fails`,
  `destructive_mqget_fails_with_mqrc_no_msg_available`,
  `destructive_mqget_fails_with_mqrc_truncated_msg_failed`, `mqget_browse_fails`,
  `mqget_browse_fails_with_mqrc_no_msg_available`,
  `mqget_browse_fails_with_mqrc_truncated_msg_failed`, `rolled_back_mqget_count`,
  `rolled_back_mqput_count`, `expired_messages`, `purged_count`, `intran_get_skipped`,
  `intran_put_skipped`.
- **Other operation counters (`rate()`):** `mqopen_count`, `mqclose_count`, `mqinq_count`,
  `mqset_count`.
- **`EXTENDED`-class counters (`rate()`):** `msg_search`, `msg_examine`, `msg_not_found`,
  `load_msg_dtl`, `correlid_mismatch_short`, `correlid_mismatch_long`, `msgid_mismatch`,
  `selection_mismatch`.

### 4.3 `nha` class (40 series) — Native HA replica statistics

Labels: `qmgr, platform, nha` (the `nha` label is the instance name, e.g. `nha-ubuntu-a1`).

**Documented here for class-completeness. The Wave-1a boards do NOT bind the `nha` class** — it is
consumed by the **Wave-2 Infra board** (Task 8), and only for its *trend* half. The Infra board's
discrete *status* (ROLE / QUORUM / GRPROLE / HASTATUS) binds the separate `cluster_nha_*`
custom-collector contract (`nha-crr-metric-contract.md`, Task 7), **not** stock `ibmmq_nha_*`
(phase0 §2 "Board 3 correction: hybrid"). Key trend signals: `ibmmq_nha_backlog_bytes` (HA replica
lag) and `ibmmq_nha_recovery_backlog_bytes` (CRR lag / RPO trend).

- **Backlog / lag (gauge):** `backlog_bytes`, `backlog_average_bytes`,
  `backlog_long_term_average_bytes`, `recovery_backlog_bytes`, `recovery_backlog_average_bytes`.
- **Log-sequence / rebase (gauge):** `acknowledged_log_sequence_number`,
  `recovery_log_sequence_number`, `recovery_rebase`.
- **Network / latency (gauge):** `average_network_round_trip_time`,
  `recovery_average_network_round_trip_time`, `log_write_latency_seconds`,
  `log_write_average_acknowledgement_latency`, `log_write_average_acknowledgement_size`,
  `log_write_size_bytes`, `log_slowest_write_since_restart`, `log_timestamp_of_slowest_write`,
  `catch_up_time_percentage`, `throttling_time_percentage`.
- **Filesystem (gauge):** `log_file_system_free_space_bytes`, `log_file_system_in_use_bytes`,
  `queue_manager_file_system_free_space_percentage`, `queue_manager_file_system_in_use_bytes`,
  `mq_fdc_file_count`.
- **Compression time (gauge):** `catch_up_log_data_average_compression_time_bytes`,
  `catch_up_log_data_average_decompression_time_bytes`,
  `recovery_log_data_average_compression_time_bytes`,
  `recovery_log_data_average_decompression_time_bytes`,
  `synchronous_log_data_average_compression_time_bytes`,
  `synchronous_log_data_average_decompression_time_bytes`.
- **Log bytes sent (counter, `rate()`):** `catch_up_compressed_log_sent_bytes`,
  `catch_up_uncompressed_log_sent_bytes`, `catch_up_log_sent_bytes`,
  `catch_up_log_decompressed_bytes`, `recovery_compressed_log_sent_bytes`, `recovery_log_sent_bytes`,
  `recovery_log_decompressed_bytes`, `synchronous_compressed_log_sent_bytes`,
  `synchronous_uncompressed_log_sent_bytes`, `synchronous_log_sent_bytes`,
  `synchronous_log_decompressed_bytes`.

## 5. Failover characteristic — the publication gap (a documented board-design input)

**During a Native HA failover the publication-driven catalog (§4) briefly reads zero, then
self-heals. This is expected, not a fault.** Observed live 2026-08-07: the active instance moved
(`a3 → a2`); the **client-mode** exporter had to reconnect to the new active instance and
**re-subscribe** to the `$SYS` resource-monitoring topics. During that window
`ibmmq_qmgr_exporter_publications` read `0` and the entire §4 catalog (CPU/RAM/MQI counts, per-queue
`depth` + put/get, `ibmmq_nha_*`) was absent — a **multi-minute gap** — until re-subscription
completed and the QM republished at the next monitoring interval. After the heal the exporter emitted
the full 238-name catalog with `exporter_publications` climbing (316+ and rising).

**The object-status catalog (§3) is unaffected** — PCF/command-queue polling continues across the
failover, so status bands, services, channel status, oldest-message-age, handles, and latency keep
reporting throughout.

**Board-design impact (spec §6, "never ship a silently-empty panel"):** trend-band panels bound to
§4 must **tolerate a failover gap** and not render it as "broken." A multi-minute hole in
`depth`/put-vs-get/message-rate during a failover is the system behaving correctly, not a dead panel.
Design inputs for the board work (#964/#965/#967):

- Prefer `connect nulls` / gap-tolerant rendering on §4 trend panels; do not alert on a short
  publication gap alone.
- Use `ibmmq_qmgr_exporter_publications` as the **collection-health signal** — a panel/annotation on
  it explains a §4 gap as "failover re-subscription in progress," turning a scary blank into a
  legible event.
- Lead health judgments that must survive a failover with the failover-resilient §3 object-status
  series; treat §4 trends as the richer-but-interruptible layer.

## 6. NOT in Prometheus — sourced from the ES event feed (Wave 1b), never PromQL

Two Queue/channel-board signals are **not stock `ibmmq_*` series** and must **never** be written as
PromQL. They come from the **Elasticsearch event feed** (Wave 1b, `es-doc-field-contract.md`),
re-confirmed live in Phase 0 and in spec §7.2. A full-catalog grep on the live exporter for
`backout`/`in_doubt`/`indoubt` returns **nothing**.

- **Per-queue backout (BackoutCount / BOTHRESH breach)** — **not** a stock series. The signal the
  Attention table wants is *"which queue has messages that exceeded the backout threshold"* — a
  per-message `MQMD.BackoutCount` vs `BOTHRESH` condition surfaced via MQ **events**, which the
  exporter does not emit. It comes from the **ES event feed**.
  - *Precision note:* the exporter **does** emit stock STATQ rollback-*operation* counters —
    `ibmmq_queue_rolled_back_mqget_count` / `_mqput_count` (§4.2) — so the Flow board's *"backout
    rate"* **trend** can bind to `rate(ibmmq_queue_rolled_back_mqget_count)`. But that is the rate of
    rolled-back MQGET/MQPUT operations, **not** the per-queue backout-threshold breach; the Attention
    "these messages are stuck in a poison-message loop" signal still comes from ES.
- **In-doubt (channel / UOW in doubt)** — **not** a stock series. Channel Attention floats in-doubt
  from the **ES event feed**, not from `ibmmq_channel_*`.

Any board panel needing the backout-threshold breach or in-doubt state binds to `$logs`
(Elasticsearch) via the Wave-1b ES contract, not to `$datasource` (Prometheus). This is the hard
boundary between the metric boards (Wave 1a) and the event feed (Wave 1b).

## 7. Sources

- **Live exporter:** `http://10.50.0.3:9163/metrics` (QM `NHAUAPP`), scraped 2026-08-07 — the full
  238-name catalog (§3/§4), the exact names/types/labels, and the failover-gap evidence in §5.
- **`phase0-observability-config.md`** (this epic, 2026-08-06) — §1 mechanisms, §2 available-vs-
  collected (~200 names incl. ~40 `ibmmq_nha_*`), §3 config levers + the coverage-is-security grant
  finding, §4 exporter collection model (`useStatus` forced on, runtime-generated names,
  `overrideCType`), §5 the `mqmon` least-privilege grant model.
- **`spec.md`** §3.2 (schema-note contract), §5 (template variables), §7.1/§7.2 (board structure +
  the counter/gauge verification and the backout/in-doubt → ES boundary).
- **Exporter source:** `ibm-messaging/mq-metric-samples` (`pkg/config/config.go`) and
  `ibm-messaging/mq-golang/mqmetric/` — names generated at runtime, no static manifest (phase0 §4).
