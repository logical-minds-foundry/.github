# Retrospective — logsearch: historical full-text log-analysis tier (epic #149)

Partners [`spec.md`](spec.md) and [`plan.md`](plan.md). Read spec → plan →
retrospective: what we set out to do, how we planned it, and honestly how it went.

## §0 — At a glance

The lab already streamed structured logs to Loki (Alloy → Loki → Grafana). This
epic added a **second, full-text analysis tier**: a dedicated `logsearch` node
(mgmt plane, `10.50.0.4`) running **single-node OpenSearch + OpenSearch Dashboards
+ Data Prepper**, a **stack-agnostic sibling of `obs`**. Alloy stays the sole
collector and gains **one fan-out sink** — the same corpus Loki receives is also
written to OpenSearch (via Data Prepper's OTLP logs source); the Loki path is
untouched. Persistence is **host-side snapshot/restore only** (the guest disk is
ephemeral; `mqlab logsearch snapshot` copies to `build/state/logsearch/` and
bring-up auto-restores). Daily `logs-YYYY.MM.DD` indices, `number_of_replicas:0`
(honest single-node green), security disabled on the mgmt-only plane (v1). **v1
deliberately ships zero seeded content** — saved searches/dashboards are the
forward work (#819).

**Work delivered**

| PR | What it did |
|---|---|
| [#930](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/930) | Foundational spikes (#826) — Alloy→OpenSearch connector feasibility + snapshot round-trip. **Named the Data Prepper fallback that the epic then needed.** |
| [#932](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/932) | Pin OpenSearch + Dashboards 3.8.0 in the shared obs manifest (#829) |
| [#933](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/933) | `opensearch` role — install + configure (#827) |
| [#934](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/934) | `opensearch-dashboards` role + `logs-*` index pattern (#828) |
| [#938](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/938) | Alloy fan-out sink to OpenSearch, gated, default off (#831) |
| [#942](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/942) | `data-prepper` role — the Alloy→OTLP→OpenSearch connector (#939, the mid-flight architecture gap) |
| [#941](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/941) | `mqlab logsearch` CLI — status/open/snapshot/restore (#833) |
| [#949](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/949) | Register + bake the `logsearch-ubuntu2404` box + topology node (#830) |
| [#951](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/951) | `site-logsearch.yml` per-run play + **fleet-wide** fan-out (#832) |
| [#955](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/955) | v1 documentation — tier, CLI, state bucket (#834) |
| [#970](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/970) | Cold-rebuild + snapshot round-trip validation runbook (#969) |
| [#973](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/973) | Doc-review sweep — close doc gaps for the tier (#818) |
| **Fixes the cold rebuild flushed out** | [#956](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/956) (#952), [#961](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/961) (#960), [#968](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/968) (#962); plus obs-side [#958](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/958) (tolerate a missing fan-out gate) |

- **Repos touched:** `mq-resiliency-lab-for-linux` (roles, box, topology, site
  play, CLI, docs, fixes); `.github` (spec/plan, this retrospective).
- **Validation:** #835 (operational cold-rebuild + round-trip) — **SUCCESS after
  fixes**; not PR-workable, closed on its outcome comment; the repeatable runbook
  is [`docs/reference/logsearch-validation-runbook.md`](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/blob/develop/docs/reference/logsearch-validation-runbook.md).
- **Follow-ups spun out:** #971 (fan-out flow diagram), #819 (follow-on brainstorm).

## §1 — How the plan evolved

**The build landed largely as planned.** Roles (opensearch, dashboards),
version pins, the Alloy fan-out block, the box + topology node, the site play, and
the CLI all shipped through clean spec→plan→implement runs. Two design decisions
and one validation dominated the rest.

- **The Data Prepper gap (the plan predated its own spike's answer).** The plan
  assumed Alloy might write to OpenSearch natively. The gating spike (#826) proved
  it **cannot** — Alloy ships only a curated OTel-Collector subset with no
  OpenSearch/Elasticsearch exporter — so the guaranteed fallback, **OpenSearch Data
  Prepper**, was the real connector. There was no plan task for it; we escalated
  the gap, filed **#939**, and folded its install/configure halves into the box
  bake (#830) and the site play (#832), beyond the plan's literal role lists.
- **Fleet-wide fan-out vs. the plan's literal wording.** Plan Task 10 said
  `hosts: logsearch` runs the alloy fan-out — which would index only the logsearch
  node's *own* logs. Spec §6/§13 wanted **"the same source set Loki receives, for
  parity"** — the whole fleet. We took it fleet-wide (a human-approved deviation),
  via a `group_vars/all` gate file (`build/work/logsearch/fanout.json`) that every
  alloy invocation inherits. The gate is **rendered eagerly at plan-build time**,
  before `site-obs.yml` configures the fleet's alloy, so fan-out flows on the first
  pass, survives per-stack observe re-applies without clobbering, and is cleanly
  inert when the file is absent.
- **The cold-rebuild validation (#835) was the load-bearing moment.** Nothing had
  ever baked and operated the tier end-to-end, and the one-pass cold rebuild
  flushed out **three latent bugs that a 100%-covered, HTTP-mocked unit suite could
  not catch**:
  1. **#952** — `build-fatbox.sh` **and** `_manifest-hash.sh` each carry their own
     per-box allowlist that had **drifted from the Python `_LOCAL_BOX_BUILDERS`
     FLEET**. #830 updated the Python side (and the test that guards *it*); the two
     shell lists rejected `logsearch-ubuntu2404` outright — the box was
     **un-bakeable**.
  2. **#960** — the dashboards install ran `opensearch-dashboards-plugin remove`
     during the (root) bake, and the CLI **refuses root without `--allow-root`**.
  3. **#962** — `mqlab logsearch snapshot` generated an **uppercase** snapshot name
     (OpenSearch requires lowercase → 400), and the fs-repo transport ran ansible
     **without `--become`**, so it couldn't read the `opensearch:opensearch 0750`
     repo. Two defects, one command, both invisible to mocked tests.
- **Operational gotchas** (now in the runbook): drive the bake from **`.venv-host`**
  (a bare `.venv/bin/mqlab` has no `ansible-playbook` → exit 127 mid-bake); and
  `vagrant destroy logsearch` can hit a **vagrant-libvirt state desync** (`Name
  … already taken` on a *destroy*), worked around via `virsh`.

## §2 — Lessons learned

- **100% unit coverage with mocked I/O is not end-to-end proof.** Every bug the
  cold rebuild found lived *below* the mock line — a real OpenSearch's name rules,
  real file permissions, a root-refusing CLI, and a shell/Python allowlist that
  only drifts when you actually shell out. The operational validation was not a
  formality; it was the only thing that could catch these.
- **Don't duplicate an allowlist across languages.** The box name lived in one
  Python dict and two shell `case` statements, and the test guarded only the dict.
  We added cross-drift regression tests, but the real fix is a single source of
  truth for the box fleet.
- **A one-pass cold build in the target environment is the acceptance bar** — the
  same lesson #114 learned. "Ships green in CI" and "builds from nothing on the
  real hypervisor" are different claims.
- **Plan against the spike's findings, not ahead of them.** The Data Prepper
  connector was knowable at spike time; the plan's role lists predated it. Catching
  it meant re-reading spec §6 against the plan mid-flight.

## §3 — Compromises & tradeoffs

- **v1 ships zero seeded content** — deliberate. The tier is usable by hand;
  saved searches/dashboards are defined by the forward brainstorm (#819), not
  guessed at now.
- **Security disabled, mgmt-plane only** — per the epic's security-boundary tenet;
  hardening is a #819 topic, not a v1 requirement.
- **Fleet-wide fan-out over the plan's literal node-scoped wording** — a conscious,
  approved deviation to meet the spec's parity intent.
- **The validation was not a clean single pass** — it required three fixes
  mid-flight. That is the validation *working*: after the fixes, the tier builds
  and operates one-pass. Recorded honestly as SUCCESS-after-fixes.
- **The fan-out *flow* is not yet drawn** — `02-inside-lab-vm.html` has no
  telemetry-ingest layer at all, so a lopsided OpenSearch-only arrow would mislead;
  the node was added, the flow deferred to a scoped redesign (#971).

## §4 — New problems & opportunities

- **The three cold-rebuild bugs** (#952/#960/#962) — all fixed and merged.
- **#971** (open) — visualize the Alloy→Loki **and** Alloy→Data Prepper→OpenSearch
  fan-out in the inside-the-lab-VM diagram (a flow-layer redesign).
- **Single-source-of-truth for the box fleet** (opportunity) — collapse the
  Python `_LOCAL_BOX_BUILDERS` and the two shell allowlists so #952-class drift
  can't recur.
- **The vagrant-libvirt `destroy` state desync** — captured as a runbook gotcha
  with a `virsh` workaround; a tooling quirk to watch, not yet owned by a task.

## §5 — What's next

- **#819 — the follow-on brainstorm** (the forward axis, to run collaboratively):
  a query/investigation catalog (channel-retry frequency, recurrence, time-of-day
  patterns) as saved searches; obs + log dashboards side-by-side; a security-posture
  review against the boundary tenet; retention & disk-space management modeled on
  baked-image GC; and synthetic-corpus extraction. Each likely spins out its own
  epic — v1 exists precisely so this brainstorm defines what we build on top.
- **#971** — the fan-out flow diagram.
- The tier is live and usable by hand today; everything further is net-new value on
  top of a proven, reproducible base.
