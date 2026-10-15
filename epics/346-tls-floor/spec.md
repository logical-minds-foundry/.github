# Spec: Enterprise TLS floor — post-quantum key exchange and maximum TLS on every hop carrying MQ data

- **Epic:** [logical-minds-foundry/.github#346](https://github.com/logical-minds-foundry/.github/issues/346)
- **Origin:** the MQ 10.0.1 adoption review, [mq-resiliency-lab-for-linux#1410](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/1410) (`docs/reference/mq-10.0.1-adoption.md` §4 and §8), part of epic [#315](https://github.com/logical-minds-foundry/.github/issues/315).
- **Depends on:** [#341 — MQ Web (Console + REST) as a first-class, secured lab component](https://github.com/logical-minds-foundry/.github/issues/341).
- **Repo:** `logical-minds-foundry/mq-resiliency-lab-for-linux` (the lab).
- **Status:** brainstormed with the human on 2026-10-15. This epic runs late, as one of the last before public release, so its first task re-baselines this spec against the lab as it is then (§6.1).

## 1. Why

The lab exists to show IBM MQ at its leading edge. For security, that means answering one question an enterprise architect needs answered: **what is the highest TLS floor a heterogeneous MQ estate can hold?** The floor is the lowest level any hop carrying MQ data can drop to. The goal is to push it as high as current MQ and its surrounding stacks allow, to run the lab at it, and to document exactly why it is not higher.

**Threat model [judgment, grounded in public guidance].** "Harvest now, decrypt later": encrypted traffic captured today can be stored and decrypted once a cryptographically relevant quantum computer exists. Messaging infrastructure carries long-lived sensitive data, so post-quantum (PQ) key exchange is a baseline requirement, not a nice-to-have. IBM's own guidance frames PQ adoption as preparation that should start now ([quantum-safe hardening](https://www.ibm.com/docs/en/SSYHRD_10.0.0/secure/quantumsafe_hardening.html)).

**Why a floor, not a ceiling (the human's direction).** In an enterprise, the bar below which nothing can drop matters more than the ability to raise it selectively on niche systems. The lab therefore optimizes the **minimum** each domain guarantees, not the maximum a single component can reach.

## 2. Definitions

- **Hop:** one TLS-protected network path, identified by its endpoints and protocol (for example, app client → app QM over `APP.SVRCONN`).
- **Domain:** a group of hops that share one floor (§3).
- **Floor:** for a domain, the minimum protocol, cipher suite, key-exchange group and certificate strength that every hop in the domain negotiates, on both hosts.
- **Register:** the per-setting record (§4.2), the epic's core evidence.
- **Named exception:** a single component that cannot reach its domain's floor, recorded with evidence. It does not lower the floor (§4.3).
- **Finding:** any result above the floor that fails, or any gap between documented and actual behaviour. Findings become register rows, tasks, or upstream reports (§8).

## 3. Scope: domains and hops

Each domain has its own floor. "Current" is the state recorded by #1410 ([data, repo]) and must be re-checked at re-baseline (§6.1).

### A. MQ's own TLS (one single bar for the whole domain)

| Hop | Current [data, repo] |
|---|---|
| App client → app QM (`APP.SVRCONN`) | TLS 1.3 (`ANY_TLS13_OR_HIGHER`), mutual, classical key exchange |
| Exporter → app and svc QMs (`MON.SVRCONN`, JSON CCDT) | TLS 1.3, mutual, classical |
| Inter-QM channels, both sides, plus `SVC.SVRCONN` | TLS 1.3, mutual, classical |
| Native HA within-group replication | **TLS 1.2** (`ANY_TLS12`), classical |
| Native HA CRR (and IRR, where used) replication | **TLS 1.2**, classical |

**[data]** MQ supports PQ key exchange on channels and C clients from 10.0 (`SSLQS` / `SSLQSR`, MQCD `QuantumSafeAlgorithm` / `QuantumSafeRequired`, CCDT `quantumSafeAlgorithm` / `quantumSafeRequired`), and on Native HA / CRR / IRR replication from 10.0.1 (`NativeHALocalInstance` `QuantumSafeAlgorithm` / `QuantumSafeRequired`, TLS 1.3 required). Sources: [quantumsafe_tls](https://www.ibm.com/docs/en/SSYHRD_10.0.0/secure/quantumsafe_tls.html), [DEFINE CHANNEL](https://www.ibm.com/docs/en/SSYHRD_10.0.0/refadmin/q085520_.html), and #1410 §3.3.

### B. MQ data on other TLS stacks

| Hop | Current [data, repo] |
|---|---|
| RDQM HA and DR replication | Kernel TLS via `tlshd` (`crtmqm -re`); PQ capability: no data |
| MQ web server: Console and REST (Liberty) | To be defined by #341; TLS 1.3 today when enabled; PQ capability: no data |

### C. The telemetry ingest plane

Everything that carries MQ-derived metrics, logs or events from the MQ hosts into the observability tier:

| Hop | Current [data, repo] |
|---|---|
| Prometheus → MQ exporter HTTP endpoint | Plain HTTP |
| Prometheus → node_exporter (incl. the HA/DR collectors' textfile metrics) | Plain HTTP |
| Log and event collection on MQ hosts → Data Prepper (Alloy, OTLP/gRPC) | Plain text (`tls { insecure = true }`, `ssl: false`) |
| Data Prepper → OpenSearch | `http://localhost:9200` |

**Superseded decision.** The original lab design ([`docs/specs/2026-06-03-mq-cluster-lab-design.md`](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/blob/develop/docs/specs/2026-06-03-mq-cluster-lab-design.md) §10) declares a "mgmt-only telemetry plane, no TLS in v1". This epic supersedes it, and the superseding is recorded in that spec.

### Not in scope

- Human access to the observability UIs (Grafana, OpenSearch Dashboards). That is the lab's viewing layer, not the system being instrumented.
- MQTT, AMS and other advanced MQ features. The lab demonstrates core queue-manager HA/DR.
- ML-DSA (post-quantum) certificates. **[data]** MQ 10.0.1 documents none. Recorded as a gap register row, to revisit at a later MQ level.

## 4. Rules

### 4.1 The floor

1. **IBM's quantum-safe hardening guide is the floor** for domain A ([quantumsafe_hardening](https://www.ibm.com/docs/en/SSYHRD_10.0.0/secure/quantumsafe_hardening.html)). No hop in domain A runs below it. For domains B and C, the floor starts at the strongest documented setting of each stack.
2. **Above the floor, try the strictest value the stack accepts.** CNSA 2.0 supplies the candidate values, for example AES-256-only suites, P-384 ECDSA certificates and pure ML-KEM. CNSA 2.0 is a source of candidates, not the bar.

### 4.2 The register

Every security setting in scope gets a register row with these columns:

| Column | Content |
|---|---|
| Domain / hop(s) | where the setting applies |
| Setting | the knob (for example `SSLQSR`, `CipherSpec`, `-groups`) |
| IBM / stack recommendation | the documented recommended value, with a source URL |
| Strictest tried | the most secure value attempted |
| Result | works / fails / not offered, on each host, with evidence |
| Shipped | the value the lab runs |
| Why not higher | the reason, if shipped ≠ strictest tried |

### 4.3 Ship rule

- **Within a domain:** ship the strictest value that works on **both hosts** (macOS arm64 and cloud x86_64) for every hop in the domain. A value that works on one host only is a finding and does not ship.
- **Across domains:** domains are independent. A limitation in one domain never lowers another domain's floor.
- **Named exceptions:** if one component cannot reach its domain's floor, it gets a named exception (component, value reached, reason, evidence, upstream finding). The domain's floor stays where it is.
- **Decisions with the human:** whether a limitation becomes a named exception or a lower domain floor is decided with the human. The agent never lowers a floor on its own.

### 4.4 No legacy

TLS 1.2, classical-only key exchange, plain-text paths and compatibility fallbacks are **removed**, not left selectable. Users who need them can modify the lab.

## 5. Proving the floor

### 5.1 `mqlab tls audit <stack>`

A new read-only, glass-box verb. For every in-scope hop it reports the negotiated protocol, cipher suite and key-exchange group, and whether that meets the domain floor or a named exception.

- **MQ channels:** `DISPLAY CHSTATUS` `SSLCIPH` / `SSLQS`.
- **Native HA / CRR:** IBM's documented status fields (confirmed by the spike).
- **Telemetry endpoints:** a probe from the obs node using `openssl s_client` with explicit `-groups`.
- **RDQM:** whatever `tlshd` or kernel TLS exposes, or "not observable", stated honestly.

It exits non-zero if any hop is below its floor and not covered by a named exception. Optionally it emits a per-hop metric for Prometheus. Dashboard panels belong to the dashboard-review epic.

### 5.2 Downgrade tests

For each domain, a deliberately weak client must be **refused**: classical-only key exchange, TLS 1.2, and, where the register adopts it, a non-AES-256 suite. A downgrade that is accepted fails validation.

### 5.3 Cold-rebuild validation

There is one round per domain, each on both hosts, following the #1474 / #1475 pattern (per-host validation tasks with a full stack × OS matrix). `mqlab tls audit` and the downgrade tests are added to every row.

| Round | After | Hosts / rows |
|---|---|---|
| A | phases 1–3 | both hosts, full matrix |
| C | phase 4 | both hosts, full matrix |
| B | phase 5 | RDQM rows on cloud x86_64, plus the mqweb surface defined by #341 |

### 5.4 Cost

Bootstrap time and `qm e2e` latency are recorded per row against the #1474 / #1496 baselines, as register data. Cost informs; it does not decide (the human's direction: the floor is about security, not speed).

## 6. Shape of the work

### 6.1 Re-baseline (first task)

Before any implementation, bring this spec and the plan up to date with the lab as it is when the epic starts:

- the MQ level and its quantum-safe documentation;
- #341's final mqweb design;
- changes to stacks, components, PKI or observability;
- the current-state tables in §3.

Changes are folded into this spec and the plan by a docs PR, and the human approves them.

### 6.2 Phase 0: spike

The spike answers the unknowns cheaply, on one spike stack per host. Its code is throwaway unless it is promoted later. It produces the register's first draft, per-domain floor proposals for the human, upstream findings, and the concrete rollout task list.

- **A:**
  - Do the lab's OpenSSL-made PKCS#12 keystores support PQ, or do they need `runmqakm -pqc true` or a different format?
  - Which `SSLQS` values work (`HYBMLKEM`, `MLKEM`)?
  - How is the negotiated algorithm observed on each hop?
  - What are the exact JSON CCDT spellings?
  - Does the `ibmmq` client's `QuantumSafeAlgorithm` work?
  - Is the behaviour the same on arm64?
  - Above-the-floor candidates: pinning `TLS_AES_256_GCM_SHA384`, P-384 ECDSA certificates, and `AllowedQuantumSafeAlgorithms` restricting inbound key exchange.
- **B:**
  - Can RHEL 10's `tlshd` / GnuTLS negotiate a hybrid ML-KEM group?
  - What does RDQM's `-re` path expose?
  - What can Liberty on the bundled Java runtime (Semeru) do for the #341 mqweb surface?
- **C:** for the Go components (Prometheus, node_exporter, the exporter's HTTP server, Alloy) and the Java components (Data Prepper, OpenSearch) at the versions the lab runs:
  - What TLS versions and PQ groups are supported?
  - How is the negotiated group observed?
  - **[data, to verify]** Go negotiates `X25519MLKEM768` by default from Go 1.24 (<https://go.dev/doc/go1.24#crypto-tls>).

### 6.3 Phases

1. **PKI foundation** (only if the spike requires it): keystore format, PQ-capable keystores, and stronger certificates if the register adopts them. This is one change in the `lab-pki` role and its consumers.
2. **MQ channels and clients (A).** Two steps, in IBM's safe order:
   1. **Request:** `SSLQS` on every initiating end (SDR channels; clients through the MQCD; the exporter through the CCDT). Then prove every running channel shows a non-`NONE` `SSLQS`.
   2. **Require:** `SSLQSR(REQUIRED)` everywhere, plus any adopted above-the-floor settings.
3. **Native HA / CRR / IRR (A):** TLS 1.3 (`CipherSpec`, `GroupCipherSpec`) with `QuantumSafeAlgorithm` and `QuantumSafeRequired=Yes`. TLS 1.2 is removed.
4. **Telemetry plane (C):**
   - a telemetry PKI from `lab-pki`;
   - TLS with **mutual TLS** on every MQ-host scrape target;
   - TLS on Alloy → Data Prepper and on Data Prepper → OpenSearch (OpenSearch's security plugin for transport TLS only);
   - PQ wherever the stack supports it;
   - the original spec §10 marked superseded.
5. **RDQM and mqweb (B):** the strictest `tlshd` setting the spike found possible, or a named exception; and the mqweb surface from #341 at the domain floor.

Each phase updates its register rows and `tls-coverage.md`, and removes the legacy option it replaces.

## 7. Deliverables

1. **`docs/reference/tls-floor.md`:** the per-domain floors, the full register, named exceptions, the threat model and sources (data vs judgment). Written for an architect running a mixed MQ estate. `tls-coverage.md` links to it.
2. **The lab running at the floor by default** on both hosts, with legacy settings removed.
3. **`mqlab tls audit`** and the downgrade tests, built into validation from then on, so later changes cannot silently weaken the floor.
4. **Updated site docs** (getting-started, operate, reference), through the docs-review bookend.

## 8. Findings and upstream

- Every failure above the floor becomes a register row. Real defects become tasks under the epic.
- Upstream issues (IBM MQ, `ibmmq`, Go, Java, `tlshd` / GnuTLS, Liberty) get a **draft report for the human** to rewrite and file, following the #1483 pattern. The agent never posts upstream.

## 9. Acceptance

- `docs/reference/tls-floor.md` is merged, with a complete register for every in-scope hop.
- Every domain runs at its agreed floor on both hosts. Every gap is either a named exception, approved by the human, or a recorded finding.
- `mqlab tls audit` passes, and every downgrade test is refused, in all three cold-rebuild validation rounds (§5.3).
- No legacy TLS setting remains selectable in the lab.
- Upstream findings are drafted, and filed by the human or explicitly deferred.

## 10. Sequencing and dependencies

- **After #341** (mqweb). This epic is planned as one of the final epics before public release.
- **Order inside the epic:** re-baseline → spike → phase 1 → phase 2 (request, then require) → phase 3 → validation round A → phase 4 → validation round C → phase 5 → validation round B → docs review → retrospective. Phases 4 and 5 may overlap phases 2–3 where the spike shows they are independent.

## 11. Risks

- **Third-party PQ support lags.** The Go exporter, the Java components or `tlshd` may not offer PQ at the versions the lab runs. Mitigation: named exceptions with upstream findings; the domain floor holds.
- **Keystore format churn.** If PQ requires IBM-made keystores, the PKI role changes substantially. Mitigation: the spike answers this first, and phase 1 isolates it.
- **arm64 parity.** IBM's ARM64 builds may lag x86-64 for some feature. Mitigation: host parity is a ship rule; a gap becomes a finding, not a host-specific floor.
- **Drift before start.** This epic starts late. Mitigation: the re-baseline task (§6.1).
- **Validation cost.** Three cold-rebuild rounds on two hosts. Accepted: the floor claim is only as good as its evidence.
