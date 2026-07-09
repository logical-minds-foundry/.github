# Resilient MQ message gateway — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Prove, in the lab against a live 3+3, that an MQ-native archive (a two-put-under-syncpoint outbound copy + a mirrored inbound copy) gives a trading gateway honest, observable "replay the unconfirmed" — and that the team's async-log design silently loses exactly the messages it exists to protect.

**Architecture:** An **independent "parasite" repo** whose cloud sessions run inside the existing lab VM (proven cross-VM config). Its Ansible adds **only its own** round-trip queues to the running queue manager and counterparty; it ships its **own** small CLI and a responder/dummy-app, and reuses **none** of the lab's tooling (`mqlab`, the HA/DR `validate` framework stay the lab's). Work proceeds in tracks: **A Scaffolding** (design-independent, starts now) → **B Verify-before-build spikes** (gate demo correctness) → **C Demo build** (one counterparty first) → **D MQ-native research** → **E** optional Rung-0 silent-loss reproduction.

**Tech Stack:** IBM MQ 9.4 (MQI: `MQPUT`/`MQGET` under `MQPMO_SYNCPOINT`, `MQCMIT`, `MQMD.MsgId`/`CorrelId`); Python 3 + **pymqi** (native MQI bindings — gives real syncpoint, two-queue-in-one-UOW, and app-controlled `MsgId`, faithfully exercising the same MQI verbs the C++ product will; prototyping in Python rather than the C++ target is **deliberate** — the design lives in the MQI-interaction algorithm, not the language, per spec §2); Ansible (the repo's own site playbook adding its queues); Grafana (panels added to the existing messaging-flow board); Vergil-managed repo (`vergil.toml`, `vrg-*`, `vrg-validate`).

- **Design spec:** `epics/49-resilient-mq-gateway/spec.md` (this directory).
- **Epic:** `logical-minds-foundry/.github#49`. **Docs task:** `#50`.
- **Implementation repo:** a **new** repo in `logical-minds-foundry` — working name `mq-gateway-replay-lab` (confirm the name in Task A1). All Track-A/C/E paths below are relative to that new repo; Track-D writes into `.github`.

## Global Constraints

- **Anonymization (absolute).** No employer-identifiable content in any artifact — code, commits, diagrams, dummy data. Generic vocabulary only: "the application team", "the app-team lead", "the alternate log hosts", "Counterparty A/B/…", invented queue/host names. (spec § anonymization notice)
- **Parasite / boundary rule.** The repo configures **only what it owns** (its own queues, its own apps) and **declares-and-verifies** what it doesn't (the QM, the counterparty). Never mutate the lab's config; never restart the lab's QM. A failed precondition is a loud signal, not a silent fix. (spec §2)
- **MQ-native & minimalist.** No new vendor products, no bolt-ons, no "magic" exits unless proven. The two-put-under-syncpoint is the mechanism; streaming queues are deliberately not used. AMS is out. (spec §2, §6, §8, §10)
- **The one-UOW rule (correctness-critical).** Outbound: the archive PUT shares the transmit PUT's unit of work, with the **same MsgId** on both. **Never** two separate commits — a crash between them is I1 silent loss. (spec §6, invariant I1)
- **Verify before you build on it.** Resolve Task B1 (MsgId preservation) and B2 (CorrelId=MsgId) before Track C depends on them. (spec §13)
- **Vergil workflow.** `vrg-git`/`vrg-gh`; commit with `vrg-commit --type <t> --scope <s> --message <m>`. No direct commits to `develop`/`main`; feature branch per issue. Validation is one command: `vrg-container-run -- vrg-validate`.
- **Cold-rebuild acceptance gate.** Any change that provisions queues onto the live lab (Track A2, and C tasks that touch MQSC) is accepted only after it re-applies cleanly onto a freshly-(re)built lab — reproducible Ansible, no hand-edited VM state.
- **IBM Docs at build time.** Confirm MQI/MQSC flags and semantics against IBM Docs 9.4 via `python3 tools/ibm_doc_cache.py <url>` (browser-UA cache) — WebFetch 403s on `ibm.com/docs`.
- **One counterparty first.** Build and demonstrate a single counterparty end-to-end before generalizing; per-counterparty topology (spec §11 Q6b) is deferred.

---

## Track A — Scaffolding (design-independent; start now)

### Task A1: Create the gateway-experiment repo + wire the cross-VM dependency

Stand up the independent, Vergil-managed repo whose sessions run inside the existing lab VM.

**Files (new repo):**
- Create: `vergil.toml` (declare the `[vm]` dependency on the lab VM — the proven cross-VM config), `README.md`, `CLAUDE.md` (repo guidance + the anonymization + parasite rules), `.gitignore` (covers `build/`, `*.env`, `secrets/`).
- Create: `MEMORY.md` (policy header via the memory-init convention).

**Interfaces:**
- Produces: a working repo whose `vrg-container-run -- vrg-validate` passes empty, and whose cloud session opens **inside the lab VM** (shares the lab's running QM network).

- [ ] **Step 1: Confirm the repo name** with the human (working name `mq-gateway-replay-lab`); create the GitHub repo in `logical-minds-foundry` and the Vergil profile.
- [ ] **Step 2: Author `vergil.toml`** declaring the cross-VM dependency on the lab VM (the existing "someone else's VM" config — trivial, already used by other repos). Pin tool versions to match the lab where the demo needs them (MQ client, Python, pymqi).
- [ ] **Step 3: Add `CLAUDE.md`** stating the anonymization rule, the parasite/boundary rule, "reuses none of the lab's tooling", and the Vergil workflow.
- [ ] **Step 4: Verify** a cloud session opens inside the lab VM and can resolve the lab's QM host (declare-and-verify: `ping`/`dspmq` reachability check, read-only). Commit.

**Acceptance:** session runs in the lab VM; `vrg-validate` green; nothing in the lab was modified.

### Task A2: Site playbook — add the experiment's own round-trip queues to the running QM

Idempotent Ansible that adds **only the experiment's** queues to the live queue manager and counterparty, and declares-and-verifies everything it doesn't own.

**Files (new repo):**
- Create: `ansible/site.yml`, `ansible/roles/gw-queues/tasks/main.yml`, `ansible/roles/gw-queues/vars/main.yml` (queue names, all invented/generic), `ansible/inventory/lab.yml` (points at the lab's running QM host).

**Interfaces:**
- Consumes: the lab's running QM (by hostname/QM name, passed as vars — never hardcoded).
- Produces (MQSC on the live QM, all app-owned, generic names): `GW.CPB.SEND` (remote-queue def → counterparty), `GW.CPB.SEND.ARCHIVE` (outbound archive, `DEFPSIST(YES)`), `GW.CPB.REPLY` (local reply queue), `GW.CPB.REPLY.ARCHIVE` (inbound archive). Later tasks open these by name.

- [ ] **Step 1: Precondition check (declare-and-verify).** A task that asserts the QM is running and the counterparty channel exists; **fail loudly** if not (do not create anything the experiment doesn't own).
- [ ] **Step 2: Confirm queue attributes against IBM Docs 9.4** (`DEFINE QLOCAL`/`QREMOTE`, `DEFPSIST`, transmission-queue resolution) via `ibm_doc_cache.py`.
- [ ] **Step 3: Author `gw-queues` role** creating the four queues idempotently (re-runnable; `runmqsc` with `DEFINE ... REPLACE` guarded to the experiment's own object names only).
- [ ] **Step 4: Apply to the live lab**, then re-apply to prove idempotency (no changes on second run).
- [ ] **Step 5: Cold-rebuild check** — after a lab rebuild, `ansible/site.yml` re-establishes the queues one-pass. Commit.

**Acceptance:** the four generic queues exist on the live QM; re-run is a no-op; the lab's own objects are untouched; loud failure if a precondition is missing.

### Task A3: The experiment's own CLI + app skeletons

A small CLI (start-of-day / run / end-of-day) and the process skeletons — **not** `mqlab`, **not** the `validate` framework.

**Files (new repo):**
- Create: `src/gwlab/__init__.py`, `src/gwlab/cli.py` (subcommands `start-of-day`, `run`, `end-of-day`), `src/gwlab/mqi.py` (thin pymqi helpers: connect, put-syncpoint, get-syncpoint, browse, commit), `src/gwlab/config.py` (queue names from one source), `tests/test_cli.py`, `pyproject.toml`.

**Interfaces:**
- Produces: `gwlab start-of-day|run|end-of-day` entrypoints; `gwlab.mqi` helpers consumed by Tracks B/C. `mqi.put_syncpoint(qname, msgdata, msgid=None) -> bytes(msgid)`, `mqi.get_syncpoint(qname, wait_ms) -> Message`, `mqi.browse(qname) -> Iterator[Message]`, `mqi.commit()`, `mqi.backout()`.

- [ ] **Step 1: Write the failing test** for `cli` argument routing (`start-of-day`/`run`/`end-of-day` dispatch to named handlers) — `tests/test_cli.py`.
- [ ] **Step 2: Run it, confirm it fails** (`vrg-container-run -- vrg-validate` or `uv run pytest tests/test_cli.py -v`).
- [ ] **Step 3: Implement `cli.py`** dispatch + stub handlers; `config.py` reading queue names from one source (no hardcoding).
- [ ] **Step 4: Run tests green.** Commit.

**Acceptance:** CLI dispatches; queue names come from one source; `vrg-validate` green. (Handlers are filled by Track C.)

---

## Track B — Verify-before-build spikes (gate Track C correctness)

### Task B1: Spike — MsgId preservation across the outbound two-put in one UOW

Prove the load-bearing mechanism on the live QM before building the demo on it.

**Files (new repo):**
- Create: `spikes/msgid_uow/spike.py`, `spikes/msgid_uow/FINDINGS.md`.

**Interfaces:**
- Produces: a decided mechanism — **(a)** gateway-assigned `MsgId` (app sets `MQMD.MsgId`, no `MQPMO_NEW_MSG_ID`), or **(b)** QM-generated read back from `MQMD` after the first `MQPUT` (pre-commit) — recorded in `FINDINGS.md` and consumed by Task C1.

- [ ] **Step 1: Confirm against IBM Docs 9.4** (`ibm_doc_cache.py`) that `MQPUT` returns the resolved `MsgId` in `MQMD` **before** `MQCMIT`, and that an app may supply its own `MsgId` (interaction with `MQPMO_NEW_MSG_ID`).
- [ ] **Step 2: Spike (a) — gateway-assigned MsgId.** PUT to `GW.CPB.SEND` and `GW.CPB.SEND.ARCHIVE` in one UOW with an app-set `MsgId`; `MQCMIT`; browse both archives and assert **identical MsgId**.
- [ ] **Step 3: Spike (b) — read-back.** PUT to `GW.CPB.SEND` with `MQPMO_NEW_MSG_ID`, capture `MQMD.MsgId`, PUT the copy to the archive with that MsgId, `MQCMIT`; assert identical.
- [ ] **Step 4: Negative control (the hole).** Two *separate* commits with an induced crash between → show the archive missing the sent message (I1 silent loss) — this is the failure the demo will contrast against.
- [ ] **Step 5: Write `FINDINGS.md`** recommending (a) or (b) with evidence. Commit.

**Acceptance:** one UOW yields identical MsgId on both queues; the two-commit variant demonstrably loses; mechanism chosen for C1.

### Task B2: Spike — CorrelId echo on the reply path

Confirm the reconciliation key: that a reply carries `CorrelId` = the request's `MsgId`.

**Files (new repo):**
- Create: `spikes/correlid_echo/spike.py`, append to `spikes/.../FINDINGS.md`.

**Interfaces:**
- Produces: confirmation (or refutation) that `reply.CorrelId == request.MsgId` for this path — recorded for Task C3's reconciliation. If refuted, the fallback (parse the app body) is noted as a *cost*, not a blocker (spec §6/§11 Q7).

- [ ] **Step 1: Confirm the request-reply `CorrelId`/`MsgId` convention against IBM Docs 9.4** and against how the counterparty simulator replies.
- [ ] **Step 2: Round-trip a message** through `GW.CPB.SEND` → counterparty sim → `GW.CPB.REPLY`; assert `reply.CorrelId == request.MsgId`.
- [ ] **Step 3: Record the outcome** (and, if not echoed, specify the body field to reconcile on instead). Commit.

**Acceptance:** the reconciliation key is confirmed (header) or the body-parse fallback is specified.

---

## Track C — Demo build (one counterparty; depends on B1/B2)

### Task C1: Outbound path — atomic two-put producer with preserved MsgId

**Files (new repo):**
- Create: `src/gwlab/outbound.py`, `tests/test_outbound.py`. Modify: `src/gwlab/cli.py` (`run` wires the producer).

**Interfaces:**
- Consumes: `gwlab.mqi`, B1's chosen MsgId mechanism.
- Produces: `outbound.send(trade) -> msgid` — one UOW PUT to `GW.CPB.SEND` + `GW.CPB.SEND.ARCHIVE`, same MsgId, `MQCMIT`. `run` drives dummy trades in batches at a configurable rate.

- [ ] **Step 1: Write the failing test** — `send()` leaves one message on the archive and one transmitted, both with the same MsgId (browse archive; assert xmit consumed by the channel or present pre-channel), and a forced backout leaves **neither** (atomicity).
- [ ] **Step 2: Run it, confirm it fails.**
- [ ] **Step 3: Implement `outbound.send()`** using B1's mechanism; batch driver in `run`.
- [ ] **Step 4: Run tests green;** verify on the live QM. Commit.

**Acceptance:** every archived message has an identical-MsgId transmitted twin; backout is all-or-nothing.

### Task C2: Inbound path — responder + mirrored archive + drop injection

**Files (new repo):**
- Create: `src/gwlab/responder.py` (the dummy counterparty-facing app / "client"), `src/gwlab/inbound.py` (gateway get→archive), `tests/test_inbound.py`. Modify: `cli.py`.

**Interfaces:**
- Consumes: `gwlab.mqi`, C1's MsgId scheme, B2's echo finding.
- Produces: `inbound.drain()` — destructive GET on `GW.CPB.REPLY` under syncpoint + PUT copy to `GW.CPB.REPLY.ARCHIVE` in the **same UOW** + `MQCMIT`. `responder.run(drop_rate)` consumes transmitted messages and replies with `CorrelId=request.MsgId`, **dropping a configurable fraction** (the injected missing-confirmation).

- [ ] **Step 1: Write the failing test** — after `drain()`, the reply is in the inbound archive and gone from the reply queue, in one UOW (backout leaves it on the reply queue, nothing archived).
- [ ] **Step 2: Run it, confirm it fails.**
- [ ] **Step 3: Implement `inbound.drain()` and `responder.run(drop_rate)`** (drop injection deterministic by seed varied per run/index — no `Math.random`-style nondeterminism in tests).
- [ ] **Step 4: Run tests green;** verify round-trip on the live QM with `drop_rate=0` (all confirmed) and `drop_rate>0` (some unconfirmed). Commit.

**Acceptance:** inbound archive is the authoritative "processed" ledger; the responder can manufacture missing confirmations on demand.

### Task C3: Reconciliation-by-browse — the unconfirmed set

**Files (new repo):**
- Create: `src/gwlab/reconcile.py`, `tests/test_reconcile.py`.

**Interfaces:**
- Consumes: `gwlab.mqi.browse`, B2's key.
- Produces: `reconcile.unconfirmed() -> list[Unconfirmed]` — browse `GW.CPB.SEND.ARCHIVE` and `GW.CPB.REPLY.ARCHIVE`, match outbound `MsgId` against reply `CorrelId` (header path; body-parse fallback per B2), return the unmatched with age. Browses **archives only** — never the live queues. Also `reconcile.metrics()` for the exporter (counts, oldest-age).

- [ ] **Step 1: Write the failing test** — given seeded archives (N sent, M confirmed), `unconfirmed()` returns exactly the N−M unmatched, with ages; and it opens **no** handle on the live queues.
- [ ] **Step 2: Run it, confirm it fails.**
- [ ] **Step 3: Implement `reconcile.py`** (browse-first/next; match; age from `PutDate`/`PutTime`).
- [ ] **Step 4: Run tests green;** verify against a live drop-injected run. Commit.

**Acceptance:** deterministic unconfirmed set from archive browse; zero contact with live queues.

### Task C4: Observability — extend the messaging-flow board with the archive panels

**Files (new repo):**
- Create: `src/gwlab/exporter.py` (expose `reconcile.metrics()` as Prometheus text or push to the lab's metrics tier by declared contract), `dashboards/gw-counterparty.json` (Grafana panels), `dashboards/README.md`.

**Interfaces:**
- Consumes: `reconcile.metrics()`.
- Produces: a **panel set added to the existing messaging-flow board** (not a standalone board): unconfirmed-backlog-over-day (hero), send-vs-confirm rate, unconfirmed-set table — **per counterparty**, QM name/counterparty as variables from one source. The published mockup (`https://claude.ai/code/artifact/e2a3f94e-100a-48c1-ada0-6fd2620acf50`) is the design reference.

- [ ] **Step 1: Write the failing test** for the dashboard generator (panels reference the metric names the exporter emits; QM/counterparty are template variables, never hardcoded).
- [ ] **Step 2: Run it, confirm it fails.**
- [ ] **Step 3: Implement the exporter + dashboard JSON** matching the mockup's panels; declare-and-verify the metrics tier (don't reconfigure the lab's Prometheus — expose/register per its documented contract).
- [ ] **Step 4: Run tests green;** eyeball the panels on the live board during a drop-injected run. Commit.

**Acceptance:** the archive panels render live on the existing board, per counterparty, with the unconfirmed-set detail; live path unaffected.

### Task C5: Start-of-day / end-of-day clearing

**Files (new repo):**
- Modify: `src/gwlab/cli.py` (`start-of-day`, `end-of-day` handlers), `src/gwlab/dayboundary.py`; Create: `tests/test_dayboundary.py`.

**Interfaces:**
- Produces: `dayboundary.clear_archives()` — empties `*.ARCHIVE` queues (the intraday retention model, spec §R12); `start-of-day` asserts empty, `end-of-day` clears after a reconciliation check (refuse to clear if unconfirmed>0 unless `--force`, and say why).

- [ ] **Step 1: Write the failing test** — `end-of-day` refuses when the unconfirmed set is non-empty (loud message), clears when empty or `--force`.
- [ ] **Step 2: Run it, confirm it fails.**
- [ ] **Step 3: Implement `dayboundary.py` + handlers.**
- [ ] **Step 4: Run tests green;** demonstrate the clear live on the board. Commit.

**Acceptance:** SOD/EOD clearing works, is safe (won't silently drop an unreconciled day), and is visible on the board.

### Task C6: The demonstration — induce, observe, quantify

**Files (new repo):**
- Create: `demo/run_demo.md` (runbook), `demo/scenario.py` (orchestrates a scripted business day at accelerated time).

**Interfaces:**
- Consumes: the whole Track-C stack.
- Produces: a repeatable scripted run — SOD clear → batches flowing (`drop_rate=0`, backlog flat) → **inject the reply stall** mid-run → backlog climbs, the unconfirmed set populates, the board shows it live → reconcile lists exactly which messages → **replay (C7) resends them, the counterparty dedups, backlog drains to zero** → EOD clear.

- [ ] **Step 1: Write `scenario.py`** driving the accelerated day and the drop injection at a scripted point.
- [ ] **Step 2: Run end-to-end on the live 3+3;** capture the board (screenshot to `build/temp/`) at the backlog climb.
- [ ] **Step 3: Write `run_demo.md`** — exact steps to reproduce, and the talking points (what Rung 0 would have lost here vs. what this holds).
- [ ] **Step 4: Commit.**

**Acceptance:** a one-command scripted demo reproduces the reply-stall story on the live lab, with the board telling it.

### Task C7: Replay & idempotent resend (R4 resend half + invariant I2)

The other half of "replay the unconfirmed": resend them, and prove the resend is **delivered once**.

**Files (new repo):**
- Create: `src/gwlab/replay.py`, `tests/test_replay.py`. Modify: `src/gwlab/responder.py` (dedup on MsgId), `src/gwlab/cli.py` (`replay` subcommand).

**Interfaces:**
- Consumes: `reconcile.unconfirmed()` (C3), `outbound.send` (C1), `responder` (C2).
- Produces: `replay.resend(unconfirmed) -> count` — resends each unconfirmed message through the outbound two-put **reusing its original MsgId** (so the resend is identifiable and the counterparty can dedup). `responder` gains a **seen-MsgId set** so a replayed message is **delivered once** (I2), with the dedup horizon = intraday (matching R12).

- [ ] **Step 1: Write the failing test** — given a set of unconfirmed messages, `resend()` re-puts each with its **original MsgId**; and the responder, having already seen that MsgId, does **not** double-process it (assert delivered-once).
- [ ] **Step 2: Run it, confirm it fails.**
- [ ] **Step 3: Implement `replay.resend()`** (re-put by original MsgId via `outbound.send`) **and responder dedup** (seen-MsgId set, persisted so a responder restart doesn't reprocess; horizon = intraday).
- [ ] **Step 4: Run tests green;** on the live QM, run a drop-injected day → replay → confirm backlog drains to zero and **no message is processed twice**. Commit.

**Acceptance:** the unconfirmed set is resent by MsgId; the counterparty dedups (I2 demonstrated — delivered once); backlog drains to zero on the board. This is the claim the whole epic exists to prove.

---

## Track D — MQ-native research writeup (feeds the app-team doc)

### Task D1: MQ-native mechanisms report

**Files (`.github`):**
- Create: `epics/49-resilient-mq-gateway/research/mq-native-archive.md`.

**Interfaces:**
- Produces: a cited report (IBM Docs 9.4 via `ibm_doc_cache.py`) covering: the two-put mechanism; **streaming queues** (why deliberately not used; whether a copy can come off a transmission queue); **COA/COD** reports (MQ-level vs business confirmation); persistent messaging/logging; **native dedup feasibility without exits** (spec §8, §11 Q12); and a one-line "adjacent paths only if already operated" note (Kafka/FIX/Chronicle — no endorsement).

- [ ] **Step 1: Fetch and cache** the relevant IBM Docs 9.4 pages.
- [ ] **Step 2: Write the report**, each claim tied to `content.txt` + `source_url`; separate **data** (what the doc says) from **judgment**.
- [ ] **Step 3: Cross-link** from spec §8. Commit (in `.github`, under the docs task's follow-up or a linked task).

**Acceptance:** every mechanism claim is sourced; recommendations stay inside the MQ-native boundary.

---

## Track E — (stretch) Rung-0 silent-loss reproduction

### Task E1: Minimal async-log gateway to reproduce host-loss silent loss

Optional deeper arm (spec §12): stand up a minimal Rung-0-style gateway (local log + async copy to an "alternate log host", manual restart) and reproduce **host loss with an un-shipped log entry** → a message sent but unrecoverable — contrasted directly with the MQ-native option holding.

**Files (new repo):**
- Create: `rung0/async_log_gateway.py`, `rung0/DEMO.md`.

**Interfaces:**
- Produces: a scripted fault (kill the host between send and log-ship) that leaves a sent-but-unrecorded message; the reconciliation cannot see it — the concrete silent-loss the spec argues.

- [ ] **Step 1: Implement the minimal async-log gateway** (single writer, async copy, no quorum).
- [ ] **Step 2: Script the host-loss-before-ship fault;** show the lost message is invisible on recovery.
- [ ] **Step 3: Write `DEMO.md`** contrasting with Track C. Commit.

**Acceptance:** the silent-loss third state is reproduced on demand and visibly contrasted with the MQ-native option.

---

## Dependencies

- **A1 → A2 → A3** (repo, then queues, then CLI/app skeletons).
- **A2 → B1, B2** (spikes need the queues + counterparty).
- **B1 → C1**; **B2 → C3**; **A3 → C1..C7**; **C1, C2 → C3 → C4**; **C1, C2, C3 → C7**; **C1..C7 → C6**.
- **D1** independent (research), feeds the app-team doc. **E1** optional, after C6.
- **Gated on app-team answers (spec §11), not on this plan:** the inbound delivery-to-app handoff (Q1–5), ordering/active-active (Q6), counterparty topology (Q6b), retention/DR/encryption scope. The demo builds the *archive + reconciliation + observability* core, which does not depend on those answers.

## Self-Review

**Spec coverage:** §5 Rung-0 teardown → E1 (repro) + C6 talking points. §6 two-put/inbound/MsgId → B1, C1, C2. **R4 replay-the-unconfirmed → C3 (identify) + C7 (resend); I2 idempotent replay → C7 (counterparty dedup, delivered-once).** §7 Rung 2 → out of scope for the demo (design discussion only; noted). §8 research → D1. §9 observability extension → C4 (mockup is the reference). §10 data-at-rest → carried as an app-team question (Q11); no build. §11 questions → the deliverable payload (spec), not code; §11 Q7/Q12 verified in B2/D1. §12 proof plan → Track C + E1. §13 scaffolding + verify items → Track A + B. R12 intraday retention → C5.

**Placeholder scan:** no "TBD/handle appropriately"; where a task's internal mechanism depends on a spike (C1 on B1), the acceptance criteria are stated precisely and the dependency is explicit — a sequencing fact, not a placeholder. Repo name is an explicit A1 decision, not a placeholder.

**Type consistency:** `mqi.put_syncpoint/get_syncpoint/browse/commit/backout` (A3) are the names consumed in B/C; `outbound.send`, `inbound.drain`, `responder.run(drop_rate)`, `reconcile.unconfirmed/metrics`, `dayboundary.clear_archives` are used consistently where referenced. Queue names (`GW.CPB.SEND[.ARCHIVE]`, `GW.CPB.REPLY[.ARCHIVE]`) are defined in A2 and reused verbatim.
