# macOS/arm64 startup reliability & bring-up hardening — implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: use `vergil:issue-implement` per
> GitHub task. Each numbered task below maps to a task within one issue under epic
> `logical-minds-foundry/.github#249` — issue **#1161** = Tasks 1–2 (A), **#1162** =
> Tasks 3–4 (B budgets), **#1163** = Task 5 (B resumability), **#1164** = Tasks 6–7
> (G). Steps use checkbox (`- [ ]`) syntax.

**Goal:** Make a cold `mqlab bootstrap nativeha-ubuntu --no-dr` **reliably
reproducible on macOS/arm64** (target 5/5) by cutting the concurrent JVM
cold-starts that compound the nested-virt tax, bounding every readiness budget
fail-loud, making each phase resume cleanly, and hardening the boot/DNS layer —
without regressing the x86 cloud.

**Architecture:** `mqlab bootstrap` drives four phases `net → vms → provision →
observe` (`src/mqlab/phases.py:646`). JVM-heavy services (Liberty/mqweb,
OpenSearch, Dashboards, Data Prepper) each pay a ~7× nested cold-start tax that is
CPU-active-but-slow (not I/O — epic #70 already removed the install I/O). The fix
is the **PR #1151 mqweb template** — non-blocking start + generous bounded
`TimeoutStartSec` + play-level `serial:` + bounded non-fatal `wait_for` readiness
gate — generalized to the remaining concurrent cold-starts, plus bounded fail-loud
budgets, clean phase re-entry, and boot/DNS ordering + bounded boot-retry.

**Tech Stack:** Ansible roles/plays (`serial:`, `wait_for`, `uri`, systemd unit
templates), `src/mqlab/phases.py` + `orchestrator.py` (Python phase/command
runner), `lab/topology.yaml` (`boot_batch`), Vagrant + libvirt/KVM (via Lima on
macOS), `pytest` (guardrail test), `vrg-container-run -- vrg-validate`.

**Spec:** `epics/249-startup-reliability/spec.md` (this directory). Read it
alongside this plan — §4 is the code grounding every task builds on.

## Global Constraints

- **Reliability over speed; adequate, not fast.** Favour a reproducible cold boot
  over a faster one (spec §2–§3).
- **Bounded and fail-loud.** No unbounded wait, no `TimeoutStartSec=infinity`, no
  swallowed failure. A service the message/data path depends on **fails loud** on
  a missed budget; one it does not may warn **non-fatally** — but the budget is
  always explicit and bounded (spec §3, repo "no silent failures").
- **Generalize the #1151 template; do not invent a new mechanism** (spec §4.2).
- **Cross-platform parity is hard.** Must benefit / never regress the x86 cloud;
  budgets stay generous enough the fast cloud never trips them. Proven by VAL-B.
- **`phases.py` stays pure** (emits commands, no side-effects; docstring
  `phases.py:7-15`) — boot-retry goes in the orchestrator/command runner, not the
  phase builders.
- **Cold rebuild is the acceptance gate; lint-green ≠ done** (repo cold-rebuild
  doctrine). The human runs the cold-rebuild validation tasks.
- **Validation is `vrg-container-run -- vrg-validate`** — the only validation
  command. It runs the pytest suite + 100 % branch coverage; no bare `pytest` /
  `uv run` outside it, and `uv run` never enters a runtime/provision path.
- **Commit with `vrg-commit --type <t> --scope <s> --message <m>`**; branch per
  issue off `develop` (`feature/<issue>-<slug>`); PRs into `develop`. Land each
  audit/diagnosis as a dated `docs/reports/` note.

## Dependency graph

```text
[#250 spec + plan lands]
   ├── #1161  A  : Task 1 (audit/instrument) → Task 2 (serialize)      ─┐
   ├── #1162  B  : Task 3 (budgets+fatality) → Task 4 (guardrail test) ─┤
   ├── #1163  B  : Task 5 (clean phase resumability)                   ─┼─ VAL-A #1154 arm64 5/5
   └── #1164  G  : Task 6 (diagnose) → Task 7 (fix boot/DNS)           ─┘   VAL-B #1155 x86 parity
                                                                             (both blocked-by #1161..#1164)
                                                        └─ Docs review #1153 ─ Retrospective #251
                                                           Follow-on brainstorm #252 (recorded in retro §5)
```

The four impl issues are mutually independent (parallel frontier). **Coordination
note:** Tasks 5 and 6–7 both edit `src/mqlab/phases.py`; the second to land
rebases onto the first (routine). A's Task-1 instrumented run is the shared
timing evidence B (§budgets) and G (§diagnosis) reference.

---

### Task 1 (issue #1161, A): Audit + instrument the concurrent JVM cold-starts

Establish empirically *where* ≥2 JVM-heavy services cold-start concurrently — per
node and cross-node within a phase — before changing anything. Diagnosis-first
(spec §3).

**Files:**

- Create: `docs/reports/2026-09-20-jvm-coldstart-concurrency-audit.md`
- Read: `src/mqlab/phases.py` (`_vms_build_steps:308`, `_provision_build_steps:444`,
  `_observe_build_steps:526`, logsearch prepend `cli.py:2115`), `lab/topology.yaml`
  (node vCPU/mem; `logsearch` 2 vCPU / 6 GiB `:172`), the mqweb plays
  (`ansible/_nativeha-*.yml`, `_pcmk-*.yml`, `_rdqm-*.yml`), and the JVM roles
  (`opensearch`, `opensearch-dashboards`, `data-prepper`, `mqweb`).

**Interfaces:**

- Consumes: a full cold `mqlab bootstrap nativeha-ubuntu --no-dr` on the arm64
  host (the human runs the lab — see repo doctrine; the agent prepares the
  instrumentation and interprets the transcript/journals).
- Produces: the audit note enumerating each concurrent cold-start point + its
  serialization prescription — the input to Task 2.

- [ ] **Step 1: Enumerate the concurrent cold-start points from the code.** At
  minimum: (a) `logsearch` cold-starts **OpenSearch + Dashboards + Data Prepper**
  on one 2-vCPU node (spec §4.3); (b) the **seven mqweb plays without `serial:`**
  (`_nativeha-cluster-ha.yml:28`, `_nativeha-dr-replication.yml:34`,
  `_nativeha-ubuntu-dr-replication.yml:33`, `_pcmk-cluster-ha.yml:167`,
  `_pcmk-dr-replication.yml:45`, `_rdqm-cluster-ha.yml:36`,
  `_rdqm-dr-replication.yml:40`) — only `_nativeha-ubuntu-cluster-ha.yml:33` has it;
  (c) any cross-node JVM overlap within the observe phase (logsearch trio + obs
  Grafana) or provision phase (mqweb across QM nodes).
- [ ] **Step 2: Instrument one cold bootstrap and capture the evidence** — per-
  service cold-start start/end (systemd `journalctl -u <svc>`
  / `systemd-analyze`), which services were mid-cold-start simultaneously, and
  whether any hit a timeout warning. Record CPU-active-but-slow vs any I/O, to
  confirm the tax is the cause here too (spec §1). Actively monitor the run — a
  hang and a slow op look identical; treat a stall as a failure to investigate.
- [ ] **Step 3: Write the audit note** — the enumerated points, the measured
  concurrency, and the per-point serialization prescription (which `serial:` value,
  which dependency order for the logsearch trio: OpenSearch first, then
  Dashboards + Data Prepper). Data vs judgment separated (repo doctrine).

---

### Task 2 (issue #1161, A): Serialize the remaining cold-starts with the #1151 template

Apply the proven pattern (spec §4.2) to each point Task 1 found.

**Files:**

- Modify: the seven mqweb plays lacking `serial:` (Task 1 Step 1b) — add the
  play-level `serial:` used by `_nativeha-ubuntu-cluster-ha.yml:33`.
- Modify: the observe/logsearch bring-up so the logsearch JVM trio cold-start in
  dependency order rather than concurrently — the ordering lever is in
  `src/mqlab/phases.py` (`_observe_build_steps` / logsearch prepend) and/or the
  logsearch playbook/role sequencing (`ansible/roles/{opensearch,
  opensearch-dashboards,data-prepper}`). Keep the non-blocking-start + bounded
  readiness-gate shape from `mqweb/tasks/main.yml:103`.
- Reference (template, do not modify): `ansible/_nativeha-ubuntu-cluster-ha.yml:33`,
  `ansible/roles/mqweb/tasks/main.yml:69,103-119`.

**Interfaces:**

- Consumes: Task 1's prescription.
- Produces: serialized cold-starts consumed (proven) by VAL-A.

- [ ] **Step 1: Extend `serial:` to the mqweb plays.** For each of the seven plays,
  add the play-level `serial:` (default `1`; a higher `serial: N` is acceptable
  where Task 1 shows a node can take two — sized to evidence, not dogma, spec §8).
- [ ] **Step 2: Serialize the logsearch JVM trio.** Ensure OpenSearch reaches its
  readiness gate before Dashboards + Data Prepper cold-start (they depend on
  OpenSearch anyway), so the 2-vCPU node never pays three JVM taxes at once.
  Preserve each service's bounded readiness gate; do not remove fail-loud where the
  data path depends (that is Task 3's call, not this one's).
- [ ] **Step 3: Validate.**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS — the plays/templates still lint and render
(`tests/test_templates_render.py`), no coverage regression.

- [ ] **Step 4: Commit.**

```bash
vrg-commit --type fix --scope bootstrap \
  --message "serialize remaining concurrent JVM cold-starts via the #1151 pattern (#1161)"
```

> Cold-boot proof (serialized cold-starts, no timeout warnings) is VAL-A; a green
> `vrg-validate` is necessary, not sufficient (spec §7).

---

### Task 3 (issue #1162, B): Inventory + right-size the readiness budgets & fatality

Make every JVM-service budget explicit, bounded, sized to the nested reality, and
fatal iff the data/message path depends on it (spec §3, §4.3).

**Files:**

- Create: `docs/reports/2026-09-20-startup-budget-inventory.md`
- Modify (as the inventory dictates): the systemd unit templates / inline units and
  their paired Ansible waits — `ansible/roles/mqweb/templates/mqweb.service.j2:13`
  + `mqweb/tasks/main.yml:103`; `opensearch/tasks/install.yml:120` (unit) +
  `opensearch/tasks/configure.yml:38`; `opensearch-dashboards/tasks/install.yml:90`
  + `configure.yml:51`; `data-prepper/tasks/install.yml:98` + `configure.yml:28`;
  and the peripheral units currently defaulting to 90 s where that is too tight for
  a JVM-adjacent service.

**Interfaces:**

- Consumes: Task 1's measured cold-start times (right-size to reality, not guess).
- Produces: the reconciled budgets the guardrail test (Task 4) pins.

- [ ] **Step 1: Build the full inventory** — every service, its systemd
  `TimeoutStartSec` (or the 90 s default where unset), its Ansible readiness wait
  (timeout / `retries×delay`), and its current fatality. Seed from spec §4.3.
- [ ] **Step 2: Right-size each budget to the nested reality.** Generous-but-
  bounded (headroom over the measured ~7× cold-start, no `infinity`). Reconcile the
  OpenSearch unit-vs-wait inconsistency (unit 180 s vs 900 s gate) so the effective
  budget is coherent. Ensure no heavy service silently rides the 90 s default.
- [ ] **Step 3: Set fatality by data-path dependence.** Fail-loud where the
  message/data path depends on the service; non-fatal-with-warn (the mqweb
  `failed_when:false` + `debug` shape) where it does not. Document the call per
  service in the note.
- [ ] **Step 4: Write the inventory note** (data vs judgment separated).
- [ ] **Step 5: Validate.** `vrg-container-run -- vrg-validate` → PASS.
- [ ] **Step 6: Commit.**

```bash
vrg-commit --type fix --scope bootstrap \
  --message "right-size startup readiness budgets to the nested reality; fatality by data-path dependence (#1162)"
```

---

### Task 4 (issue #1162, B): Startup-budget guardrail test

Make the budget doctrine executable so a later edit can't silently drift a budget
to `infinity` or the 90 s default. Mirror `tests/test_logsearch_budgets.py`
(spec §4.5).

**Files:**

- Create: `tests/test_startup_budgets.py`
- Reference: `tests/test_logsearch_budgets.py` (`_load_tasks`, `_unit_content`,
  `BUDGET_SECONDS`), `tests/test_opensearch_render.py` (readiness-budget assert).

**Interfaces:**

- Consumes: the reconciled budgets from Task 3.
- Produces: `test_startup_budgets` — the guardrail every future budget edit relies
  on staying green.

- [ ] **Step 1: Write the test.** For each JVM-heavy service (and the mqweb/OpenSearch
  unit values not yet guarded): parse the unit template / inline unit, assert
  `TimeoutStartSec` is present and within the reconciled bound (a `MIN <= v <= MAX`
  range, not just `!= 180`), and assert the paired Ansible readiness wait's
  timeout / `retries×delay` matches Task 3. Cover the fatality expectation where it
  is statically checkable (e.g. `failed_when: false` present/absent per service).
- [ ] **Step 2: Confirm RED then GREEN.** If Task 3 already landed the budgets on
  the same branch, the test is GREEN on write; if authored first (true TDD), it is
  RED against the pre-Task-3 values, then GREEN after. Either way finish GREEN with
  100 % branch coverage (all helper branches exercised — repo coverage gate).

Run: `vrg-container-run -- vrg-validate`
Expected: PASS — `test_startup_budgets` green, 100 % branch coverage holds.

- [ ] **Step 3: Commit.**

```bash
vrg-commit --type test --scope bootstrap \
  --message "add startup-budget guardrail test pinning bounded TimeoutStartSec + readiness waits (#1162)"
```

---

### Task 5 (issue #1163, B): Clean phase resumability (`--from <phase>`)

`--from`/`--only` already exist (`cli.py:2153`); the gap is **non-idempotent
intra-phase steps** behind coarse per-phase `satisfied()` probes (spec §4.1). Make
each phase re-enter cleanly after a mid-phase failure.

**Files:**

- Modify (per finding): `src/mqlab/phases.py` (the phase step builders and/or
  `satisfied` probes where a partial-completion state is mis-read) and/or the roles
  a phase invokes where a step is not re-runnable.
- Reference: `net-up.sh` idempotency precedent (`phases.py:196-204`, #974);
  `site-dns.yml` cold-boot guards.

**Interfaces:**

- Consumes: the four-phase model; the stateless probe-resume (`first_unsatisfied`,
  `phases.py:659`).
- Produces: clean `--from observe` / `--from provision` re-entry (proven by an
  induced-stop test, VAL-A exercises it end-to-end).

- [ ] **Step 1: Induce a mid-observe stop and resume.** Interrupt the observe
  phase partway (e.g. after logsearch is up but before Grafana health), then
  `mqlab bootstrap nativeha-ubuntu --no-dr --from observe`; confirm it completes
  cleanly with no half-state and no duplicate/conflicting work.
- [ ] **Step 2: Induce a mid-provision stop and resume.** Interrupt after
  NIC/DNS but before QM-up, then `--from provision`; confirm clean completion.
- [ ] **Step 3: Fix each non-idempotent step surfaced.** Prefer making the step
  idempotent (or the probe correctly partial-aware) over widening a probe to skip
  work — the induced stop must genuinely complete cleanly (spec §8 risk). Keep
  `phases.py` pure.
- [ ] **Step 4: Validate.** `vrg-container-run -- vrg-validate` → PASS (add/adjust
  a phase-runner unit test if a `phases.py` behaviour changed; hold 100 % branch
  coverage).
- [ ] **Step 5: Commit.**

```bash
vrg-commit --type fix --scope bootstrap \
  --message "make provision/observe phases re-enter cleanly on --from after a mid-phase failure (#1163)"
```

---

### Task 6 (issue #1164, G): Diagnose the IP-lease + DNS boot flakiness

Diagnosis-first before the boot-layer fix (spec §3, §4.4).

**Files:**

- Create: `docs/reports/2026-09-20-boot-layer-flakiness-diagnosis.md`
- Read: `lab/Vagrantfile:38,62-76` (boot_timeout, NIC/DHCP), `lab/topology.yaml:135`
  (`boot_batch`) + `:503` (commons order) + `:178-189` (infra nodes),
  `src/mqlab/phases.py:139-157,227-261,295-339` (all_vms order, batching,
  `vagrant up`), `src/mqlab/orchestrator.py:2,67` (fail-loud, no retry),
  `src/mqlab/platforms.py:28,106` (TCG boot timeout),
  `ansible/roles/host-resolver/tasks/main.yml`, `ansible/site-dns.yml:12-53`.

**Interfaces:**

- Consumes: cold-boot observation of the vms phase on arm64.
- Produces: the diagnosis note prescribing the Task-7 fix.

- [ ] **Step 1: Reproduce & characterise the IP-lease timeout.** Confirm the
  `Fog::Errors::TimeoutError` is the **default management-network DHCP lease** wait
  during `vagrant up` (lab NICs are DHCP-disabled, `Vagrantfile:66`), and how
  `boot_batch: 4` + TCG-slow boots widen the window. Capture how often and in which
  batch it fires.
- [ ] **Step 2: Reproduce & characterise the DNS `no route to host`.** Confirm the
  `infra-client` DNS node boots at the **tail** of the order (commons
  `[obs_box,probe,svc,app,infra]`, `topology.yaml:503`) after its dependents, and
  whether the failure is (a) a dependent resolving before infra's BIND is serving,
  or (b) a still-settling multi-NIC infra-client unreachable, or both.
- [ ] **Step 3: Write the diagnosis note** with the prescribed fix set (which of:
  `boot_batch` sizing, bounded boot-retry, infra-first ordering) and the rationale
  (data vs judgment).

---

### Task 7 (issue #1164, G): Fix the boot/DNS layer per the diagnosis

Apply the Task-6 prescription. Any subset of the three levers, driven by evidence.

**Files:**

- Modify (per diagnosis): `lab/topology.yaml` (`boot_batch`, and/or the
  boot/commons ordering so the infra/DNS node boots **before** its dependents);
  `src/mqlab/orchestrator.py` and/or the command runner (a **bounded** retry/backoff
  around the `vagrant up` step — kept out of the pure `phases.py`); and/or
  `src/mqlab/phases.py` `all_vms`/batching if ordering is expressed there.
- Reference: existing provision-phase retries (`site-dns.yml:46-48`,
  `mq-client/tasks/main.yml:70-72`) as the bounded-retry precedent.

**Interfaces:**

- Consumes: Task 6's diagnosis.
- Produces: a boot phase that reaches end-of-vms without IP-lease/DNS failures
  (proven by VAL-A's repeated cold boots).

- [ ] **Step 1: Order the infra/DNS node ahead of its dependents** (if Step 2 of
  Task 6 confirmed the race) — boot `infra` early rather than appending it to the
  tail, so every guest's sole resolver is up first.
- [ ] **Step 2: Add a bounded boot-retry** around the `vagrant up` step (if the
  lease timeout is transient) — retry with backoff up to a bounded cap, then
  fail-loud (never an unbounded loop; consistent with the orchestrator contract,
  spec §8). Do not put retry logic in the pure `phases.py`.
- [ ] **Step 3: Re-size `boot_batch`** only if the evidence shows a smaller batch
  removes the lease pressure without an unacceptable wall-clock cost (parity: must
  not regress the cloud, where the larger batch is fine).
- [ ] **Step 4: Validate.** `vrg-container-run -- vrg-validate` → PASS (unit-test
  the retry/ordering logic; hold 100 % branch coverage).
- [ ] **Step 5: Commit.**

```bash
vrg-commit --type fix --scope bootstrap \
  --message "harden VM boot: infra-first ordering + bounded boot-retry for transient IP-lease timeouts (#1164)"
```

> ≥3 cold boots reaching end-of-vms with no IP-lease/DNS failures is the Task-7
> acceptance; VAL-A's 5 runs are the reproducibility proof.

---

## Validation (operational) tasks

Created with `vrg-issue-create --kind validation --blocked-by …` (not PR-workable;
close on an attested `Outcome: SUCCESS` comment, never fabricated). Both are
blocked-by **all four** impl issues (#1161, #1162, #1163, #1164).

- **VAL-A — arm64 reproducibility gate (#1154).** **5 consecutive** clean cold
  `mqlab bootstrap nativeha-ubuntu --no-dr` runs on the Apple-silicon host.
  SUCCESS = all 5 reach all four phases (net/vms/provision/observe) green within
  budget, **no per-component timeout failures**. On FAILURE: record the evidence,
  file follow-on fix task(s), leave this task and the epic open. (Precondition: the
  A/B/G changes are merged and the lab is rebuilt to include them — cold-rebuild
  doctrine.)
- **VAL-B — x86 cloud parity (#1155).** At least one cold rebuild on the x86 cloud
  host after the changes; SUCCESS = **no regression** (still green, no new
  failures). Human-run in a cloud session. (Precondition: the A/B/G changes are
  merged and the cloud lab is rebuilt to include them.)

## Terminal bookends

- **Docs review (#1153, in `mq-resiliency-lab-for-linux`)** — sweep human-facing
  docs (esp. bring-up / bootstrap-phase and any startup/timeout guidance) for drift
  from the reliability work; spawn per-repo doc tasks where docs live elsewhere.
  Runs **before** the retrospective.
- **Retrospective (#251, terminal)** — `vergil:epic-retrospective`; refuses to run
  until every other child is closed. Its docs PR closes the epic; §5 records the
  follow-on brainstorm outcome.
- **Follow-on brainstorm (#252)** — seed the C/E/F/D follow-on epic(s); recorded in
  the retrospective, not a terminal gate.

## Self-review

**Spec coverage:**

- §5 A (serialize) → Task 1 (audit) + Task 2 (serialize via #1151 template). ✅
- §5 B budgets → Task 3 (inventory + right-size + fatality) + Task 4 (guardrail). ✅
- §5 B resumability → Task 5 (clean `--from` re-entry). ✅
- §5 G → Task 6 (diagnose) + Task 7 (boot_batch / bounded retry / infra ordering). ✅
- §6 sequencing (parallel frontier; #1163/#1164 `phases.py` overlap) → dependency
  graph + coordination note. ✅
- §6 validation (arm64 5/5, x86 parity, cold-rebuild gate) → VAL-A / VAL-B. ✅
- §7 acceptance (serialized cold-starts, bounded fail-loud budgets, clean resume,
  no boot failures, 5/5, no regression, docs reconciled) → per-task acceptance +
  VAL tasks + #1153. ✅

**Placeholder scan:** no TBD/TODO. The deliberately-open values are the *specific*
budget numbers (Task 3, resolved by the measured cold-start times) and the *chosen
subset* of G levers (Task 7, resolved by the Task-6 diagnosis) — both are
evidence-gated by design (mirroring #236's Task-6 gate), not placeholders. Report
paths carry today's date (2026-09-20); implementers may adjust the date to the day
they land the note.

**File/name consistency (verified against the repo this session):**
`src/mqlab/phases.py` (`:308/:444/:526/:646/:659`), `cli.py:2153/2161`,
`orchestrator.py:67`, `platforms.py:28`, `lab/topology.yaml:135/503`,
`lab/Vagrantfile:66`, `ansible/_nativeha-ubuntu-cluster-ha.yml:33`,
`ansible/roles/mqweb/{templates/mqweb.service.j2:13,tasks/main.yml:103}`,
`ansible/roles/opensearch/tasks/{install.yml:120,configure.yml:38}`,
`tests/test_logsearch_budgets.py`, `tests/test_opensearch_render.py` — all match
the code as read for this plan.
