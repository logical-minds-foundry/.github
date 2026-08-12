# Extract `mq-resiliency-observability` — a standalone, installable observability component — Design

> **Status:** design, first pass — brainstormed 2026-07-15.
> **Date:** 2026-07-15
> **Author:** Phillip Moore (with Claude)
> **Epic:** [logical-minds-foundry/.github#79](https://github.com/logical-minds-foundry/.github/issues/79)
> **Documentation task:** [logical-minds-foundry/.github#80](https://github.com/logical-minds-foundry/.github/issues/80)
> **Parent roadmap:** the parked component-extraction roadmap
> [`docs/specs/2026-06-27-component-extraction-roadmap-design.md`](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/blob/develop/docs/specs/2026-06-27-component-extraction-roadmap-design.md)
> ([#368](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/368)).
> This epic is the **concrete kickoff of that roadmap's component #1** — its
> conventions, six-step migration gate, and metrics contract are inherited here,
> not re-derived.
> **Builds on** the lab observability foundation
> [`docs/specs/2026-06-10-lab-observability-design.md`](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/blob/develop/docs/specs/2026-06-10-lab-observability-design.md)
> (#103), which established Prometheus + Grafana on the `obs` VM, the
> node_exporter fleet, and the textfile-collector pattern this component harvests.

---

## 1. Why this exists

The lab, built at breakneck pace, contains a genuinely valuable piece of software
that is currently **lost inside it**: a set of collectors that turn MQ HA/DR
posture — which IBM exposes only as CLI text (`dspmq`, `rdqmstatus`, `crm_mon`,
`drbdsetup`) — into Prometheus metrics. Nothing off-the-shelf does this. It lives
today as `src/mqlab/*` modules and Ansible roles, entangled with lab-specific
scaffolding, with no independent identity, tests-as-a-unit, or release.

This epic is the **first replay of a repeatable "harden-and-extract" pattern**:
take one ad-hoc lab-grown capability and make it a first-class, fully-tested,
standalone product in a single epic — after which **the lab dogfoods the
published artifact** instead of carrying the code. It pays off three ways:

1. **A public reference / prototype.** A small, comprehensible, well-tested
   component developed fast at home — usable as the frame of reference for a
   from-scratch, by-hand reimplementation in a corporate environment that will
   not consume external OSS for the foreseeable future (and may, someday, consume
   the published artifact directly — engineered for, not assumed).
2. **Lab scalability.** The lab shrinks: it installs a separately-maintained
   component rather than owning its code. Each future extraction shrinks it
   further.
3. **A proving ground for new tooling.** The component must ship as **both** an
   RPM and a Debian package — a capability the Vergil toolkit does not have today.
   Building it here prototypes tooling that a follow-on epic extracts into Vergil
   proper.

## 2. Charter, scope & non-goals

### 2.1 Charter — the non-MQI supplement

The stock IBM `mq_prometheus` exporter owns everything reachable through the **MQI**
(queue depth, channel status, connection counts, queue-manager status). This
component is deliberately the **complement**: it maps **non-MQI operational
state** — state that exists only as **CLI command output** or as **local
filesystem / installation / configuration state** — into Prometheus metrics.

That boundary is *why* the collectors are **stdlib-only** and need no MQI client:
verified, `src/mqlab/nativehastate.py`, `rdqmstate.py`, and `clusterstate.py`
import only the Python standard library (`subprocess`, `xml.etree`, …) and shell
out to CLIs. No PyMQI, no compiled dependency, no MQ client headers. This is the
single most important architectural fact: **the crown-jewel collectors install
anywhere `python3` and the relevant MQ/cluster CLIs are on `PATH`.**

### 2.2 In scope

- **Three HA/DR state collectors** + their systemd timer units:
  - `nativehastate` → `cluster_nha_*` (MQ Native HA, via `dspmq -o nativeha`).
  - `rdqmstate` → `cluster_rdqm_*` (RDQM + DRBD, via `rdqmstatus`/`drbdsetup`).
  - `clusterstate` → `cluster_*` (Pacemaker/Corosync/DRBD/SBD, via
    `crm_mon`/`drbdsetup`).
- **The metrics contract** (§3.3): the emitted `cluster_*` families **and** the
  required scrape-side labels the dashboards depend on.
- **A metadata-driven `render-dashboards` generator** (§3.4) plus the **portable
  dashboards**: the stock-only `qmboard`/`messagingboard` (no collector
  dependency) and the cluster/HA boards (which ride the contract).
- **Dual-format OS packaging** (§3.5): `.rpm` **and** `.deb` from one source
  base, via a format-agnostic core + per-format adapters.

### 2.3 Non-goals

- **Lab-instrumentation collectors** stay in the lab: `netstate`/`net-reach`
  (libvirt-coupled fabric), the `lab_network_health` recording rule, and the
  `app_requester` synthetic-workload textfile. They are scaffolding, not product.
- **The log pipeline** (`alloy`/`loki`/`mq-diag-logging`) — a separate future
  component (`mq-resiliency-logging`, roadmap #368 component #2). Not this epic.
- **The generic event handler** — a separate future epic. **FFST
  *appearance-as-event* belongs there; FFST *count/age metrics* belong here**
  (§6). The seam between event and metric is explicit and deliberate.
- **Stock infrastructure is configured, never re-shipped:** `mq_prometheus`,
  `node_exporter`, Prometheus, and Grafana. The component documents the wiring
  (including the scrape-side label contract); it ships none of these binaries.
- **The MQI / REST path** (PyMQI, `pymqrest`) — out of scope by construction
  (§2.1). A future collector needing MQ internals via the MQI would be a
  different component.
- **Manual-reimplementation ergonomics are not a code design constraint.** We do
  **not** shape the product so a human with no automation can hand-rebuild it.
  That concern is served solely by the out-of-band adopter playbook (§7).

## 3. Architecture

### 3.1 Package shape

One installable package (source: pure-Python-stdlib), delivered as `.rpm` and
`.deb`. It carries multiple entry points on multiple host roles:

- **On MQ / cluster nodes:** the collector modules + their systemd timer units.
- **On the obs / build host:** the `render-dashboards` generator.

One artifact with multiple entry points is fine; whether it later splits into
sub-packages is an implementation detail deferred to the build, not decided here.

### 3.2 Collectors — per-technology modules, fail-loud activation

The package carries a collector **per technology**, activated by **explicit
configuration, with auto-detect as a convenience layered on top**. Explicit
config is the reliable backbone; detection never overrides it.

Detection resolves by **precedence and exclusion**, not by treating the probes
as independent — because they are not. RDQM **bundles its own Pacemaker/DRBD**,
so on an RDQM node both `rdqmstatus` **and** `crm_mon` answer (verified:
`rdqmstate.py` itself parses `crm_mon --one-shot --output-as=xml`). Naive
"any-probe-that-answers" detection would double-activate the generic
`clusterstate` collector on every RDQM node and double-emit overlapping
`cluster_*`/`cluster_drbd_*` series. The resolution order:

1. **RDQM** (`rdqmstatus` present) → `rdqmstate` **only**; explicitly suppress the
   generic `clusterstate` even though `crm_mon` answers (RDQM's Pacemaker is
   internal to it).
2. **Native HA** (`dspmq -o nativeha` succeeds) → `nativehastate`.
3. **Standalone Pacemaker** (`crm_mon` present **and not** RDQM) → `clusterstate`.

**Fail loud only on genuinely unresolvable combinations** — not on RDQM's
legitimate both-probes-answer case, which precedence handles. An unrecognized
stack surfaces a detectable signal and refuses to emit wrong-shaped metrics (the
§8 boundary principle applied to detection).

Collection mechanism (unchanged from the lab, and deliberately conservative):
each collector writes a node_exporter **textfile** `.prom` **atomically**
(temp-file + `rename`) into the textfile directory, driven by a **systemd timer**
(~5 s cadence). node_exporter only *reads* the file, so the privileged producer
and the unprivileged exporter stay cleanly separated. A failed or stale write
never re-publishes a last-known value; each collector emits a
`*_last_write_timestamp` the dashboards alert stale on.

### 3.3 The metrics contract

The contract has two halves, both versioned inside the component:

1. **Emitted metrics** — the `cluster_*`, `cluster_nha_*`, `cluster_rdqm_*`,
   `cluster_drbd_*` families, their labels, and value semantics. Because the
   collectors and the dashboards live in **one repo**, this half is a
   **single-repo consistency invariant**, enforced by a CI test: *every metric a
   panel queries is one a collector emits.*
2. **Required scrape-side labels** — the labels the dashboards' PromQL assumes
   but the collectors do **not** emit: notably the `groups` label (from Ansible
   inventory groups) and host-name patterns. A stock Prometheus will not have
   these unless the operator configures relabeling. This is **the single biggest
   trap for an external adopter** (roadmap #368 §9): the dashboards render
   **empty** against a stock Prometheus. Mitigation: the component ships a
   documented example scrape/relabel snippet, and the README leads with the
   requirement.

The contract pins not only metric *names* but specific resource-name label
*values* — the Pacemaker/DRBD resource names and the QM names used as resource
scopes. The collector that emits a value and the panel that queries it must be
parameterized **together** from one source (§3.4).

### 3.4 The dashboard generator — metadata-driven

`render-dashboards` takes a **profile** — QM names, node/group selectors,
datasource UIDs, network planes, and the Pacemaker/DRBD **resource** names — and
emits Grafana dashboard JSON. It replaces today's hardcoded `lab_*_dashboard()`
entry points (`src/mqlab/{dashboard,clusterboard,messagingboard,qmboard}.py`),
which bake in lab specifics.

**Source-of-truth rule:** the generator derives its profile from the existing
`src/mqlab/stacks.py` source of truth for QM names (`lab_stacks()`; QM names are
derived as `f"{short}APP"` / `f"{short}SVC"`), plus the cluster/DRBD resource
names `stacks.py` does not itself cover — **never a new, competing parameter
source**. In the lab, `stacks.py` feeds the profile; an external adopter supplies
their own profile values. *(Note: the parent roadmap #368 §4.3 names this file
`setups.py`, which does not exist; that stale reference should be corrected in
the member repo — the SOT landed as `stacks.py` via #351/#353.)*

Two dashboard families fall out of this, and they sequence differently (§4):

- **Stock-only boards** (`qmboard`, `messagingboard`) query only `ibmmq_*` — they
  have **no dependency on this component's collectors**. They need only the stock
  exporter scraped and the §3.3 relabeling correct.
- **Cluster/HA boards** (`clusterboard`, and the per-technology HA views) ride
  the `cluster_*` contract and therefore gate on the collectors existing.

### 3.5 Dual-format packaging — a framework, up front

The component must ship **both** `.rpm` (RHEL) and `.deb` (Debian/Ubuntu) from a
single source base. This is required not only by external adopters but by the
lab's **own mixed fleet**: Native HA runs on both `nha-ubuntu` and `nha-rhel`
nodes; RDQM is RHEL-only. The lab cannot dogfood the component on a cold rebuild
without both formats.

**Design the packaging as a format-agnostic core + per-format adapters up
front**, even though slice 1 implements only the RPM adapter:

- A single declarative description of the package (files, install paths, systemd
  units, dependencies, metadata) feeds **format adapters**. The one-to-many shape
  — where the bulk of the complexity lives — is designed from the start, so
  adding a format is adding an adapter, not reworking the core.
- **Extensible to a third format.** RPM + DEB cover 100% of the current use case,
  but the framework must not foreclose a future format.
- **Slice 1 implements the RPM adapter and forks-and-stubs the Debian adapter at
  every divergence** (a conditional plus a `TODO(deb)` marker), so the Debian
  pass slots in rather than forcing a redo. Wherever we do something
  RPM-specific, we mark where Debian will differ.
- **Forward-engineered for extraction.** This packaging capability is prototyped
  here and is expected to be lifted into the Vergil toolkit by a follow-on epic
  (§10). Candidate builders — a single-config multi-format tool (`nfpm`, `fpm`)
  vs. native `rpmbuild` + `dpkg-deb` — and the publish channel are **open
  questions** settled in the packaging sub-brainstorm (§11), not here.

### 3.6 Package lifecycle & the node_exporter textfile boundary

The collectors physically depend on one thing this component does **not** own:
node_exporter's **textfile collector** must read the directory the collectors
write `.prom` files into. If that wiring is wrong, the collectors run happily and
the metrics **silently never appear** — the same failure shape as the §3.3 label
trap. But node_exporter's configuration is *across the ownership boundary* (§8):
it may be a package-owned or config-managed file we cannot safely touch. So this
component **declares-and-verifies; it does not mutate**:

- **The textfile directory is a required, explicit configuration input** — no
  assumed default path. We write into the directory the operator designates as
  the one node_exporter watches.
- **Install-time gate** (RPM `%post` / deb `postinst`): **fail the install
  loudly** if the directory is unspecified or not writable by the collector's
  service user. Never silently proceed against an unusable path.
- **No cross-boundary mutation by default.** We do **not** patch node_exporter's
  config to point it at us. We *may* document, and offer as an explicit,
  consent-gated **opt-in** helper, a way to register a textfile directory — but
  only for operators whose node_exporter config is theirs to patch. Default is
  declare-and-verify.
- **Shared-path caution.** The textfile directory is shared with other collectors
  and packages. We document the required permissions (the lab's `02775` setgid +
  `systemd-tmpfiles` pattern, commit `5496637`) but we do **not** assume we can
  `chmod` a directory we don't own.
- **Runtime self-check backstop.** Each run verifies the directory is writable; on
  failure it emits a loud health signal and a stale `*_last_write_timestamp`,
  never a silent gap.
- The **README integration section** states the boundary plainly: we write where
  you tell us; wiring node_exporter to watch it is the integrator's
  responsibility.

**Clean removal.** Package removal (`rpm -e` / `apt remove`) is a first-class
concern, not deferred: `%preun`/`%postun` + `prerm`/`postrm` **stop and disable
the timers** and **remove the artifacts we own** — our units and the `.prom`
files *we* wrote (stale textfiles left behind would keep feeding last-known
values into Prometheus, a lie). Removal touches **only what we own**: the shared
textfile directory itself and other packages' files are left untouched — the
uninstall mirror of the boundary rule.

## 4. Build order

Every slice runs the roadmap's **six-step migration gate** (#368 §6):

1. **Extract & scrub** — lift the code into the new repo; strip lab assumptions;
   parameterize; define the public interface (README + contract).
2. **Build + CI** — package build; the component's own tests, including the
   contract-consistency test.
3. **Publish** — cut a `0.x` release; semver from day one. *(Cutting a release is
   a human-gated step — the agent never publishes.)*
4. **Repoint the lab (dogfood)** — the lab installs the *published* artifact.
5. **Delete the in-lab copy** — remove `src/mqlab/<module>` / the local role; the
   deletion proves there is no silent fallback or drift.
6. **Cold-rebuild proof** — a full VM cold rebuild passes one-pass. Lint-green ≠
   extracted.

### 4.1 Slice 1 — Native HA, end-to-end

Native HA is the exemplar because `nativehastate.py` is the most self-contained
collector (no cross-collector dependency; RDQM's collector reuses
`clusterstate`'s DRBD parser). Drive it through all six steps, implementing the
**RPM adapter** and **fork-and-stubbing Debian** (§3.5). Slice 1 proves the whole
machinery — extraction, packaging, publish, dogfood, cold rebuild — on real,
high-value code.

### 4.2 Slice 2+

- **RDQM** (RHEL → exercises the **`.rpm`** path in anger) and **Pacemaker/DRBD**,
  behind the auto-detect/config pillar (§3.2).
- The **Debian adapter** completed — "done" requires both `.rpm` **and** `.deb`.
- The **portable dashboards**: stock-only boards first (no collector dependency),
  then the cluster/HA boards on the contract.

### 4.3 Validation — per-package install checks + a final cold rebuild

Validation is **two-tier**, matching cost to purpose:

- **Per-package install validation (fast).** Each published package is proven by
  installing it on a **single ad-hoc VM build** — the `.rpm` on a RHEL VM
  ([#644](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/644)),
  the `.deb` on an Ubuntu VM
  ([#645](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/645))
  — confirming timers active and metrics scraping. **No full lab rebuild is
  needed to validate a package.** Each is blocked-by its format's dogfood tasks.
- **Final full-lab cold rebuild (integration).** One full VM cold rebuild of the
  **entire lab** comes up one-pass with the published component installed across
  the mixed RHEL/Ubuntu fleet — inherently exercising **both** formats in a
  single bring-up. This is the standing cold-rebuild acceptance gate and the last
  thing to close; it is filed as a **new** validation task blocked-by #644, #645,
  and the dashboards work.

## 5. Repository bootstrap & the cross-org tooling dependency

The `mq-resiliency-observability` repo does not exist yet; creating it is
[#82](https://github.com/logical-minds-foundry/.github/issues/82). Repo creation
via `vrg-github-repo-init` is currently interactive (only `repo`/`--adopt`/
`--visibility` are flags; ~10 wizard answers are runtime prompts), so full
automation depends on **exposing every prompt as a CLI flag** —
[`vergil-project/vergil-tooling#2382`](https://github.com/vergil-project/vergil-tooling/issues/2382).

That dependency is **cross-org**, so it is a **loose, prose-level reference, not a
`Blocked-by` link** (native sub-issue links do not cross orgs). The sequence:
land #2382 → create the repo non-interactively → dogfoot the scripted path; if #2382 is not ready when we reach #82, fall back to the interactive wizard with a
prepared answer table.

## 6. Discovery — other CLI-only / filesystem-only MQ metrics

The charter (§2.1) generalizes beyond clustering: **what MQ-level operational
state is CLI-only or filesystem-only and absent from `mq_prometheus`?**

**This epic's §6 work is discovery-only.** Its *implementation* scope stays the
clean extraction of the three existing collectors (the harden-and-extract
thesis); it does **not** build any new collector. The discovery task produces a
written, prioritized candidate list; **building** the top candidate (FFST
count/age) is spun into slice work or a successor epic by the follow-on
brainstorm ([#81](https://github.com/logical-minds-foundry/.github/issues/81)) —
"extract what exists," not "extract + invent." Initial candidates, in rough
priority order:

- **FFST/FDC files** in `/var/mqm/errors` — **count** and **oldest-age** metrics.
  When FFSTs start appearing they arrive in bursts; a count-and-age pair is a
  genuine drive-to-zero signal, rarely reported on. *(The **appearance** of an
  FFST is an event for the separate event-handler epic; the **count/age** are
  metrics for this component — the explicit event/metric seam.)*
- `dspmqtrn` — in-doubt transactions.
- `dspmqver` / `dspmqinst` — installation / patch-level inventory.
- `dmpmqcfg` — configuration drift.
- `dspmqspl` — security-policy state.

This list is expected to **grow post-publish**; the component is built to
increment. We get the architectural footholds now, not the full catalogue.

## 7. The adopter playbook — out-of-band, disposable

A separate deliverable, deliberately **not** part of the product repo: a one-off
markdown that documents a **manual, no-AI, UI-first** rebuild of a dashboard in a
from-scratch corporate Grafana — including a procedure to **test whether JSON
import is available** (and, if not, a UI-only fallback and the ask to raise with
that platform's owners). It is tactical and **disposable** — an artifact the
author handles by hand, not a maintained product doc, and explicitly **not** a
constraint on the product's design (§2.3). It is tracked as a task in this epic
but may be pulled into its own independent brainstorm; it gates nothing.

## 8. Cross-cutting concerns

- **OSS ownership boundary.** The component configures only what it owns and
  **declares-and-loudly-verifies** what it does not. It **never** edits a
  consumer's `qm.ini`, **never** restarts a queue manager, **never** mutates
  node_exporter's config (§3.6), and never assumes the operator's start/stop
  mechanism. In the lab we own everything; an external
  adopter does not, and the code holds that line. A monitored-but-misconfigured
  target surfaces a health signal, never a silent empty panel.
- **Fail loud — no stale panels.** A failed scrape reads as `up == 0` (a red
  tile), never a last-known-good value; collectors emit `*_last_write_timestamp`;
  auto-detect fails loud on ambiguity (§3.2).
- **Secrets.** No IBM binaries, entitlements, keystores, or licenses in the
  published repo; the stock exporter is built-from-source against the MQ redist
  client (documented as wiring, not shipped). The repo's existing secrets policy
  carries forward unchanged.

## 9. Testing & definition of done

- **Pure logic** — the collectors' parse functions, the scrape-target renderer,
  and `render-dashboards` — unit-tested to the repo's **100% branch** bar.
- **Contract-consistency CI test** — every metric a panel queries, a collector
  emits (§3.3, half 1).
- **Packaging** — `ansible-lint` / the pipeline's package-build checks.
- **Detection precedence** — tests that an RDQM-shaped environment activates
  `rdqmstate` and suppresses `clusterstate`, that standalone Pacemaker activates
  `clusterstate`, and that config overrides detection (§3.2).
- **Install / uninstall boundary** — the install-time gate fails loud on an
  unspecified or unwritable textfile directory, and package removal stops+disables
  the timers and removes only our own artifacts (§3.6).
- **Per-package install validation** on an individual VM (fast), plus a **final
  full-lab cold-rebuild** integration proof (§4.3).

A component is **"extracted"** (from #368 §8) only when **all** hold: published
release **+** the lab installs the published artifact **+** the in-lab copy is
deleted **+** a cold rebuild passes one-pass **+** the README documents the
off-the-shelf wiring, including the scrape-side label contract. **License: MIT.
Versioning: semver, starting `0.x`; `1.0` only after the lab dogfoods through a
cold rebuild.** Working repo name `mq-resiliency-observability` is a placeholder;
the final naming pass precedes repo creation but does not block this spec.

## 10. Follow-on (anticipated)

Recorded now, decided at the closing follow-on-brainstorm
([#81](https://github.com/logical-minds-foundry/.github/issues/81)):

- **Extract the dual-format OS-packaging tooling into Vergil proper**
  (`vergil-project`) — the headline follow-on. This epic prototypes it; the
  reusable capability belongs in the toolkit.
- The **generic event handler** epic (home for FFST appearance-as-event).
- **`mq-resiliency-logging`** (roadmap #368 component #2).
- The remaining CLI-only / filesystem-only MQ metrics surfaced by §6.

## 11. Open questions

1. **Auto-detect mechanism** — exact probes and multi-stack / ambiguity handling
   (the fail-loud contract of §3.2).
2. **Packaging tool + publish channel** — `nfpm` vs `fpm` vs native
   `rpmbuild`+`dpkg-deb`; GitHub Releases first vs. yum/apt repos or COPR; and
   the shape of its Vergil extraction. **Its own sub-brainstorm** (§3.5, §10).
3. **`render-dashboards` host packaging** — one package with multiple entry
   points vs. sub-packages (deferred to the build).
4. **Final repo naming pass** — placeholder accepted; naming precedes repo
   creation (#82).

## 12. Cross-epic links

- Parent roadmap:
  [mq-resiliency-lab-for-linux#368](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/issues/368).
- Tooling dependency (cross-org, loose):
  [vergil-project/vergil-tooling#2382](https://github.com/vergil-project/vergil-tooling/issues/2382).
- Future: generic-event-handler epic; `mq-resiliency-logging` component.
