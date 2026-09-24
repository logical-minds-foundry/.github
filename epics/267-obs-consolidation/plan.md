# Consolidate the logsearch tier onto the obs node — implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: use `vergil:issue-implement` per
> GitHub task (each task below maps to one issue under epic
> `logical-minds-foundry/.github#267`). Steps use checkbox (`- [ ]`) syntax.

**Goal:** Run the whole observability platform (metrics **and** logs) on one
adequately-resourced `obs` node — fold the logsearch tier (OpenSearch + Dashboards +
Data Prepper) onto `obs`, retire the `logsearch` node/box/group, combine resources
(→ 4 vCPU / 10 GB) — so the observe phase becomes reliable on nested arm64 and the
paused VAL-A (`mq-resiliency-lab-for-linux#1154`) can proceed.

**Architecture:** `obs` and `logsearch` are two Ubuntu nodes with two baked boxes
(`bake-obs.yml` / `bake-logsearch.yml`) brought up in the observe phase by two
configure plays (`site-obs.yml` / `site-logsearch.yml`). We merge them: one box
(`obs-ubuntu2404`) bakes the union; one node runs both; the observe phase brings
metrics + logs up in one sequence with localhost inter-service links. JVM heaps are
explicitly bounded to coexist in 10 GB, and Dashboards' `opensearch.requestTimeout`
is raised to the nested reality to kill the saved-objects migration deadlock.

**Tech Stack:** `lab/topology.yaml`, `ansible/bake-obs.yml` + the opensearch/
dashboards/data-prepper roles, `ansible/site-logsearch.yml` / `site-obs.yml`,
`src/mqlab/{phases.py,cli.py,dns.py,...}`, the fat-box builder
(`lab/boxes/build-fatbox.sh`), `pytest` (guardrails), `vrg-container-run --
vrg-validate`.

**Spec:** `epics/267-obs-consolidation/spec.md` (this directory). Read it alongside;
§4 is the code grounding, §5 the design, §5.2/§6 the box-fit gate.

## Global Constraints

- **Adequate-over-starved, iterate:** start `obs` at the combined 4 vCPU / 10 GB;
  bump only if a cold-rebuild shows pressure (spec §2–§3).
- **Bounded & fail-loud, sized to nested reality:** JVM heaps explicitly capped;
  Dashboards request/readiness timeouts generous-but-bounded (spec §3).
- **Reuse the #70 bake model; simplify by co-location** (inter-service links become
  localhost). Don't invent parallel tooling.
