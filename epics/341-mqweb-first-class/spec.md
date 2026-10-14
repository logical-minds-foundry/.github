# Spec: MQ Web (Console + REST) as a first-class, secured lab component

- **Epic:** [logical-minds-foundry/.github#341](https://github.com/logical-minds-foundry/.github/issues/341)
- **Documentation task:** [.github#342](https://github.com/logical-minds-foundry/.github/issues/342)
- **Absorbs:** [.github#266](https://github.com/logical-minds-foundry/.github/issues/266) (per-node vs client-mode mqweb; closed as superseded)
- **Builds on:** epic [#39](../39-mqweb-data-plane/spec.md) (mqweb is data-plane infrastructure; canonical endpoints)
- **Amends:** the lab design spec `docs/specs/2026-06-03-mq-cluster-lab-design.md` §8.3, whose "minimal auth registry … security is out of scope" becomes first-class security
- **Repo:** `logical-minds-foundry/mq-resiliency-lab-for-linux` (the lab)
- **Status:** design (brainstorm output, 2026-10-14), pending pushback and review

## 0. How to read this spec

Every claim is labelled as one of:

- **[data]** something a cited source actually says;
- **[judgment]** reasoning built on top of the data;
- **[decision]** a human decision taken in the brainstorm.

IBM sources are MQ **10.0.x** topics (the lab pins MQ 10.0.1.0, `lab/mq-version`). They are cited
as `[Sn]` (see §10) and cached in the lab's shared cache bucket under
`build/cache/refs/ibm-docs/` via `tools/ibm_doc_cache.py`.

**Standing rule [decision].** The design follows IBM's documented recommendations to the
letter. Every departure is listed in the deviations register (§8), with its reason and risk.

## 1. Why

- **[data]** IBM MQ 10.0.1 deprecates MQ Explorer and "intends to remove it in a future release
  with no new functional enhancements" [S1]. The 10.0.1 announcement says the MQ Console "is
  intended to become the primary operational and interactive administration experience for MQ"
  [S2].
  - S2 is a JavaScript-rendered page that was fetched with curl, so it is a **human-verify**
    item.
  - **[decision]** The human attests the Console as the strategic replacement from IBM's
    public announcement.
- **[data]** The Console and the administrative REST API run in one Liberty server, **mqweb**,
  installed from the MQ installation's `MQSeriesWeb` component [S3]. They share one user
  registry, and roles decide who gets which [S4].
  - **[judgment]** REST is therefore exactly as strategic as the Console. Both are first-class.
- **[data, lab]** mqweb is **gated off** by default (`mqweb_enabled: false` in
  `ansible/group_vars/all/admin.yml`, #1171/#1188). The reason was that per-node Liberty
  cold-starts failed on the 2-vCPU QM nodes (#1154).
  - Those measurements predate the huge-page fix for the arm64 nested-virtualization page-fault
    cost (#1240/#1241).
  - As a result, a default bootstrap today runs neither the Console nor REST.
- **[data, lab]** The existing per-node `mqweb` role already provides:
  - org-CA TLS (#250);
  - JSON logging into Loki (#282);
  - a Console/REST binding split (#255/#258);
  - per-node serial start with a readiness gate (#1148/#1169);
  - topology-derived endpoints (`mqlab rest render`, epic #39).
  - **[judgment, human]** About 75–80% of the work is already done. This epic re-enables
    mqweb and completes it, and the main new work is security.

## 2. Decisions

### 2.1 Per-node local mqweb; client mode rejected

- **[decision]** mqweb runs **locally on every QM node**: both sites, every stack, and svc-sim.
  This is the shape the lab design §8.3 and epic #39 already ratified.
- Client mode (one mqweb managing every queue manager remotely) was the brainstorm's first
  direction. It was rejected on IBM's documentation:
  - **[data]** `setmqweb remote` connections serve "the IBM MQ Console and the messaging REST
    API". Their `visibility` is `messaging`, `console`, or both, with no administrative value
    [S5].
  - **[data]** "To administer remote queue managers, you must configure the administrative REST
    API gateway so that at least one queue manager in the same installation as the mqweb server
    acts as a gateway queue manager", which needs transmission queues and sender/receiver
    channels [S6].
  - **[data]** The stand-alone MQ Web Server, IBM's only queue-manager-less mqweb, has no
    administrative REST API [S3].
  - **[decision]** No gateway queue manager is added. A single HA/DR queue manager must be
    manageable either locally or as a client, never through another queue manager.
- **[judgment]** Local mode also brings two capabilities that remote mode lacks:
  - start and stop of the queue manager;
  - the Console's Native HA and replication status view, documented under *local* queue
    managers [S7].

### 2.2 Security model

**Users and roles.** Users live in a Liberty basic registry in `mqwebuser.xml`; the lab has no
LDAP.

| User | Role [S8] | Security context on the QM [S8] | Who uses it | Password |
|---|---|---|---|---|
| `mqadmin` | `MQWebAdmin` | the OS user that started mqweb (`mqm`) | human | `mqadmin` |
| `mqreadonly` | `MQWebAdminRO` | the OS user that started mqweb; display, inquire and browse only | human | `mqreadonly` |
| `mqoperator` | `MQWebUser` | **its own OS identity**; authorized by MQ OAM | human | `mqoperator` |
| `mqlabrest` | `MQWebAdmin` | the OS user that started mqweb | `mqlab` and `pymqrest` tooling | generated lab secret |

The rules behind the table:

- **[decision] Anonymous access is retired.** The `special-subject EVERYONE` binding and the
  `mqweb_console_anonymous` variable are deleted.
  - **[data]** IBM documents only `no_security.xml` as its "no security" route, and that route
    also disables HTTPS [S4].
  - **[judgment]** The lab's `EVERYONE` binding is undocumented Liberty configuration that gives
    `mqm`-level administration to anyone who can reach port 9443.
- **[decision] Lab credential convention.**
  - Accounts a human logs into use **password = username**.
  - The machine identity uses a real secret generated by `lab/scripts/lab-secret.sh` and stored
    in the build state directory. It is runtime-injected and never committed.
  - The lab is ephemeral and holds no data. The deviation is recorded as §8 D2.
- **[data]** IBM says to encode registry passwords with Liberty's `securityUtility` [S9].
  - **[decision]** The lab does this for every user, including the trivial ones. It costs nothing
    and keeps the configuration IBM-shaped.
- **[data]** `MQWebUser` users "can perform only the operations that the user ID is granted to
  perform on the queue manager". The registry user ID "must have an identical user ID on the
  system", in the same case [S10], and is at most 12 characters [S8]. All four names fit.
  - **[decision]** `mqoperator` is created as an OS account on **every node of every stack** by
    the existing `mq-authz-accounts` role, so it survives failover.
  - **[decision]** `mqoperator` gets **display-only** grants through MQSC `SET AUTHREC` (the lab
    convention): `+connect +inq +dsp` on the queue manager, `+dsp +inq +browse` on queues. The
    grants live next to each stack's MQSC.
  - **[judgment]** The two read-only tiers deliberately demonstrate defence in depth:
    - `mqreadonly` is IBM's standard role, and its writes are refused **by mqweb**;
    - `mqoperator` writes are refused **by the queue manager** (OAM), so a compromised or
      misconfigured mqweb still cannot write as that user.
- **[data]** The mqweb transport and session defaults are [S11, S12]:
  - `httpsPort=9443`, `httpPort=-1` (HTTP disabled);
  - `ltpaExpiration=120` minutes, `secureLtpa=true`;
  - `mqRestCsrfValidation=true`; CORS is off unless configured.
  - **[decision]** Keep IBM's defaults. HTTPS only, CSRF on, CORS off, LTPA at the defaults.
- **[data]** MQ 10.0.1 removes the `ltpaKeysPassword` property in favour of
  `ltpa_keys_password` [S1]. The lab does not set either today. If it sets one, it uses the
  10.0.1 name.
- **[data, lab]** TLS stays as it is: the org-CA `mqweb` entity certificate served over TLS 1.3
  (#250).
- **[decision]** `httpHost` stays `*`. The VIP floats, so a single-valued `httpHost` cannot bind
  only to the VIP on standby nodes (epic #39 §5). This departs from IBM's default of `localhost`
  [S11] and is recorded as §8 D1.

### 2.3 Endpoint discovery (how clients find the live mqweb)

- **[data]** On Native HA, "there must be a way to connect clients … to the active instance. You
  can use a comma-separated list of three possible addresses in order to locate the currently
  active instance" [S13].
- **[data, lab]** mqweb on a replica instance answers. REST reports the queue manager as
  `"state":"replica"` there, and as `"running"` on the active instance
  (`docs/reference/mqweb-endpoint-verification.md`).

**[decision] The discovery contract:**

| Stack | Clients address mqweb by |
|---|---|
| RDQM, Pacemaker | the per-site data-plane VIP, as in epic #39 |
| Native HA (no VIP, and none is added) | a per-site **candidate list** of instance endpoints (`mqlab rest render`) plus a **state probe** |
| svc-sim | its `net-ext` address, as in epic #39 |

The Native HA state probe works like this: query each candidate, use the one that reports
`running`, and search again on a connection failure or a `replica` answer. This is the HTTP
analogue of IBM's address-list model [S13].

- **[judgment]** A health-checked load balancer in front of the three instances is the usual
  production answer for HTTP clients that cannot run discovery, such as a browser. It is
  documented as the alternative and is **not built**. A VIP for Native HA is rejected because
  IBM does not need one [S13].
- **[decision]** For humans, `mqlab` resolves the endpoint:
  - `mqlab console open <stack>` resolves the live instance and points a port-forward relay at
    it, following the Grafana relay pattern in `mqlab.relay`;
  - it prints the URL and the sample logins.
- **[decision]** A reusable discovery helper in the MQ API client libraries is **out of scope**.
  It is the epic's follow-on brainstorm,
  [.github#344](https://github.com/logical-minds-foundry/.github/issues/344).

## 3. Open questions the spike must answer (gate)

The first implementation task is a **measurement and behaviour spike**. Its answers are recorded
in a findings document and amended into this spec before further tasks start. **Any surprise
pauses the epic for discussion [decision].**

1. **Per-node cold-start and memory.** With mqweb enabled on every node, measure time-to-ready
   and RSS per node on the current platform, on both macOS/arm64 and cloud x86. Does the
   one-pass cold rebuild hold?
   - **[data]** IBM publishes no mqweb memory or heap guidance (none was found; `setmqweb
     properties` has no heap property [S11]).
2. **Native HA replica behaviour.** What do the Console and REST show on a replica and on a
   CRR recovery-group instance? Do the Native HA and replication status views [S7] work on the
   active instance?
3. **REST version.** The 10.0.x examples use `/ibmmq/rest/v3/`, while the lab's runbook uses
   `/v2/`. Does `v2` still work on 10.0.1? Which version does `pymqrest` (`>=1.2,<2`) use?
4. **`MQWebUser` behaviour.** Confirm that `mqoperator` reads succeed and that its writes are
   refused by the queue manager with `MQRC 2035`, visible as an authority event in Loki.

## 4. Scope

**In scope:**

- every registered stack in `lab/topology.yaml`: `pcmk-ubuntu`, `rdqm-rhel`, `nativeha-ubuntu`,
  `nativeha-rhel-crr`, and svc-sim;
- both sites;
- macOS/arm64 and cloud x86.

**Out of scope:**

- **Client-mode mqweb and gateway queue managers** (§2.1).
- **Network and firewall enforcement** (deferred lab-wide, epic #39 §6).
- **A REST discovery helper in the client libraries** (follow-on #344).
- **Grafana's anonymous admin.** It conflicts with the new credential convention and gets its
  own ad-hoc task.
- **MFT roles.** The lab runs no MFT.

## 5. Task breakdown

Implementation tasks are filed after the plan, in the lab repo, linked under #341.

| # | Task | Kind | Blocked by |
|---|---|---|---|
| T1 | **Spike:** answer §3 with mqweb enabled on a cold rebuild; findings document PR; spec amended | impl (findings PR) | #342 |
| T2 | **Registry and roles:** rewrite `mqwebuser.xml.j2` with 4 users and 3 roles, `securityUtility`-encoded passwords, the `mqlabrest` lab secret; delete `EVERYONE` and `mqweb_console_anonymous` | impl | T1 |
| T3 | **`mqoperator`:** OS account on every node via `mq-authz-accounts`, plus display-only `SET AUTHREC` grants per stack | impl | T1 |
| T4 | **On by default:** mqweb on every QM node, keeping serial start and the readiness gate; delete the `mqweb_enabled` gate and `tests/test_mqweb_disabled.py`; tests assert the role is always present | impl | T2, T3 |
| T5 | **Human and tooling access:** resolve the live Native HA instance by state probe; `mqlab console open <stack>` relay plus login printout; tooling moves to `mqlabrest`; REST version per T1 | impl | T4 |
| T6 | **Cold-rebuild validation**, recording cold-start and memory figures | validation | T5 |
| T7 | **Live-lab acceptance:** rewrite `docs/reference/mqweb-endpoint-verification.md` (§6) and run it | validation | T6 |

**Bookends:**

- documentation, [.github#342](https://github.com/logical-minds-foundry/.github/issues/342);
- documentation review, [lab#1490](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/1490);
- follow-on brainstorm, [.github#344](https://github.com/logical-minds-foundry/.github/issues/344);
- retrospective, [.github#343](https://github.com/logical-minds-foundry/.github/issues/343).

## 6. Acceptance

The following run on a cold-rebuilt lab, for every stack and both sites:

1. **Running.** mqweb is active on every QM node; no gate exists.
2. **No anonymous access.** An unauthenticated Console or REST request is refused.
3. **Per-role allow/deny, through both the Console and REST:**
   - `mqadmin` can read and write;
   - `mqreadonly` can read, and its writes are refused by mqweb;
   - `mqoperator` can read, and its writes are refused by the queue manager (`MQRC 2035` and an
     authority event in Loki);
   - `mqlabrest` drives `mqlab rest` successfully.
4. **Discovery.**
   - VIP stacks follow the queue manager through HA failover and DR cutover.
   - Native HA discovery picks the `running` instance after an in-site failover and after a
     cross-site switchover.
   - `mqlab console open` lands on the live instance.
5. **TLS.** The served certificate is the org-CA `mqweb` certificate, using TLS 1.3.
6. **Cold rebuild.** The bootstrap is one-pass, and cold-start and memory figures are recorded.

## 7. Risks

- **The per-node Liberty cost has not truly gone away.**
  - Mitigation: T1 measures it first and gates the epic.
  - **[judgment]** If it fails, the existing serial-start design limits the damage, and the
    remaining options come back for discussion rather than being chosen silently.
- **The REST v2 → v3 move breaks the tooling.** Mitigation: T1 checks it, and T5 absorbs the
  change.
- **[data]** With RDQM, the Console shows nothing RDQM-specific, and Pacemaker-style external HA
  is not integrated with the Console [S14]. This is expected and documented, not a defect.

## 8. Deviations from IBM guidance

| # | IBM guidance | Lab choice | Why | Risk |
|---|---|---|---|---|
| D1 | `httpHost` defaults to `localhost` [S11] | `*` | The VIP floats, so mqweb must answer on the VIP whichever node holds it (epic #39 §5) | mqweb answers on every NIC, including management. Plane enforcement is deferred lab-wide |
| D2 | Basic-registry users with real passwords (implied by [S9, S10]) | Human logins use password = username | Ephemeral lab with no data; frictionless human use; all mechanisms stay wired (§2.2) | None beyond rebuilding the lab. **Never carry this to a real deployment** |
| D3 | mqweb normally runs where its queue managers are administered, with no stated sizing [S11] | Six or more Liberty JVMs on 2-vCPU nested-virtualization nodes | Lab scale | Cold-start contention; measured by T1 |

Anything T1 finds that departs from IBM guidance is added here.

## 9. Relationships

- **Supersedes** #266's open question; that issue is closed.
- **Unblocks** the follow-on brainstorm #344, a discovery helper for `pymqrest` and the other MQ
  API libraries.
- **Coordinates with** `#252` (name-based certificate and verified REST, under `#13`). Nothing
  here blocks it.

## 10. Sources

Cache paths are relative to `build/cache/refs/ibm-docs/` in the lab's main checkout.

- **S1** IBM MQ 10.0.1, deprecated/stabilized/removed features —
  <https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=1001-deprecated-stabilized-removed-features-in-mq>
  (`ibm-mq/10.0.x/1001-deprecated-stabilized-removed-features-in-mq/`)
- **S2** IBM announcement, IBM MQ 10.0.1 —
  <https://www.ibm.com/new/announcements/ibm-mq-10-0-1-advancing-enterprise-messaging-with-ai-mqtt-5-0-and-enhanced-security>
  (curl'd, not cached; **human-verify**)
- **S3** The MQ Console and REST API overview — <https://www.ibm.com/docs/SSYHRD_10.0.0/techoview/mqconsole-and-restapi.html>
  (`SSYHRD/10.0.0/techoview-mqconsole-and-restapi/`)
- **S4** Console and REST API security — <https://www.ibm.com/docs/SSYHRD_10.0.0/secure/q127970_.html>
  (`SSYHRD/10.0.0/secure-q127970_/`)
- **S5** `setmqweb remote` — <https://www.ibm.com/docs/SSYHRD_10.0.0/refadmin/setmqweb_remote.html>
  (`SSYHRD/10.0.0/refadmin-setmqweb_remote/`)
- **S6** Remote administration using the REST API — <https://www.ibm.com/docs/SSYHRD_10.0.0/administer/q131070_.html>
  (`SSYHRD/10.0.0/administer-q131070_/`)
- **S7** Viewing queue manager status in the Console — <https://www.ibm.com/docs/SSYHRD_10.0.0/administer/console_viewstatus.html>
  (`SSYHRD/10.0.0/administer-console_viewstatus/`)
- **S8** Roles on the Console and REST API — <https://www.ibm.com/docs/SSYHRD_10.0.0/secure/q127950_.html>
  (`SSYHRD/10.0.0/secure-q127950_/`)
- **S9** Console and REST API security configuration (password encoding with `securityUtility`) — <https://www.ibm.com/docs/SSYHRD_10.0.0/secure/q127930_.html>
  (`SSYHRD/10.0.0/secure-q127930_/`)
- **S10** Configuring a basic registry — <https://www.ibm.com/docs/SSYHRD_10.0.0/secure/q127972_.html>
  (`SSYHRD/10.0.0/secure-q127972_/`)
- **S11** `setmqweb properties` — <https://www.ibm.com/docs/SSYHRD_10.0.0/refadmin/setmqweb_properties.html>
  (`SSYHRD/10.0.0/refadmin-setmqweb_properties/`)
- **S12** CSRF protection for the REST API — <https://www.ibm.com/docs/SSYHRD_10.0.0/secure/q128710_.html>
  (`SSYHRD/10.0.0/secure-q128710_/`)
- **S13** Native HA overview, "Readiness – locating the current active instance" —
  <https://www.ibm.com/docs/en/SSYHRD_10.0.0/configure/Native_HA_overview.html>
  (`SSYHRD/10.0.0/configure-Native_HA_overview/`)
- **S14** HA and DR configuration — <https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=configuring-high-availability-disaster-recovery>
  (`ibm-mq/10.0.x/configuring-high-availability-disaster-recovery/`)
