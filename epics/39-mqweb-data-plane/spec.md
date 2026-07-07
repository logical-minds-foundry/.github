# mqweb REST as data-plane infrastructure on every QM — design spec

- **Epic:** `logical-minds-foundry/.github#39`
- **Design task:** `logical-minds-foundry/.github#40`
- **Origin:** `mq-resiliency-lab-for-linux#252` under epic `logical-minds-foundry/.github#13` (MQ security hardening)
- **Unblocks:** `mq-resiliency-lab-for-linux#252`
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-07-07

## 1. Problem & motivation

The IBM MQ **mqweb** endpoint (the administrative REST API and the browser
Console, Liberty on `9443/HTTPS`) is the control surface co-located with each
queue manager. Its deployment *shape* already matches the authoritative decision
in `mq-resiliency-lab-for-linux` `docs/specs/2026-06-03-mq-cluster-lab-design.md`
§8.3 — *stateless `mqweb` per node, each administering the local QM, published to
clients as the floating data-plane VIP that follows the QM*. But three things
drifted, letting mqweb be **perceived and addressed as a management /
observability-plane service** rather than as part of the QM infrastructure it
fronts:

1. **It has no plane discipline.** The role binds `httpHost=*`
   (`ansible/roles/mqweb/tasks/main.yml:62`, `templates/mqwebuser.xml.j2:38`), so
   mqweb answers on *every* NIC, including the mgmt plane `net-mgmt`
   (`10.50.0.0/24`).
2. **Its canonical client address is undefined and stale.** `src/mqlab/apply.py`
   takes the REST base URL as a hand-passed CLI argument (`apply.py:50-51`); the
   docstring and bring-up runbook example is `https://10.30.0.10:9443` — the
   **legacy net-client plane** that no longer exists in `lab/topology.yaml`. Nothing
   derives the correct data-plane address.
3. **The plane taxonomy never classified it.** `docs/reference/dns-fqdn-inventory.md`
   declares the mgmt and observability planes IP-only but **never lists the mqweb
   `9443` endpoint at all** — neither "migrate to FQDN" nor "stay literal IP." Left
   unclassified, it defaulted into the Watcher bucket.

The user's judgment: placing mqweb on the management plane is an architectural
mistake, because **mqweb is part of the infrastructure being instrumented, not
part of the instrumentation observing it.** This epic makes reality faithful to the
already-correct §8.3 decision. It **ratifies and enforces**; it does not redesign.

## 2. Doctrine & principles

This epic inherits the lab's established plane doctrine (epic #21 spec §2):

- **Management is "The Watcher" (Uatu).** `net-mgmt` observes every node and
  carries Ansible control, but **nothing routes through it**. The realism budget is
  spent on the **app↔service (data) axis**, not the watcher plane. mqweb — an
  admin/content control surface for a live QM — belongs on that data axis. Treating
  it as a mgmt-plane service conflates the observer with the observed.
- **Mimic production where it matters.** In a real estate, admins reach a QM's REST
  API at the service address (a VIP or the live node), not over an out-of-band
  management network that exists only for lab instrumentation.
- **Minimal dependency.** The correction adds no new runtime service and no new
  network-enforcement dependency (see §6).

## 3. Grounding: current mqweb architecture

Established from the code (see §1 citations and below):

- **Placement — already correct in shape.** mqweb is a per-node systemd service
  (`roles/mqweb/templates/mqweb.service.j2`), applied on every Pacemaker and RDQM
  node in both sites and on `svc-sim`
  (`ansible/_pcmk-cluster-ha.yml:137`, `_pcmk-dr-replication.yml:40`,
  `_rdqm-cluster-ha.yml:20`, `_rdqm-dr-replication.yml:25`; svc via the `mq-qmgr`
  role `roles/mq-qmgr/tasks/main.yml:24-26`). Each administers whatever local QM is
  present. Never centralized.
- **Native HA — mqweb is entirely absent.** No Native HA play applies the `mqweb`
  role (`mq-nativeha` includes only `mq-diag-logging`). This is a gap against the §1
  "REST on **every** queue manager" hard requirement
  (`docs/specs/2026-06-03-mq-cluster-lab-design.md:192,1084`).
- **Planes** (`lab/topology.yaml`): `net-mgmt` `10.50.0.0/24` (Watcher / Ansible),
  `net-data-a/b` `10.10.1/2.0/24` (app data; VIPs pcmk `.200`, rdqm `.100`),
  `net-ext` `10.60.0.0/24` (inter-business WAN to the SVC counterparty), plus
  hb/san/wan fabrics. `10.30.0.x` is the **legacy** net-client plane, no longer in
  the topology.
