# macOS/arm64 nested-virt startup reliability & bring-up hardening — design spec

- **Epic:** `logical-minds-foundry/.github#249`
- **Design task (this spec + the plan):** `logical-minds-foundry/.github#250`
- **Implementation tasks (in `mq-resiliency-lab-for-linux`):** `#1161` (A),
  `#1162` (B — budgets), `#1163` (B — resumability), `#1164` (G)
- **Validation (operational):** arm64 reproducibility `#1154`; x86 parity `#1155`
- **Doc-review bookend:** `mq-resiliency-lab-for-linux#1153`
- **Retrospective (terminal):** `logical-minds-foundry/.github#251`
- **Follow-on brainstorm (seed C/E/F/D):** `logical-minds-foundry/.github#252`
- **Prior art (cited, not re-trod):** epic `#70` (bootstrap-performance — baked the
  static software into per-role boxes, eliminating per-run install **I/O**); epic
  `#198` (logsearch min-footprint — sized the OpenSearch/Dashboards/Data-Prepper
  900s readiness budgets, #1034/#1040); PR `#1151` (the mqweb serialization
  fix this epic generalizes).
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-09-20

## 1. Problem & motivation

macOS/arm64 is the go-forward platform for this lab (moving off the expensive x86
cloud box), and it must be **released for public use** on a ~2-week runway. Today
a cold `mqlab bootstrap nativeha-ubuntu --no-dr` on the Apple-silicon host **fails
intermittently** at several independent points:

- **mqweb SIGKILLed** at its systemd start timeout (before PR #1151);
- **OpenSearch / Dashboards / Data Prepper** missing their 15-minute readiness
  budgets;
- **VM boot IP-lease timeouts** (`Fog::Errors::TimeoutError`) during the
  concurrent batch boot;
- **infra-client DNS `no route to host`** when a dependent races the DNS node.

Bring-up is not reliably reproducible, which is a release blocker.

### Root cause — the nested-virtualization JVM cold-start tax

This is **not a resource shortage**, and adding CPU/RAM does not fix it. The
macOS host has *more aggregate resources than the x86 cloud box, yet the cloud
comes up cleanly and macOS does not*. This session's investigation isolated the
constraint to the **nested-virt tax on JVM cold-start**:

- **~7× the cloud on JVM cold-start** — OpenSearch cold-start ~225 s on 6 vCPU
  (macOS → Lima → libvirt/KVM → guest) versus ~30 s on the flat-KVM cloud.
- **CPU-active-but-slow, proven *not* resource-starved** — 6 vCPU with an
  otherwise-idle host is still slow.
- **Proven *not* I/O-blocked** — `iowait ~0`, no D-state pile-up, disk reads ~0
  during the slow window. (This is the material difference from epic #70, whose
  bottleneck *was* I/O — see §2.)
- **Concurrent bring-up compounds the per-service tax into hard timeouts.** Three
  JVMs cold-starting at once on one 2-vCPU node, or several heavy guests booting
  in one batch, each pay the tax simultaneously and blow their budgets.

The flat-KVM cloud never pays this tax, which is why it stays green.

### Relationship to epic #70 (this is the next layer down)

Epic #70 established that a cold bootstrap was **I/O-bound** (iowait 38 % avg /
97 % peak, D-state pile-up) because 12 nested VMs installed the same software
every run; its fix **baked the static software into per-role boxes**, removing
that install I/O. This epic addresses what baking *did not*: with install-I/O
gone, the residual macOS/arm64 failure mode is **CPU-active-but-slow JVM
cold-start under nested virt**, compounded by concurrency. #70 stopped doing
wasteful I/O; #249 stops paying the JVM tax all at once. The two are complementary
and #70's "concurrency/fork tuning — sequenced last" note (its §4.4) is the thread
this epic picks up.

## 2. Goal & non-goals

**Goal: reliable, reproducible bring-up.** *Adequate*, not fast. This is a
**functional-evaluation environment**; nested virt (vert-on-vert) is the accepted
dev-time tradeoff — real performance testing would be bare metal. **Reliability
and reproducibility rank above raw speed.**

