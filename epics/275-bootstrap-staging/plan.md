# Measurement-Driven Bootstrap Staging — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `mqlab bootstrap` self-report where its time and host contention go, then add per-environment staging levers so cold boot can be tuned — from data — to reliable-and-fast on macOS/arm64 (and compared against x86 cloud).

**Architecture:** Instrumentation FIRST (a `PerfRecord` collected in the orchestrator + a background host-contention sampler, emitted as JSON + human summary to the run record). THEN a per-environment profile layer (`MQLAB_ENV=macos|cloud`) over `lab/topology.yaml` that selects lever values (`boot_batch`, obs `cpus`). THEN a parallel-validation harness that runs the same commit on both arches and diffs their perf reports. Lever *tuning* is the evidence-driven iteration loop, not a fixed code change — those are follow-on tasks filed as the data dictates.

**Tech Stack:** Python 3.12 (`src/mqlab`, `uv run`), pytest (100% branch coverage, `vrg-validate`), ansible/vagrant/libvirt (lab), YAML (`lab/topology.yaml`).

**Spec:** `epics/275-bootstrap-staging/spec.md` (in the `.github` repo; travels with this plan).

## Global Constraints

- All work in the **mq-resiliency-lab-for-linux** repo, feature branch off `develop`, `vrg-commit`, PR via `vrg-pr-workflow report-ready`. One task = one PR.
- Validation is `vrg-container-run -- vrg-validate` ONLY; must stay green (100% branch coverage, ruff format/E501@100col, ansible-lint).
- Runtime CLI invokes companion tools by bare name; never embed `uv run` in runtime code (`uv run` is dev-loop only).
- No new hardcoded `build/<X>` paths — use `mqlab build path`.
- Perf report is **additive and non-fatal**: instrumentation failure must NEVER fail a bootstrap (wrap sampling in try/except that degrades to "sample unavailable", never a swallowed silent error at the lab layer — log it into the report).
- One variable at a time: the *tuning* of `boot_batch`/obs-vCPU is NOT in these tasks; it is the iteration loop (validation #1200 + evidence-driven follow-ons).

---

### Task 1: PerfRecord model + per-phase/per-step timing capture

Build the perf data model and collect the timings the orchestrator already measures.

**Files:**
- Create: `src/mqlab/perf.py`
- Create: `tests/test_perf.py`
- Modify: `src/mqlab/orchestrator.py` (thread a `PerfSink` through `run_steps`)

**Interfaces:**
- Produces: `perf.PerfRecord` (dataclass) with `.add_step(phase: str, label: str, seconds: float, retries: int)`, `.add_milestone(name: str, seconds: float)`, `.to_json() -> str`, `.human_summary() -> str`; and `perf.PerfSink` (Protocol) with `.step(phase, label, seconds, retries) -> None`. A no-op `perf.NullSink` for callers that don't collect.
- Consumes (from run_steps): the existing per-step `elapsed` and the step's `label`; the phase name (add a `phase: str` field to `CommandStep`, defaulting to "" so existing callers are unaffected until Task 3 sets it).

- [ ] **Step 1: Write the failing test** — `tests/test_perf.py`

```python
from mqlab.perf import PerfRecord

def test_perf_record_collects_steps_and_serialises():
    rec = PerfRecord(stack="nativeha-ubuntu", started_at=1000.0)
    rec.add_step(phase="vms", label="vms up [1/3]", seconds=154.95, retries=0)
    rec.add_step(phase="vms", label="vms up [3/3]", seconds=310.0, retries=2)
    rec.add_milestone(name="opensearch_green", seconds=33 * 60)
    data = __import__("json").loads(rec.to_json())
    assert data["stack"] == "nativeha-ubuntu"
    assert data["phases"]["vms"]["seconds"] == 154.95 + 310.0
    assert data["phases"]["vms"]["retries"] == 2
    assert data["milestones"]["opensearch_green"] == 33 * 60
    assert "vms" in rec.human_summary()
    assert "opensearch_green" in rec.human_summary()
```

- [ ] **Step 2: Run test to verify it fails** — `uv run pytest tests/test_perf.py -v` → FAIL (no module `mqlab.perf`).

- [ ] **Step 3: Write `src/mqlab/perf.py`** — a frozen-ish dataclass aggregating steps by phase (sum seconds, sum retries, keep the per-step list), milestones as a name→seconds map, `to_json()` (stable key order), and `human_summary()` (a phase table + milestones + any `notes` for degraded samples). Include a `PerfSink` Protocol and `NullSink`.

- [ ] **Step 4: Run test to verify it passes** — `uv run pytest tests/test_perf.py -v` → PASS.

- [ ] **Step 5: Thread the sink through `run_steps`** — add `perf: PerfSink = NullSink()` param and `phase` field to `CommandStep` (default ""); after `renderer.ok(...)`, call `perf.step(step.phase, step.label, elapsed, retries)`. `_run_step_with_retry` already knows the retry count — return it alongside `elapsed` (or expose via the step). Add a test asserting `run_steps` feeds a fake `PerfSink` the right (phase,label,seconds,retries) tuples.

- [ ] **Step 6: Run + Commit** — `vrg-container-run -- vrg-validate` green; `vrg-commit --type feat --scope perf --message "PerfRecord model + per-step/phase timing capture in run_steps (#<TASK1>)"`.

---

### Task 2: Host-contention sampler (vCPU steal + host CPU + I/O)

A background sampler that, during a bootstrap, periodically records each running guest's vCPU steal% and the host's CPU/I/O, into the PerfRecord.

**Files:**
- Create: `src/mqlab/perfsampler.py`
- Create: `tests/test_perfsampler.py`
- Modify: `src/mqlab/perf.py` (add `PerfRecord.add_sample(t, host_cpu, host_iowait, guests: dict[str, GuestSample])`)

**Interfaces:**
- Consumes: a `SampleSource` Protocol — `.host() -> HostSample` and `.guest(name) -> GuestSample` — so the sampler is testable with a fake source (real impl shells `virsh nodecpustats`/`nodeinfo` for the host and reads each guest's `/proc/stat` steal via ssh, using the same key/opts the driver uses).
- Produces: `perfsampler.Sampler(record, source, guests, interval=15.0)` with `.start()` / `.stop()` (a daemon thread), degrading a failed probe to a recorded `note`, never raising into the bootstrap.

- [ ] **Step 1: Write the failing test** — a `FakeSource` returning canned host/guest samples; assert that a `Sampler` with `interval=0` ticked N times records N samples with the expected steal values, and that a source raising on one guest records a note but keeps sampling.

- [ ] **Step 2: Run → FAIL.**

- [ ] **Step 3: Implement `perfsampler.py`** — thread loop: every `interval`, snapshot host + each guest via the source, `record.add_sample(...)`; wrap each probe in try/except → `record.note(...)`. `.stop()` joins with a timeout.

- [ ] **Step 4: Run → PASS.**

- [ ] **Step 5: Real `SampleSource`** — host via `virsh nodecpustats --percent`/`nodeinfo`; guest steal via `ssh <guest> awk '/^cpu /{print $9}' /proc/stat` deltas. Cover with a test that parses fixed `/proc/stat` + `virsh` output strings (no live calls).

- [ ] **Step 6: Commit** — `vrg-commit --type feat --scope perf --message "host-contention sampler: vCPU steal + host CPU/IO into PerfRecord (#<TASK2>)"`.

---

### Task 3: Wire perf capture into bootstrap + emit the report + milestones

Make `mqlab bootstrap` actually build a PerfRecord, run the sampler, tag each phase's steps with their phase name, capture the OpenSearch/boot milestones, and write the report to the run record.

**Files:**
- Modify: `src/mqlab/phases.py` (set `CommandStep.phase` on emitted steps; expose the observe milestone hooks)
- Modify: `src/mqlab/cli.py` (bootstrap entry ~line 108: construct PerfRecord, start/stop Sampler around `run_steps`, write `<runrecord>/perf-<ts>.json` + print `human_summary()`)
- Modify: `tests/` (cli/phases tests)

**Interfaces:**
- Consumes: `PerfRecord`, `PerfSink`, `Sampler` (Tasks 1–2); the run-record dir from `mqlab build path` (reuse the existing bootstrap output location — do NOT hardcode).
- Produces: a `perf-<ts>.json` + human summary per bootstrap; milestones `opensearch_bound`, `opensearch_green`, `data_prepper_ready`, `dashboards_ready`, and per-VM `boot:<name>` seconds + `boot_retries:<name>`.

- [ ] **Step 1: Failing test** — assert each `Phase.build_steps` tags its steps with the phase name (`net`/`vms`/`provision`/`observe`); assert bootstrap writes a `perf-*.json` whose `phases` keys are exactly those four.

- [ ] **Step 2: Run → FAIL.**

- [ ] **Step 3: Set `phase=` on every emitted CommandStep** in `phases.py` (each phase's `build_steps` stamps its own name). Milestones for observe: derive `opensearch_bound`/`_green` from the existing readiness-wait step boundaries (the wait step's start→success elapsed); boot times from the vms-phase per-batch step timings + `_BOOT_RETRY` counts.

- [ ] **Step 4: Construct + emit in cli.py** — build `PerfRecord(stack, started_at)`, start `Sampler(record, RealSource(...), guests=all_vms(...))`, pass `perf=record` into `run_steps`, stop the sampler in a `finally`, then `write_text(perf_json)` + `renderer.output(record.human_summary())`. All wrapped so a perf failure never aborts the bootstrap.

- [ ] **Step 5: Run → PASS; `vrg-validate` green; Commit** — `feat(perf): emit per-bootstrap perf report (timings + steal samples + OpenSearch/boot milestones) (#<TASK3>)`.

> After Task 3, run ONE instrumented cold bootstrap on macOS and read the report — this is the first real data (records the macOS bottleneck shape). No lever change yet.

---

### Task 4: Per-environment profile layer (MQLAB_ENV) over topology

Let the same stack run with different lever values per platform, so macOS can be throttled while cloud stays maximal — without forking topology.

**Files:**
- Modify: `lab/topology.yaml` (add an optional `env_profiles:` map: `{macos: {...overrides}, cloud: {...overrides}}`)
- Modify: `src/mqlab/stacks.py` or wherever topology is loaded (apply the `MQLAB_ENV` profile overrides onto the base topology at load)
- Create: `tests/test_env_profiles.py`

**Interfaces:**
- Consumes: `os.environ["MQLAB_ENV"]` (default `""` → base topology unchanged; unknown value → fail loud, no silent default).
- Produces: the effective topology with the selected profile's overrides applied (deep-merge: a profile may set `boot_batch`, and per-node `cpus`). Everything downstream (phases `_boot_batch`, node vCPU) reads the effective topology, so no other call site changes.

- [ ] **Step 1: Failing test** — base topology `boot_batch=4`, obs `cpus=12`; with `MQLAB_ENV=macos` and a profile `{boot_batch: 2, nodes: {obs: {cpus: 8}}}`, the effective topology yields `boot_batch==2` and obs `cpus==8`; with `MQLAB_ENV` unset it's unchanged; with `MQLAB_ENV=bogus` it raises `ValueError` (no silent default).

- [ ] **Step 2: Run → FAIL.**

- [ ] **Step 3: Implement the deep-merge profile apply** at topology load, gated on `MQLAB_ENV`; fail loud on an unknown env name.

- [ ] **Step 4: Run → PASS.**

- [ ] **Step 5: Seed empty profiles + doc** — add `env_profiles: {macos: {}, cloud: {}}` (no overrides yet — values are filled by the evidence-driven iteration, one at a time) with a comment pointing at the spec. Guardrail test: `env_profiles` keys ⊆ `{macos, cloud}`.

- [ ] **Step 6: Commit** — `feat(topology): MQLAB_ENV per-environment profile overrides (macos|cloud) (#<TASK4>)`.

---

### Task 5: Parallel-validation harness + perf-report diff

One command to run the cold-bootstrap VAL and, given two run records (macOS + cloud), diff their perf reports for the bottleneck-shape comparison.

**Files:**
- Create: `src/mqlab/perfdiff.py` (+ a `mqlab perf diff <a.json> <b.json>` CLI subcommand in `cli.py`)
- Create: `tests/test_perfdiff.py`
- Create: `docs/development/perf-and-staging.md` (how to run the VAL on each arch with `MQLAB_ENV`, collect the two `perf-*.json`, and read the diff — with the grain-of-salt caveat: directional, not apples-to-apples)

**Interfaces:**
- Consumes: two `PerfRecord` JSONs (Task 1 format).
- Produces: `perfdiff.diff(a: dict, b: dict) -> DiffReport` (per-phase seconds delta + ratio, per-milestone delta, top steal contributors on each side) and a human table; the CLI prints it. NO pass/fail verdict — it's a comparison aid, read with judgment.

- [ ] **Step 1: Failing test** — two canned reports where phase `observe` is 33 min on A and 2 min on B; assert `diff` reports the observe delta + ratio and flags observe as the dominant divergence; assert it tolerates a milestone present on one side only.

- [ ] **Step 2: Run → FAIL.**

- [ ] **Step 3: Implement `perfdiff.py` + the `mqlab perf diff` subcommand.**

- [ ] **Step 4: Run → PASS.**

- [ ] **Step 5: Write `docs/development/perf-and-staging.md`** — the parallel-run procedure (same commit, `MQLAB_ENV=macos` vs `=cloud`, collect reports, `mqlab perf diff`), the iteration discipline (one lever at a time), and the grain-of-salt caveat.

- [ ] **Step 6: `vrg-validate` green; Commit** — `feat(perf): perf-report diff + parallel-validation harness + docs (#<TASK5>)`.

---

## After the plan lands

Tasks 1–5 give us instrumentation + the per-env lever mechanism + the comparison harness. The **tuning itself** — shrinking `boot_batch`, reducing obs vCPU, adjusting phase overlap — is the iteration loop under validation #1200: change ONE lever in the `macos` profile, run the parallel VAL, read `mqlab perf diff`, keep/revert on evidence. Each lever change that proves out is its own small PR (one variable). A structural re-architecture is filed only if the data shows a wall these levers cannot move.

## Self-Review

- **Spec coverage:** Component 1 (instrumentation) → Tasks 1–3; Component 2 (per-env levers) → Task 4 (mechanism) + the iteration loop (values); Component 3 (parallel validation) → Task 5 + validation #1200; Component 4 (iterate) → "After the plan lands" + #1200. Testing/acceptance → each task's `vrg-validate` + #1200's consecutive-pass gate.
- **No placeholders:** lever *values* are intentionally empty in Task 4 Step 5 — that is the evidence-driven design, not a TODO; every code task has concrete files/interfaces/tests.
- **Type consistency:** `PerfRecord`/`PerfSink`/`Sampler`/`SampleSource`/`diff` names are used consistently across Tasks 1→5.
