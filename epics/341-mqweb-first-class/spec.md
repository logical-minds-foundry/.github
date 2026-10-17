# Spec: MQ Web (Console + REST) as a first-class, secured lab component

- **Epic:** [logical-minds-foundry/.github#341](https://github.com/logical-minds-foundry/.github/issues/341)
- **Documentation task:** [.github#342](https://github.com/logical-minds-foundry/.github/issues/342)
- **Absorbs:** [.github#266](https://github.com/logical-minds-foundry/.github/issues/266) (per-node vs client-mode mqweb; closed as superseded)
- **Builds on:** epic [#39](../39-mqweb-data-plane/spec.md) (mqweb is data-plane infrastructure; canonical endpoints)
- **Amends:** the lab design spec `docs/specs/2026-06-03-mq-cluster-lab-design.md` §8.3, whose "minimal auth registry … security is out of scope" becomes first-class security
- **Repo:** `logical-minds-foundry/mq-resiliency-lab-for-linux` (the lab)
- **Status:** design (brainstorm output, 2026-10-14). Amended 2026-10-16 with the T1 spike's
  cloud x86 findings ([.github#358](https://github.com/logical-minds-foundry/.github/issues/358)):
  §2.2 Console login, §2.3 discovery, §3 answers, §5 bug tasks, §6, §7 risks and §8 deviations.

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

**[decision] No machine identity.** No lab tooling authenticates to REST today, and future REST
administration tools will run as `mqadmin` or `mqreadonly`; their design belongs to the later
Python API work. The one programmatic consumer this epic adds, the Native HA state probe
(§2.3), authenticates as `mqreadonly`.

The rules behind the table:

- **[decision] Anonymous access is retired.** The `special-subject EVERYONE` binding and the
  `mqweb_console_anonymous` variable are deleted.
  - **[data]** IBM documents only `no_security.xml` as its "no security" route, and that route
    also disables HTTPS [S4].
  - **[judgment]** The lab's `EVERYONE` binding is undocumented Liberty configuration that gives
    `mqm`-level administration to anyone who can reach port 9443.
- **[decision] Lab credential convention.**
  - Accounts a human logs into use **password = username**.
  - Any credential that is not a human login uses a real secret generated by
    `lab/scripts/lab-secret.sh`, stored in the build state directory, runtime-injected and never
    committed. Examples are the AES passphrase, the keystore password, and the LTPA keys password
    if one is set.
  - The lab is ephemeral and holds no data. The deviation is recorded as §8 D2.
  - **[decision] The operator-facing edge stays trivially easy.** The Console login a lab user
    types is `mqadmin`/`mqadmin` for the admin, and the same password = username pattern holds
    for every sample user (`mqreadonly`/`mqreadonly`, `mqoperator`/`mqoperator`). Security inside
    the lab is modelled properly (IBM's roles, S16 encoding, the OAM-enforced `MQWebUser` tier);
    only the human login at the operator-facing edge is made convenient. `mqlab console open`
    prints these logins (§2.3).
- **[data]** IBM's procedure for every password in `mqwebuser.xml` has four rules [S9, S16]:
  - Encrypt it: "encrypt the password by using the securityUtility command … Plain text
    passwords added to the mqwebuser.xml file are not secure".
  - Use one key per server: "You must use the same encryption key to encrypt all passwords used
    by the same mqweb server".
  - Create the key with `securityUtility generateAESKey --createConfigFile=<file>
    --key=<strongPassphrase>`, protect the key file, and include it from `mqwebuser.xml` with
    `<include location="<keyFile>"/>`.
  - Encode each password with `securityUtility encode --encoding=aes --aesConfigFile=<keyFile>
    <password>`.
- **[decision] The lab follows S16 to the letter for every password in the file**: the three
  human users and the **keystore password**, which until now was inserted in plain text.
  - The passphrase is a real lab secret, `mqweb_aes_passphrase`, generated by `lab-secret.sh` and
    declared in each stack's `secrets:`. It replaces `mqweb_admin_password`.
  - The AES key file is generated **once**, on the first QM node, and stored in build state
    (0600). It is copied to **every** QM node (owner mqm, 0600), so an encoded string decodes on
    any node the QM can run on. `mqwebuser.xml` includes it.
  - **Encode once and reuse.** Each password is encoded a single time with that key, and the
    encoded string is stored in build state. The template only inserts stored strings.
    - **[judgment]** `securityUtility` output may differ between calls for the same input.
      Encoding at template time would change `mqwebuser.xml` on every run and trigger the role's
      restart-on-change.
    - A test asserts that a second render is byte-identical.
- **[data]** `MQWebUser` users "can perform only the operations that the user ID is granted to
  perform on the queue manager". The registry user ID "must have an identical user ID on the
  system", in the same case [S10], and is at most 12 characters [S8]. All three names fit.
  - **[decision]** `mqoperator` is created as an OS account on **every node of every stack**, so
    it survives failover and a cross-site switchover.
    - The account comes from the `mq-authz-accounts` role.
    - **[data, lab]** Today that role runs only in `site-nativeha-ubuntu.yml`, `site-pcmk.yml` and
      `site-rdqm.yml`. This epic **extends** it to `site-nativeha.yml` (the `nativeha-rhel-crr`
      arm: every instance, both sites, including the CRR recovery group) and to svc-sim.
    - A guard test asserts that every play hosting a queue manager applies the role, so a new
      stack cannot silently miss it.
  - **[data]** IBM's minimum authorities for `MQWebUser` users [S15] are:
    - queue manager: `+dsp +connect`;
    - `SYSTEM.ADMIN.TOPIC`: `+sub +dsp`;
    - `SYSTEM.REST.REPLY.QUEUE`: `+put +get +browse +inq`;
    - `SYSTEM.ADMIN.COMMAND.QUEUE`: `+put`;
    - plus authority on the objects the user works with.
  - **[decision]** `mqoperator` gets exactly that minimum, plus **display-only** object access
    (`+dsp +inq +browse` on queues; `+dsp` on the other object types it should see). The grants
    are made with MQSC `SET AUTHREC` (the lab convention) and live next to each stack's MQSC.
    - **[judgment]** `+put` on the command queue does not weaken the tier. The command server
      authorizes each command as the message's user, so an `mqoperator` write is refused by OAM.
    - **[data]** S15 is written for remote queue managers. IBM documents no separate minimum
      for local mode. This was searched in the 10.0.x and 9.4.x REST administration, messaging,
      roles, security and system-object topics during alignment on 2026-10-14.
    - **[judgment]** The mechanism is the same locally: PCF on `SYSTEM.ADMIN.COMMAND.QUEUE`,
      replies on `SYSTEM.REST.REPLY.QUEUE`. So S15's list is applied, and T7 proves it
      empirically.
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
- **[data, lab]** mqweb runs and answers on every Native HA instance, but which REST query is
  asked matters. The T1 spike measured this on MQ 10.0.1.0, identically on `v1`, `v2` and `v3`
  (lab `docs/reference/mqweb-spike-findings.md` §4.2,
  [lab#1499](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/1499)):
  - the **per-QM** query `GET /ibmmq/rest/v3/admin/qmgr/<QM>` answers 200 with
    `state: running` only on the active instance. On every replica, and on the CRR
    recovery-group leader, it answers **HTTP 500** `MQWB0004E` with reason 2543
    `MQRC_STANDBY_Q_MGR`;
  - the **list** query `GET /ibmmq/rest/v3/admin/qmgr?attributes=*` answers 200 on every
    instance. Each record carries `name` and `state` (`running`, `replica` or
    `recoveryGroupLeader`), and its `extended` block carries `role`, `groupRole` and
    `instanceName`, among others.
  - The older runbook note that REST reports `"state":"replica"` on a replica
    (`docs/reference/mqweb-endpoint-verification.md`) holds only for the list query.

**[decision] The discovery contract:**

| Stack | Clients address mqweb by |
|---|---|
| RDQM, Pacemaker | the per-site data-plane VIP, as in epic #39 |
| Native HA (no VIP, and none is added) | a per-site **candidate list** of instance endpoints plus a **state probe** |
| svc-sim | its `net-ext` address, as in epic #39 |

**[decision] The "follow the active" algorithm.** The Native HA state probe is the HTTP analogue
of IBM's address-list model [S13]. It is written as a reusable client algorithm: the lab
recommends it to REST API applications, and the lab's REST API wrapper implements it later
([.github#344](https://github.com/logical-minds-foundry/.github/issues/344)).

1. Take the candidate list (one endpoint per Native HA instance in the site) and ask each
   candidate in turn, authenticated, for the **list** query
   `GET /ibmmq/rest/<REST_API_VERSION>/admin/qmgr?attributes=*`. The per-QM query is never used
   for discovery, because a healthy replica answers it with HTTP 500 (see the data above).
2. In the answer, select the record whose `name` is the queue manager. Do not rely on its
   position in the list.
3. Classify the candidate by that record's `state`:

   | Answer | Meaning | Action |
   |---|---|---|
   | `running` | the live instance | **use it**, stop searching |
   | `replica` or `recoveryGroupLeader` | a valid member that is not live | skip |
   | any other `state`, no record for the QM, an unparseable body, any other HTTP status, or a transport or TLS failure | unreachable or unknown | skip |
   | HTTP 401 or 403 on the list query | the login is wrong: a misconfiguration, not a node problem | **fail loud**, stop |

4. If no candidate is live, fail loud, naming the queue manager and each candidate's answer. Never
   fall back to a replica or to the first candidate.
5. Re-run discovery whenever the chosen endpoint stops answering or stops reporting `running`
   (for example after a failover or a DR cutover). In this epic the only implementation,
   `mqlab console open`, re-runs discovery when the user re-runs it (the accepted quirk below);
   automatic re-resolution belongs to the client helper in #344.

- **[judgment]** On the CRR recovery site every instance reports `recoveryGroupLeader` or
  `replica`, so the algorithm correctly finds nothing live there until a cross-site switchover
  makes that site live.
- **[judgment] Why the algorithm has to exist.** A service-location mechanism is meant to turn a
  queue manager name into connection details. Native HA without a VIP leaves that job to the
  client, so every REST client must carry this logic. The lab specifies it once here, so that
  applications and the later wrapper share one definition.
- **[judgment]** The list query's `role`, `groupRole` and `instanceName` attributes can be shown
  to the user as diagnostics, but the classification uses `state` only.

- **[decision] Probe TLS validation.** The state probe verifies the mqweb certificate chain
  against the lab's app-org CA but does **not** check the hostname.
  - **[data, lab]** The `mqweb` certificate is `CN=mqweb` with no IP SAN (#250), and the probe
    connects by IP.
  - **[data]** Name-based certificates were the goal of `mq-resiliency-lab-for-linux#252`. That
    issue was closed as completed on 2026-07-07, but the certificate never shipped (checked
    2026-10-14). The work, including re-enabling the probe's hostname check, is tracked by the
    ad-hoc task `mq-resiliency-lab-for-linux#1498`.
  - This is not an IBM deviation: IBM prescribes nothing for a lab client's TLS checks.
- **[judgment]** A health-checked load balancer in front of the three instances is the usual
  production answer for HTTP clients that cannot run discovery, such as a browser. It is
  documented as the alternative and is **not built**. A VIP for Native HA is rejected because
  IBM does not need one [S13].
- **[decision] The endpoint derivation stays; the `mqlab rest` verb goes.**
  - The topology-derived endpoint logic from epic #39 (`src/mqlab/rest.py`) stays as an
    **internal library**, renamed (for example `mqweb_endpoints.py`). It provides the VIPs and
    candidate lists that `mqlab console open` uses.
  - The `mqlab rest render` verb and its `build/work/rest/endpoints.json` have no consumer, so
    they are removed.
  - The unused `pymqrest` dependency (`pyproject.toml`, `uv.lock`) and the shelved
    `CN=pymqrest` lab-PKI entity (`src/mqlab/cli.py`) are removed with them. REST client tooling
    is designed later, with the Python API work.
- **[decision] Human access is two hops.**
  - **[data, lab]** vergil-vm installs port-forward relays **statically**, from `vergil.toml`
    `port_forwards` records, when the dev VM is built (`src/mqlab/relay.py`). They cannot follow
    a queue manager that moves.
  - **[data, lab]** The dev VM has interfaces on both data planes (`virbr-data-a` and
    `virbr-data-b`), so it reaches every mqweb without using the management plane.
  - **Hop 1:** one static vergil forward per VM profile, for example `9443|127.0.0.1:19443`,
    takes the workstation into the dev VM. It is added to `vergil.toml`; the dev VM is rebuilt
    once.
  - **Hop 2:** `mqlab console open <stack>` resolves the live instance with the state probe and
    starts a local forwarder from the VM's `127.0.0.1:19443` to that instance's data-plane
    `:9443`. It prints the URL and the sample logins (`mqadmin`/`mqadmin`,
    `mqreadonly`/`mqreadonly`, `mqoperator`/`mqoperator`; §2.2). TLS passes straight through, so
    the browser sees the real `mqweb` certificate.
  - **[data]** The Console requires a login: an unauthenticated `GET /ibmmq/console/` answers 401
    and only `login.html` loads, on 10.0.1.0 (spike findings §4.4).
- **[decision] Accepted quirk, documented for users.** Getting from the real world into the
  ephemeral lab cannot be transparent:
  - on macOS, the workstation browses `https://localhost:<forwarded port>`;
  - on a cloud host, it browses the cloud VM's address and the same port;
  - after an HA failover or a DR cutover, the user re-runs `mqlab console open`.

  Grafana's static forward works only because obs never moves; a queue manager does.
- **[decision]** This epic specifies the "follow the active" algorithm and implements it only in
  the `mqlab` state probe. A reusable discovery helper in the MQ API client libraries is **out of
  scope**. It is the epic's follow-on brainstorm,
  [.github#344](https://github.com/logical-minds-foundry/.github/issues/344), which starts from
  the algorithm above.

## 3. Questions the spike answers (gate)

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
   `/v2/`. Does `v2` still work on 10.0.1? The state probe and the runbook use whichever version
   is current.

The `MQWebUser` behaviour is deliberately **not** a spike question. IBM documents it
[S8, S10, S15], and the account it needs is built by T3, which comes after the spike. It is
proven in T7's acceptance (§6 step 3).

### 3.1 The spike's answers (cloud x86)

T1 ([lab#1499](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/1499))
ran a cold `nativeha-rhel-crr` bootstrap on the cloud x86 host with mqweb on, and recorded the
results in the lab's `docs/reference/mqweb-spike-findings.md`. The nodes were 2 vCPU / 2 GiB, with
`mqweb.service` `TimeoutStartSec=1800`.

1. **Cost and one-pass rebuild.** [data, findings §3]
   - The bootstrap completed in **one pass**, in 29 min 22 s, with no re-run and no retries.
   - First start, measured by the role's readiness gate: about **21 s per node**. With the gate
     run serially, the six-node arm adds about 2 min.
   - Warm restart: median about **6.6 s**, at most 8.4 s.
   - Java RSS: about **245 MB at ready**, settling to about **170 MB** within 60 s. About 1 GB
     stays available on each 2 GiB node.
   - **[judgment]** Per-node mqweb is cheap on cloud x86, and D3 (§8) holds there.
2. **Native HA replica behaviour.** [data, findings §4]
   - mqweb runs, and the Console application is served, on every instance, including replicas
     and the recovery-group leader.
   - The per-QM REST query answers HTTP 500 (reason 2543 `MQRC_STANDBY_Q_MGR`) on every
     non-active instance. The list query reports `replica` or `recoveryGroupLeader` there. §2.3
     records the detail, and the discovery contract now uses the list query.
   - Without credentials the Console answers 401, even though the lab still renders the
     `EVERYONE` binding; only `login.html` loads. **[judgment]** This does not matter, because T2
     retires anonymous access.
   - Still open: whether the Native HA and replication status views [S7] render on the active
     instance, and what a replica shows after login. They need a browser and move to T7.
3. **REST version.** [data, findings §5] `v1`, `v2` and `v3` answer identically on 10.0.1.0;
   `v4` answers 404. **`REST_API_VERSION=v3`** [judgment: the version the 10.0.x documentation
   uses].

**The surprise and its resolution.** The spike's provisional verdict was PAUSE, because the plan's
probe used the per-QM query and would have treated a healthy replica's HTTP 500 as a login failure.

- **[decision]** The human accepted the recommended fix, now written into §2.3 as the "follow the
  active" algorithm.
- **[decision]** The macOS/arm64 half of the spike is **deferred** to end-of-epic validation and
  folded into T6 (cold rebuild, cost figures, the REST table re-run) and T7 (the Console views in
  a browser). The macOS lab is busy with another effort, and the human accepts the small risk that
  arm64 behaves differently from x86 (§7). On arm64 that half runs on **`nativeha-ubuntu`**, the
  native arm64 Native HA + CRR arm (QM `NHAUAPP`), not `nativeha-rhel-crr`: the RHEL stacks are
  x86-only (§4, `.github#360`).
- **[decision]** The cloud x86 results gate T2 and T3. With this amendment the gate is **PASS on
  cloud x86**.

## 4. Scope

**In scope:**

- every registered stack in `lab/topology.yaml`: `pcmk-ubuntu`, `rdqm-rhel`, `nativeha-ubuntu`,
  `nativeha-rhel-crr`, and svc-sim;
- both sites;
- macOS/arm64 and cloud x86, under the lab's platform policy:
  - **cloud x86:** every stack above, RHEL and Ubuntu;
  - **macOS/arm64:** the Ubuntu stacks (`pcmk-ubuntu`, `nativeha-ubuntu`) and svc-sim only.
  - **[data]** IBM ships MQ for RHEL on x86 only, so there are no RHEL-on-ARM MQ binaries,
    although RHEL itself runs on ARM. IBM does ship MQ for Ubuntu on ARM. `mqlab bootstrap`
    refuses a RHEL stack on an aarch64 host
    ([lab#847](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/847)).
  - **[note]** Earlier drafts sent the macOS/arm64 checks to the RHEL stacks. That was an
    oversight, which surfaced when
    [lab#1595](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/1595)
    ran. `.github#360` corrected it.

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
| T1 | **Spike:** answer §3 with mqweb enabled on a cold rebuild; findings document PR; spec amended (cloud x86 done, §3.1; the macOS/arm64 half moves to T6/T7) | impl (findings PR) | #342 |
| T2 | **Registry and roles:** rewrite `mqwebuser.xml.j2` with 3 users and 3 roles; follow S16 (one AES key generated once and distributed to every node; the 3 user passwords plus the keystore password encoded once and stored in build state), with a byte-identical re-render test; `mqweb_aes_passphrase` replaces `mqweb_admin_password` in the stacks' `secrets:`; delete `EVERYONE` and `mqweb_console_anonymous` | impl | T1 |
| T3 | **`mqoperator`:** extend `mq-authz-accounts` to `site-nativeha.yml` and svc-sim, with a guard test that every QM play applies it; IBM-minimum plus display-only `SET AUTHREC` grants per stack [S15] | impl | T1 |
| T4 | **On by default:** mqweb on every QM node, keeping serial start and the readiness gate; delete the `mqweb_enabled` gate and `tests/test_mqweb_disabled.py`; tests assert the role is always present | impl | T2, T3 |
| T5 | **Human access:** state probe (as `mqreadonly`) implementing the §2.3 "follow the active" algorithm over the list query, on top of the internal endpoint library; `mqlab console open <stack>` with the two-hop forward (`vergil.toml` record plus local forwarder) and the login printout; remove the `mqlab rest` verb, `endpoints.json`, the `pymqrest` dependency and the `CN=pymqrest` PKI entity; REST `v3` per T1 | impl | T4 |
| T6 | **Cold-rebuild validation**, recording cold-start and memory figures on both platforms; includes the spike's deferred macOS/arm64 half | validation | T5 |
| T7 | **Live-lab acceptance:** rewrite `docs/reference/mqweb-endpoint-verification.md` (§6) and run it; includes the deferred Console views (§3.1) | validation | T6 |
| B1 | **Bug [lab#1569](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/1569):** a bootstrap run from a git worktree forks the shared state bucket, because `lab-secret.sh` and the Ansible callers derive `build/state` from the script path | impl (bug) | none |
| B2 | **Bug [lab#1570](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/1570):** `lab-secret.sh` accepts option-like and unknown names and silently mints secrets for them | impl (bug) | none |

T1–T7 are filed as lab issues
[#1499](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/1499)–[#1505](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/1505),
in order. B1 and B2 were found by the spike. They are independent of T2–T5 and can run in
parallel with them.

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
     authority event in Loki). This holds on every stack, including `nativeha-rhel-crr` and
     svc-sim.
4. **Discovery.**
   - VIP stacks follow the queue manager through HA failover and DR cutover.
   - The state probe, authenticating as `mqreadonly` and following §2.3, picks the `running`
     Native HA instance after an in-site failover and after a cross-site switchover. It skips
     replicas and the recovery-group leader without error.
   - `mqlab console open` lands on the live instance from the workstation, and re-running it after
     a failover follows the move.
5. **No REST leftovers.** The `mqlab rest` verb, `endpoints.json`, the `pymqrest` dependency and
   the `CN=pymqrest` PKI entity are gone.
6. **TLS.** The served certificate is the org-CA `mqweb` certificate, using TLS 1.3.
7. **Idempotent.** Re-provisioning a live lab neither changes `mqwebuser.xml` nor restarts mqweb.
8. **Cold rebuild.** The bootstrap is one-pass, and cold-start and memory figures are recorded,
   on macOS/arm64 (Ubuntu stacks only, §4) as well as cloud x86 (the spike measured cloud x86
   only; §3.1).

## 7. Risks

- **The per-node Liberty cost has not truly gone away.**
  - Mitigation: T1 measures it first and gates the epic. On cloud x86 it measured cheap (§3.1).
  - **[judgment]** If it fails, the existing serial-start design limits the damage, and the
    remaining options come back for discussion rather than being chosen silently.
- **macOS/arm64 differs from cloud x86.** The arm64 half of the spike is deferred to T6/T7.
  - **[decision]** The human accepts this risk as small.
  - **[judgment]** The historic macOS failure (#1154) happened at cold start, so T6's cold
    rebuild on macOS is the check. If it fails, the epic pauses for discussion, as T1 would have.
  - On arm64 the check runs on the Ubuntu stacks, with `nativeha-ubuntu` standing in for
    `nativeha-rhel-crr` (§4).
- **The REST v2 → v3 move.** The only consumers this epic leaves are the state probe and the
  runbook. Settled by T1: `v3` (§3.1).
- **[data]** With RDQM, the Console shows nothing RDQM-specific, and Pacemaker-style external HA
  is not integrated with the Console [S14]. This is expected and documented, not a defect.

## 8. Deviations from IBM guidance

| # | IBM guidance | Lab choice | Why | Risk |
|---|---|---|---|---|
| D1 | `httpHost` defaults to `localhost` [S11] | `*` | The VIP floats, so mqweb must answer on the VIP whichever node holds it (epic #39 §5) | mqweb answers on every NIC, including management. Plane enforcement is deferred lab-wide |
| D2 | Basic-registry users with real passwords (implied by [S9, S10]) | Human logins use password = username | Ephemeral lab with no data; frictionless human use; all mechanisms stay wired (§2.2) | None beyond rebuilding the lab. **Never carry this to a real deployment** |
| D3 | mqweb normally runs where its queue managers are administered, with no stated sizing [S11] | Six or more Liberty JVMs on 2-vCPU nested-virtualization nodes | Lab scale | Cold-start contention. Measured by T1 on cloud x86: about 21 s first start and about 170 MB steady RSS per node (§3.1). macOS/arm64 is measured by T6, on the Ubuntu stacks (§4) |

**T1 added no deviation [data, findings §6].** The replica behaviour in §2.3 is not a deviation:
IBM prescribes no REST discovery method for Native HA, and the "follow the active" algorithm
applies IBM's address-list model [S13] [judgment]. Anything T6 or T7 finds on macOS/arm64 that
departs from IBM guidance is added here.

## 9. Relationships

- **Supersedes** #266's open question; that issue is closed.
- **Unblocks** the follow-on brainstorm #344, a discovery helper for `pymqrest` and the other MQ
  API libraries. §2.3's "follow the active" algorithm is its starting point.
- **Spike-found bugs** lab#1569 and lab#1570 (§5 B1, B2) are filed under this epic.
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
- **S15** Authorizing users to administer queue managers in the Console (minimum `MQWebUser` authorities) —
  <https://www.ibm.com/docs/SSYHRD_10.0.0/administer/authorizing_to_administer_remote_qmgrs_console.html>
  (`SSYHRD/10.0.0/administer-authorizing_to_administer_remote_qmgrs_console/`)
- **S16** Securing your mqweb server configuration passwords by using the securityUtility command —
  <https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=mcras-securing-your-mqweb-server-configuration-passwords-by-using-securityutility-command>
  (`ibm-mq/10.0.x/mcras-securing-your-mqweb-server-configuration-passwords-by-using-securityutility-command/`)
