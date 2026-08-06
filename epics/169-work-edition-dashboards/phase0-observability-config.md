# Phase 0 — MQ Prometheus observability config verification & least-privilege grant model

- **Epic:** `logical-minds-foundry/.github#169` · **Phase-0 task:** `#173`
- **Status:** research & live verification complete (2026-08-06); recommendations feed #173 implementation
- **Method:** the running `nativeha-ubuntu` lab (live `mq_prometheus` exporter + `dspmq`/`runmqsc` +
  `amqsrua`) cross-checked against **IBM MQ 9.4 docs** and the **`ibm-messaging/mq-metric-samples` /
  `mq-golang` source**. A credibility artifact (spec §3.1): claims trace to a live probe, an IBM 9.4
  `source_url`, or a source file+line.

## 0. Executive summary

Verifying (not assuming) the config was the right call — it produced two **corrections** to
mid-session claims and a set of **real, actionable gaps**, and it is what turned up two security bugs
(#935, #936) already fixed.

- **The exporter collects far more than the first (broken-exporter) snapshot implied** — CPU, DISK,
  STATMQI, STATQ resource publications; QSTATUS/CHSTATUS object status; topics; subscriptions; and the
  full Native HA replica/recovery statistics — ~200 distinct `ibmmq_*` metric names on this QM.
- **Corrections (own them):** `useStatus` is **not off** — it is deprecated and **forced on** in the
  exporter; and `ibmmq_nha_*` **does** flow from the stock exporter (the earlier "none" was purely the
  crash-looping exporter of #936).
- **Genuine maximal-coverage gaps** remain: accounting (needs a *separate* collector), per-queue
  coverage is publication-driven (idle queues can be invisible), the `EXTENDED` queue class is excluded
  by default, `showInactiveChannels` is off, statistics *messages* are not consumed, and STATAPP is not
  collected.
- **The security/grant model is undocumented upstream** — the exporter repo specifies *what objects it
  touches* but ships **no `setmqaut`**. §5 pins the least-privilege model empirically.

## 1. The three monitoring mechanisms (who reads what)

A "maximal" deployment must treat these as **three distinct sources**, not one:

| # | Mechanism | How the exporter gets it | IBM 9.4 source |
|---|-----------|--------------------------|----------------|
| 1 | **Resource-monitoring publications** (`$SYS/MQ/INFO/QMGR/...`, the `amqsrua` data) | pub/sub subscribe | `topic=stmat-monitoring-system-resource-usage-by-using-amqsrua-command`; `topic=trace-metrics-published-system-topics` |
| 2 | **Real-time object status** (`DISPLAY QSTATUS` / `CHSTATUS`) | PCF poll over `SYSTEM.ADMIN.COMMAND.QUEUE` | `topic=reference-display-qstatus-display-queue-status`; `topic=reference-display-chstatus-display-channel-status` |
| 3 | **Statistics & accounting messages** (`SYSTEM.ADMIN.STATISTICS.QUEUE` / `.ACCOUNTING.QUEUE`) | **NOT consumed by default** — needs `useStatistics` (statistics only) or a separate `amqsmon`-style collector | `topic=messages-accounting-statistics-message-reference` |

Publication cadence is subscriber-driven (~10 s; the true interval is in each message's
`MQIAMO64_MONITOR_INTERVAL`).

## 2. Available vs collected (live, this QM)

**What the QM publishes (`amqsrua` classes, live):** `CPU`, `DISK`, `STATMQI`, `STATQ`, `STATAPP`,
`NHAREPLICA`.

**Current QM monitoring posture (`DIS QMGR`, live):** `MONQ(MEDIUM)` · `MONCHL(MEDIUM)` ·
`MONACLS(QMGR)` · `STATMQI(ON)` · `STATQ(ON)` · `STATCHL(MEDIUM)` · `STATINT(1800)` · **`ACCTMQI(OFF)`**
· **`ACCTQ(OFF)`** · `ACCTINT(1800)`. So real-time monitoring (Mech. 2) and statistics-message
production (Mech. 3) are **on**; accounting is **off**. (Note the QM *initial* defaults for all of
MONQ/MONCHL/STATMQI/STATQ/ACCT* are `OFF` — the lab has deliberately enabled most of them.)

**What the exporter emits (live, ~200 names):** rich `ibmmq_qmgr_*` (CPU, RAM, log/DISK, all MQI call
counts and byte counters, subscriptions), `ibmmq_queue_*` (depth, oldest-message-age, handles,
uncommitted, get/put/browse counters — QSTATUS **and** STATQ fields), `ibmmq_channel_*` (status,
substate, xmitq-time, nettime, bytes/buffers/batches), `ibmmq_topic_*`, `ibmmq_subscription_*`, and
**~40 `ibmmq_nha_*`** (replica + recovery-group stats, incl. `ibmmq_nha_recovery_backlog_bytes` = the
CRR lag / RPO trend, and `ibmmq_nha_backlog_bytes` = HA replica lag).

**Board 3 correction:** the infra board is a **hybrid** — lag/throughput/latency *trends* come from
stock `ibmmq_nha_*`; only the discrete *status* (ROLE / QUORUM / GRPROLE Live·Recovery / HASTATUS)
still needs the `dspmq -o nativeha` collector (#79).

## 3. Genuine gaps & the maximal-coverage levers

Per the "collect everything, then prune" strategy, each lever is annotated keep/prune. Levers are
exporter config unless noted; config keys per `mq-metric-samples/pkg/config/config.go`.

| Lever | Now | To maximize | Yields | Mechanism / reader | Keep / prune |
|-------|-----|-------------|--------|--------------------|--------------|
| **Accounting** `ACCTMQI`/`ACCTQ` | OFF | `ALTER QMGR ACCTMQI(ON) ACCTQ(ON)` **+ build an accounting collector** (exporter can't read the accounting queue — confirmed in source) | per-connection & per-queue MQI op counts/bytes/times by app/user | Mech. 3 → **NEW collector** | **follow-on, skip for MVP** — statistics are Prometheus-native, accounting is not (dedicated add-on); not needed at the target employer now (filed as a separate task) |
| **Queue coverage / idle visibility** | most queues invisible | grant `mqmon +dsp +inq` over the queue namespace (§3 key finding) | all queues incl. backed-up idle ones (`oldest_message_age` while idle) | Mech. 1/2 → exporter + authz | **RESOLVED** — it was authority, not activity |
| **`EXTENDED` queue class** | excluded by default (`queueSubscriptionSelector`) | add `EXTENDED` | 9.4.2+ L2/L3 queue diagnostics (msg-search/examine/skip internals) | Mech. 1 → exporter | **keep** |
| **`showInactiveChannels`** | `false` | `true` | defined-but-inactive / **stopped** channels (a dashboard-critical signal) | Mech. 2 → exporter | **keep** |
| **Statistics *messages*** (`useStatistics`) | not consumed | `useStatistics=true` **or** an `amqsmon` collector | STATMQI/STATQ/STATCHL interval records | Mech. 3 | **likely prune** — overlaps the pubs the exporter already has; forces `monitoredQueues=*` |
| **STATAPP** (per-app) | not collected | (subscriber-driven; tied to uniform-cluster app balancing) | per-application instance stats | Mech. 1 → exporter | **N/A this topology** (Native HA, not a uniform cluster) |
| **`MONCHL` level** | MEDIUM | `HIGH` | higher channel-status sampling rate | Mech. 2 → QM attr | **consider** (MONQ has no level distinction; MONCHL does) |
| **AMQP / MQTT channels** | off | `monitoredAMQPChannels` / `monitoredMQTTChannels` | protocol channel status | Mech. 2 → exporter | **out of scope** — the lab does not use AMQP or MQTT and will not until a real-world need arises (decision 2026-08-06); leave off |

**Key finding — coverage IS the security model (RESOLVED 2026-08-06).** Despite
`monitoredQueues *,SYSTEM.*`, most queues were invisible — **because the least-privilege `mqmon`
identity only held `+dsp` on `APP.REPLY`**; the exporter cannot inquire a queue it has no authority on.
Proof: defining `TEST.IDLE.PROBE`, granting `mqmon +dsp +inq` via **`SET AUTHREC`**, and forcing
rediscovery made it appear immediately (11 series) reporting `oldest_message_age` climbing while
**idle** (67 s, no activity) — so `useStatus`/QSTATUS **does** cover idle-but-nonempty queues. It was
*authority*, not publication-gating. **Grant `mqmon +dsp +inq` across the monitored queue namespace and
every queue — including backed-up idle ones — is visible.** This is precisely the least-privilege
**MQMon security profile** (§5) — the opposite of the industry "run the monitor as `mqm`" cop-out.
Caveat: `rediscoverInterval` defaults to **1h**, so newly-created queues lag until rediscovery — lower
it or accept it for stable topologies.

## 4. Sourcing note on the exporter's collection model

- `useObjectStatus`/`useStatus` is **deprecated and forced on** (`VerifyConfig` hard-sets
  `UseStatus=true`; `pkg/config/config.go:383–388`). QSTATUS/CHSTATUS/TPSTATUS/SBSTATUS collection is
  therefore always active.
- Metric names are **generated at runtime** from publication descriptions (`mq-golang/mqmetric/mapping.go`
  `FormatDescriptionHeuristic` + the `mHeur` map) and coded status constants — there is **no static
  manifest**; enumeration = scrape-and-grep (as done here). `mqmetric/metrics.txt` documents classes,
  not concrete series.
- `overrideCType=true` (default this major version) splits counters vs gauges correctly — consistent
  with the live `# TYPE` checks (put/get = counters → `rate()`).

## 5. The MQMon least-privilege security profile (a first-class deliverable)

**The architectural stance:** the industry default is to run the monitor as `mqm` (superuser) and skip
authority entirely — a cop-out that hands a monitoring tool full control of the queue manager it
watches. This lab does it **right**: a scoped, documented, least-privilege identity (`mqmon`, mapped
from `MON.SVRCONN` via SSLPEERMAP) with exactly the authorities the exporter needs and nothing more.
The exporter repo ships **no `setmqaut`** at all (whole-tree grep is empty) — it documents only the
*objects touched* — so this profile fills a real upstream gap and is a reusable, portable deliverable.

**Expressed as `SET AUTHREC` (MQSC), not `setmqaut` (CLI):** they are equivalent, but `SET AUTHREC`
runs **remotely** and lives **in the same templated MQSC as the object definitions**, so authority
travels with the objects (rule, 2026-08-06). Model pinned empirically (#936, the §3 coverage test) +
the object list, per data source:

| Data source | Object | Authority | Confirmed |
|-------------|--------|-----------|-----------|
| all | qmgr | `+connect +inq +dsp` | #936 live |
| Mech. 2 (PCF status/discovery) | `SYSTEM.ADMIN.COMMAND.QUEUE` | `+put` | #936 live |
| Mech. 2 (PCF replies) | `SYSTEM.DEFAULT.MODEL.QUEUE` (or configured `replyQueue`) | `+dsp +inq +get` | #936 |
| Mech. 1 (resource pubs) | `SYSTEM.ADMIN.TOPIC` (the `$SYS` tree) | `+sub` | #936 live (replaced the wrong `SYSTEM.BASE.TOPIC` placeholder) |
| Mech. 1/2 (per-queue) | monitored queue namespace (`SET AUTHREC PROFILE('**') OBJTYPE(QUEUE)`, or per-critical-queue) | `+dsp +inq` | **confirmed — this is the coverage gate (§3); without it a queue is invisible** |
| Mech. 2 (channel status) | channels via the command server (PCF) | *covered by command-queue `+put`* — no per-channel authrec needed | live (4 channels visible without per-channel grants) |
| Mech. 3 (statistics), *if* `useStatistics` | `SYSTEM.ADMIN.STATISTICS.QUEUE` | `+get` | to add |
| Mech. 3 (accounting collector) | `SYSTEM.ADMIN.ACCOUNTING.QUEUE` | `+get` (the collector's own identity) | to add |

**QM-object tuning (not grants), per the exporter README/TUNING:** `MAXHANDS` high enough for the
non-durable subscriptions (or use `durableSubPrefix`); `replyQueue` model `MAXDEPTH` ≈ 1 min of
publications, `DEFPSIST(NO)`; **`USEDLQ(NO)` on `SYSTEM.ADMIN.TOPIC`** to avoid flooding the DLQ.

## 6. Open verification items → next steps

1. **Idle-queue visibility — RESOLVED (§3).** It was `mqmon` authority, not activity. `TEST.IDLE.PROBE`
   granted via `SET AUTHREC` appeared with `oldest_message_age=67 s` while idle. Fix = the namespace
   grant in §5/§7.
2. **Accounting collector — deferred to a filed follow-on task.** An `amqsmon`-based non-MQI collector
   reading `SYSTEM.ADMIN.ACCOUNTING.QUEUE` (the #79 pattern) + `ACCTMQI/ACCTQ` enablement. Not built in
   the MVP.
3. **Confirm live defaults Agent A could only source from older docs:** `STATINT`/`ACCTINT`=1800,
   `ACCTQ`=OFF, `STATACLS` default — `DISPLAY QMGR` on a 9.4 QM.
4. **STATAPP** — requires a uniform cluster; out of scope for the Native HA topology.

## 7. Recommendations (feed #173 implementation)

**Posture — open the fire hoses (lab), then dial back.** Enable everything collectible and observe;
the worst case is a full disk, then we prune. Maximal coverage first, tuning second.

- **Grant `mqmon` the monitored-queue namespace** (`SET AUTHREC PROFILE('**') OBJTYPE(QUEUE)
  GROUP('mqmon') AUTHADD(DSP, INQ)`) — this is the **coverage fix** (§3) *and* the MQMon security
  profile (§5) in one. Idle queues then appear; no `mqm` cop-out.
- **Adopt a YAML config file for the exporter (DECIDED 2026-08-06).** Migrate the `mq-exporter` role
  from ExecStart CLI flags to a rendered `mq-exporter-<qm>.yaml`, with the unit invoking
  `mq_prometheus -f <yaml>`. Rationale: (1) `EXTENDED` and the per-class filters have **no CLI flag**,
  so YAML is the only clean way to reach them; (2) the fire-hose option explosion makes a stack of
  `-ibmmq.*` flags unmanageable; (3) **work already runs a YAML config**, so the lab's config transfers
  directly — the YAML becomes part of the **portable deliverable** alongside the dashboards JSON and the
  grant contract. The launcher (systemd on `mon-probe` vs an MQ `SERVICE` object) is orthogonal; stays
  systemd for the off-cluster client-mode exporter.
- **Enable in that YAML / QM config:** `EXTENDED` queue class; `showInactiveChannels=true`;
  `MONQ(HIGH)`/`MONCHL(HIGH)`/`STATCHL(HIGH)` (applied live 2026-08-06); lower `rediscoverInterval` so
  new queues are picked up promptly.
- **Accounting → follow-on, skipped for the MVP.** Statistics are Prometheus-native (the exporter
  collects them); **accounting is not** — it needs a dedicated add-on collector (`amqsmon`, #79
  pattern). It is not needed at the target employer now, so per minimal-MVP-complexity it is deferred
  to a separate task (filed), not built here.
- **Ship the MQMon security profile (§5) as templated `SET AUTHREC`** co-located with the object MQSC —
  the portable, documented least-privilege contract; fills a genuine upstream gap.
- **Prune later:** revisit `useStatistics` (redundant with pubs) and any high-cardinality series once
  live.

## 8. Sources

IBM MQ 9.4 (cached under `build/refs/ibm-docs/ibm-mq/9.4.x/`, `source_url` in each `meta.json`):
amqsrua, published-system-topics, DISPLAY QSTATUS, DISPLAY CHSTATUS, real-time-monitoring &
attributes-that-control, accounting-statistics-message-reference, ALTER QMGR. Exporter source:
`ibm-messaging/mq-metric-samples` (`pkg/config/config.go`, `cmd/mq_prometheus/{config,exporter}.go`,
`config.common.yaml`, `TUNING.md`, `metrics.txt`) and `ibm-messaging/mq-golang/mqmetric/`
(`mqif.go`, `mapping.go`, `status.go`, `discover.go`). Live evidence: `nativeha-ubuntu`, 2026-08-06.
