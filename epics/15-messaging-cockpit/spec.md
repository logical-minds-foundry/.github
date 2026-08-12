# Design: Messaging-layer cockpit — MQ flow + live application round-trip

- **Status:** Approved (brainstorm output, pushback-reviewed)
- **Date:** 2026-07-01
- **Author:** Phillip Moore (with Claude)
- **Epic:** `logical-minds-foundry/.github#15` (this doc is the epic spec)
- **Relationship:** the messaging-layer peer of the cluster cockpits
  (`src/mqlab/clusterboard.py`, #219/#279/#287/#417). Those show the
  infrastructure *under* MQ; this shows what is happening *inside* MQ.

---

## 1. Why this exists

The cluster cockpits answer *"is the infrastructure under MQ healthy?"* — nodes,
HA role, quorum, replication, storage. They hit a sweet spot: a status band, node
matrices, a failover timeline, and integrated logs, all on one screen.

This epic builds the **next layer up**: *"is messaging actually flowing?"* A
per-stack dashboard that shows the queue managers, queues, and channels **plus**
the live application round-trip — request→reply rate, failures, latency, and the
actual app/service log lines — using the same design DNA, retargeted from
`cluster_*`/infra metrics to `ibmmq_*` MQ metrics and a new app round-trip signal.

A precondition falls out of this: for the cockpit to have anything to show, the
**workload must be live**. So this epic also makes the application and service
run continuously by default — a steady stream of request/reply messages flowing
through every stack from the moment it boots.

## 2. Scope and epic decomposition

This is a **finite epic**. Near-term tasks (this spec details Task ③; Tasks ①–②
are its prerequisites, surfaced during pushback review):

- **① Per-stack MQ exporter + scrape (prerequisite).** Today there is one
  exporter pair on fixed ports 9157/9158 for the single active stack, and the
  Prometheus scrape config is hardcoded to those ports (§3.1). For per-stack
  boards to all show data — and to honor the **all-four-stacks-concurrent** goal
  — deploy one exporter pair per stack on the topology alloc ports (9157–9164)
  and render the Prometheus scrape config per-stack. The exporter **wildcards all
  objects** (§3.1).
- **② Always-on workload + round-trip signal (prerequisite).** Promote
  `app_requester` from a manually-run client into a managed, continuously-
  streaming systemd service that emits round-trip **logs** (→ Loki) and a
  round-trip **metric** (→ Prometheus) (§3.2, §6). Independently valuable: live
  traffic + metrics + logs, provable before any board consumes them.
- **③ First-level flow overview board (this spec's headline deliverable).** The
  per-stack messaging-flow overview, consuming ① and ②.
- **④ (future) per-QM detail dashboard** (`lab-messaging-qm`).
- **⑤ (future) per-channel detail dashboard** (`lab-messaging-channel`) — channel
  state integrated with network metrics.
- **⑥ (future) per-queue detail dashboard** (`lab-messaging-queue`) — depth, age,
  storage.
- **⑦+ (future) workload realism** (§6.3): burst/variable patterns, slow-network
  simulation to force queuing, DR message-loss scenarios, realistic message
  profiles.

Tasks ①–③ are specified here; ④+ are named, not detailed. The overview (③) is
built with the **drill-down seam designed in** — queue/channel rows and flow-strip
hops carry Grafana data links to the (future) detail-dashboard UIDs — but the
detail dashboards themselves are later tasks. This mirrors the clusters, which
also have no drill targets yet.

## 3. Telemetry

### 3.1 MQ metrics — wildcard-scraped, per stack

The `mq_prometheus` exporter (`ansible/roles/mq-exporter`) runs client-mode
against the app QM and the SVC QM. Two corrections land in Task ① (both are
current-state gaps, not board work):

1. **Per-stack instances + scrape.** Deploy one exporter pair per stack on its
   alloc ports (9157–9164, from `lab/topology.yaml` `alloc:`), and render the
   Prometheus scrape config per-stack (today `site-obs.yml` and
   `prometheus.yml.j2` hardcode 9157/9158 — pcmk only). This is what lets every
   per-stack board light up, including all four concurrently.
2. **Wildcard all objects.** The exporter monitors **all** queues and channels,
   **including `SYSTEM.*`** — not a cherry-picked list. Filtering is a *display*
   concern, never a *scrape* concern. This removes any exporter↔board drift and
   gives the system-object series for free — e.g. `SYSTEM.CLUSTER.*` queue churn,
   a proven leading indicator of clustering problems. At lab scale the extra
   cardinality is negligible; production-style object filtering is a decision we
   deliberately do **not** make here.

Metrics are **QM-keyed** (labels `qmgr` / `queue` / `channel`), so they follow a
failover automatically. The board reads (non-exhaustive, since everything is
scraped): `ibmmq_qmgr_status`, `ibmmq_qmgr_connection_count`, the QM interval
put/get counts, `ibmmq_channel_status_squash`, `ibmmq_queue_depth`,
`ibmmq_queue_mqput_mqput1_count` / `ibmmq_queue_mqget_count`.

### 3.2 The application round-trip signal (Task ②)

`clients/app_requester.py` is currently copied to a node and run by hand
(`site-distributed-shared.yml:67`). Task ② makes it a managed service and adds two
lab-native signals:

1. **Logs → Loki.** Run `app_requester` as a systemd service (unit
   `mq-app-requester`) so its `[N] round-trip <ms> <- <reply>` lines flow
   journald → Alloy → Loki, exactly like `svc-responder`.
2. **Metrics → Prometheus (node-exporter textfile).** The requester writes a
   node-exporter textfile (`/var/lib/node_exporter/textfile/`, which the role
   already creates): counters `app_roundtrip_total` and
   `app_roundtrip_failures_total`, plus round-trip **latency as a histogram**
   (`app_roundtrip_latency_ms_bucket/_sum/_count`). `app-client` already runs
   node-exporter (scraped by the `node` job), so **no new scrape config** for this
   signal. The histogram — not a single gauge — is what feeds the flowing latency
   heatmap and, later, percentiles (§5.4).

### 3.3 Acknowledged gaps (not filled here)

Per-channel *message rates* (only QM-level rates exist) and AMQERR file-based
error logs (not tailed) remain gaps; the overview does not depend on them. They
are candidates for the future detail dashboards.

## 4. Architecture

### 4.1 A new module, sharing the cluster toolkit

`src/mqlab/messagingboard.py` — a **new module** importing the clusterboard
primitives (`matrix`, `_stat`, `_timeseries`, `_state_timeline`, `_logs_panel`,
`_title_banner`, `_row_header`, the color-mapping vocabulary, `_log_level_var`).
`clusterboard.py` is already ~1500 lines; the messaging board is a different
*layer* (MQ objects, not infra); and a clean, QM-parameterized module serves the
component-extraction roadmap (#368).

### 4.2 One board per stack, from a spec map

The `_NHA_ARM_SPEC` pattern. A `_MSG_STACK_SPEC` maps each stack to its messaging
identity (`app_qm`, `svc_qm`, groups, uid, title). QM names come from each stack's #351 short-derived `qm_app`/`qm_svc`; channel/queue names are lab constants — no
QM literal hardcoded in the builder. Renders `lab-messaging-pcmk`,
`lab-messaging-rdqm`, `lab-messaging-nativeha-rhel`, `lab-messaging-nativeha-ubuntu`.

### 4.3 Render + deploy path

Same as the cluster boards: `cli.py` render sites write `lab-messaging-<stack>.json`
into `build/work/grafana/dashboards/`; the `grafana` role copies each into place;
one board provisioned per stack. `lab-status` gains a drill link to each stack's
messaging board; deeper dedup of `lab-status`'s existing MQ section is out of scope.

## 5. Layout — flow-oriented (top → bottom)

### 5.1 Title banner

`Messaging Layer · <STACK> · app-client ⇄ <APP_QM> ⇄ <SVC_QM> ⇄ svc-sim`.

### 5.2 ① Status band — flow indicators (not depths)

Compact tiles for **flow health**: App QM up · SVC QM up · round-trip success % ·
message rate (msg/s) · error/failure rate. Deliberately **no summed queue depth** —
per-queue depth is a per-queue concern (queues matrix / future per-queue board),
not a meaningless aggregate.

### 5.3 ② The message flow

A horizontal strip of **positioned status tiles** (built from `_stat` + text
connectors, not a custom viz, so it stays generic): `app-client → APP.SVRCONN →
[APP QM] → SDR chl → [SVC QM] → SVC.SVRCONN → svc-sim`, with the return
`APP.REPLY` leg. Each hop colors on its own metric (channel status squash / queue
depth). Signature panel; iterate on its look after first render.

### 5.4 ⟳ Round-trip timeline (the signature panel)

The failover-timeline analog for messaging: a left-to-right flowing time series of
**message rate f(t)** and **failure rate f(t)**, with round-trip **latency** as a
heatmap/histogram over the same window. This is where burst workloads, induced
queuing (slow network), and DR message-loss (§6.3) become visible. Exact
visualization is settled by experiment during build; the histogram metric (§3.2)
keeps rate/failure/latency/percentile options all open.

### 5.5 ▤ Round-trip logs

`_logs_panel` over `{unit=~"mq-app-requester|mq-svc-responder"} |~ ${level}` (the
shared `$level` toggle), interleaving the requester's round-trip lines with the
responder.

### 5.6 Queues & Channels matrices

Queues: depth · put/s · get/s per queue (each row data-links to
`lab-messaging-queue`, future). Channels: status · type per channel (each row
data-links to `lab-messaging-channel`, future). Because scraping is wildcard, these
matrices can optionally surface `SYSTEM.*` objects too (e.g. cluster system-queue
depth) — display-filtered, not scrape-filtered.

## 6. The always-on workload (Task ②)

### 6.1 Both ends run continuously, by default

Both `svc-responder` (already a service) and `app-requester` (promoted from a
manual client) are managed systemd services, **enabled + started on provision**,
so every stack boots with a steady nonstop request→reply stream and the cockpit
shows live data immediately.

### 6.2 v1 is deliberately laminar

v1 streams at a **fixed rate with a fixed message shape** — "good enough" noise to
make the board live, explicitly *not* a faithful model of the real application.

### 6.3 Designed for iteration (config seam) — the interesting future

The requester carries a small config seam (**rate**, **message size/shape**, a
**fault hook**) so the workload-realism tasks (⑦+) drop in without a rewrite:

- **Burst / variable-rate patterns** — bursting messages fast is how queues build;
  the §5.4 timeline is what shows that queuing behavior.
- **Slow-network simulation** — deliberately throttle a link to force queuing and
  study back-pressure.
- **DR message-loss scenarios** — combine induced queuing with a DR cutover to
  simulate and measure message loss during failover.
- **Realistic message profiles** — sizes/shapes closer to the real app.

None of these are built in v1; the seam and the histogram-backed timeline just
keep them cheap to add.

## 7. Drill-down seam (designed, not built)

Queue/channel matrix rows and the flow-strip hop tiles carry Grafana **data links**
to the future detail-dashboard UIDs (`lab-messaging-qm`, `lab-messaging-channel`,
`lab-messaging-queue`). Targets are stubs until Tasks ④–⑥.

## 8. Testing & acceptance

- **Unit** — `tests/test_messagingboard.py`: per-stack parameterization (right QM
  names in the right selectors, no cross-stack leak); flow-strip tiles resolve the
  correct `channel=`/`queue=` labels; status band uses rate/failure metrics (no
  depth-sum); the logs panel selector covers both app + svc units; drill-down data
  links carry the correct target UIDs. A unit test for the round-trip textfile
  emitter's histogram/counter format. Task ①: a test that the rendered scrape
  config covers every stack's alloc ports and the exporter args wildcard objects.
- **Gate** — `vrg-container-run -- vrg-validate` green, incl. 100% branch coverage.
- **Live acceptance (the human's gate)** — a cold/warm-rebuild bring-up shows the
  board rendering real data with the steady stream flowing (cold-rebuild
  acceptance gate). Lint-green ≠ done for the lab-facing half.

## 9. Non-goals (v1)

- The three detail dashboards (Tasks ④–⑥) — seam only.
- Workload realism (Tasks ⑦+) — laminar stream only; seam only.
- AMQERR log tailing and per-channel message-rate metrics (§3.3).
- Reworking `lab-status`'s existing MQ section beyond adding a drill link.

## 10. References

- `src/mqlab/clusterboard.py` — primitives + the `_NHA_ARM_SPEC` pattern.
- `ansible/roles/mq-exporter`, `ansible/site-obs.yml`,
  `ansible/roles/prometheus/templates/prometheus.yml.j2` — the exporter + scrape
  layer Task ① reworks.
- `clients/app_requester.py`, `clients/svc_responder.py` — the workload.
- `ansible/roles/alloy` — journald → Loki shipping.
- `ansible/roles/node-exporter` — textfile collector dir (round-trip metric sink).
- Component-extraction roadmap #368 — why this is its own module.
