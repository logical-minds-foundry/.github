# Discovery-native MQ administration — HA-transparent admin tooling + AUTHREC migration

Epic: [logical-minds-foundry/.github#165](https://github.com/logical-minds-foundry/.github/issues/165)

## §0 — Thesis

Administrative tooling must reach a queue manager the way an **application** does: by
handing MQ a way to *locate* the QM — a connection-name list / CCDT for Native HA, the
VIP for Pacemaker-managed arms — and letting the client connection **discover the active
instance at the moment the operation runs**. No admin operation may act on a
*previously-discovered* location; that assertion is stale the instant HA leadership moves.

In parallel, the lab's authorization surface moves off the local-only `setmqaut` control
command onto the remotely-administrable MQSC **`SET AUTHREC`**. That collapses the entire
admin surface — configuration *and* authorization — into one uniform path:
**client-mode MQSC → dynamic active-instance discovery**, with no node-local execution.

## §1 — Problem

The reproduce-until-reliable campaign (#876, under lifecycle-reliability epic `.github#104`)
surfaced a class of failover-window races the current design does not handle:

- **#891 — stale pinned active.** `site-nativeha-ubuntu.yml` detects the active instance
  **once** (`set_fact active_host`) and then five tasks `delegate_to: active_host`
  (our-side MQSC, render authz, apply authz, `setmqaut`, refresh authz). Native HA raft can
  re-elect **between** the pin and the applies — common while the group is still stabilizing
  at cold-build — leaving `active_host` a **replica**. The delegated `runmqsc` then fails
  `AMQ8478E` (instance is a replica), and the existing `until`/`retries: 30` cannot recover
  because they re-hit the **same** stale host; they exhaust and fail the bootstrap. The QM
  itself is healthy (`QUORUM 3/3`) — only the pinned target is wrong.
- **#888 — readiness band-aid (merged, interim).** A per-task `AMQ8146E` ("QM not available
  yet") retry on the authz apply. It patches the *readiness* window but not the *leadership*
  window, and it is exactly the kind of per-task special-casing this epic retires.
- **#472 — client side of the same window.** IBM MQ automatic client reconnection protects
  only an *already-established* connection; the initial `MQCONNX` is not retried. During a
  leader election no instance accepts connections, so an application's initial connect
  fast-fails. Clients need a bounded initial-connect retry over the CONNAME list — the
  client half of the same "connect to a moving HA resource" contract.

Root cause, stated once: **the lab treats "where is the QM active" as a fact to compute
and cache, when it is a property to discover per-operation.** Applications already do this
correctly; administrative tooling does not.

## §2 — Design principles

1. **Per-operation dynamic discovery.** Every "mutate this QM" operation discovers the
   active instance *for itself, at the moment it runs*. Nothing trusts a cached
   `active_host`. This is the general rule; §3 gives the mechanism.
2. **Connect like an application.** Remote-capable operations (MQSC) run in **client mode**
   against the QM's *connection identity* — a CCDT / CONNAME list for Native HA (which has
   no VIP; clients try the instance endpoints and land on the active), the **VIP** for
   Pacemaker-managed arms (pcmk/rdqm), where the address already follows the active. Same
   mechanism the lab's applications and the client-mode `mq_prometheus` exporter already use.
3. **Bring-up stability gate — A *and* B, not either.** Per-op discovery is the general
   rule, but at cold-build leadership churns right after `crtmqm`/start; even a
   discover-and-apply op can have the active move *between* its discover and its apply. So
   after **defining** the QM, wait for leadership to **quiesce** before firing the change
   sequence — and each op in that sequence *still* discovers dynamically rather than trusting
   the gate's snapshot. The gate shrinks the vulnerable window; per-op discovery closes it.
4. **`setmqaut` → `SET AUTHREC`.** `setmqaut` is a local-only control command predating
   remote OAM administration; `SET AUTHREC` is MQSC and remotely administrable. Migrating
   the authorization surface to AUTHREC removes the *one* node-local execution mode, so the
   whole admin surface becomes uniformly client-mode-MQSC.
5. **HA only.** Discovery assumes the live (HA) side. DR is a deliberate **manual** failover
   and is never auto-discovered — an admin op targets the HA group, not the DR peer.
6. **No stale assertions anywhere.** The `active_host` `set_fact` + `delegate_to` pattern is
   retired, not merely guarded. #888's `AMQ8146E` retry is retired. Any remaining need to
   "know which node" is answered by discovery at use time.

## §3 — Architecture

### 3.1 The discovery-native MQSC apply

The reference primitive: **apply an MQSC file to a QM by its connection identity, in client
mode, retrying the connect over the CONNAME list until it lands on the active** — the admin
analogue of an application connecting. No `delegate_to`, no SSH, no pinned host.

- **Native HA** (`nativeha-ubuntu`, `nativeha-rhel`): a CCDT / CONNAME list enumerating the
  instance endpoints. The MQ client tries them and connects to the active; a bounded
  connect-retry rides out the election window (the admin sibling of #472). `runmqsc -c`
  (client mode) is the candidate transport — **spike required** (§5, research).
- **Pacemaker arms** (`pcmk`, `rdqm`): the QM's **VIP** already moves with the active, so a
  client connection to the VIP is inherently discovery-native. These arms are *reconciled*
  to the same connect-like-an-app model rather than rebuilt.

### 3.2 The authorization surface → `SET AUTHREC`

Replace the lab's `setmqaut` grants with MQSC `SET AUTHREC` so authorization is applied over
the same client-mode-MQSC path as everything else. **Contingent on §5 research** confirming
AUTHREC covers the grant surface the lab uses (object types, principals, `+setall`/context
authorities, the DLQ grant, etc.). If a residual grant genuinely has no AUTHREC equivalent,
it is documented as the sole node-local exception rather than silently retaining the old mode.

### 3.3 The bring-up stability gate

After `crtmqm`/start and before the change sequence, poll the Native HA group until
leadership has **quiesced** (a stable active across N consecutive observations / a settle
interval — exact predicate decided at plan time). This is a bring-up convenience that shrinks
the churn window; it does **not** license the downstream ops to trust its snapshot.

## §4 — Scope

**In scope**

- The discovery-native MQSC apply primitive + its adoption on `nativeha-ubuntu` (reference).
- `setmqaut` → `SET AUTHREC` migration of the lab's authorization surface.
- The bring-up stability gate on the Native HA arms.
- Fan-out to `nativeha-rhel` — **note: Stage 2 authorization is net-new there** (the epic
  #74 fan-out never reached that arm), so this is *add authz via AUTHREC*, not a migration.
- Reconcile `pcmk`/`rdqm` to the connect-like-an-app model (VIP-based; largely confirming
  they already are, plus retiring any pinned-node admin steps).
- Retire the interim pins/retries (`active_host` pin, #888 `AMQ8146E` retry).
- **#472** — client initial-connect retry / CONNAME-list discipline (the client half).
- **Cold-rebuild validation** proving the failover-window race no longer recurs.

**Out of scope**

- DR auto-discovery (DR remains a deliberate manual failover).
- Re-architecting non-QM admin surfaces (cluster/DRBD/Pacemaker tooling) beyond what the
  QM-connection model touches.

## §5 — Research / spikes (resolve before or early in implementation)

1. **`runmqsc -c` + CCDT/CONNAME-list reaches the Native HA active cleanly.** The lab already
   runs client-mode `mq_prometheus` with a CCDT, so the plumbing exists; confirm `runmqsc`
   client mode connects to the active instance and rides the election window with a bounded
   connect-retry. Fallback if a gap is found: a discovery wrapper that resolves the active
   (e.g. `dspmq`) then connects — still per-op, never pinned.
2. **`setmqaut` vs `SET AUTHREC`.** Modernity and tradeoffs; confirm AUTHREC covers the
   lab's grant surface. Hypothesis (to verify): AUTHREC is the modern, remote-administrable
   path and `setmqaut` predates remote OAM administration. Output: a go/no-go on full
   migration + an itemized grant-by-grant mapping.

## §6 — Success criteria

- A Native HA cold rebuild applies all QM configuration **and** authorization with **no**
  `AMQ8478E`/`AMQ8146E` and **no** pinned-`active_host` dependency — repeatably, across
  induced/observed leadership churn (reproduce-until-reliable, campaign #876 discipline).
- The `active_host` `set_fact`/`delegate_to` pattern and #888's retry are **gone** from the
  Native HA path.
- Authorization is applied via `SET AUTHREC` over client-mode MQSC (or the documented
  exception set from §5.2).
- Clients (#472) survive an initial-connect during a leader election via bounded CONNAME-list
  retry.

## §7 — Risks & tradeoffs

- **`runmqsc -c` gap.** If client-mode `runmqsc` cannot cleanly reach the active, the
  discovery-wrapper fallback (§5.1) applies — more moving parts but still discovery-native.
- **AUTHREC coverage gap.** If some grant has no AUTHREC form, it stays node-local as a
  documented exception (§3.2) — the design degrades gracefully rather than blocking.
- **Fan-out asymmetry.** `nativeha-rhel` authz is net-new; pcmk/rdqm are mostly reconciliation.
  Sequencing (§4) implements the reference arm first to de-risk before fan-out.
