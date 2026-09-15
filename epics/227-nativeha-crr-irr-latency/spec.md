# Dual-mechanism Native HA — CRR (async) vs IRR (strict-sync), latency-tunable — design spec

- **Epic:** `logical-minds-foundry/.github#227`
- **Documentation task:** `logical-minds-foundry/.github#228` (this spec + the plan)
- **Prior art (arm we extend):** `logical-minds-foundry/.github#88` (native-HA RHEL box),
  `#219` (MQ 9.4.5→10.0 Native HA CRR upgrade + version centralization)
- **Status:** design (brainstorm output), pending pushback + human review
- **Date:** 2026-09-15

## 1. Problem & motivation

The Native HA arm today runs **one** queue manager per stack, replicated **CRR**
(Cross-Region Replication — asynchronous) from a Live group to a Recovery group in a
second region. Async CRR keeps replication off the commit path, so it costs almost
nothing in latency — but it leaves a **message-loss window** equal to the replication
lag on an unplanned failover. For a settlement-style workload, losing messages during a
DR event is a serious, expensive problem.

IBM MQ 10.0 also offers **IRR** (In-Region Replication) with **`SyncConsistency=Strict`**:
the Live instance will not acknowledge a commit until the Recovery group has durably
logged the data. That yields **RPO 0** even on unplanned failover — no message loss —
at the cost of putting a cross-site round-trip on the **critical path of every persistent
commit**. IBM documents IRR strict-sync as usable at roughly **5 ms** inter-site latency.

**That 5 ms is a single operating point taken off a trade-off surface, not a hard limit.**
Where the surface sits depends on message size, offered rate, and volume — the same way
"N queue managers per appliance" collapses the difference between a few large busy queue
managers and a swarm of tiny idle ones. When a vendor publishes one boiled-down number,
the data behind it is gone. **This epic exists to recover that curve** — to measure what
5 ms, 10 ms, 20 ms actually cost, find the knee where strict-sync throughput falls off,
and let an application/infrastructure owner choose an operating point from evidence
rather than accept a default. A team that can absorb the tax gets no-message-loss DR;
this lab lets them see the tax before they sign up for it.