- **VIP realization.** The data-plane VIP is a Pacemaker `IPaddr2` floating IP in
  the QM resource group, following the QM (`roles/mq-pcmk-qmgr/tasks/main.yml:181-184`).
  It exists only on the active node. Liberty `httpHost` is single-valued, so mqweb
  cannot bind *only* the VIP and still run on standby nodes — to answer on the
  floating VIP at all, the socket must be `*`.
- **No observability component consumes mqweb REST.** The cockpit collectors read
  local `crm_mon`/`drbd`; the MQ Prometheus exporter connects to the QM over a
  SVRCONN (client mode), not REST. `alloy` *tails* mqweb's `messages.log` — i.e.
  mqweb is correctly the **observed**, never the observer. Re-homing mqweb's
  canonical address therefore breaks no instrumentation.
- **No network enforcement exists.** The lab has no firewall of any kind
  (nftables/iptables/ufw/firewalld absent); planes are separated only by being
  distinct libvirt NICs.
- **The lab's own content-apply does not use REST.** All six content-bearing site
  plays apply QM objects via Ansible **`runmqsc`** on the active instance — not over
  REST. `src/mqlab/apply.py` (+ `content/{qm-main,svc-sim}.yaml`) is a **legacy
  Phase-B** artifact (the single-QM `10.30.0.10`/`10.20.0.50` model that predates the
  #351 four-stack topology), referenced only by its test, the Phase-B/DR plans, and
  the stale bring-up runbook. mqweb REST is thus *enabled* on every QM (the §1
  capability requirement) but not *consumed* by current provisioning. This diverges
  from design §8's intended pymqrest content plane — a divergence this epic does not
  close (see §8).

## 4. The correction

Ratify §8.3 and align reality to it via three corrections, plus one explicit
binding decision.

### 4.1 Classify mqweb as data-plane infrastructure

Enter the mqweb `9443` endpoint into `docs/reference/dns-fqdn-inventory.md` as a
**data-plane infrastructure** surface, closing the "unclassified → defaulted to
Watcher" gap:

- pcmk / RDQM stacks → the QM's **data-plane VIP** (eligible for the VIP's
  generated DNS name, e.g. `pcmk-vip.client.com`, which the #21 DNS work already
  emits).
- Native HA stacks → the **active instance's data-plane node IP** (the per-node
  list; no VIP exists).

State the plane explicitly in `docs/specs/2026-06-03-mq-cluster-lab-design.md`
§8.3 (and the §1 scope note) so the classification is unambiguous going forward.

### 4.2 Establish the canonical published REST address

Define, for each QM, the **canonical published mqweb endpoint** — the one
authoritative address anything connecting to that QM's REST/Console uses — derived
from `lab/topology.yaml`:

- pcmk / RDQM → the QM's **data-plane VIP** `:9443`.
- Native HA → the **active instance's** `<node-IP>:9443`, discovered at runtime by
  the existing *"find the active Native HA instance"* step already used to place
  MQSC on the live instance (`ansible/site-nativeha-ubuntu.yml:26`). (Topology yields
  the candidate node list; the active member is a *runtime* resolution, not a static
  literal.)
- `svc-sim` → **special case:** it models the *counterparty*, lives on `net-ext`
  (`10.60.0.50`, the inter-business WAN) with no app-data-plane presence, and in a
  real estate we would not administer its mqweb at all. Its mqweb is administered
  (lab-only) over its `net-ext` address and is explicitly **outside** the "our data
  plane" framing.

**Who consumes this address.** Not current provisioning — content-apply is Ansible
`runmqsc` (§3). The canonical endpoint serves: human admins (Console / ad-hoc REST),
the §1 requirement that REST be *available and addressable* on every QM, and future
pymqrest-based tooling (including #252). Record it authoritatively in the FQDN
inventory (§4.1) and as a topology-derived value, following the QM-as-a-variable /
de-hardcoding principle already driving the cockpit work — the endpoint comes from
one source, never a hardcoded literal.

**Legacy cleanup, honestly.** `src/mqlab/apply.py`, `content/{qm-main,svc-sim}.yaml`,
and the bring-up runbook's `10.30.0.10:9443` example are legacy Phase-B. Update or
retire them so they no longer masquerade as the live content path — do **not** re-aim
`apply.py` as though it drives current provisioning (it does not).

### 4.3 Native HA mqweb coverage (the real build)

Add the `mqweb` role to the Native HA plays so mqweb is **present and running on
every** instance (rhel + ubuntu arms; site-A HA + site-B CRR — six instances per
arm) as the stateless, always-on per-node service. Note the Native HA semantics:
only the **active** instance's QM is serving (the other site-A members are raft
replicas; the site-B trio is a CRR recovery QM), so **admin is served by the active
instance's mqweb** — reached via the active-instance resolution step (§4.2). We do
not claim symmetric admin across all instances. This closes the §1 gap (REST
*available* on every QM). No VIP and no new discovery machinery are introduced.

