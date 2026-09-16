# Discovery-native MQ administration — HA-transparent admin tooling + AUTHREC migration

Epic: [logical-minds-foundry/.github#165](https://github.com/logical-minds-foundry/.github/issues/165)

## §0 — Thesis

This epic delivers two small, security-adjacent improvements to how the lab **builds**
its queue managers. They are logically independent — one is a configuration-representation
change, the other a bring-up stability fix — but both touch the same authorization/admin
code path and are small enough to ship together under one umbrella rather than split into
separate epics.

1. **Uniform MQSC configuration (the primary motivator).** Standardize the lab's queue-manager
   configuration — object definitions **and** authorization — on a single format, MQSC.
   Today authorization is applied with the `setmqaut` **control command** (a CLI invocation
   per grant); migrating it to the MQSC **`SET AUTHREC`** command collapses the whole
   configuration of an object into one templatable representation instead of "MQSC plus a
   snippet of shell." This is easier to template, compartmentalize, and reason about — and it
   completes a migration the lab already began: the `mqmon` grants moved to `SET AUTHREC` in
   `authz.mqsc.j2` under the standing repo rule (2026-08-06) that a security profile should
   *travel co-located with the object definitions* it protects. This epic extends that rule to
   the remaining (`mqapp`/`mqsvc`) grants.
2. **Bootstrap targeting that follows leadership.** The Native HA arms pin the active
   instance **once** at the start of the admin sequence and then act on that pinned host for
   every step. Raft can re-elect between the pin and a later step, so the pin goes stale and
   the operation lands on a replica. Replace the one-time pin with a **per-operation
   re-resolution** of the active, packaged as one reusable apply-with-retry primitive.

Both are **build-time** concerns. Neither changes how MQ administration works at runtime;
this epic is about making the lab's cold build uniform and reliable.

## §1 — Problem

The reproduce-until-reliable campaign (#876, under lifecycle-reliability epic `.github#104`)
surfaced a failover-window race, and the lab's authorization surface is split across two
representations. Concretely:

- **#891 — stale pinned active.** `ansible/site-nativeha-ubuntu.yml` detects the active
  instance **once** (`find the active Native HA instance` → `set_fact active_host`, lines
  88–112) and then six tasks target that pinned host (our-side/app MQSC, render authz MQSC,
  apply authz MQSC, the `setmqaut` grant loop, `REFRESH SECURITY`, and the event-monitor
  role — lines 114–230). Native HA raft can **re-elect between the pin and a later apply** —
  common while the group is still stabilizing at cold-build — leaving `active_host` a
  **replica**. The delegated `runmqsc` then fails **`AMQ8478E`** (rc 20, "Standby queue
  manager") and the existing `until`/`retries: 30` cannot recover, because every retry
  re-hits the **same** stale host; it exhausts and fails the bootstrap. The QM itself is
  healthy (`QUORUM 3/3`) — only the pinned target is wrong.
- **`AMQ8146E` readiness retries (#435, #888) — a *different* race, currently patched
  separately.** The raft layer assigns `ROLE(Active)` a beat before the QM admin interface
  reaches `Running`, so `runmqsc` in that window fails **`AMQ8146E`** (rc 20, "queue manager
  not available"). This was patched with per-task `until: rc in [0,10] / retries: 30 / delay:
  5` on the our-side MQSC (#435, `site-nativeha-ubuntu.yml:135-143`) and later mirrored onto
  the authz apply (#888, `:166-183`). These readiness retries handle the *not-ready* window
  but not the *leadership-moved* window, and they are per-task special-casing.
- **Split authorization representation.** Authorization is applied two ways today. Most grants
  use the `setmqaut` control command in a per-grant loop driven by `authz_grants`
  (`ansible/group_vars/all/authz.yml`); the loop is copy-pasted across the **three arms that
  have Stage-2 authz today** (`site-nativeha-ubuntu.yml:188-201`, `site-rdqm.yml:404-418`,
  `roles/mq-pcmk-qmgr/tasks/main.yml:142-166`). The fourth arm, `nativeha-rhel-crr`
  (`site-nativeha.yml`), has **no** `setmqaut` loop and no Stage-2 authz at all — authz is
  net-new there (§3.3). But the `mqmon` grants **already** moved to MQSC `SET AUTHREC`
  (`authz_mqmon_grants`, `authz.yml:64-69`) and travel in the shared, arm-agnostic
  `authz.mqsc.j2`. So the lab already has one foot in AUTHREC; the rest is a half-finished
  migration.

Root cause of the race, stated once: **the lab treats "where is the QM active" as a value to
compute and cache, when it is a property to re-check per operation.** Root cause of the split:
**the AUTHREC migration was started (mqmon) but never completed.**

## §2 — Design principles

1. **Per-operation re-resolution, not a cached pin.** Each admin step that must run on the
   active instance re-resolves the active *for itself*. Nothing trusts a `set_fact active_host`
   that was computed earlier in the play. §3.1 gives the mechanism.
2. **Follow leadership; wait for readiness — in one loop.** The two bring-up races
   (`AMQ8478E` leadership-moved, `AMQ8146E` not-ready-yet) are both transient `rc 20`
   conditions. A single retry loop that, on each attempt, re-resolves the active, targets it,
   applies the MQSC, and retries on `rc 20` — subsumes **both**, and therefore replaces the
   #435/#888 readiness retries as well as the #891 stale pin. Fail-loud is preserved: a
   genuine non-`[0,10]` rc still fails hard.
3. **One format: MQSC.** Object configuration and authorization are expressed as MQSC. The
   `setmqaut` control command is migrated to `SET AUTHREC`, completing the migration the
   `mqmon` grants already began, so each object's whole configuration lives in one
   templatable place.
4. **Stay in bindings mode; keep it simple.** Admin MQSC continues to run locally on the
   active instance via `su - mqm -c "runmqsc <QM>"`. Client-mode `runmqsc -c` was considered
   and **rejected** — see §3.4.
5. **HA only.** Targeting assumes the live (HA) side. DR is a deliberate **manual** failover
   and is never auto-resolved — an admin op targets the HA group, not the DR peer.

## §3 — Architecture

### 3.1 The reusable "apply MQSC to the active Native HA QM" primitive

The core of Deliverable 2. A single reusable Ansible include — call it the *apply-to-active*
primitive — that takes an MQSC payload (inline or a rendered file) and applies it to the
queue manager, **re-resolving the active instance on every attempt** and retrying transient
`rc 20`:

- Resolve the active with `dspmq -m <QM> -o nativeha -x`, matching the per-instance detail
  line `INSTANCE(...) ROLE(Active)` (the resolution already in use at
  `site-nativeha-ubuntu.yml:88-94`, robust to which instance answers, per #722).
- `delegate_to` the resolved active and run `su - mqm -c "runmqsc <QM>"` (bindings mode).
- On `rc 20` (whether `AMQ8478E` — moved, so the *next* attempt re-resolves and follows it —
  or `AMQ8146E` — not ready, so the next attempt waits), retry with bounded
  `retries`/`delay`. On `rc in [0,10]` succeed; on any other rc fail loud.

Because resolution happens **inside** the retry, a leadership move mid-sequence is *followed*
rather than fatal, and the readiness window is *waited out* — one mechanism, both races. The
six pinned steps in `site-nativeha-ubuntu.yml` are rewritten to call this primitive; the
`set_fact active_host` pin and the standalone #435/#888 retries are removed. Being a single
include, it also dissolves the copy-paste of the discover-and-target dance across the Native
HA arms.

**The event-monitor role is an explicit special case.** It is gated
`when: inventory_hostname == active_host` (`site-nativeha-ubuntu.yml:221-230`) because
`delegate_to` is invalid on `include_role`, and it runs **several sequential `runmqsc` tasks**
(enable LOGGEREV, per-queue perf events, `DEFINE SERVICE` — `mq-event-monitor/tasks/service.yml`
`:59/:74/:90`) plus local `qm.ini` reads. The single-shot apply-to-active primitive therefore
cannot wrap it. Instead it is covered by **block-level re-resolution + retry**: the whole
guarded include is wrapped so that on `rc 20` (a leadership move *during* the role's multi-task
run) the active is re-resolved and the role's tasks are re-run. The role's tasks are already
idempotent (`REPLACE`, `CONTROL(QMGR)`, `AMQ8737W`/rc 10 tolerated), so re-running is safe.
This keeps the "no stale targeting anywhere" guarantee intact for the one consumer the
single-shot primitive can't reach.

### 3.2 The authorization surface → `SET AUTHREC`

Deliverable 1. Move the `authz_grants` surface from the `setmqaut` loop into MQSC `SET AUTHREC`
statements carried by the already-shared, already-arm-agnostic `authz.mqsc.j2` — the same
template that already carries the migrated `mqmon` grants. Delete the four per-arm `setmqaut`
loops. Each grant's `setmqaut -t <type> [-n <obj>] -g <group> <+auths>` maps to
`SET AUTHREC OBJTYPE(<type>) [PROFILE(<obj>)] GROUP('<group>') AUTHADD(<AUTHS>)`.

- **Preflight coverage check (folds in the old §5 research).** Before implementing, confirm
  `SET AUTHREC` covers the exact grant surface — in particular the `mqsvc` **`+setall`**
  context authority on the qmgr, `APP.REPLY`, and the DLQ (`SYSTEM.DEAD.LETTER.QUEUE`), which
  maps to `AUTHADD(SETALL)`. The `mqmon` precedent gives high confidence AUTHREC is the right
  path; this check just confirms the specific authorities and produces a grant-by-grant map.
  If (unexpectedly) a grant has no AUTHREC form, it is documented as the sole retained
  `setmqaut` exception rather than silently split — no silent fallback.
- **Preserve `REFRESH SECURITY`.** The `REFRESH SECURITY TYPE(AUTHSERV)` step that today
  follows the `setmqaut` loop (`site-nativeha-ubuntu.yml:203-212`) reloads the OAM cache and
  **must be retained** — folded into the authz apply, still re-resolving the active — or
  AUTHREC changes may not take effect until the next QM restart (grants that look applied but
  aren't live). It is also one of the six pinned consumers, so it is covered by §3.1.

### 3.3 Arm scope

- **Deliverable 1 (AUTHREC) — all four arms.** Every arm has a `setmqaut` loop to migrate:
  `nativeha-ubuntu`, `nativeha-rhel-crr` (`site-nativeha.yml`), `rdqm` (`site-rdqm.yml`),
  `pcmk` (`roles/mq-pcmk-qmgr`). On **`nativeha-rhel-crr` authorization is net-new** — the
  epic #74 Stage-2 fan-out never reached that arm — so there it is *add authz via AUTHREC*,
  not a migration. Net-new authz there includes ensuring the `mqapp`/`mqmon`/`mqsvc` OS
  accounts + groups (the same identities the other arms use) are provisioned on every
  `nha_rhel_crr` instance — `SET AUTHREC GROUP('…')` and the SSLPEERMAP `MCAUSER` targets
  require those groups to exist on each failover node (the OS-replication caveat).
- **Deliverable 2 (apply-to-active primitive) — the two Native HA arms only.**
  `nativeha-ubuntu` (reference) then `nativeha-rhel-crr`. These are the only arms with the
  raft re-election churn. `rdqm` (per-host `rdqm_is_active` + `when`-guard, plus a VIP) and
  `pcmk` (LUN-owner `run_once` + a VIP) target the active differently and do **not** exhibit
  the stale-pin failure; they are **verified unaffected**, not rewritten.

### 3.4 Why not client-mode `runmqsc -c` (considered and rejected)

The original framing of this epic proposed running admin MQSC in **client mode**
(`runmqsc -c`) against a CONNAME list, letting the client connection discover the active — the
way the `mq_prometheus` exporter already connects. It was rejected during design:

- **It changes nothing without also dropping discovery.** An Ansible task always runs *from*
  some host. Bindings mode fuses "where the task runs" with "where the QM is active" (bindings
  `runmqsc` only works on the active node). Client mode *decouples* them — but only pays off
  if the task then runs from a **fixed** host and stops resolving the active at all. Merely
  swapping `su - mqm -c runmqsc` for `runmqsc -c` while still targeting a resolved host
  changes nothing.
- **The payoff is undercut by an irreducible seed.** The first MQSC — the one that `DEFINE`s
  the listener and channels — has no channel to connect over yet and (in Native HA) must run
  on the active anyway. Client mode cannot eliminate node-local admin; it only shrinks it.
- **It has a real cost and no other use case.** Client-mode admin needs a dedicated admin
  SVRCONN channel + admin cert + CHLAUTH map that don't exist today (APP.SVRCONN stops being
  admin-capable once CHLAUTH+SSLPEERMAP map the app cert to non-privileged `mqapp`). That
  configuration overhead buys us nothing beyond this one problem — which per-operation
  re-resolution (§3.1) solves in bindings mode with no new channel or identity.

## §4 — Scope

**In scope**

- The reusable *apply-to-active* primitive (§3.1) and its adoption on `nativeha-ubuntu`
  (reference), retiring the `active_host` pin and the #435/#888 standalone retries there.
- Fan-out of the primitive to `nativeha-rhel-crr`.
- `setmqaut` → `SET AUTHREC` migration across all four arms (§3.2, §3.3), including the
  net-new authz on `nativeha-rhel-crr`, via the shared `authz.mqsc.j2`.
- Verification that `rdqm` and `pcmk` are unaffected by the targeting change.
- **Cold-rebuild validation** proving the failover-window race no longer recurs (bookend
  `mq-resiliency-lab-for-linux#913`).

**Out of scope**

- **Client-mode `runmqsc -c` admin** — considered and rejected (§3.4).
- **A separate bring-up "stability gate"** — the original design's "wait for leadership to
  quiesce" step. The §3.1 retry loop follows leadership and waits for readiness, so a separate
  gate is redundant.
- **#472 (client initial-connect retry).** This is the *client-application* half of the
  HA-connect story (retry loops in `clients/app_requester.py` et al.), a separate subsystem
  from the lab's admin tooling. Per #472 itself it is to be implemented independently, ahead
  of the future detailed client-connectivity design. It remains its own triage item, not part
  of this epic.
- DR auto-resolution (DR remains a deliberate manual failover).
- Re-architecting non-QM admin surfaces (cluster/DRBD/Pacemaker tooling) beyond what the
  authorization/QM-config path touches.

## §5 — Success criteria

- The fix is **demonstrably exercised**, not merely observed clean (bookend #913). Because
  bring-up leadership churn is non-deterministic — a clean rebuild may never re-elect
  mid-sequence, and the #876 campaign's own caveat is that "backstops didn't have to *fire*
  this pass" — acceptance is two-pronged: **(a)** an **induced** mid-sequence leader move
  (e.g. step down / `endmqm` the current active *during* the admin sequence) that the sequence
  must **follow and complete** with no `AMQ8478E` failure; **plus (b)** the statistical
  reproduce-until-reliable rounds (#876) as the durability backstop.
- A Native HA cold rebuild applies all QM configuration **and** authorization with **no**
  `AMQ8478E` and **no** `AMQ8146E` reaching a failure, and with **no** `set_fact active_host`
  dependency.
- The `active_host` `set_fact`/`delegate_to` pattern and the standalone #435/#888 readiness
  retries are **gone from the Native HA path**, replaced by the single §3.1 primitive. This
  criterion is Native-HA-scoped: `rdqm`/`pcmk` **retain** their own `AMQ8146E` readiness
  handling (e.g. `site-rdqm.yml:387-402`), which D2 does not touch.
- Authorization on all four arms is applied via `SET AUTHREC` in `authz.mqsc.j2`; the **three**
  existing `setmqaut` loops (`nativeha-ubuntu`, `rdqm`, `pcmk`) are removed and
  `nativeha-rhel-crr` gains authz net-new — no fourth loop exists to remove. (For any grant
  with no AUTHREC form, that grant is retained as an explicitly documented exception — no
  silent split.)
- `nativeha-rhel-crr` gains its previously-missing Stage-2 authorization, via AUTHREC.
- `rdqm` and `pcmk` continue to build clean (targeting change verified not to regress them).

## §6 — Risks & tradeoffs

- **Residual intra-attempt window.** Re-resolving *inside* the retry (rather than pinning)
  means a leadership move between a single attempt's resolve and its `runmqsc` simply fails
  that attempt with `rc 20` and is retried — so the window is handled by construction, not
  merely shrunk. The tradeoff is one extra `dspmq` per attempt, which is negligible at
  bring-up.
- **AUTHREC coverage gap.** If some grant has no `SET AUTHREC` form (not expected, given the
  `mqmon` precedent and `AUTHADD(SETALL)`), it stays `setmqaut` as a documented exception —
  graceful degradation, never a silent fallback.
- **Fan-out asymmetry.** `nativeha-rhel-crr` authz is net-new (add, not migrate); `rdqm`/`pcmk`
  are verification only. The reference arm (`nativeha-ubuntu`) is implemented and validated
  first to de-risk before fan-out.
- **Coupling two deliverables.** D1 and D2 are independent and are tracked as separate tasks;
  bundling them under one epic trades a little purity for avoiding a second round of
  epic/spec/plan overhead. Accepted deliberately.