*(Data vs judgment: IBM's ~5 ms guidance and the CRR/IRR taxonomy are cited from the
IBM MQ 10.0 docs the lab has cached — §2. The commit-path/throughput reasoning in this
section is **a-priori physics (judgment)**; replacing it with measured numbers is the
epic's whole point.)*

## 2. Terminology & grounding

### 2.1 CRR vs IRR (IBM MQ 10.0)

Pinned from the cached IBM 10.0 docs (`build/refs/ibm-docs/ibm-mq/10.0.x/`, per
`docs/reports/2026-09-14-mq10-nativeha-crr-upgrade-facts-spike.md:107-113`):

- A **Native HA group** is three log-replicating instances (one active), **synchronous**
  raft replication **within** the group, over the dedicated heartbeat NIC.
- **CRR (Cross-Region Replication)** pairs a **Live** group with a **Recovery** group in
  *different regions* over **asynchronous** replication. *This is the arm the lab has.*
- **IRR (In-Region Replication)** is the same Live/Recovery two-group model for two data
  centres *within one region*, close enough to run **synchronous strict consistency**
  (`SyncConsistency=Strict`). Not supported in containers (irrelevant here — the lab runs
  on VMs).

The lab already caches IBM's IRR *upgrade* doc (`irr-upgrading-native-ha-configurations`)
but **not** an IRR *setup/config* reference. Confirming the exact IRR bring-up mechanics
against IBM 10.0 is therefore the first task (§9, Task 1) — this spec does **not** assert
IRR config details it has not verified.

### 2.2 The current CRR arm (established from code)

- **Stack** `nativeha-rhel` — `short: NHAR`, QM **`NHARAPP`**, `provision:
  ansible/site-nativeha.yml`, cutover via `site-nativeha-switchover.yml`
  (`lab/topology.yaml:446-464`). QM names are **derived** from `short`
  (`<short>APP`), never literal (`lab/topology.yaml:398-399`, `src/mqlab/stacks.py`).
- **Nodes** `nha-rhel-a1/a2/a3` (site A, Live) + `nha-rhel-b1/b2/b3` (site B, Recovery),
  `rhel/9.6-x86_64`, shared-nothing (no `extra_disk` — raft-log replication).
- **Networks** (`lab/networks/*.xml`, `lab/topology.yaml:284-372`): `net-hb-a`
  (172.16.1.0/24) / `net-hb-b` (172.16.2.0/24) carry **HA replication on port 9414**;
  `net-wan` (10.99.0.0/24, bridge **`virbr-wan`**) carries **CRR replication on port
  9415**; `net-data-a/b`, `net-mgmt`. net-wan is isolated (no `<forward>`/DHCP) and its
  XML comment already earmarks it for netem.
- **qm.ini stanzas** written by the shared role:
  - `NativeHAInstance` (per node): `Name`, `ReplicationAddress=<hb-ip>(9414)` — intra-group
    synchronous HA (`ansible/roles/mq-nativeha/tasks/main.yml:78-88`).
  - `NativeHALocalInstance`: `CipherSpec`, `CertificateLabel`, `KeyRepository`,
    `GroupName`, `GroupRole`, `GroupLocalAddress=(9415)` — CRR local identity
    (`crr.yml:15-27`).
  - `NativeHARecoveryGroup`: `GroupName` (peer), `ReplicationAddress=<ip>(9415),…`,
    `Enabled` — CRR peer (`crr.yml:29-41`). **No `SyncConsistency` key exists today.**
- **Composition** (`ansible/_nativeha-dr-replication.yml`): create the site-B Recovery
  group (`nha_start: false` so it adopts the Live QMId), apply TLS, write Live vars
  (`this_group: Live`, `this_role: live`, `peer_wan_addrs: [b's]`, `recovery_enabled: Yes`)
  and Recovery vars (`recovery_enabled: No`), then restart **Recovery-first-then-Live**
  (load-bearing). The `peer_wan_addrs` are **hardcoded literal IP lists** in
  `_nativeha-dr-replication.yml` *and* `site-nativeha-switchover.yml`.
- **Version floor:** `main.yml:18-28` asserts MQ ≥ **9.4.4** with a real `version()`
  compare (admits 10.0).
- **Substrate reality:** IBM ships **no ARM build of MQ** — MQ is emulated on Apple
  Silicon regardless of guest OS, which is why the Ubuntu arm is a macOS convenience run
  *without* DR. This DR-and-timing epic therefore runs on **RHEL Native HA on cloud x86**,
  where MQ is native and timing is trustworthy.

## 3. Design

### 3.1 The pair — rename the CRR member, add the IRR sibling

Replication **mode** becomes the distinguishing axis, encoded in the 4-char `short`
(QM names derive from it; well under MQ's 48-char limit):

| Member | Stack | `short` | QM name | Groups | Nodes | Replication |
|---|---|---|---|---|---|---|
| CRR (async) | `nativeha-rhel` **→ `nativeha-rhel-crr`** | `NHAR` **→ `NHARC`** | `NHARAPP` **→ `NHARCAPP`** | `nha_rhel_a/b` **→ `nha_rhel_crr_a/b`** | `nha-rhel-a1..3/b1..3` **→ `nha-rhel-crr-*`** | Cross-region, async |
| IRR (strict-sync) | **new** `nativeha-rhel-irr` | `NHARI` | `NHARIAPP` | `nha_rhel_irr_a/b` | `nha-rhel-irr-*` | In-region, `SyncConsistency=Strict` |

The existing arm gets a **full rename** to the explicit CRR member — stack key, `short`,
QM, **and the node/group names**. Renaming the nodes/groups is not gratuitous churn: it
*serves* a load-bearing constraint — each arm's nodes are uniquely named by arm precisely
so all arms can **coexist**, which this epic actively exercises (CRR and IRR running
together). Leaving CRR's nodes as bare `nha-rhel-*` beside `nha-rhel-irr-*` would break
that symmetry. The rename is **atomic** (see plan Task 2): every reference moves at once —
topology nodes/groups, the switchover `qm_app` default, `alloc.app_unit`, exporter ports,
the Grafana board selectors, `observability.yml` group conditionals, and the composing
playbooks' group literals — so the CRR stack works after the rename; Task 4 then replaces
those group literals with the parameterized extra-vars. The IRR sibling mirrors the
`nativeha-ubuntu` "clone with different fields" precedent (`lab/topology.yaml:470-488`) on
its own six nodes and a fresh host-octet block.

### 3.2 Two parallel independent stacks

The two queue managers run as **two independent Native HA stacks, each on its own six
nodes** (12 nodes total). A stack stays what it is today — *one queue manager, one
implementation* — so there is no multi-QM-per-node special case to build. You build CRR,
or you build IRR; you want both, you run both stacks. This is the
minimum-necessary-complexity choice: co-residency was considered and **rejected**, because
sharing nodes would special-case the stack model and cascade port- and QMId-adoption
complexity for no real gain.

- **Node footprint.** CRR keeps the six `nha-rhel-*` nodes; the IRR stack adds six more on
  a fresh, collision-free host-octet block across every plane (CRR is `.9x`, Ubuntu `.1x`).
  The cloud VM is **resized before any build begins** — a stated pre-build pause, targeting
  **≥64 GiB** — to carry 12 nodes comfortably; the current box is deliberately not the
  ceiling.
- **Apples-to-apples holds.** The nodes are identical flavour and both stacks attach to the
  **same `virbr-wan`** (distinct IPs), so **one `mqlab netem` setting shapes both
  identically** (§3.4). The only differences between the two stacks are the names and the
  single `SyncConsistency` config line — everything else is identical.
- **Measurement gains.** With separate nodes there is no inter-QM resource contention, so
  the benchmark (§3.5) can run the two stacks **in parallel at the same injected latency**
  (tightest temporal apples-to-apples) or one-at-a-time — a Task-6/7 choice, not a
  constraint.
- **No port juggling.** Each stack owns its nodes, so both keep the standard listener
  1414 / HA-replication 9414 / CRR-WAN 9415 ports — no per-QM port reallocation.

### 3.3 One codebase, mode as a single config knob

The CRR↔IRR difference is **one configuration value**, not a forked implementation. Keep
the logic **OS-agnostic and single-sourced** in the shared `mq-nativeha` role (so the
Ubuntu arm inherits it for free — the standing factoring constraint), and drive the mode
from the stack's own topology entry:

- **An explicit `replication_mode` field per stack** (`async` | `strict`; default `async`
  for back-compat — no magic parsing of the QM name). The **same** `provision`/switchover
  playbooks read it — there is **no** separate `site-nativeha-irr.yml`. In `crr.yml`'s
  `NativeHARecoveryGroup` block, `strict` conditionally adds the single line
  **`SyncConsistency=Strict`** (exact key/placement confirmed by Task 1); `async`
  reproduces today's CRR byte-for-byte (regression-guarded).
- **De-hardcode `peer_wan_addrs`.** Today the site-A/site-B WAN IP lists are literals in
  `_nativeha-dr-replication.yml` and `site-nativeha-switchover.yml`. Derive them from
  topology per stack, so the IRR sibling's distinct net addresses are not a third copy of
  hardcoded IPs.
- **Reuse the bring-up ordering.** Recovery-first-then-Live restart, the `nha_start`
  Recovery gate, and the switchover `GroupRole` flip are mode-independent and reused; Task
  1 confirms whether strict-sync adds any ordering constraint (e.g. a Strict Live instance
  refusing to start if Recovery is unreachable).
- **Version floor.** If IBM documents a higher MQ floor for IRR strict-sync, raise/gate
  the `main.yml` assertion accordingly (Task 1 input).

### 3.4 The latency knob

A runtime, host-side latency injector on the cross-region link only:

- **Goal (not a pinned mechanism):** a **symmetric one-way delay D** on the cross-region
  path such that **round-trip ≈ 2D** — what a commit actually pays — applied to the
  `virbr-wan` plane only. The HA/heartbeat nets (`virbr-hb-a/-b`) are **never** shaped
  (intra-group replication is always same-site). The exact `tc` attach point — a root
  qdisc on the `virbr-wan` bridge, per-tap `netem`, or an `ifb` device for the ingress
  direction — is a **Task-5 implementation detail**, determined empirically and arbitrated
  by the acceptance probe, because `netem` directionality on a bridge is a known gotcha and
  the spec should not pre-commit a recipe that may not behave as written. Today `virbr-wan`
  runs the default `fq_codel` (no shaping) — greenfield.
- **Scope:** **delay only** in this epic. Jitter and packet-loss are the natural next knobs
  on the same verb — deliberately deferred (§5).
- **Interface:** a new **`mqlab netem`** Typer sub-app (following the `add_typer` +
  per-command pattern in `src/mqlab/cli.py`; shells `tc` on the libvirt host the way the
  `dr` group shells scripts): `mqlab netem set --delay <ms>` / `mqlab netem clear` /
  `mqlab netem show`. Runtime-tunable so a sweep changes latency **between runs** without
  re-provisioning (baking it in would force a rebuild per data point). Because both stacks
  share `virbr-wan`, one setting governs both identically.

### 3.5 The benchmark harness

- **Purpose-built benchmark client** (new, e.g. `clients/bench_client.py`) — the existing
  `clients/app_requester.py` self-describes as "good-enough noise, not a faithful app
  model" (`app_requester.py:16-20`) and is **not** built upon for precision numbers. The
  new client does controlled runs: **warmup**, fixed-N **persistent** messages under
  syncpoint, steady-state throughput, and latency **percentiles** (p50/p95/p99/max). It
  *reuses* the isolated, side-effect-clean pieces of `app_requester.py` — `RoundTripStats`,
  `render_prom`, the atomic `write_textfile`/`publish_metrics`, and the `_RECONNECT_REASONS`
  Native-HA-failover reconnect logic — and keeps emitting node-exporter textfile metrics
  for live dashboards, but its **primary output is a structured results artifact** (JSONL/
  CSV) for offline analysis.
- **Metrics that reveal the sync tax:** (a) **sustained throughput ceiling** and (b)
  **commit / round-trip latency percentiles**. Operational definition of the ceiling (so
  two runs agree): *the highest offered rate at which, over a fixed steady-state window
  after warmup, queue depth stays bounded (no upward trend) and p99 commit latency stays
  finite.* Latency percentiles (p50/p95/p99/max) are reported at **~80 % of that ceiling**.
  The window length and the 80 % fraction are tunable at alignment, but the definition is
  **frozen before Task 6 builds the client**. The knee of (a) vs injected latency is the
  headline of the curve.
- **Sweep runner:** drives the matrix `{mode: CRR, IRR} × {latency}` (this epic: one
  representative message size and rate; the client *supports* size×rate axes for the
  follow-on surface), setting `mqlab netem` between points, one QM at a time, into the
  results artifact.
- **First-pass experiment + comparison report:** one meaningful latency sweep (e.g.
  {0, 5, 10, 20, 40} ms one-way) at a representative message size/rate, CRR vs IRR,
  producing the epic's initial cost numbers and a written report. **Fidelity claim:**
  *relative* overhead (IRR vs CRR, identical hardware, identical injected latency) — not
  absolute production throughput. Native x86 (no emulation) makes even relative timing
  trustworthy.

## 4. Binding decisions (explicit)

1. **Substrate: RHEL Native HA on cloud x86**, native MQ (no emulation). Ubuntu is a
   macOS convenience arm, out of scope here; new logic stays OS-agnostic in the shared
   role so Ubuntu inherits it.
2. **Mode encoded in `short`, OS preserved:** CRR = `NHARC`/`NHARCAPP` (rename in place),
   IRR = `NHARI`/`NHARIAPP` (new sibling). App selects sync vs async by QM name.
3. **Two parallel independent stacks** (per §3.2) — CRR on its six nodes, IRR on six more
   (12 total); the VM is resized (≥64 GiB target) before any build. Both share `virbr-wan`;
   no co-resident multi-QM provisioning. (Co-residency considered and rejected.)
4. **One codebase, mode as a config field** — an explicit per-stack `replication_mode:
   async|strict` drives the **same** playbooks (no forked IRR playbook); de-hardcode
   `peer_wan_addrs`; `strict` adds the single line `SyncConsistency=Strict` (exact form per
   Task 1). `async` is byte-for-byte the current CRR.
5. **Latency knob = `mqlab netem`, delay-only, cross-region path, runtime-tunable** —
   symmetric one-way D → RTT ≈ 2D; HA nets never shaped; the exact `tc` attach point is a
   Task-5 detail arbitrated by a **measured-RTT-both-directions** probe. Jitter/loss
   deferred.
6. **Purpose-built benchmark client**, not an extension of `app_requester.py`; primary
   output a structured results artifact + a comparison report.
7. **First pass only:** one latency sweep at a representative size/rate. The full
   size×rate×latency surface, RPO/loss-window measurement, and automated DR validation are
   explicit non-goals (§5).

## 5. Scope & non-goals

**In scope:** the IRR-setup facts spike; the CRR rename + IRR sibling; mode-parameterizing
the shared role; the `mqlab netem` delay knob; the purpose-built benchmark client + sweep
runner; a first-pass CRR-vs-IRR latency sweep + comparison report; a cold-rebuild
validation of the extended arm; and the doc / doc-review / retrospective bookends.

**Non-goals (recorded follow-on / forward-axis work):**

- **Jitter and packet-loss** on the netem knob (phased right behind delay).
- The **full multi-dimensional cost surface** (message-size × rate × latency); this epic
  runs a first pass, though the client is built to sweep those axes.
- Any **RPO / message-loss demonstration** or **automated DR validation & reporting**. We
  trust the architecture — CRR loses a replication-lag window; IRR strict-sync is zero-loss
  by design — so *proving* it programmatically is a separate epic, not a gate here.
- The **Ubuntu / pcmk / RDQM arms** — untouched; Ubuntu inherits the shared-role changes
  passively but is not built or validated here.

## 6. Decisions settled in pushback, and what remains for alignment

**Settled (pushback, 2026-09-15):**

- **Topology: two parallel independent stacks** (§3.2) — co-residency rejected for
  minimum-necessary-complexity.
- **netem: goal-stated, not mechanism-pinned** (§3.4) — attach point is a Task-5 detail
  verified by a measured-RTT-both-directions probe.
- **Throughput-ceiling metric: operationally defined** (§3.5) — definition frozen before
  Task 6; numbers settled at alignment (below).

**Settled (alignment, 2026-09-15):**

- **CRR full rename** (§3.1) — stack key, `short`, QM, **and** node/group names all move,
  atomically, to serve the arm-coexistence constraint.
- **Benchmark numbers** (tunable in a Task-8 dry-run): persistent **2 KiB** messages;
  one-way latency sweep **{0, 5, 10, 20, 40} ms** (RTT 0–80 ms); **30 s warmup / 120 s
  steady-state measure**; latency percentiles reported at **80 % of the measured
  throughput ceiling**.
- **Results artifact + comparison report location:** `docs/reports/`.

**Remaining for Task 1 (the only pre-build unknown):**

1. **IRR exact config & version floor** — resolved by the Task-1 facts spike before any
   build; the spec deliberately leaves the stanza/port/`SyncConsistency`-placement/floor
   specifics to that spike.

## 7. Verification & acceptance

- **CRR unchanged by the refactor:** with `replication_mode: async`, the rendered `qm.ini`
  and CRR behaviour are byte-for-byte today's (regression check on the rename + mode split).
- **IRR pair forms and replicates strict-sync:** `NHARIAPP` comes up as a Live+Recovery
  pair with `SyncConsistency=Strict`; `dspmq -o nativeha` shows healthy CRR/IRR state on
  both members; a persistent PUT is confirmed present on the Recovery group before the
  commit returns (the strict-sync guarantee, observed once as a sanity check — not the
  swept RPO proof, which is out of scope).
- **Both stacks reachable in parallel:** an app selects `NHARCAPP` or `NHARIAPP` by name
  and round-trips through each; both stacks run concurrently on the resized host.
- **Latency knob works:** `mqlab netem set --delay 10ms` measurably raises cross-site RTT
  to ≈20 ms across `virbr-wan` — **measured both directions by probe**, not merely
  `tc qdisc show` — and leaves the HA nets untouched; `clear` restores `fq_codel`.
- **First-pass curve produced:** the sweep runs the `{CRR,IRR} × {latency}` matrix at a
  representative size/rate, emits the results artifact, and the comparison report shows the
  IRR-vs-CRR overhead and where (if within range) the strict-sync knee appears — reported
  as an outcome, not a threshold.
- **Cold-rebuild validation:** a full cold rebuild of the extended arm on cloud x86 comes
  up green in one pass (the epic's `validation` operational task).

## 8. Relationships

- **Extends:** `#88` (native-HA RHEL box), `#219` (MQ 10.0 CRR upgrade + version pin).
- **Long-line #145 §7.3 / phase-A §567:** `tc netem` WAN latency injection — long-planned,
  never delivered — is finally built here (the `net-wan.xml` earmark).
- **Spawns (forward axis, recorded in the retrospective):** jitter/loss knob; the full
  size×rate×latency surface; RPO/loss-window measurement; automated DR validation &
  reporting.

## 9. Task breakdown (provisional — finalized in the plan)

Implementation tasks land in `mq-resiliency-lab-for-linux`, linked under `#227`:

1. **IRR-setup facts spike** — confirm IRR bring-up mechanics (stanzas, `SyncConsistency`
   placement, ports, version floor, any strict-sync start-ordering) against IBM MQ 10.0
   docs via `tools/ibm_doc_cache.py`; cache the IRR *setup* reference. Gates the build.
2. **Rename `nativeha-rhel` → CRR member** (`NHAR`→`NHARC`, `NHARAPP`→`NHARCAPP`) and
   handle the migration blast radius (switchover default, alloc/app_unit, exporter ports,
   dashboard folder, app config, box wiring). `async` behaviour byte-for-byte preserved.
3. **Mode-parameterize the shared role** — `replication_mode` var, de-hardcode
   `peer_wan_addrs`, `SyncConsistency=Strict` for `strict`.
4. **Add the IRR stack** `nativeha-rhel-irr` (`NHARI`/`NHARIAPP`) on its **own six nodes** —
   topology entry with `replication_mode: strict`, a fresh host-octet block on every plane,
   shared `virbr-wan` attachment, reusing the same shared playbooks.
5. **`mqlab netem` delay knob** — the Typer sub-app; determine the `tc` attach point on the
   cross-region path empirically (bridge / per-tap / ifb) and verify symmetric RTT both
   directions; set/clear/show.
6. **Purpose-built benchmark client + sweep runner** — controlled persistent-message
   throughput/latency with percentiles; structured results artifact.
7. **First-pass experiment + comparison report** — the `{CRR,IRR} × {latency}` sweep and
   the written report (the headline deliverable).

Operational task (in `mq-resiliency-lab-for-linux`, linked under `#227`):

- **Cold-rebuild validation `#1098`** — extended arm provisions green cold on cloud x86;
  blocked-by the implementation tasks.

Bookend tasks (already created):

- **`.github#228` Documentation** — this spec + the plan (first task; its PR publishes them).
- **`mq-resiliency-lab-for-linux#1097` Documentation review** — reconcile the Native HA
  reference/site docs with the CRR/IRR dual-mechanism arm (closing, pre-retrospective;
  spawns per-repo doc tasks as the sweep discovers them).
- **`.github#229` Retrospective** — terminal gate; records the forward-axis follow-ons.