## 5. Binding decision (explicit)

`httpHost=*` **stays.** mqweb is a stateless pass-through; leaving it listening on
all NICs is an accepted configuration. The correction is *canonical addressing +
classification*, not socket-level or network-level restriction. This is consistent
with the lab deliberately not modeling firewalls, and with the floating-VIP
constraint in §3 that forces a `*` socket anyway. mqweb becomes **architecturally**
part of the data-plane infrastructure; nothing hard-blocks the mgmt plane, and that
is acceptable for this epic.

## 6. Scope & non-goals

**In scope — every stack:** pcmk-ubuntu, pcmk-rhel (A+B), RDQM (A+B), Native HA
(rhel + ubuntu, A+B), svc-sim.

**Non-goals (explicit):**

- **Network / firewall enforcement.** Real per-plane isolation (e.g. nftables so
  `9443` is refused on mgmt) is **deferred to a follow-on epic**, seeded by this
  epic's closing brainstorm task (#41). It should be framed lab-wide, around what we
  actually want to filter between planes — especially the inter-business /
  counterparty boundary on `net-ext` — to make the lab more reflective of a
  real-world configuration.
- **Name-based cert + verified-REST flip.** That is `#252` under `#13`. This epic
  *unblocks* it (plane + address + name become settled) but does not perform it. The
  verify-flip is gated on external `pymqrest #518` (widen `verify_tls` to accept a
  CA bundle), which is temporarily parked while pymqrest's publishing is migrated to
  the org — deliberately kept **off this epic's critical path**.
- **Changing how observability treats mqweb.** `alloy` already tails its logs
  correctly (mqweb = observed); no change.

## 7. Verification & acceptance

Per stack:

- The canonical published endpoint is reachable and authenticates on the expected
  address (VIP for pcmk/RDQM; active-instance node IP for Native HA; `net-ext` for
  svc-sim) — verified by a direct probe (e.g. `openssl s_client` / a REST GET), with
  no reliance on a hand-passed or legacy literal.
- **Survives failover:** pcmk/RDQM — the VIP moves to the new owner and the endpoint
  follows; Native HA — a switchover moves the active instance and the resolution step
  re-points to the new active member's mqweb.
- mqweb is **present and running on every** Native HA instance; the active instance's
  mqweb serves admin (no claim of symmetric admin across replicas).
- mqweb is classified as data-plane infrastructure (svc-sim as the counterparty
  special case) in the FQDN inventory and design §8.3/§1.

Because this touches provisioning, the **cold-rebuild acceptance gate** applies: a
full VM cold rebuild must prove the changes one-pass; lint-green is not "done."

## 8. Relationships

- **Unblocks `#252`** (under `#13`): once plane + address + name are settled, #252
  shrinks to cert-SAN + verify-flip, gated only on `pymqrest #518`.
- **Spawns** a lab-wide firewall / plane-enforcement epic via closing task `#41`.
- **Candidate future epic — dogfood the REST content plane.** Design §8 intended QM
  content to be applied through `pymqrest.ensure_*` against the REST API, but the
  stacks shipped Ansible `runmqsc` (§3). Migrating content-apply `runmqsc` → pymqrest
  across all stacks would realize that intent and make `apply.py` a live consumer of
  the canonical endpoint this epic establishes. It is **out of scope here** (a much
  larger, orthogonal change) and noted as a possible follow-on.

## 9. Task breakdown

Implementation tasks (land in `mq-resiliency-lab-for-linux`, linked under `#39`):

1. **Classify + document** mqweb's plane — FQDN inventory entry (svc-sim as the
   counterparty special case) + design §8.3/§1 statement.
2. **Establish the canonical published REST address** — topology-derived value +
   authoritative FQDN-inventory record; honestly update/retire the legacy
   `apply.py`/`content/`/runbook rather than re-aiming them as the live path.
3. **Native HA mqweb coverage** — `mqweb` role onto every instance (both arms, both
   sites) as the always-on per-node service; wire active-instance resolution so the
   published endpoint reaches the active member's mqweb.
4. **Cross-stack verification + failover proof** (+ cold-rebuild acceptance).

Bookend tasks (in `.github`, already created):

- **`#40` Documentation** — this spec + the plan (first task; its PR publishes them).
- **`#41` Follow-on brainstorm** — seed the firewall / plane-enforcement epic +
  what-shipped review (closing).
- **`#42` Documentation review** — verify the human-facing / versioned site docs
  reflect the mqweb-plane correction (final close gate).
