# Epic 275 — Measurement-driven bootstrap staging for cold-boot reliability + speed

## Why

Cold bootstrap of the lab is unreliable and slow on macOS/arm64 nested-virt. This session
isolated a stack of causes and fixed the first several (merged): the entropy-blocking
launcher (#1190/#1192), the metrics-burst starving OpenSearch (#1194: serialize observe +
obs 12 vCPU), and a too-short readiness budget (#1197: 15 → 40 min). But bootstrap still
fails and crawls, and every wall has the same shape: **the 24-core host is oversubscribed
when too much of the bootstrap runs concurrently** — guests booting/provisioning/running
their MQ workloads while the obs node cold-starts its JVMs.

We have RULED OUT the usual suspects: the JVM and its flags (`-XX:-UseCompactObjectHeaders`
A/B both green in 16 s), disk/page-cache (survives `drop_caches`), memory (~0.5 G of 10 G),
entropy (#1190), and simple CPU-share (a `CPUQuota=160%` cap on an *idle* host still greens
in 10 s). Measured extremes:

- OpenSearch reaches green in **~15 s** on a freshly-rebooted obs with the host uncontended.
- OpenSearch takes **~20–33 min** during a full bootstrap (all 9 guests active).
- app-client's network failed to come up across **3 boot retries** in the concurrent
  `vagrant up` batch — the boot phase is unreliable under the same contention.

The remaining mechanism — hypervisor **vCPU steal** vs **I/O** vs **memory-bandwidth**
contention — is **unmeasured**, and will differ between macOS/arm64 (a laptop hypervisor
oversubscribed across nested guests) and x86 cloud (dedicated cores).

## Goal

Make cold bootstrap **reliable AND fast enough to be useful** on macOS/arm64, finding the
sweet spot **from data**, and validate every change **in parallel on macOS + x86 cloud**.
The optimizations may be **macOS-specific** — cloud may need none ("pound the crap out of
the cloud, let it all start at once").

## Non-goal

An upfront bring-up re-architecture. We do NOT have the metrics to drive one, and guessing
risks optimizing the wrong bottleneck — which differs per arch. A targeted re-architecture is
a *possible outcome* of the data, not a starting assumption. (Cart before horse otherwise.)

## Approach — instrument → tune levers → validate in parallel → iterate

### 1. Perf instrumentation (foundation; macOS-usable immediately)

Every `mqlab bootstrap` emits a **perf report** (structured JSON + human summary in the run
record) capturing WHERE the time and contention go:

- **per-phase wall-clock** (net / vms / provision / observe) and per-step timings (the
  orchestrator already prints e.g. `vms up [1/3] 154.95s` — capture them structurally);
- **milestones**: per-VM boot time and boot-retry counts; OpenSearch time-to-bind and
  time-to-green; Data Prepper / Dashboards ready times;
- **host-contention samples** (~every 15 s across the run): each guest's **vCPU steal %**
  (`/proc/stat`) and load; the **host** total CPU + I/O wait (`virsh nodeinfo` /
  `nodecpustats`, host `iostat` where reachable).

The report is the artifact every iteration reads and the two arches are diffed on. This is
the new capability that finally distinguishes steal vs I/O vs memory-bandwidth, per arch.

### 2. Staging levers (measured hypotheses, per-environment overridable)

Small, mostly-existing knobs, treated as hypotheses to measure — not fixed decisions — and
**overridable per environment** so macOS is throttled while cloud stays maximal:

- **`boot_batch`** (topology, default 4) — shrink to reduce the concurrent-boot flake
  (r6's app-client). Cost: slower boot; the report shows the tradeoff.
- **obs vCPU** (12) — OpenSearch needs ~1.6 cores and the metrics burst is already serialized
  (#1194), so 12 may mostly add host oversubscription now. Measure whether reducing it cuts
  steal without regressing observe.
- **provision → observe overlap** — ensure OpenSearch's cold-start gets a genuinely quiet
  window (it is already first in observe, #1194; verify the handoff doesn't leave other nodes
  hot).

Override mechanism: an environment/profile layer over `lab/topology.yaml` (e.g. `MQLAB_ENV=
macos|cloud`) selecting per-arch lever values, so the same stack runs throttled on macOS and
unthrottled on cloud.

### 3. Standing parallel validation (macOS + x86 cloud, both available now)

x86/macOS parity already exists — the cloud runs the lab today (#1155 is a *deferred
reliability-validation* for a grandparent epic, NOT a functionality blocker). So parallel
validation starts immediately: build the **same commit/branch on both platforms**, run one
VAL definition on each, and diff the perf reports. The one-off cross-arch comparison (#1196)
folds into this standing harness.

**Compare with a grain of salt.** The two run on fundamentally different underlying hardware
(a cloud host vs a macOS laptop), VM architectures, and network stacks. We scale the guests
approximately the same (vCPU/memory), but wall-clock is **directional, not apples-to-apples**
— the value is in the *shape* of the bottleneck (where steal/I/O/time concentrates), not exact
seconds. DIFFERENT sweet spots per arch are expected, and the macOS-specific optimizations may
be unnecessary on cloud.

### 4. Iterate — one variable at a time, evidence-driven

Change **one** lever → build the same commit on both → validate → read the perf diffs →
keep/revert/tune → repeat until **reliable (5 consecutive clean cold bootstraps)** AND
**fast enough to be useful** on each arch. Reliability is more than one data point, so a
"clean" claim requires the consecutive-pass count, not a single green.

Discipline is load-bearing here: there are many interacting variables (boot concurrency, per-
node vCPU, phase overlap, budgets) and the relationships are **non-linear** — more vCPU is not
more performance (that is exactly why obs was bumped to 12 and why it is now a *tuning*
candidate). We do NOT batch changes or tune everything at once; each change is justified by
evidence from the report before the next. Any obs-vCPU reduction happens only on measured
evidence that it improves things. If the data reveals a structural limit, THEN scope a targeted
re-architecture driven by the numbers.

## Testing

- Perf instrumentation: unit-test the perf-record emit/parse; assert the sampler captures
  steal + phase timings.
- Staging levers: guardrail tests on the topology/orchestrator knobs and the per-env override
  selection.
- The sweet spot is proven by the parallel VAL (parts 3–4), not by unit tests.

## Acceptance

- macOS: **5 consecutive clean** cold `nativeha-ubuntu --no-dr` bootstraps, each within a
  "useful" wall-clock — the target set **from the instrumentation** once it shows the floor,
  not guessed up front.
- Per-arch perf reports produced and diffed (macOS + x86 cloud, both from the same commit);
  the macOS bottleneck **named from the data**, read with the different-hardware grain of salt.

## Relationships

Serves epics `logical-minds-foundry/.github#249` (nested-virt startup reliability) and `#267`
(consolidation VAL #1177); absorbs the one-off cross-arch comparison (#1196) into the standing
harness. x86/macOS parity already exists, so this needs no prerequisite — `#1155` (x86
cold-rebuild reliability) is a *deferred grandparent-epic validation* we loop back to after
this stabilization work, not a dependency. This is the exclusive focus for the next few days,
to stabilize the platform for all follow-on lab work.
