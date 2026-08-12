# DNS hostname resolution for the lab — design spec

- **Epic:** `logical-minds-foundry/.github#21`
- **Design task:** `mq-resiliency-lab-for-linux#454`
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-07-02
- **Topology reference:** corrected network diagram (see the review artifact)

## 1. Problem & motivation

Live triage on the running `nativeha-ubuntu` stack (#454) found that IBM MQ
performs a **reverse (PTR) lookup on channel peer IP addresses** on the
connection path. The lab's private IPs have no reverse mapping, so the host
resolver forwards the PTR query to an uplink that cannot answer it and **times
out at ~10 s** per lookup (`AMQ9788W: Slow DNS lookup`, `getnameinfo`). Every MQ
(re)connect pays that tax per instance IP it touches — flooding logs and, for a
resiliency lab, **crippling client reconnection across a Native HA failover**.

A tactical `/etc/hosts` stopgap has already shipped to unblock the live lab.
This epic **replaces** that stopgap with a production-shaped DNS infrastructure,
for two reasons:

1. **Fidelity.** Nobody runs an estate of `/etc/hosts` files in production; they
   run DNS. The lab exists to model real-world MQ resiliency, and the interaction
   between MQ and the naming layer is part of that reality.
2. **Strategy.** The security workstream will use hostname-based channel
   authentication (CHLAUTH). Per IBM, hostname CHLAUTH rules are matched **only**
   when `REVDNS(ENABLED)` (the default) **and** working PTR records exist. So
   reverse DNS is a **functional requirement** of the security guides, not a
   cosmetic nicety.

## 2. Doctrine & principles

- **Mimic production where it matters; cut corners only in supportive
  scaffolding.** The observability network (which spans both sides — something no
  real deployment would do) is admitted, deliberate unrealism that exists purely
  for lab instrumentation. DNS is on the *matters* side of that line.
- **Management is "The Watcher" (Uatu).** The `net-mgmt` plane observes every
  node and carries Ansible control, but **nothing routes through it**; keeping it
  non-transit is our responsibility to enforce. The realism budget is spent on
  the **app↔service axis**, not the watcher plane.
- **Minimal dependency.** Anything that must survive a deliberate DNS outage
  (a host's own identity, the HA/replication fabric) must **never** depend on
  DNS.
- **Failure-domain independence.** DNS is its own failure domain (a dedicated
  infra node per side), so it can later be failed independently to measure MQ's
  sensitivity to resolution loss.

## 3. Corrected topology (grounding)

This spec is grounded in `lab/topology.yaml` — the single source of truth for the
lab's nodes, planes, sites, and addressing. Two facts reshaped the design and
must be understood before the architecture:

1. **The network is already segregated.** The data plane is **not** a shared
   client/service subnet needing a split. `net-data-a` (10.10.1.0/24) is **site A
   (live)** and `net-data-b` (10.10.2.0/24) is **site B (DR)** — the HA+DR axis.
   Four HA arms (pcmk, rdqm, nha-ubuntu, nha-rhel) each run a site-A and a site-B
   group; `app-client` is dual-homed on both data planes and rides the cross-site
   DR cutover. Pacemaker/RDQM arms carry **VIPs** (e.g. pcmk `10.10.1.200` /
   `10.10.2.200`, rdqm `10.10.1.100`); Native HA arms carry **no VIP**.
2. **The cross-organization boundary already exists** as its own routed plane:
   **`net-ext` (10.60.0.0/24)** — the inter-business WAN between our estate and
   the SVC counterparty's partner VIP (`vip_ext 10.60.0.10`; `svc-sim` at
   `10.60.0.50`).

**Consequence:** there is nothing to re-segment. The original "split the data
plane" task (former Task A) is **dropped**. The planes below already exist:

| Plane | Subnet | Role | DNS |
|---|---|---|---|
| `net-mgmt` | 10.50.0.0/24 | OOB admin + Ansible control — The Watcher; non-transit | bootstrap only |
| `net-data-a` | 10.10.1.0/24 | Site A (live) data; client-facing VIPs | forward + PTR |
| `net-data-b` | 10.10.2.0/24 | Site B (DR) data; DR VIPs | forward + PTR |
| `net-ext` | 10.60.0.0/24 | Inter-business WAN (our estate ↔ SVC) | forward + PTR (cross-org) |
| `net-wan` | 10.99.0.0/24 | Cross-site DR replication path | IP / hosts |
| `net-hb-a/b` | 172.16.1-2.0/24 | Heartbeat / cluster membership fabric | hosts (IP-configured) |
| `net-san-a/b`, `net-san-rhel-a` | 10.40.1-3.0/24 | Storage / DRBD replication | IP / hosts |

## 4. Key research findings (grounding)

All citations are IBM MQ **9.4.x** docs, fetched via `tools/ibm_doc_cache.py`
and cached under `build/refs/ibm-docs/ibm-mq/9.4.x/`. **Data** (what a source
literally says) is separated from **judgment** (our inference). Items that could
not be machine-fetched (IBM Support/APAR pages) are marked **unverified**.

- **Server reverse-DNS surface — one switch.** `ALTER QMGR REVDNS` (default
  **ENABLED**, inbound-TCP-only, demand-driven) feeds exactly two documented
  consumers: **(a)** hostname-based CHLAUTH rule matching and **(b)** host names
  in error messages. `REVDNS(DISABLED)` silently makes hostname CHLAUTH rules
  fail to match.
- **Client reverse-DNS surface — no knob.** There is **no** documented
  client-side control to disable or tune the reverse lookup (verified across the
  `mqclient.ini` TCP stanza, the Channels stanza, and the complete MQ
  environment-variable list). `REVDNS` is inbound-server-only. **The client
  surface can only be fixed at the OS resolver layer** — correct, fast PTR
  records (or a fast-failing resolver).
- **No config-file timeout covers the resolver.** No `qm.ini`/`mqclient.ini`
  keyword bounds or disables reverse DNS; `Connect_Timeout` bounds the socket
  `connect()`, **not** the resolver call.
- **FQDN surface is nearly universal.** `CONNAME`, `LOCLADDR`, listener
  `IPADDR`, LDAP `AUTHINFO CONNAME`, Native HA `ReplicationAddress`, mqweb
  `httpHost`, and RDQM node `Name` all accept an FQDN and forward-resolve it. The
  only genuine IP-only holdouts are the **RDQM floating/VIP** and the RDQM
  `rdqm.ini` replication/heartbeat link fields.
- **Native HA client connectivity — IBM documents the 3-address list, not a
  VIP.** *"When connecting clients and channels to a Native HA queue manager
  running outside of OpenShift or Kubernetes… You can use a comma-separated list
  of three possible addresses… to locate the currently active instance."* A
  floating IP is explicitly scoped to RDQM/Appliance (Pacemaker moves the VIP+QM
  as a unit); the single-address pattern exists only in Kubernetes. A VM VIP for
  Native HA would be a **custom** build, **not IBM-blessed**.
- **Client multi-CONNAME algorithm.** Addresses are tried in **listed order**
  (only `CLNTWGHT` reorders). A **refused** peer (TCP RST — a Native HA replica)
  is skipped **immediately, regardless of `Connect_Timeout`**; a
  **down/blackholed** peer costs a full `Connect_Timeout` (**20 s** default) each.
  CCDT-exclusive selection features: queue-manager groups, `CLNTWGHT`,
  `AFFINITY`.
- **Server-to-server differs.** A SDR/SVR channel walks the CONNAME list
  top-to-bottom (one full walk = one `SHORTRTY`/`LONGRTY` attempt), hard-stops
  and requires a **manual restart** on exhaustion, and rolls back the in-flight
  batch on failover. (The commonly-cited `SHORTRTY`/`LONGRTY` defaults are **not**
  published in the 9.4 docs; they live in `SYSTEM.DEF.SENDER` — confirm live.)
- **Reconnect root cause of #454.** Automatic client reconnection protects only
  an **already-established** connection — *"the MQCONNX call itself is not tried
  again if it fails."* Captured as a client-code requirement in **#472**.
- **APAR IT27381** (stalled PTR → HA partition) is **MQ Appliance-only**; the
  Linux equivalent is a **hypothesis to test** — a motivation for the deferred
  fault-injection work. *(unverified — IBM Support page not machine-fetchable.)*

## 5. Target architecture

### 5.1 Two businesses, two domains

The lab models the classic relationship where **our** business is the **client**
of an external **service** run by a counterparty. The boundary is the existing
`net-ext` inter-business WAN, and the domains follow the *relationship*:

- **`client.com` — our entire estate** (the side we instrument). Everything on
  our side of `net-ext`: `app-client`, **all our queue managers** (pcmk / rdqm /
  nha, sites A **and** B), `obs`, `mon-probe`, and our infra node.
- **`service.com` — the counterparty** (mocked, not instrumented). Everything on
  the far side of `net-ext`: `svc-sim` and its partner VIP. It handles messages
  for us; we model it, we don't measure it.

**Note (corrects an earlier framing):** the client/service line is **not**
app-vs-QM — the app and our QMs are *both* `client.com`. It is **us vs. the
counterparty across `net-ext`.** The security payoff lands exactly there: the
cross-org channel auth the guides teach is at the **`client.com` ↔ `service.com`
boundary over `net-ext`** (our QMs authenticating the counterparty and
vice-versa). The `app-client` → our-QM hop is internal to `client.com`.

We model only the DNS layers the two sides actually touch — no root-server or
deep-delegation realism.

### 5.2 Naming scheme — Option A (function-suffix, per-domain flat zone)

Each multi-homed host exposes a **function-indicating** name per interface, plus
a base name that is its primary service-facing identity:

- Per-interface **A records**, one per NIC, suffixed by the **plane name from
  `topology.yaml`**: `<host>-data-a`, `<host>-data-b`, `<host>-mgmt`,
  `<host>-hb-a`, `<host>-ext`, etc.
- **Base name is a CNAME → the host's primary (service-facing) interface** — for
  a site-A QM that is its `-data-a` address. The generator derives "primary" from
  the node's role in `topology.yaml`.
- **`-mgmt`, `-hb-*`, `-san-*`, `-wan` are never reachable via the base name**
  (no leaking internal/observation planes into the primary identity).
- **Service (VIP) names.** Pacemaker/RDQM VIPs get a single service FQDN
  (`<stack>-vip.client.com` → the VIP); the partner-facing `vip_ext` gets a name
  resolvable from `service.com`. **Native HA** has no VIP: clients dial the
  **three per-instance FQDNs** (`<qm>-a1-data-a`, `-a2`, `-a3`) in `CONNAME`.

### 5.3 BIND9 authoritative DNS on per-side infra nodes

- **The one net-new component is a per-side infrastructure node.** Our-side
  `infra` is authoritative for **`client.com`** (the large zone: our whole
  estate); a **mock** service-side infra is authoritative for **`service.com`**
  (small: `svc-sim` + VIP), modeling "their" DNS so that *"the counterparty's DNS
  is down"* can be fault-injected independently later. Each infra node forwards
  cross-org lookups to the other.
- The infra node is **DNS (+ future LDAP and other failable naming/auth
  services)** — **not** a gateway/router (routing already exists; nothing to
  re-segment).
- **Reverse zones.** Our infra is authoritative for the reverse zones of our
  internal planes (`net-data-a`, `net-data-b`, and — for admin readability —
  `net-mgmt`). The **`net-ext` reverse zone is the one that spans the org
  boundary**; it gets a single designated authority (default: our-side infra owns
  `0.60.10.in-addr.arpa`, holding the `svc-sim` PTR too). The exact split
  (single-owner vs. delegation) is a plan-time detail for Task B.
- BIND chosen for production fidelity (real `SOA`/`NS`/`A`/`CNAME`/`PTR`
  semantics) and for the **realistic failure modes** the later fault-injection
  work needs.

### 5.4 Zone source-of-truth

- Zones are **generated, never hand-edited.** An Ansible role reads
  **`lab/topology.yaml`** and templates the forward and reverse zone files from
  it — single source of truth, no drift.
- **Good news from the topology review:** `topology.yaml`'s per-node `nics:` map
  already carries the **plane-named interfaces** (`net-data-a`, `net-mgmt`, …) —
  i.e. the **function labels the generator needs already exist.** The only new
  data is a per-node **`org`** tag (`client` | `service`) selecting the forward
  zone; the suffix + address for every interface come straight from `nics:`.
- The BIND zone file is a **generated artifact** in BIND's mandated format; we
  author only YAML.

### 5.5 Resolver & minimal `/etc/hosts`

- Guests point their resolver at their side's infra node **by IP**
  (netplan/`resolv.conf`); `nsswitch` stays `files dns`.
- **`/etc/hosts` holds only what must survive a DNS outage:**
  1. `localhost` (v4/v6).
  2. The host's **own** identity (FQDN + short name → primary address).
  3. The **HA/replication fabric** — heartbeat and replication-plane peer names
     (Pacemaker/Corosync peers, Native HA replication addresses). The HA fabric
     must not depend on the breakable DNS service; a deliberate DNS outage must
     not partition a cluster or stall replication. *(Formalizes what
     `pcmk-cluster` and `rdqm-ha` already do.)*
- **Everything on the app↔service axis is DNS-only** and is *allowed* to fail
  under fault injection. The dividing line is exactly **"must-survive-DNS-loss"
  vs "we-want-to-measure-DNS-loss."**

### 5.6 Native HA client connectivity

- **No VIP for Native HA.** Clients list the **three per-instance FQDNs** in
  `CONNAME`. Reject a single multi-A "service name" (MQ may not exhaustively try
  every A record). **CCDT deferred** to the future client-connectivity effort —
  the DNS design only makes the three FQDNs exist and be listable.
- **VIPs are retained on the Pacemaker/RDQM arms** (Pacemaker moves the floating
  IP + QM as a unit): a single service FQDN → the VIP. The lab thus teaches that
  the correct client-connection name shape depends on the HA technology.

## 6. Task breakdown

The epic is **complete when Tasks B and C land** (plus the infra nodes, delivered
in B). Each is its own spec→plan→build with a **cold-rebuild acceptance gate**
(lint-green ≠ done). *(Former Task A — network re-segmentation — is dropped: the
review showed the network is already segregated.)*

- **Task B — BIND9 + the infra nodes.** Add the per-side infra node(s); stand up
  `named` authoritative for `client.com` (and the mock `service.com`); generate
  forward + reverse zones from `lab/topology.yaml`; configure resolvers on all
  guests; seed the minimal `/etc/hosts` bootstrap. **Remove the tactical stopgap
  hosts only after DNS is proven** to replace them (no coverage gap). Cold-reboot
  validation.
- **Task C — FQDN migration sweep.** Point the lab's configuration at the DNS
  names where they make sense; hunt down remaining raw IPs (greppable after B).
  **Guiding principle: err toward minimal dependency** — e.g. configure heartbeat
  channels with **literal IPs** (no DNS or hosts dependency for the HA fabric),
  keeping the IP+name in `/etc/hosts` only for admin readability. Where FQDN vs.
  IP is a genuine 50/50, decide case-by-case, defaulting to least-dependency.
  Cold-reboot validation.

## 7. Explicitly out of scope (separate later efforts)

- **DNS fault-injection capability** — the systematic "break DNS and measure MQ's
  sensitivity" work (including testing the IT27381-on-Linux hypothesis, and
  independently failing `service.com` DNS). Its own major iteration and brainstorm.
