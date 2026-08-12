# Stage 2 — Internal MQ Authorization — Design

> **Status:** design, first pass — brainstormed + pushback-hardened 2026-07-13.
> **Date:** 2026-07-13
> **Author:** Phillip Moore (with Claude)
> **Epic:** [logical-minds-foundry/.github#74](https://github.com/logical-minds-foundry/.github/issues/74)
> (harvested from the Stage 2 design task
> [logical-minds-foundry/mq-resiliency-lab-for-linux#249](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/249),
> which rolls up the **MQ security hardening** epic
> [logical-minds-foundry/.github#13](https://github.com/logical-minds-foundry/.github/issues/13)).
> **Relationship:** **Stage 2 of 2**, the authorization layer on top of Stage 1
> connection TLS
> ([`docs/specs/2026-06-17-connection-tls-pcmk-ubuntu-design.md`](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/blob/develop/docs/specs/2026-06-17-connection-tls-pcmk-ubuntu-design.md)
> in the member repo). Stage 1 *authenticated* every connection (mutual TLS +
> `SSLPEER`); Stage 2 *acts* on those identities. Anchored on the
> identity/authorization research brief
> ([`docs/reports/2026-06-24-ibm-mq-client-identity-authorization.md`](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/blob/develop/docs/reports/2026-06-24-ibm-mq-client-identity-authorization.md),
> #347).

---

## 1. Why this exists

Stage 1 secured and **cert-authenticated** every connection of the `pcmk-ubuntu`
arm, but deliberately left internal authority wide open: every QM runs
`CHLAUTH(DISABLED)`, `CONNAUTH(' ')`, and every channel/SVRCONN carries a fixed
`MCAUSER('mqm')`. The certificates *assert* an authenticated identity to each
queue manager; nothing acts on it. That is *authenticated but unauthorized* — a
conscious lab stepping stone, indefensible in production.

Stage 2 closes that gap on **our** queue manager: retire `MCAUSER('mqm')` /
`CHLAUTH(DISABLED)`, map each authenticated certificate DN to a **non-privileged
service account** (`CHLAUTH SSLPEERMAP`), and lock the queues down to a **minimal
authority surface** per identity (`setmqaut` / OAM). The research brief (#347)
settles the *pattern* — deny-all back-stop → `SSLPEERMAP` cert-DN → low-privilege
per-partner service account → minimal `setmqaut` to the group. This spec applies
that pattern to the lab's actual topology and derives the minimal grant surface
empirically.

## 2. Scope & non-goals

**In scope:** enforced authorization on our app QM **`qm_app`** (concretely
`PCMKAPP`, §3) on the `pcmk-ubuntu` arm — `CHLAUTH(ENABLED)`, a deny-all
back-stop, DN→account `SSLPEERMAP` rules, OS service accounts, and minimal
`setmqaut` grants, plus the validation that proves both authorized flows and
*denied* attempts (including after failover).

**Non-goals:**

- **The `qm_svc` counterparty (`SVCQM`) is out of scope — deliberately.** It is
  the black-box counterparty scaffold: test infrastructure that stands in for the
  remote side (one shared `SVCQM` serves every arm, #446) and generates response
  messages. It is **not the thing being instrumented**. It keeps
  `CHLAUTH(DISABLED)` / `MCAUSER('mqm')`. We secure the subject, not the scaffold.
- **The other arms** (`nativeha`, `nativeha-ubuntu`, `rdqm`) — the model here is
  authored arm-agnostically (§7) so fan-out is a delta, but only `pcmk-ubuntu` is
  *committed* by this spec. Fan-out is a follow-up task (§9).
- **CONNAUTH / passwords** — authentication is already carried by the
  certificates (Stage 1). Layering password auth is an orthogonal *authentication*
  concern, explicitly deferred (§4).
- **LDAP authorization** — the intended clean end-state, but a separate future
  brainstorm gated on the lab's infra nodes hosting a directory (§6, §9).
- **mqweb REST authorization** — `pymqrest`→mqweb is governed by mqweb roles, not
  channel `MCAUSER`; a separate posture-check task (§9).

## 3. The authorization surface (what gets secured on our app QM)

> **Naming — variable-driven, never hardcoded (#350/#351).** The lab derives QM
> and inter-QM channel names from topology, not literals. Our app QM is
> **`qm_app`** (from `short: PCMK` → concretely **`PCMKAPP`** on this arm); the
> counterparty is **`qm_svc`** — the *single shared* Business-B QM **`SVCQM`**
> (#446). The inter-QM channels are **`chl_to_app`** (RCVR into us, concretely
> `SVCQM.PCMKAPP`) and **`chl_to_svc`** (SDR out). The client SVRCONN names
> (`APP.SVRCONN`, `MON.SVRCONN`) and the local queues (`APP.REPLY`,
> `SVC.REQUEST`) *are* literals in the roles. Every CHLAUTH/`setmqaut`/channel
> snippet below is authored against the **variables**; concrete values appear only
> as illustration. Hardcoding `SSLPEERMAP` on a wrong literal channel name would
> silently miss and drop the channel to the deny-all back-stop.

Split the counterparty's own framing — **client connections vs the server
(QM↔QM) connection**. On `qm_app` there are exactly three inbound channels:

| Kind | Channel | Peer cert DN (Stage 1) | Identity role |
|---|---|---|---|
| client | `APP.SVRCONN` | `O=app-org, OU=apps` | our application client |
| client | `MON.SVRCONN` | `O=app-org, OU=ops` | monitoring / observability |
| server | `{{ chl_to_app }}` (RCVR, e.g. `SVCQM.PCMKAPP`) | `O=svc-org` | the remote counterparty ("service app") |

There is **no generic "inter-QM" identity.** The receiver that takes messages
*from the remote QM* does not need an anonymous channel identity; it needs *the
counterparty's assigned identity*. We know that remote QM is this one
counterparty, so we assign it an ID and pin the receiver's `MCAUSER` to it, with
**minimal** access — not the over-broad authority receiver channels are usually
given. That counterparty ID **is** the "service app." The real duality is
**client-app vs service-app**, and inter-QM dissolves into service-app.

## 4. Identities & namespace

Three OS service accounts, one per role. A coherent, parallel, `mq`-namespaced
scheme, each ≤12 characters (so an adopted-context ID is never truncated), each
in its own private primary group:

| Role | Channel on `qm_app` | Service account | Group |
|---|---|---|---|
| client app | `APP.SVRCONN` | `mqapp` | `mqapp` |
| monitoring | `MON.SVRCONN` | `mqmon` | `mqmon` |
| service app (remote counterparty) | `{{ chl_to_app }}` RCVR | `mqsvc` | `mqsvc` |

`mqmon` is close to `mqm` but not too close; `mqsvc` echoes the existing SVC /
counterparty naming already in the lab. No collisions with existing accounts.

**CONNAUTH stays disabled.** The certificate is the credential — Stage 1 mutual
TLS + `SSLPEER` is strong (possession-of-object) authentication. Stage 2's thesis
is *authorization*, not another authentication factor, and a password is just
another secret to provision and rotate across the 3+3 nodes. CONNAUTH is recorded
here as an explicit non-goal / possible future defense-in-depth demonstration.

## 5. The mechanism on our app QM

Authored against the topology **variables** (§3), applied in order:

1. **Enable channel authentication:** `ALTER QMGR CHLAUTH(ENABLED)` (CONNAUTH
   stays `' '`). Keep MQ's V8 default `BLOCKUSER(*MQADMIN)` rule — no inbound
   channel may assert a privileged ID.
2. **Deny-all back-stop, first:**

   ```text
   SET CHLAUTH('*') TYPE(ADDRESSMAP) ADDRESS('*') USERSRC(NOACCESS)
   ```

   Anything not explicitly mapped ends the channel immediately.
3. **Strip the fixed `MCAUSER('mqm')`** from the three inbound channel
   definitions (set blank). The *only* path to an identity becomes the CHLAUTH
   map; there is no admin identity left to fall back to. (Blank `MCAUSER` is safe
   *only because* of the deny-all back-stop — an unmatched channel is denied, not
   left to assert its own ID.)
4. **Per-channel `SSLPEERMAP` rules** assign the non-privileged accounts (channel
   names are the variables; the inter-QM name **must** be `{{ chl_to_app }}`, not
   a literal, or the rule misses and the counterparty drops to the back-stop):

   ```text
   SET CHLAUTH('APP.SVRCONN')      TYPE(SSLPEERMAP) SSLPEER('O=app-org,OU=apps') USERSRC(MAP) MCAUSER('mqapp')
   SET CHLAUTH('MON.SVRCONN')      TYPE(SSLPEERMAP) SSLPEER('O=app-org,OU=ops')  USERSRC(MAP) MCAUSER('mqmon')
   SET CHLAUTH('{{ chl_to_app }}') TYPE(SSLPEERMAP) SSLPEER('O=svc-org')         USERSRC(MAP) MCAUSER('mqsvc')
   ```

5. **The receiver asserts ownership:** `{{ chl_to_app }}` gets `PUTAUT(DEF)` so
   put-authority keys off `mqsvc` — the counterparty's self-asserted
   `MQMD.UserIdentifier` is disregarded. We decided who they are; we do not trust
   what they stamp. (See the `+setall` research item, §8.)

### 5.1a Cutover safety — no self-lockout

Enabling a deny-all back-stop is the classic CHLAUTH footgun, so the cutover is
provably safe here:

- **The inbound census is a closed set of three** — `APP.SVRCONN`, `MON.SVRCONN`,
  and `{{ chl_to_app }}` — all explicitly mapped. There is **no
  `SYSTEM.ADMIN.SVRCONN`** or other remote admin MQI channel.
- **Administration is immune.** QM config is applied via **local `runmqsc` in
  bindings mode as `mqm`** (`src/mqlab/rest.py`: "QM content via Ansible runmqsc,
  not REST"), which does not traverse a channel — so no CHLAUTH rule can lock out
  administration.
- **Apply atomically.** All maps + the back-stop land in one `runmqsc` batch
  before any channel restart / `REFRESH SECURITY`, so no legitimate channel
  bounces through the deny-all window.

### 5.1 The SSLPEER→SSLPEERMAP granularity lift

Stage 1's channel `SSLPEER` is deliberately coarse — org-only
(`tls_peer_client = O=app-org` for the SVRCONN clients). Stage 2's `SSLPEERMAP`
reads the **full** DN, using the `OU` to discriminate `mqapp` (`OU=apps`) from
`mqmon` (`OU=ops`). The certificates already carry those OUs; Stage 2 simply
begins acting on them. This is exactly the property the partial-DN PKI was built
for: a coarse authentication gate, a fine authorization map, no cert re-issue.

## 6. The OAM substrate & provisioning

- **OS accounts + private groups**, created idempotently by Ansible
  (`ansible.builtin.group` then `ansible.builtin.user`) on **every** QM node.
  No login shell, no password — they exist only to be OAM principals.
- **Group-based authorization** (MQ's Linux default): `setmqaut` grants attach to
  the **group** and are managed via membership. A mapped `MCAUSER` must resolve to
  a real OS principal with a primary group or `setmqaut` returns `AMQ7026E`.
- **The resiliency invariant is first-class.** OAM authority records live in the
  QM data (on the shared LUN here) and are keyed by group **name**; the group must
  exist on whatever node the QM fails over to. "The identity substrate must be
  consistent across every failover target, or authorization silently breaks after
  a failover" is a stated invariant of the arm (verified positively post-failover,
  §10) — the OS-account provisioning targets every node precisely so it holds.
- **Interim by design.** A `group_vars/all/authz.yml` file (mirroring `tls.yml`)
  is the single source for accounts, DN→account maps, and grant sets. Real OS
  accounts are a **deliberate interim**: the clean end-state is **LDAP
  authorization** (§9), at which point the `SHORTUSR` identities no longer need to
  exist on any QM host, the per-node account provisioning is retired, and the
  design simplifies. The OS-account provisioning is written so that flip is a
  subtraction, not a rewrite.

## 7. The authority model (`setmqaut`, grounded in the real queues)

The app flow uses the QREMOTE `SVC.REQUEST` (→ routes to `qm_svc` via the
`qm_app`-named XMITQ) and `APP.REPLY` (`QLOCAL`, where replies land); `HA.TEST` is
the failover smoke queue. These local queue names *are* literals in the role. All
grants to the account's private **group**, minimal:

| Account | QMGR | Queues | Explicitly denied |
|---|---|---|---|
| `mqapp` | `+connect +inq` | `SVC.REQUEST` `+put`; `APP.REPLY` `+get +inq +browse` | no SYSTEM/admin object; cannot `get` from `SVC.REQUEST` |
| `mqmon` | `+connect +inq` | `+sub` on the resource-monitoring `$SYS` topic tree; `+dsp +inq` on reported queues | **no `put`, no `get` anywhere** — the least-privilege showcase |
| `mqsvc` | `+connect` | `APP.REPLY` `+put`; **a dead-letter queue** `+put` (see below) | no `get`; no `SVC.REQUEST`, `HA.TEST`, or admin object |

**These grant sets are a starting hypothesis of the minimal surface, not a proven
floor.** The design intent is to derive the true minimal surface *empirically* —
build it, exercise it, and add only what the running arm proves is required
(§8, §10). The `mqmon` set in particular (exact `$SYS` metric-topic subscribe +
`+dsp` needs) is pinned against the **live exporter** during implementation; it is
the fiddliest and will not be guessed blind.

### 7.1 Dead-letter authority — a forked, research-contingent grant

A receiver MCA that cannot put a message to its target queue (queue full,
`PUT(DISABLED)`, wrong target) falls back to the **dead-letter queue**. If `mqsvc`
lacks `+put` there, that DLQ put is denied and the channel goes **RETRY/STOPPED** —
an *authorization* lockdown creating a latent *availability* failure. So `mqsvc`
needs *some* DLQ put authority. **Which DLQ forks on research** (§8):

- **If** MQ supports a **dedicated per-channel / per-remote-QM DLQ** → give the
  counterparty its own DLQ (e.g. `DLQ.SVCQM`), grant `mqsvc +put` on *that* only,
  and leave the shared `SYSTEM.DEAD.LETTER.QUEUE` untouched. This is the preferred
  outcome — securing the queues is exactly *why* you stop dumping every
  counterparty into one shared DLQ everyone reads.
- **If** MQ's DLQ is **strictly QM-level** (`ALTER QMGR DEADQ(...)` — you may
  *rename* the one queue but cannot have per-channel DLQs, so all traffic
  multiplexes) → grant `mqsvc +put` on that single DLQ, whatever it is named.

The lab does nothing with the DLQ today, so this is also the forcing function for
a proper DLQ story (§9 follow-up). Undeliverable-message behavior gets an induced
test (§10).

## 8. Known corner case + open research

The whole point is to derive a **working minimal surface area** and then extract
it (the reusable authorization component, per the extraction roadmap). We expect
to find corner cases where a specific right is genuinely required — and to
distinguish those from a counterparty's mis-posture.

**Research item — the `+setall` demand (new).** A counterparty once told us we had
to grant `+setall` (context authority) on the **receiver channel**, *even for the
ID we were citing* — which sounded wrong. The research brief flagged exactly this
as an unverified gap (its §5 / §11 open question 1):

- Being asked for `+setall`/`+setid` on a receiver strongly implies **their side
  runs `PUTAUT(CTX)`** — the MCA passes the *message-context* identity through, and
  to *stamp* that context the MCA user must hold context authority. That is in
  direct tension with our `PUTAUT(DEF)` "assert ownership" posture (§5).
- **To establish:** exactly when `PUTAUT(CTX)` forces `+setall`/`+setid` on the
  MCA user; whether our `PUTAUT(DEF)` choice avoids it entirely for inbound; and
  whether *sending* to a counterparty that runs `PUTAUT(CTX)` obliges anything on
  our side. IBM's `PUTAUT` docs returned HTTP 403 during the #347 pass, so this
  needs direct `ibm.com/docs` verification (use the repo's IBM-doc cache tool).

This is tracked as a research follow-up (§9). It may or may not change the
minimal surface for `pcmk-ubuntu`; either way, exercising the arm is what settles
it.

**Research item — per-channel vs QM-level DLQ (new, §7.1).** The `mqsvc` DLQ grant
forks on a mechanism question I will *not* assert without verification: can a
receiver channel / remote QM be routed to its **own** dead-letter queue, or is the
DLQ **strictly QM-level**? What I am confident of *(data)* is the QM-level knob
`ALTER QMGR DEADQ(...)`. What I am *not* sure of *(judgment)* is whether MQ exposes
any clean per-channel/per-remote-QM DLQ selection — it may require a DLQ-handler
rule or a channel exit. **To establish:** verify against IBM docs (per-channel DLQ
feasibility) before choosing the §7.1 branch. If it can't be done, the fallback is
the single shared DLQ grant — the honest, documented outcome.

## 9. Follow-up tasks (new, this epic)

1. **Fan Stage 2 out to `nativeha` + `rdqm`** — apply the arm-agnostic model
   (accounts, DN maps, grant sets) to each arm; the delta is *which nodes* get the
   accounts and *binding the maps to each arm's channel names* (`NHARAPP.NHARSVC`,
   `RDQMAPP`, …). Mirrors the Stage 1 #212 seam.
2. **Brainstorm: add LDAP authorization** — extend the lab's infra nodes with an
   LDAP directory, switch to `idpwldap` authorization, retire the per-node OS
   accounts, and simplify (§6). The cleaner, "right way" end-state.
3. **Research: the `+setall` / `PUTAUT(CTX)` question** (§8) — verify against live
   IBM docs and, ultimately, the running arm.
4. **Research + brainstorm: per-channel vs QM-level DLQ** (§7.1/§8) — settle the
   mechanism, then choose the dedicated-DLQ vs shared-DLQ grant branch.
5. **Brainstorm: DLQ reporting & handling framework for the lab** — the lab does
   nothing with the dead-letter queue today; this hardening is the forcing
   function. Not *strictly* security, but it overlaps hard (securing the queues is
   *why* you stop using the shared system DLQ), so it belongs in this epic.
6. **Confirm mqweb REST authorization posture** — `pymqrest`→mqweb is governed by
   mqweb roles, not `MCAUSER`; verify it is not wide-open admin (outside this
   spec's MCAUSER/OAM model).

## 10. Validation (negative + positive + post-failover)

The point of authorization is **provable denial**, and this feeds the live-lab
validation framework epic (#38). Acceptance is stated as induced-and-asserted
outcomes:

- **Negative (induced denial):**
  - `mqmon` attempts `MQPUT` to `APP.REPLY` (or `SVC.REQUEST`) → `2035
    MQRC_NOT_AUTHORIZED`.
  - A client presenting a cert DN that maps to nothing → the deny-all back-stop
    ends the channel (`AMQ9777` / channel blocked).
  - `mqapp` reaching a non-granted queue or admin object → `2035`.
  - **Assert-ownership demo:** the counterparty stamps `mqm` (or any privileged
    string) in `MQMD.UserIdentifier`; `PUTAUT(DEF)` ignores it, the message is
    authorized as `mqsvc`, and it can *only* reach `APP.REPLY`.
  - **Undeliverable-message / DLQ path (§7.1):** induce a reply that cannot be put
    (e.g. `APP.REPLY` `PUT(DISABLED)`), confirm what `mqsvc`'s DLQ authority must be
    for the receiver to survive rather than go `RETRY/STOPPED` — this is how the
    §7.1 fork is settled empirically.
- **Positive (legit flows survive):**
  - The app trade flow completes end-to-end (`app-client` → `qm_app` → `qm_svc` →
    responder → `APP.REPLY`).
  - The exporter still scrapes metrics.
  - Counterparty replies still land in `APP.REPLY` over the RCVR.
- **Resiliency (post-failover):**
  - Trigger a Pacemaker failover; re-run one positive + one negative check on the
    new active node — authorization **still holds** (the per-node account
    provisioning, §6, is what makes it hold).

These land as induced-and-asserted checks — the first consumers of the #38
`validate` pattern, or interim scripted functional checks. The **cold-rebuild
acceptance gate** applies (the arm must come up one-pass on a fresh VM with
authorization enforced).

## 11. Roles & files touched (pcmk-ubuntu)

- **New:** `group_vars/all/authz.yml` (accounts, groups, DN→account maps, grant
  sets — the single source, mirroring `tls.yml`); an OS-account provisioning task
  set (Ansible `group`/`user`) targeting the QM nodes; a shared, arm-agnostic
  CHLAUTH/`setmqaut` MQSC snippet.
- **Extended:** `mq-pcmk-qmgr` — `ALTER QMGR CHLAUTH(ENABLED)`; strip
  `MCAUSER('mqm')`→blank on `APP.SVRCONN`/`MON.SVRCONN`; add the deny-all
  back-stop, the three `SSLPEERMAP` maps, `PUTAUT(DEF)` on the `{{ chl_to_app }}`
  RCVR, and the `setmqaut` grants; the our-side RCVR map in `inter-qm.mqsc.j2`.
- **Untouched (out of scope):** `their-side.mqsc.j2` (the `qm_svc` counterparty,
  `SVCQM`) — the black-box scaffold stays `CHLAUTH(DISABLED)` / `MCAUSER('mqm')`.

## 12. Definition of done & next steps

- This design captured, reviewed, committed (PR into `develop`, issue #249).
- The arm-agnostic authorization model authored as shared vars/snippet so the
  fan-out (§9, item 1) is a delta, not a rewrite.
- Terminal step: **`writing-plans`** turns this into the pcmk-ubuntu
  implementation plan once reviewed.
- Sequenced follow-ups (§9): fan-out to `nativeha` + `rdqm`; LDAP-authorization
  brainstorm; the `+setall`/`PUTAUT(CTX)` research; mqweb REST posture check.
