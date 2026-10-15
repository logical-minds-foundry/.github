# Plan: Enterprise TLS floor — post-quantum key exchange and maximum TLS on every hop carrying MQ data

- **Epic:** [logical-minds-foundry/.github#346](https://github.com/logical-minds-foundry/.github/issues/346). **Spec:** [spec.md](spec.md).
- **Form:** compact by decision, following the #315 precedent. This epic starts late and its first task re-baselines the spec and this plan (spec §6.1), so step-level detail lives in each task's issue, written or refreshed at re-baseline. This plan fixes tasks, order, gates and done-ness.

Repo shorthand: `lab#N` = `logical-minds-foundry/mq-resiliency-lab-for-linux#N`; `.github#N` = `logical-minds-foundry/.github#N`.

**Goal:** find, apply and document the highest TLS floor per domain (A: MQ's own TLS; B: RDQM and mqweb; C: telemetry ingest), with the lab running at it on both hosts.

**Architecture:**

- A per-setting **register** is the evidence spine. Each phase raises one group of hops to its domain floor and updates the register.
- `mqlab tls audit`, plus downgrade and misconfigured-peer tests, prove the floor.
- One cold-rebuild validation round per domain closes each group.

**Tech stack:** IBM MQ (channels, MQCD/CCDT, Native HA `qm.ini`), the `lab-pki` role (OpenSSL / `runmqakm`), the `ibmmq` Python client, Go components (Prometheus, node_exporter, `mq_prometheus`, Alloy), Data Prepper, `tlshd` / kernel TLS (RDQM), Liberty (mqweb, per #341), Ansible, and the `mqlab` CLI.

## Global constraints (from the spec; every task inherits these)

- **Floor:** IBM's quantum-safe hardening guide for domain A. Strongest documented stack setting for B and C (spec §4.1).
- **Ship rule:** the strictest value that works on every host where the hop exists. Host parity applies to hops on both hosts. Single-host hops (RDQM, RHEL stacks) set their floor on the host where they exist (spec §4.3).
- **Domains are independent.** A component that falls short gets a **named exception** decided with the human. The agent never lowers a floor on its own.
- **No legacy:** TLS 1.2, classical-only key exchange, plain-text paths and fallbacks are removed, not left selectable (spec §4.4).
- **Scope:** only hops that cross a network. Loopback hops that don't touch MQ get an N/A register row (spec §3 C).
- **Every changed setting updates its register row** (spec §4.2) and `tls-coverage.md`.
- **Citations:** each claim is labelled data or judgment, with full URLs. IBM Docs are read via `tools/ibm_doc_cache.py`, pinned to the lab's MQ level.
- **Upstream findings** are drafted for the human. Agents never post upstream (spec §8).
- **Validation:** `vrg-container-run -- vrg-validate` is the only code gate. Cold-rebuild rounds are the lab gate.
- **Catalog lines stay single-line `{...}`.** Version tokens appear only in the catalog and resolved box names.

## Review focus (failure modes the spec implies; each is pinned in the owning task)

1. **Requiring PQ before every initiator requests it** breaks live channels with AMQ9681. Owner **P2b**: its first step asserts, from `mqlab tls audit`, that every running channel shows a non-`NONE` `SSLQS`, and refuses to flip `SSLQSR(REQUIRED)` otherwise.
2. **The audit reports a pass for a hop it couldn't observe.** Owner **AUD**: "not observable" is a distinct result. It counts as a failure unless the register records an exception or "negative test not possible" for that hop. There's a test for each.
3. **An existing lab keeps old keystores or certificates after a PKI change** (state in `build/state`). Owner **P1**: a migration path (regenerate or refuse loudly with the `mqlab` command to run), tested from a pre-change state.
4. **CRR failover or DR cutover after PQ is required** drops replication or falls back to classical. Owner **P3**: its validation steps include a CRR site switch with `tls audit` before and after.
5. **arm64/x86 divergence found only at validation time.** Owner **S**: every spike question that applies to both hosts is answered on both hosts, and a missing answer on one host is a recorded gap.

## Task table

| ID | Issue | Kind | Run via | Blocked by | Done when |
|---|---|---|---|---|---|
| DOC | .github#347 | docs | this PR | — | Spec and plan merged |
| R | .github#349 | docs | `issue-implement` (in `.github`) | #341 closed; human says the epic starts | Spec and plan re-baselined against the current lab (MQ level, #341's mqweb design, stack/PKI/obs changes, §3 current-state tables); human-approved docs PR merged; task issues below refreshed |
| S | lab#1522 | code (spike + doc) | `issue-implement` | R | Spike answers to spec §6.2 on **both** hosts. First draft of `docs/reference/tls-floor.md` (register) merged. Per-domain floor proposals **decided by the human** (recorded on the epic). Upstream drafts written. P1–P5 issue bodies refreshed from the findings |
| P1 | lab#1523 | code | `issue-implement` | S | PKI changes the register adopts (keystore format / PQ-capable keystores / certificate algorithm) merged with a migration path, **or** closed as not needed with the spike evidence |
| AUD | lab#1524 | code | `issue-implement` | S | `mqlab tls audit <stack>` merged (MQ channels and Native HA observers first; telemetry and RDQM observers added by P4a/P4b/P5a). Downgrade and misconfigured-peer test harness merged. "Not observable" ≠ pass. The audit is **wired into the end of `mqlab bootstrap`** (observe phase): every bring-up checks the floor and fails loud naming the offending hop (spec §7.3, "from then on") |
| P2a | lab#1525 | code | `issue-implement` | P1, AUD | Every channel initiator requests PQ (SDR channels, MQCD in `mqrc` clients and the svc responder, exporter CCDT). Audit shows non-`NONE` `SSLQS` on every running channel in a live check recorded on the issue |
| P2b | lab#1526 | code | `issue-implement` | P2a | `SSLQSR(REQUIRED)` on every channel, plus adopted above-floor settings (e.g. `AllowedQuantumSafeAlgorithms`, pinned suite), guarded by review focus #1 |
| P3 | lab#1527 | code | `issue-implement` | P1, AUD | Native HA / CRR / IRR on TLS 1.3 (`CipherSpec`, `GroupCipherSpec`) with `QuantumSafeAlgorithm` and `QuantumSafeRequired=Yes`; TLS 1.2 removed; misconfigured-peer test defined |
| VA-mac | lab#1528 | validation | `issue-validate` (macOS) | P2b, P3 | SUCCESS: Ubuntu matrix cold, audit clean, downgrade/peer tests refused, cost recorded |
| VA-cloud | lab#1529 | validation | `issue-validate` (cloud agent) | P2b, P3 | SUCCESS: RHEL matrix cold (incl. a CRR site switch), the same checks |
| P4a | lab#1530 | code | `issue-implement` | VA-mac, VA-cloud | Telemetry PKI. **Mutual TLS** on every MQ-host scrape target (exporter HTTP, node_exporter), PQ where the Go stack supports it. Audit observers and **downgrade tests** for those endpoints |
| P4b | lab#1531 | code | `issue-implement` | P4a | Alloy → Data Prepper OTLP/gRPC on TLS (mutual where supported), PQ where supported. Original spec §10 marked superseded. Data Prepper → OpenSearch recorded N/A (loopback) |
| VC-mac | lab#1532 | validation | `issue-validate` (macOS) | P4b | SUCCESS: full matrix cold with telemetry audit and downgrade tests |
| VC-cloud | lab#1533 | validation | `issue-validate` (cloud agent) | P4b | SUCCESS: the same on cloud |
| P5a | lab#1534 | code | `issue-implement` | VA-cloud | RDQM replication at the strictest `tlshd` setting the spike found, **or** a human-approved named exception with evidence; audit observer or "not observable" recorded |
| P5b | lab#1535 | code | `issue-implement` | S, #341 | mqweb (Console/REST) at domain B's floor per #341's design; audit observer; downgrade test |
| VB-mac | lab#1536 | validation | `issue-validate` (macOS) | P5b | SUCCESS: mqweb surface checks on the Ubuntu stacks |
| VB-cloud | lab#1537 | validation | `issue-validate` (cloud agent) | P5a, P5b | SUCCESS: RDQM rows cold plus mqweb surface |
| F | lab#1538 | code (doc) | `issue-implement` | VB-mac, VB-cloud, VC-mac, VC-cloud | `docs/reference/tls-floor.md` final: every in-scope hop has a complete register row, floors and named exceptions are stated, cost data is in, upstream drafts are linked; `tls-coverage.md` points to it. Includes the **ML-DSA certificate gap** register row (spec §3) and a **legacy sweep** proving no TLS 1.2 / classical-only / plain-text option remains selectable (spec §9). `tls audit` plus the downgrade tests are added to the **standard validation procedure** future validation tasks follow (spec §7.3) |
| DR | lab#1519 | docs review | `issue-implement` (sweep) | F | Site, reference and dev docs reflect the floor; sibling doc tasks spawned where needed |
| RETRO | .github#348 | retrospective | `epic-retrospective` | every other child closed | `retrospective.md` merged; the epic closes |

## Order of work

1. **DOC** now. Then the epic waits, as backlog, until #341 closes and the human starts it (planned as one of the last epics before release).
2. **R:** re-baseline. Nothing else starts until it merges.
3. **S:** the spike, on both hosts. The **floor decisions gate** sits here: the human decides each domain's floor and any named exceptions before P1–P5 proceed.
4. **P1** (if needed), with **AUD** in parallel.
5. **P2a → P2b**, with **P3** in parallel with P2a/P2b (different files: `mq-nativeha` vs channel MQSC/clients).
6. **VA-mac** and **VA-cloud** in parallel (macOS locally, cloud via its agent).
7. **P4a → P4b**, then **VC-mac** and **VC-cloud**.
8. **P5a** (after VA-cloud) and **P5b** (after S and #341), then **VB-mac** and **VB-cloud**. Phase 5 may overlap phase 4.
9. **F**, then **DR**, then **RETRO**.

## Gate policy

- **Start gate:** the epic doesn't start before #341 closes. R's merge is the start of real work.
- **Floor decisions:** at S, and whenever later evidence says a knob can't reach its domain floor, the human decides between a named exception and a lower domain floor. Record each decision on the epic and in the register.
- **Cold-rebuild gate:** validation tasks close only on SUCCESS evidence, never on a merge. Each validation task's procedure comment follows #1474/#1475/#1496/#1497 (preconditions, bake, per-row steps 1–8, results template), plus `tls audit`, the downgrade/peer tests, and cost capture against the #1474/#1496 baselines.
- **Host parity:** a value that works on one host only (for a hop on both hosts) is a finding. It does not ship and does not lower the floor without a human decision.

## Defect handling

- File each defect with `vrg-issue-create --epic logical-minds-foundry/.github#346`. Implement it in its own worktree and leave it at report-ready.
- Re-run only the rows a fix affects, cold, and record why the others didn't need a re-run (the #1284 precedent).
- Upstream defects: draft the report on a tracking task (the #1483 pattern). The human files it.

## Notes for implementers

- **Known current state** (re-check at R):
  - Native HA TLS lives in `ansible/roles/mq-nativeha/tasks/tls.yml` and `crr.yml`, with `ANY_TLS12` today.
  - Channel TLS vars are in `ansible/group_vars/all/tls.yml` (`tls_cipher: ANY_TLS13_OR_HIGHER`).
  - The exporter CCDT is `exporter-ccdt.json.j2`.
  - Alloy's insecure exporter is `ansible/roles/alloy/templates/config.alloy.j2` (`insecure = true`).
  - Data Prepper's OTLP source is `ansible/roles/data-prepper/templates/pipelines.yaml.j2` (`ssl: false`).
  - OpenSearch deliberately runs without the security plugin (`ansible/roles/opensearch/templates/opensearch.yml.j2`).
- **`ibmmq` client:** `QuantumSafeAlgorithm` needs `ibmmq` ≥ 2.1.0. Add new MQCD field names to the `ibmmq_refs` scanner (#1478) so its real-binding check covers them. Every client connect goes through `mqrc/mqconn.connect_client` (#1480).
- **Native HA keystores** need a `.sth` stash (`nativeha-crr-tls-guide.md` §5). P1 must keep that working.
- **Long lab jobs:** run detached with stdin from `/dev/null`, under a log-mtime stall watch. Capture **full** e2e output (the #1474 lesson).
- **Source material:** the 10.0.1 adoption review (`docs/reference/mq-10.0.1-adoption.md` §4) has the per-hop current/best table and the IBM source list.