- **The 20 G guest-disk ceiling is hard** (`build-fatbox.sh:222-230`, #1144/#1146):
  never let the merged box's partition exceed 20 G — the box-fit gate (Task 1) is a
  prerequisite, not a routine bump.
- **Cold rebuild is the acceptance gate; lint-green ≠ done.** `vrg-validate` is the
  only validation command; `uv run` never enters a runtime path.
- **Commit with `vrg-commit`; branch per issue off `develop`; PRs into `develop`.**

## Dependency graph

```text
[#268 spec + plan lands]
   └── Task 1  box-fit gate + merged box bake  (foundation; re-bake)
         ├── Task 2  consolidation: topology + observe plays + phase code + DNS/renders/mqlab retarget (+ ref guardrail)
         └── Task 3  coexistence sizing + Dashboards requestTimeout (+ budget guardrail)
               └── VAL #1177  arm64 cold-rebuild — observe green (metrics + logs on obs)
                     └── Docs review #1176 → Retrospective #269
```

Tasks 2 and 3 both depend on Task 1 (the merged box must exist), and are otherwise
independent (topology/plays vs. heap/timeout config) — a parallel pair. VAL #1177 is
blocked-by 1–3. A half-merged `develop` still passes `vrg-validate` (static), so the
split is review-friendly; the cold-rebuild (VAL) is the integration proof.

---

### Task 1 (box): box-fit gate + merged `obs` box bake

Establish that the merged box fits the 20 G ceiling, then bake the union.

**Files:**

- Read/measure: `lab/boxes/build-fatbox.sh` (the 18 G resize, `:222-230`), the
  current `build/state/boxes/{obs,logsearch}-ubuntu2404-aarch64.box`.
- Modify: `ansible/bake-obs.yml` (add logsearch install roles), retire
  `ansible/bake-logsearch.yml`; the box fleet (`src/mqlab` `_LOCAL_BOX_BUILDERS`,
  `mqlab box`, `manifests/_shared/observability.yaml`) to drop `logsearch-ubuntu2404`.

- [ ] **Step 1 (GATE): measure the merged installed footprint.** Boot the current
  obs box and logsearch box (or inspect their images), sum the installed sizes over
  the shared base, and confirm the union fits under **18 G** (guest `virtual_size:20`).
  Record the numbers in a `docs/reports/` note.
- [ ] **Step 2 (only if it does NOT fit): scoped prerequisite** — trim (dedupe base
  tooling; share one JDK across OpenSearch/Data-Prepper where possible; strip
  caches/docs) or raise `virtual_size` fleet-wide per #1146's truncation lesson.
  Re-measure until it fits.
- [ ] **Step 3: fold logsearch install roles into `bake-obs.yml`** (OpenSearch +
  Dashboards + Data Prepper installs, the bake halves per #70), retire
  `bake-logsearch.yml` and remove `logsearch-ubuntu2404` from the box fleet/manifest.
- [ ] **Step 4: bake + validate.** `uv run mqlab box build obs-ubuntu2404` produces a
  bootable merged box under the ceiling; `vrg-container-run -- vrg-validate` green.
- [ ] **Step 5: commit.**

```bash
vrg-commit --type feat --scope box \
  --message "bake the logsearch tier into the obs box; retire the logsearch box (#<TASK1>)"
```

---

### Task 2 (consolidation): topology + observe plays + phase code + reference retarget

The core merge: one node, one observe sequence, no dangling logsearch references.

**Files:**

- Modify: `lab/topology.yaml` (remove `logsearch` node + `logsearch_box` group; grow
  `obs` to `cpus: 4`, `memory: 10240`); `ansible/site-logsearch.yml` (configure plays
  `hosts: logsearch` → the obs host) folded into the observe sequence;
  `src/mqlab/{phases.py,cli.py}` (collapse the logsearch observe prepend into obs's
  observe; fix `all_vms`/groups consumers); `src/mqlab/dns.py` + renders + the
  `mqlab logsearch` CLI namespace + any hardcoded `10.50.0.4`; Grafana datasource /
  alloy / Data-Prepper endpoints → localhost.
- Create: `tests/test_no_logsearch_node_refs.py` (the two-pronged guardrail).

- [ ] **Step 1: topology.** Remove the `logsearch` node + `logsearch_box` group; set
  `obs` `cpus: 4`, `memory: 10240`. Fix every `topology.yaml`/inventory consumer in
  `src/mqlab` (`all_vms`, groups, the observe prepend).
- [ ] **Step 2: observe plays + phase code.** Retarget `site-logsearch.yml`'s
  configure plays to the obs host and fold them into obs's single observe sequence
  (`phases.py`/`cli.py`); point inter-service links (Grafana↔OpenSearch,
  alloy→Data-Prepper, Data-Prepper→OpenSearch) at **localhost**.
- [ ] **Step 3: retarget the rest** — DNS record (drop `logsearch`), renders,
  reach-peers, dashboards, and every hardcoded `10.50.0.4` in `src/`/`ansible/`/
  `lab/`/`manifests/`.
- [ ] **Step 4: two-pronged guardrail test** (`tests/test_no_logsearch_node_refs.py`):
  assert (a) no `10.50.0.4` literal in the runtime paths, and (b) no `logsearch`
  host/group reference in topology/inventory consumers (distinct from the services'
  own cluster/box-name usage). RED against any stranded ref → GREEN when clean.
- [ ] **Step 5: validate.** `vrg-container-run -- vrg-validate` green (100 % branch
  coverage; the guardrail's helper branches exercised).
- [ ] **Step 6: commit.**

```bash
vrg-commit --type feat --scope obs \
  --message "consolidate logsearch onto obs: topology + observe + retarget refs (#<TASK2>)"
```

---

### Task 3 (tuning): JVM-heap coexistence + Dashboards request-timeout

Make the shared 10 GB node hold six services without OOM, and kill the migration
deadlock.

**Files:**

- Modify: the OpenSearch / Data Prepper heap settings (role defaults / jvm.options)
  and the Dashboards config (`opensearch.requestTimeout` + saved-objects migration
  retry budget) in `ansible/roles/{opensearch,data-prepper,opensearch-dashboards}`.
- Create: `tests/test_obs_coexistence_budgets.py` (heap caps + Dashboards timeout).

- [ ] **Step 1: cap the JVM heaps** for the shared box — bound OpenSearch (`-Xms`/
  `-Xmx`) and Data Prepper explicitly, budgeting Dashboards' Node footprint + the Go
  services (Prometheus/Grafana/Loki) + OS so the total fits 10 GB with margin.
  Record the budget in a `docs/reports/` note.
- [ ] **Step 2: raise Dashboards `opensearch.requestTimeout`** (+ migration retry) to
  the nested reality so the saved-objects migration request cannot time out and
  deadlock `.kibana_1` (the observed #249 failure).
- [ ] **Step 3: guardrail test** — assert each heap cap is set and within the shared
  budget, and Dashboards' `requestTimeout` is at/above the chosen bound (mirror
  `tests/test_startup_budgets.py` / `test_logsearch_budgets.py`).
- [ ] **Step 4: validate + commit.** `vrg-container-run -- vrg-validate` green.

```bash
vrg-commit --type fix --scope obs \
  --message "cap JVM heaps for the shared obs node; raise Dashboards requestTimeout (#<TASK3>)"
```

---

## Validation (operational) task

- **VAL #1177 — arm64 cold-rebuild (blocked-by Tasks 1–3).** Re-bake the merged obs
  box and run a clean cold `mqlab bootstrap nativeha-ubuntu --no-dr` on the
  Apple-silicon host. SUCCESS = observe green with **both** metrics
  (Prometheus/Grafana/Loki) and logs (OpenSearch/Dashboards/Data Prepper) up **on the
  obs node**, no per-component timeout or migration deadlock. Feeds the paused VAL-A
  (`#1154`). Close only on a recorded `Outcome: SUCCESS`; on FAILURE record evidence,
  file follow-on fix task(s), leave open.

## Terminal bookends

- **Docs review (`#1176`, lab repo)** — sweep human-facing docs (site build-layout /
  box-taxonomy / operate pages; the lab-bootstrap runbook; the operating-the-lab
  access doc) for drift from the retired logsearch node/box and the merged obs node;
  spawn per-repo doc tasks where docs live elsewhere. Runs before the retrospective.
- **Retrospective (`#269`, terminal)** — `vergil:epic-retrospective`; its docs PR
  closes the epic; records the follow-on (mqweb client-mode #266 half; #266 C/F).

## Self-review

**Spec coverage:**

- §5.1 topology/resources → Task 2 (topology). ✅
- §5.2/§6 box merge + box-fit gate → Task 1 (gate + merged bake). ✅
- §5.3 observe plays/phase code → Task 2 (plays + phase). ✅
- §5.4 heap coexistence + Dashboards timeout → Task 3. ✅
- §5.5 DNS/renders/mqlab retarget + two-pronged guardrail → Task 2 (steps 3–4). ✅
- §6 validation (cold-rebuild) → VAL #1177; §9 unblocks VAL-A #1154. ✅

**Placeholder scan:** no TBD/TODO. `<TASK1>`/`<TASK2>`/`<TASK3>` are filled with the
issue numbers when the tasks are filed (epic-create step 9). The heap figures and the
`requestTimeout` value are resolved in Task 3 against the measured footprint (design
intent, not placeholders).

**File/name consistency (verified this session):** `lab/topology.yaml` (obs/logsearch
nodes, `logsearch_box`), `ansible/bake-obs.yml` + `bake-logsearch.yml`,
`ansible/site-logsearch.yml`, `lab/boxes/build-fatbox.sh:222-230` (18 G/20 G),
17 hardcoded `10.50.0.4` refs across `src/`/`ansible/`/`lab/`/`manifests/`, the
`opensearch`/`opensearch-dashboards`/`data-prepper` roles — all match the code read
for this plan.