- **Detailed client-connectivity design** — API usage, timeouts, and parameters
  for reliability across HA and DR failover. Largely DNS-independent; its one
  load-bearing defect is already captured (#472).
- **Broader "when should we prefer a name over an IP" policy** beyond the Task C
  sweep.

## 8. Related work

- **#472** — Clients must implement initial-connect retry (precursor client fix;
  root cause of the #454 reconnect symptom).
- **#454** — This design task and the diagnostic origin.

## 9. Sources & verification

Fetched IBM MQ 9.4.x docs (cached under `build/refs/ibm-docs/ibm-mq/9.4.x/`):
`reference-alter-qmgr-alter-queue-manager-settings`,
`reference-set-chlauth-create-modify-channel-authentication-record`,
`mechanisms-channel-authentication-records`,
`mqclientini-tcp-stanza-client-configuration-file`,
`mqclientini-channels-stanza-client-configuration-file`,
`qmini-tcp-stanza-file`, `qmini-channels-stanza-file`,
`variables-environment-descriptions`, `reference-amq9xxx-remote-messages`,
`codes-amq-messages-multiplatforms`, `attributes-channel-mqsc-keywords-b`,
`attributes-channel-mqsc-keywords-c`, `attributes-channel-mqsc-keywords-d-l`,
`attributes-channel-mqsc-keywords-s`, `reference-define-channel-define-new-channel`,
`table-queue-manager-groups-in-ccdt`, `managers-role-client-channel-definition-table`,
`cscccddp-creating-client-connection-channel-mq-mqi-client-using-mqserver`,
`cscccddp-creating-client-connection-channel-mq-mqi-client-using-mqcno`,
`managers-channel-client-reconnection`, `restart-automatic-client-reconnection`,
`availability-native-ha`, `availability-creating-deleting-floating-ip-address`,
`reference-rdqmint-add-delete-floating-ip-address-rdqm`,
`configurations-rdqm-high-availability`,
`ha-example-deploying-simple-native-configuration-linux`,
`sdcqmk-example-configuring-native-ha-in-kubernetes-self-deployed-queue-managers`,
`reference-define-listener-define-new-listener-multiplatforms`,
`reference-define-authinfo-define-authentication-information-object`,
`api-configuring-http-host-name`.

**Unverified / needs manual fetch** (IBM Support/APAR pages, not
machine-fetchable): APAR **IT27381**; the canonical AMQ9788W per-message 9.4 page
(text corroborated via a developer Q&A snippet); the MQCNO Options reference
(quoted from 9.3.x — the 9.4.x URL did not resolve). Third-party commentary
(Colin Paice's blog) was used only for mechanism intuition, never as
authoritative.

**Empirical items to confirm live** (not documented): whether a stalled reverse
lookup blocks *other* MQ DNS work; the literal `SHORTRTY`/`LONGRTY` defaults in
`SYSTEM.DEF.SENDER`; and the precise TCP behavior of a Native HA replica's
listener (refuse vs. accept-then-reject).