**In scope (the reliability core — A/B/G):**

- **A — Serialize / stagger bring-up.** Generalize the mqweb pattern (PR #1151) so
  JVM-heavy services cold-start one-at-a-time per node and in controlled order
  across phases; cut the concurrent JVM startup that compounds the nested tax.
- **B — Bounded, fail-loud readiness budgets + clean resumability.** Audit every
  service's systemd `TimeoutStartSec` and Ansible readiness wait; size them to the
  nested reality (generous-but-bounded), fail-loud where the data/message path
  depends on the service and non-fatal where it does not; and make each bootstrap
  phase resume cleanly after a mid-phase failure.
- **G — Boot-layer hardening.** The VM-boot IP-lease timeouts and DNS
  `no route to host` flakiness — boot-batch sizing, boot-retry robustness, and
  infra/DNS bring-up ordering so dependents don't race the DNS node.

**Out of scope — deferred to the follow-on brainstorm (#252), not this epic:**

- **C — per-start cost:** OpenJ9 `-Xshareclasses` / `-Xquickstart`, OpenSearch
  startup tuning, cross-reboot cache warmth. (Options #1148 explicitly deferred.)
- **E — consolidation:** fewer/bigger VMs (e.g. merge obs + logsearch).
- **F — deep nested-virt profiling** (perf / page-faults) to target C.
- **D — selective right-sizing *where it demonstrably helps*.** The parked #1152
  observe right-size folds here — CPU was **not** the bottleneck, so #1152 is
  **not** merged as a fix.

Also unchanged and out of scope: the x86 cloud is already reliable; IBM MQ and the
`mq-metric-samples` exporter are not touched.

## 3. Doctrine & principles

- **Reliability over speed.** Every decision favours a reproducible cold boot over
  a faster one. A generous-but-bounded budget that always passes beats a tight one
  that occasionally fails.
- **Serialize the tax, don't pay it in parallel.** The nested JVM cold-start cost
  is largely fixed per service; the failure comes from paying several at once.
  Stagger the cold-starts (per node and across phases) so each pays the tax alone.
- **Bounded and fail-loud, sized to the nested reality.** No unbounded wait and no
  swallowed failure. A service the message/data path depends on **fails loud** if
  it misses its budget; a service the path does *not* depend on may warn
  **non-fatally** and let bring-up proceed — but the budget is always explicit and
  bounded (never `TimeoutStartSec=infinity`, never a silent default). This is the
  repo's "no silent failures" rule applied to bring-up.
- **Generalize the proven pattern; do not invent a new one.** PR #1151's mqweb fix
  — non-blocking start + generous `TimeoutStartSec` + play-level `serial:` +
  bounded non-fatal `wait_for` readiness gate — is the **template** (§4). A is its
  disciplined generalization, not a redesign.
- **Cross-platform parity is a hard requirement.** Everything here must *benefit*
  the x86 cloud lab and must **never regress it**. Serialization adds a little
  wall-clock on the (fast) cloud but no failure; budgets must stay generous enough
  that the fast cloud never trips them, yet bounded enough to fail loud. Validation
  runs cold rebuilds on **both** platforms.
- **Cold rebuild is the acceptance gate.** Per the repo's cold-rebuild acceptance
  doctrine, bring-up changes are accepted only after a full VM cold rebuild proves
  them one-pass. **lint-green ≠ done.**
- **Diagnose before fixing; land the diagnosis.** A/G each begin with an
  instrumented observation of the *actual* cold boot and land the findings as a
  `docs/reports/` note, so the fix is attributable to evidence, not a guess.
- **`mqlab` phases stay pure; retries live in the orchestrator.** `src/mqlab/
  phases.py` emits commands and is side-effect-free by design; boot-retry logic
  belongs in the command runner / orchestrator, not smuggled into the pure phase
  builders.

## 4. Grounding: current bring-up architecture (from the code, 2026-09-20)

Established this session by direct read of the repo.

### 4.1 The four-phase bootstrap

`mqlab bootstrap <stack> [--no-dr] [--from <phase>] [--only <phase>] [--step]`
(`src/mqlab/cli.py:2161`) drives four phases, `net → vms → provision → observe`
(`src/mqlab/phases.py:646`):

- **net** — define/start the libvirt lab networks (`_net_build_steps`,
  `phases.py:188`). CPU-free; not a reliability concern.
- **vms** — `vagrant up` the stack's members + commons + logsearch in contiguous
  `boot_batch`-sized batches (`_vms_build_steps`, `phases.py:308`; command at
  `:332`).
- **provision** — NIC config/assure → `mqlab dns render` → `site-dns.yml` → the
  stack's provision playbook (`_provision_build_steps`, `phases.py:444`). mqweb
  comes up here across the QM nodes.
- **observe** — logsearch bring-up (prepended, `cli.py:2115`) → obs stack
  (Prometheus/Grafana) → instrument cluster + host → Grafana health
  (`_observe_build_steps`, `phases.py:526`).

**Resume already exists, but only at phase granularity.** `--from`/`--only` are
implemented (`cli.py:2153`), and with neither flag the runner is stateless — it
probes live world-state (`_probe_all`) and resumes from `first_unsatisfied`
(`phases.py:659`). But each phase's `satisfied()` probe is a **coarse per-phase
boolean** (`nets`, `domains`, `qm_up`, `observe`; `build_states`, `phases.py:671`).
A phase that fails partway re-runs *every* step from the top on resume, so clean
re-entry depends entirely on the constituent steps being individually idempotent —
and that is not guaranteed today (§4.4). **This is the gap B-resumability closes:
not a missing flag, but non-idempotent intra-phase steps.**

### 4.2 The mqweb serialization pattern (PR #1151 — the template for A)

Before #1151, all three `nha-ubuntu-a*` nodes started their Liberty (mqweb) JVM at
once via a fire-and-forget `--no-block` systemd start; the concurrent cold-start
churn dragged the observe phase and SIGKILLed mqweb at its timeout. The fix, in
commit `9c54319`, has four parts:

1. **Non-blocking start** — `systemd ... no_block: true`
   (`ansible/roles/mqweb/tasks/main.yml:69`), so provision never blocks on mqweb.
2. **Generous, bounded systemd budget** — `TimeoutStartSec=1800`
   (`ansible/roles/mqweb/templates/mqweb.service.j2:13`).
3. **Play-level `serial: 1`** — `ansible/_nativeha-ubuntu-cluster-ha.yml:33`, so
   the nodes cold-start one at a time (~20 s isolated) instead of together.
4. **Bounded, non-fatal readiness gate** — the role's last task,
   `wait_for` port 9443 `timeout: 300` with `failed_when: false` and a `debug`
   warn keyed on `elapsed >= 300` (`ansible/roles/mqweb/tasks/main.yml:103-119`).
   Non-fatal is correct **here** because the message path does not depend on the
   REST/console (§4.3 contrast).

**The generalization gap:** the `serial:` half lives **per-play** — only the one
Ubuntu Native-HA play carries it; the other seven plays that include the `mqweb`
role (`_nativeha-cluster-ha.yml:28`, the DR-replication and pcmk/rdqm plays) still
start mqweb in parallel across their hosts. The readiness-gate half already lives
in the shared role, so it applies everywhere. A must close this asymmetry and
extend the same discipline to the other concurrent JVM cold-starts (§4.3).

### 4.3 JVM-heavy services & their budgets today (the B inventory seed)

| Service | Node(s) | systemd `TimeoutStartSec` | Readiness wait | Fatality | Source |
|---|---|---|---|---|---|
| mqweb / Liberty | MQ QM nodes | **1800** | `wait_for` :9443, 300 s | **non-fatal** (`failed_when:false`) | `mqweb.service.j2:13`; `mqweb/tasks/main.yml:103` |
| OpenSearch | logsearch | **180** (`Type=simple`, non-gating) | `uri` `_cluster/health`, 180×5 = **900 s** | **fatal** | `opensearch/tasks/install.yml:120`; `configure.yml:38` |
| OpenSearch Dashboards | logsearch | **900** | `uri` `/api/status`, 900 s | **fatal** | `opensearch-dashboards/tasks/install.yml:90`; `configure.yml:51` |
| Data Prepper | logsearch | **900** | `wait_for` :21892, 900 s | **fatal** | `data-prepper/tasks/install.yml:98`; `configure.yml:28` |
| qm (queue manager) | MQ nodes | 300 | — | — | `mq-qmgr/templates/qm.service.j2:6` |
| mq-exporter, app-requester, node_exporter, alloy, loki, prometheus, grafana-image-renderer, svc-responder@ | various | **unset → systemd default 90 s** | — | — | per role `*.service.j2` / inline `install.yml` |

Two facts this table surfaces:

1. **The `logsearch` node cold-starts three JVMs (OpenSearch, Dashboards, Data
   Prepper) concurrently on 2 vCPU** — the single densest concurrent-JVM point in
   the lab, and a prime A target (serialize them in dependency order:
   OpenSearch → Dashboards + Data Prepper).
2. **Budget inconsistency:** OpenSearch's *unit* is 180 s while its real gate is
   the 900 s Ansible `uri` loop (the `Type=simple` unit reports "started"
   immediately, so 180 s doesn't gate readiness); Dashboards and Data Prepper were
   already bumped to 900 s in the *unit*. Many peripheral units fall to the 90 s
   systemd default. B must reconcile unit-vs-wait budgets to the nested reality and
   make the fatality match data-path dependence.

### 4.4 Boot & DNS layer (the G seed)

- **IP-lease `Fog::Errors::TimeoutError`** originates in `vagrant-libvirt` /
  `fog-libvirt` (`lab/.vagrant/bundler/global.sol`) waiting on the **default
  management-network DHCP lease** during `vagrant up` — the modeled lab NICs are
  DHCP-disabled (`lab/Vagrantfile:66`), so it is the base management NIC's lease
  that times out when many heavy guests boot at once. Aggravated on macOS by
  TCG-slow boots (`platforms.py:28,106`, `TCG_BOOT_TIMEOUT = 1800`).
- **No boot/lease retry exists anywhere.** The orchestrator is fail-loud on the
  first non-zero `vagrant up` exit (`orchestrator.py:2,67`); the only throttle is
  `boot_batch: 4` (`lab/topology.yaml:135`; `_boot_batch`/`_batch_guests`,
  `phases.py:227-261`) plus a same-box `--no-parallel` guard
  (`_batch_shares_box`, `phases.py:295`). A transient lease timeout therefore
  aborts the whole bootstrap with no re-attempt.
- **DNS `no route to host` race.** Every guest's *sole* resolver is its org's
  infra node (`host-resolver/tasks/main.yml`), yet `infra-client` (the DNS node)
  is appended to the **tail** of the boot order (`all_vms`, `phases.py:139`;
  commons order `[obs_box, probe, svc, app, infra]`, `topology.yaml:503`) — so it
  boots *after* its dependents. Only a guest's own FQDN is in `/etc/hosts` as a
  fallback. Existing mitigations are provision-phase, Ansible-level retries
  (`site-dns.yml:12-53` `wait_for_connection` + `is-system-running`), not
  boot-layer ordering.

### 4.5 Existing guardrail-test style (the B-guardrail mirror)

`tests/test_logsearch_budgets.py` is the direct model: `_load_tasks()` parses a
role's tasks YAML, `_unit_content()` extracts the inline systemd unit from the
`ansible.builtin.copy` task by its `dest`, and it string-asserts
`TimeoutStartSec=900` present / `=180` absent, plus the `wait_for`/`uri` budgets
(`BUDGET_SECONDS = 900`). `tests/test_opensearch_render.py` renders the config
template under `StrictUndefined` and asserts `retries==180`/`delay==5`. **Gap:**
no guardrail yet covers the mqweb unit (1800 s) or gate (300 s / non-fatal), nor
the OpenSearch *unit* value — natural additions for B's guardrail test.

## 5. Scope & work breakdown (maps to the plan's tasks and the GitHub issues)

- **A — serialize remaining concurrent JVM cold-starts** → issue **#1161**, plan
  **Tasks 1–2**. Task 1: instrument one cold bootstrap, confirm the concurrent
  cold-start points empirically (logsearch's three JVMs; the seven mqweb plays
  without `serial:`; any cross-node concurrency within a phase). Task 2: serialize
  each remaining one with the #1151 template; land the audit as a `docs/reports/`
  note.
- **B — bounded fail-loud readiness budgets + fatality + guardrail** → issue
  **#1162**, plan **Tasks 3–4**. Task 3: inventory every systemd `TimeoutStartSec`
  and Ansible readiness wait, right-size to the nested reality, set fatality by
  data-path dependence; land the inventory as a `docs/reports/` note. Task 4: add
  `tests/test_startup_budgets.py` asserting each service unit template declares a
  bounded `TimeoutStartSec` in range (mirroring `test_logsearch_budgets.py`).
- **B — clean phase resumability (`--from <phase>`)** → issue **#1163**, plan
  **Task 5**. Make net/vms/provision/observe re-enter cleanly after a mid-phase
  failure — idempotently, without half-state. Induce a mid-observe and a
  mid-provision stop, resume, confirm clean completion; fix any non-idempotent step
  in `src/mqlab/phases.py` or the roles.
- **G — boot-layer hardening** → issue **#1164**, plan **Tasks 6–7**. Task 6:
  diagnose the IP-lease `Fog::Errors::TimeoutError` and the infra-client DNS
  `no route to host`; land the diagnosis as a `docs/reports/` note. Task 7: fix per
  finding — `boot_batch` sizing (`lab/topology.yaml`), boot-retry robustness (a
  bounded retry/backoff around the `vagrant up` step, in the orchestrator), and/or
  infra/DNS bring-up ordering so dependents don't race the DNS node.

**Out of scope** (restated for the implementer): C/E/F/D (§2); IBM MQ; the
exporter; the x86 cloud's already-reliable behaviour beyond proving no regression.

## 6. Sequencing & validation

**Sequencing.** The four implementation tasks (A #1161, B-budgets #1162,
B-resumability #1163, G #1164) touch largely independent surfaces and are
**mutually runnable in parallel** — no `Blocked-by` links among them. One
coordination note: **#1163 and #1164 both edit `src/mqlab/phases.py`** (resumability
touches the phase `satisfied`/step logic; G touches `boot_batch`/boot-retry), so
whichever lands second rebases onto the first — a routine rebase, called out so it
is expected. A's Task-1 instrumented cold bootstrap produces timing evidence that
usefully informs B's budget-sizing and G's diagnosis; the tasks share that one
observation rather than each forcing a serial dependency.

**Validation (two operational tasks, blocked-by all four impl tasks).** A cold
rebuild is the acceptance gate on both platforms; lint-green ≠ done.

- **VAL-A — arm64 reproducibility gate (#1154).** **5 consecutive** clean cold
  `mqlab bootstrap nativeha-ubuntu --no-dr` runs on the Apple-silicon host.
  SUCCESS = all 5 reach all four phases (net/vms/provision/observe) green within
  budget, **no per-component timeout failures**. Run locally.
- **VAL-B — x86 cloud parity (#1155).** At least one cold rebuild on the x86 cloud
  host after the changes; SUCCESS = **no regression** (still green, no new
  failures). Run in a cloud session.

Both close only on a recorded `Outcome: SUCCESS` comment (never fabricated); on
FAILURE, file follow-on fix task(s) and leave the task and the epic open.

## 7. Acceptance criteria

- **A:** each remaining concurrent JVM cold-start is serialized with the #1151
  template — the logsearch JVM trio staggered in dependency order and the `serial:`
  discipline extended to the mqweb plays that lacked it; an instrumented cold
  bootstrap shows serialized (not concurrent) cold-starts and **no timeout
  warnings**; the audit landed as a `docs/reports/` note.
- **B (budgets):** every service's systemd `TimeoutStartSec` and Ansible readiness
  wait is inventoried and right-sized to the nested reality (generous-but-bounded,
  no unset-default surprises for the heavy services, no `infinity`); fatality
  matches data-path dependence (fail-loud where the message/data path depends,
  non-fatal-with-warn where it does not); the guardrail test
  `tests/test_startup_budgets.py` goes RED→GREEN and holds each unit's bound.
- **B (resumability):** a `mqlab bootstrap … --from observe` and `--from provision`
  after an induced mid-phase stop complete cleanly and idempotently, with no
  half-state.
- **G:** ≥3 cold boots reach end-of-vms with **no IP-lease/DNS boot failures**; the
  diagnosis landed as a `docs/reports/` note; the fix is one or more of boot-batch
  sizing, bounded boot-retry, and infra/DNS ordering.
- **Reproducibility (arm64):** VAL-A green — **5/5** clean cold
  `nativeha-ubuntu --no-dr` bootstraps.
- **Parity (x86):** VAL-B green — at least one x86 cloud cold rebuild confirming no
  regression.
- **Docs reconciled** (bookend #1153): the bring-up / bootstrap-phase docs and any
  startup/timeout guidance reflect the reliability work.
- `vrg-validate` green throughout; **the cold-rebuild validations are what accept
  the epic** — lint-green is not "done."

## 8. Risks & mitigations

- **Serialization slows the (already-fine) x86 cloud.** *Mitigation:* the cost is
  bounded wall-clock, not failure; parity VAL-B proves no regression. Where a
  service is genuinely cheap on the cloud, `serial:` can be sized (e.g. `serial: N`)
  rather than forced to 1 — the pattern is a lever, not a dogma.
- **A budget sized for nested arm64 is too generous to catch a real x86 hang.**
  *Mitigation:* budgets are bounded, not unbounded, and fail-loud where the path
  depends; the guardrail test pins the range so a future edit can't quietly drift
  to `infinity` or the 90 s default.
- **Resumability fix hides a real non-idempotency behind "just re-run."**
  *Mitigation:* the acceptance is an *induced* mid-phase stop that must complete
  cleanly — fixing the non-idempotent step, not widening a probe to skip it.
- **Boot-retry masks a genuine boot failure (a real config error retried
  forever).** *Mitigation:* the retry is **bounded** with backoff and stays
  fail-loud after its budget; it is a transient-lease mitigation, not an unbounded
  loop, consistent with the orchestrator's fail-loud contract.
- **`#1163`/`#1164` merge conflict in `phases.py`.** *Mitigation:* expected and
  called out (§6); routine rebase of the second-lander.
- **The tax shifts under us** (a JVM or box version bump changes cold-start cost).
  *Mitigation:* budgets are generous-but-bounded with headroom; the reproducibility
  gate is re-run at acceptance, not assumed from an earlier run.

## 9. Follow-on (seeded to brainstorm #252, not built here)

The "make each start cheaper/smaller" set is generally applicable to the whole lab
and is deferred to a follow-on epic (or epics), brainstormed from this epic's
bookend before it closes:

- **C** — per-start cost (OpenJ9 `-Xshareclasses`/`-Xquickstart`, OpenSearch
  startup tuning, cross-reboot cache warmth; the #1148 option-3 deferral).
- **E** — consolidation (fewer/bigger VMs, e.g. merge obs + logsearch).
- **F** — deep nested-virt profiling (perf / page-faults) to target C.
- **D** — selective right-sizing *where it demonstrably helps* (the parked #1152
  observe right-size folds here — CPU was not the bottleneck, so it is not merged
  as a fix by this epic).
