# Consolidate the logsearch tier onto the obs node — design spec

- **Epic:** `logical-minds-foundry/.github#267`
- **Design task (this spec + the plan):** `logical-minds-foundry/.github#268`
- **Doc-review bookend:** `logical-minds-foundry/mq-resiliency-lab-for-linux#1176`
- **Retrospective (terminal):** `logical-minds-foundry/.github#269`
- **Validation (cold-rebuild):** `logical-minds-foundry/mq-resiliency-lab-for-linux#1177`
- **Implements:** the **consolidation half** of `logical-minds-foundry/.github#266`
  (instrumentation-plane rearchitecture) and `#252`'s "E — consolidation".
- **Unblocks:** epic `#249`'s arm64 reproducibility gate
  (`mq-resiliency-lab-for-linux#1154`).
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-09-24

## 1. Problem & motivation

The arm64 reproducibility gate for epic #249 (VAL-A, `#1154`) proved that the
observability tier is the recurring cold-bootstrap bottleneck on macOS/arm64, and
that the cause is **structural, not a bug**: the `logsearch` node is a **2-vCPU**
guest running **three JVMs at once** — OpenSearch, OpenSearch Dashboards, and Data
Prepper. On a repeated cold `nativeha-ubuntu --no-dr` run (with per-node mqweb
retired in #1171 and apt auto-updates quiesced in #1173):

- OpenSearch reached green but **barely** within its 900s budget (~21 min cold).
- **OpenSearch Dashboards deadlocked its saved-objects migration**: its request to
  OpenSearch **timed out** under nested-virt slowness, leaving `.kibana_1`
  half-created; on retry Dashboards saw `resource_already_exists` and entered
  *"Another OpenSearch Dashboards instance appears to be migrating… waiting for
  that migration to complete"* — a **permanent** wait. Dashboards never goes
  green, so the fatal observe wait would exhaust.

Splitting the observability engine across **two small, dedicated, undersized
nodes** (`obs` 2 vCPU/4 GB + `logsearch` 2 vCPU/6 GB) starves the JVMs. The host
has ample capacity (24 vCPU); the per-node 2-vCPU cap is the constraint.

## 2. Goal & non-goals

**Goal.** Run the **whole observability platform on one adequately-resourced
instrumentation node**: fold the logsearch tier onto `obs`, combining the two
nodes' vCPU/RAM. Logs are part of observability; co-locating metrics + logs is a
net **simplification** — one shared-resource machine, localhost inter-service
links — and gives the JVMs headroom instead of starving two nodes.

**In scope:** the topology merge; a merged box bake; observe-play + phase-code
retarget; JVM-heap coexistence sizing + the Dashboards request-timeout hardening;
DNS/renders/`mqlab` retarget. (§5.)

**Non-goals:**

- **mqweb client-mode** — the *other* half of #266; a sibling follow-on, not here.
- `mon-probe` and the on-`obs` `mq_prometheus` exporter — unchanged.
- The RHEL arms beyond proving no regression; the x86 cloud (already reliable).
- Deep per-JVM startup-cost tuning (`-Xshareclasses` etc., #266 "C") — not here.

## 3. Doctrine & principles

- **The instrumentation plane exists to instrument the core 3+3.** Everything
  outside the core is there to *observe* it; nobody runs the lab's obs/logsearch as
  a production topology. So co-locating and right-sizing the instrumentation engine
  is legitimate and desirable, not a compromise.
- **Adequate resources over dedicated-but-starved.** One machine with the combined
  resources beats two undersized ones. Start at the plain combined figure and bump
  only if a cold-rebuild shows pressure ("combine, then find out").
- **Bounded, fail-loud, sized to the nested reality** (inherited from #249): JVM
  heaps are explicitly bounded so services coexist without OOM; readiness/request
  timeouts are generous-but-bounded, sized to the nested cold-start cost.
- **Reuse the bake model; don't invent.** The #70 bake/configure split stands — the
  merged box bakes the union of the two software sets via the existing
  `ansible/bake-*.yml` mechanism; configure stays per-run.
- **Simplify by co-location.** Cross-node inter-service links (Grafana↔OpenSearch,
  alloy→Data-Prepper, Data-Prepper→OpenSearch) become localhost — fewer moving
  parts, not more.
- **Cold rebuild is the acceptance gate** (repo doctrine); lint-green ≠ done.
- **Cross-platform parity:** must benefit / never regress the x86 cloud.

## 4. Grounding: current architecture (from the code, 2026-09-24)

- **Two nodes** (`lab/topology.yaml`): `obs` (`platform: obs-ubuntu2404`, `cpus: 2`,
  `memory: 4096`, `net-mgmt 10.50.0.2`) runs Prometheus + Grafana + Loki + the cgo
  `mq_prometheus` exporter; `logsearch` (`platform: logsearch-ubuntu2404`,
  `cpus: 2`, `memory: 6144`, `net-mgmt 10.50.0.4`) runs OpenSearch + Dashboards +
  Data Prepper. `logsearch` is its **own group** `logsearch_box` (a declared "CORE
  observability layer", always booted).
- **Two baked boxes**, two bake playbooks: `ansible/bake-obs.yml` and
  `ansible/bake-logsearch.yml` (the #70 provision-then-snapshot model).
- **Two configure plays** in the observe phase: `ansible/site-obs.yml`
  (`hosts: obs_box`) and `ansible/site-logsearch.yml` (`hosts: logsearch`), plus
  `observability.yml` (instrument cluster nodes) and `host-obs.yml` (instrument the
  libvirt host). The observe phase **prepends** the logsearch bring-up as a separate
  node (`src/mqlab/cli.py` `_logsearch_up_steps`, ~`:977`/`:2115`).
- **Cross-node links today:** Grafana's OpenSearch datasource → `logsearch`;
  alloy (`loki.write`/`otelcol`) and the log path → Data Prepper on `logsearch`;
  Data Prepper → OpenSearch on `logsearch`. DNS carries a `logsearch` record; the
  `mqlab logsearch` CLI namespace (status/open/snapshot/restore) targets the
  `logsearch` host.
- **Dashboards migration timeout** is Dashboards' internal `opensearch.requestTimeout`
  (default ~30 s) — **not** covered by the ansible 900s readiness budgets (#1040);
  the deadlock in §1 is this internal timeout firing under nested-virt slowness.

## 5. Design

### 5.1 Topology & resources

In `lab/topology.yaml`: **remove the `logsearch` node** and the `logsearch_box`
group; **grow `obs`** to `cpus: 4`, `memory: 10240` (2+2 vCPU, 4096+6144 MB — the
plain combined figure). `obs` keeps `10.50.0.2`; `10.50.0.4` is retired. `obs_box`
becomes the single instrumentation group carrying both the metrics and log services.
Iterate RAM up only if the first cold-rebuild shows memory pressure. `mon-probe` and
all other nodes are untouched.

### 5.2 Merged box bake

Fold logsearch's **install** roles (OpenSearch + Dashboards + Data Prepper + their
JDKs) into `ansible/bake-obs.yml`, so **one box** ships the full platform; retire
`ansible/bake-logsearch.yml` and the `logsearch-ubuntu2404` box from the fleet
(`mqlab box`, `_LOCAL_BOX_BUILDERS`, manifest). The box **keeps the name
`obs-ubuntu2404`** (it is still the obs box, now complete) — avoiding a fleet-wide
rename.

**Box-disk ceiling — an early gate, not a routine bump.** The merged box grows by
the OpenSearch/Dashboards/Data-Prepper artifacts (+ bundled JDKs) on top of an obs
box that **already consumed its disk headroom** (#1145/#1147). `build-fatbox.sh`
pins the build disk at an **absolute 18 G because guests instantiate at
`virtual_size:20`, and a partition over 20 G is truncated at guest-create — the
GPT-corruption failure of #1144, fixed in #1146**. So "bump the disk" is **not**
available past 20 G. Therefore the plan's **first step is a measurement gate**:
confirm the merged *installed* footprint (union of the two current boxes over the
shared ~8.7 G base) fits under 18 G. If it does not, a **scoped prerequisite** —
either trim the footprint (dedupe base tooling, share one JDK, strip caches) or
raise the box `virtual_size` fleet-wide (handled per #1146's truncation lesson) —
runs **before** the merge. This gate protects the whole epic (everything rests on
the box building).

### 5.3 Observe plays & phase code

Retarget `site-logsearch.yml`'s **configure** plays from `hosts: logsearch` to the
obs host (`obs_box`), and **collapse the observe-phase logsearch prepend** into
`obs`'s single observe sequence (`src/mqlab/phases.py` / `cli.py`): one node, one
observe bring-up (metrics + logs). Inter-service references become **localhost**
(Grafana's OpenSearch datasource, alloy→Data-Prepper, Data-Prepper→OpenSearch),
removing the cross-node dependency.

### 5.4 Coexistence sizing + the Dashboards fix (the crux)

On the shared 10 GB node the three JVMs must coexist with the Go services
(Prometheus/Grafana/Loki + the exporter) **without OOM**:

- **Bound the JVM heaps explicitly.** OpenSearch otherwise defaults to ~50 % of
  RAM; on a shared box it must be capped (`-Xms`/`-Xmx`) with Data Prepper's heap
  and Dashboards' Node footprint budgeted so the total (plus the Go services and OS)
  fits 10 GB with margin. The plan sets concrete figures and a guardrail test.
- **Raise Dashboards' `opensearch.requestTimeout`** (and the saved-objects
  migration retry budget) to the nested reality, so the migration request does not
  time out mid-flight and deadlock `.kibana_1`. This directly targets the observed
  failure and is belt-and-suspenders alongside the faster OpenSearch responses a
  bigger node yields.

### 5.5 DNS / renders / `mqlab`

Drop the `logsearch` DNS record; retarget every logsearch reference — obs scrape
targets, reach-peers, dashboards, the `mqlab logsearch` CLI namespace, and
`all_vms`/groups/phase consumers in `src/mqlab` — to the obs host. The
`mqlab logsearch` verbs keep working, now resolving to obs.

**Concrete, two-pronged guardrail.** The retarget surface is wide (~29 files
mention `logsearch`; **17 hardcoded `10.50.0.4` literals** across `src/`,
`ansible/`, `lab/`, `manifests/`), and the IP literals are the easy miss — a
guardrail that greps only the string `logsearch` passes a stranded `10.50.0.4`. So
the guardrail test asserts **both**: (a) no `10.50.0.4` literal survives in the
runtime paths (`src/`, `ansible/`, `lab/`, `manifests/`); and (b) no `logsearch`
**host/group** reference remains in the topology/inventory consumers — distinct
from the OpenSearch/Dashboards/Data-Prepper roles' own cluster/box-name usage,
which moves *with* the services and is fine.

## 6. Sequencing & validation

Implementation order (refined in the plan): **(0) box-fit measurement gate** —
confirm the merged installed footprint fits under 18 G (§5.2); if not, the scoped
trim / `virtual_size` prerequisite runs first. Then (1) topology + box-bake merge
(the foundation, requires a re-bake); (2) observe-play + phase-code retarget +
localhost links; (3) heap coexistence + Dashboards timeout; (4) DNS/renders/`mqlab`
retarget. These are largely one coherent change set; the plan will split them into
reviewable tasks with a guardrail test where each is statically checkable.

**Validation (`#1177`, operational).** A full arm64 **cold-rebuild** of the merged
obs box, then a clean cold `mqlab bootstrap nativeha-ubuntu --no-dr`: SUCCESS =
observe green with **both** metrics (Prometheus/Grafana/Loki) and logs
(OpenSearch/Dashboards/Data Prepper) up **on the obs node**, no per-component
timeout or migration deadlock. This feeds the paused VAL-A (`#1154`). A green
`vrg-validate` is necessary, not sufficient (cold-rebuild doctrine).

## 7. Acceptance criteria

- `logsearch` node/box/group retired; `obs` runs the full platform at 4 vCPU / 10 GB
  (or higher if the cold-rebuild shows pressure), with bounded, coexisting JVM heaps.
- One merged box (`obs-ubuntu2404`) bakes metrics + logs; `bake-logsearch.yml` and
  the `logsearch-ubuntu2404` box are gone from the fleet.
- The observe phase brings up metrics + logs on `obs` in one sequence; inter-service
  links are localhost; no dangling `logsearch` references (DNS, renders, `mqlab`).
- Dashboards reaches green (no migration deadlock) on a cold rebuild.
- **VAL `#1177` green** on arm64; the paused VAL-A `#1154` can proceed; **no x86
  regression**.
- Human-facing docs reconciled (bookend `#1176`).

## 8. Risks & mitigations

- **10 GB is too tight for all six services + OS.** *Mitigation:* explicit heap
  caps + a guardrail; iterate RAM up (the "combine, then find out" contract) — the
  host has ample capacity.
- **Dashboards still deadlocks despite consolidation.** *Mitigation:* the
  `requestTimeout` bump (§5.4) directly targets it, independent of node size.
- **Merged box overflows the hard 20 G guest-disk ceiling** (#1144/#1146) — "bump
  the disk" is unavailable past 20 G. *Mitigation:* the §5.2 / §6 **early
  measurement gate**; if it doesn't fit, a scoped trim (dedupe base tooling / share
  one JDK / strip caches) or a `virtual_size` bump runs before the merge.
- **A missed `logsearch` reference breaks bring-up** — especially a stranded
  hardcoded `10.50.0.4`, which a name-only grep misses. *Mitigation:* the §5.5
  **two-pronged guardrail** (no `10.50.0.4` literal in runtime paths; no `logsearch`
  host/group reference in topology/inventory consumers); the cold-rebuild proves it
  end-to-end.
- **x86 regression.** *Mitigation:* the change is platform-agnostic (obs/logsearch
  are Ubuntu everywhere); parity is re-checked (no x86 regression) before closing.

## 9. Relationships & follow-on

- Implements #266's **consolidation half** and #252's "E". The **mqweb client-mode**
  half of #266 remains a sibling follow-on (its own future epic).
- **Unblocks** epic #249's VAL-A (`#1154`) — once the merged obs node makes observe
  reliable, the arm64 5/5 reproducibility gate can be re-run and closed.
- Deferred (not here): #266 "C" per-JVM startup-cost tuning; #266 "D" further
  right-sizing beyond the combined figure if the cold-rebuild demands it.
