# Retrospective — HA-only (no-DR) bootstrap (epic #188)

## §0 At a glance

We set out to add a lighter-footprint bring-up path — `mqlab bootstrap <stack> --no-dr` — that starts only a stack's HA site (skipping the DR/site-B guests and DR provisioning), because under a full lab load the DR guests starve the host (sustained CPU steal) and IBM MQ Native HA cannot hold quorum. We shipped a **stateless, flag-based mechanism** — a `dr_groups` topology marker + effective-member computation threaded through the bootstrap phases, plus `dr_enabled`-gated provision playbooks — wired for **all four HADR stacks**, documented in the versioned site docs, and **proven on real hardware** by a cold-lab acceptance.

### Work delivered

| PR | Repo | What it did |
|---|---|---|
| `.github#194` | `.github` | spec + plan for the epic |
| `#1003` | lab | **Wave 1** — the `--no-dr` mechanism (flag + fail-loud guard, `dr_groups` marker, effective-members, phase threading, provision gates) wired for `nativeha-ubuntu` |
| `#1006` | lab | **Wave 2** — `pcmk-ubuntu` (Option B: drop the DR cluster nodes, keep the SAN/DRBD mirror) |
| `#1004` | lab | **Wave 2** — `rdqm-rhel` (5 site-B touch-points gated) |
| `#1005` | lab | **Wave 2** — `nativeha-rhel` (single DR-import gate) |
| `#1007` | lab | **docs** — document `--no-dr` in getting-started, the operate runbook, and the command reference |
| *(no PR)* | lab | **#996 validation** — cold-lab acceptance, recorded as a SUCCESS comment |

- **Repos touched:** 2 — `logical-minds-foundry/.github`, `logical-minds-foundry/mq-resiliency-lab-for-linux`
- **Sub-issues:** 8 (spec+plan · 4 implementation waves · validation · docs-review · retrospective)
- **PRs merged:** 6, plus 1 validation task (no PR)
- **Releases cut:** none (integration to `develop`)
- **Span:** opened and closed **2026-08-11** — a same-day epic.

## §1 How the plan evolved

The plan's spine held. The mechanism (Wave 1) landed exactly as designed: a stateless boolean threaded from the CLI through `_bootstrap_run` into the phase builders, keyed on a `dr_groups` marker, with `dr_enabled | default(true)` provision gates that keep the default (no-flag) path byte-for-byte identical. Two of the three Wave-2 stacks followed the "repeat the stack-local half" recipe cleanly — `nativeha-rhel` needed a single DR-import gate (its RHEL playbook, unlike the Ubuntu twin, has no authz `:_b` play), and `rdqm-rhel` needed five site-B touch-points gated (the DR import plus several `rdqm_a:rdqm_b` plays), each reasoned through host-pattern-drop vs play-level `when`, with the `qm-create` script's own site-B auto-detection correctly left untouched.

The real deviation was **`pcmk-ubuntu`**. The plan (and the epic's stated approach) assumed every stack "already cleanly splits its HA half from its DR half, so the gates are minimal." That held for the Native HA stacks (site-A is three self-contained raft nodes) but was **false for Pacemaker/SAN**, whose HA storage is a *cross-site DRBD pair*: the HA building block `_pcmk-cluster-ha.yml` — which the plan said must stay ungated — itself touches site-B (the `pcmk_ubuntu` aggregate spans all four groups; a `san_b` DRBD-secondary play; and a `san_a` skip-initial-sync that requires *both* DRBD peers connected, confirmed by the drbd-san role's own note). A faithful "drop all of site B" would have needed new single-peer-DRBD promotion logic in a historically bug-prone area, provable only by a human-operated cold rebuild. Rather than ship that blind, execution **stopped and surfaced the premise error**, and the maintainer chose **Option B**: `dr_groups: [pcmk_b]` drops only the three heavy DR *cluster* nodes and keeps `san_b` up, so the DRBD pair stays two-peer and untouched — a deliberate, recorded deviation from the epic's "no DR replication" criterion, traded for zero storage surgery and the dominant footprint win.

## §2 Lessons learned

- **"Cleanly separable HA/DR" is a per-mechanism property, not a universal one.** Shared-nothing HA (Native HA raft) separates trivially; shared-storage HA (Pacemaker/DRBD) does not, because the HA layer itself spans sites. Any future "do X to the HA site only" feature must be checked against each mechanism's storage topology, not inferred from group naming.
- **Stopping to surface a false premise beat powering through.** The pcmk investigation cost one agent run but avoided shipping a known-broken, UNREACHABLE-on-provision playbook or an unvalidated single-peer-DRBD path. There, the escalation — not code — was the deliverable.
- **A `--limit`-less provision relies entirely on per-play `hosts:` + gates**, and the honest acceptance signal is a PLAY RECAP where the site-B nodes show `ok=0 unreachable=0` (present in inventory, never contacted). That recap, plus the absence of the `_nativeha-ubuntu-dr-replication` play, is what proved "no DR ran."
- **Parallel fan-out with honesty-first subagents works.** Three Wave-2 stacks ran in isolated worktrees; the two mechanical ones finished unattended, and the one with a real design blocker stopped and reported rather than faking a green result.

## §3 Compromises & tradeoffs

- **`pcmk-ubuntu --no-dr` still runs DR replication (Option B).** Keeping `san_b` up means the `san_a → san_b` DRBD mirror keeps replicating — a knowing deviation from "no DR replication." Accepted because the alternative (single-peer DRBD promotion) is new, risky storage code and the footprint win is dominated by the three fat cluster boxes anyway. A "true zero-replication pcmk `--no-dr`" (Option A) is left unbuilt.
- **Re-run idempotency and the satisfied-probes are not `--no-dr`-aware.** Wave 1 scoped `no_dr` into the phase *builders* only, not the satisfied-probes; a re-run of `bootstrap --no-dr` re-runs an idempotent vms/observe pass rather than reporting "already satisfied." Consistent with the stateless design; not worth the larger change.
- **RHEL live acceptance deferred.** `rdqm-rhel` and `nativeha-rhel` are x86-only; their code + unit tests are validated, but the live cold-rebuild acceptance for those two awaits an x86 host.

## §4 New problems & opportunities

Surfaced by the live cold-lab validation (#996) — all filed under **epic #104 (Lab lifecycle reliability)**:

- **#1008** — `commons up` doesn't ensure baked boxes (whereas `bootstrap` does); box-dependent bring-up paths should universally depend on the auto-bake/ensure tooling.
- **#1009** — `bootstrap`/`commons up` exit 5 on the grafana relay-heal step restarting a non-existent `.socket` unit (the `.service` runs fine), returning non-zero after a fully successful bring-up.
- **#1010** — the cold-boot staleness nudge conflates the host-mounted (persistent) `build/state` stamp age with VM age, and its "`vrg-vm rebuild` keeps it fresh" remedy is false.

From the PR flow:

- **`vergil-tooling#2750`** — `vrg-pr-await` hangs forever on an orphaned check-run (terminal-but-`BLOCKED` not detected); a recent GitHub-side change made this recurrent.

Opportunity: the pcmk premise-correction argues for a lightweight "does this stack's HA layer touch site-B?" audit as a precondition for any future site-scoped lab feature.

## §5 What's next

- **Option A — true zero-replication `pcmk-ubuntu --no-dr`** (single-peer DRBD promotion): a candidate follow-on epic if the "no DR replication" criterion becomes load-bearing for Pacemaker/SAN.
- **x86 live acceptances** for `rdqm-rhel` / `nativeha-rhel` when a RHEL host is available.
- The four reliability/tooling issues above are tracked (epic #104; `vergil-tooling#99`) for independent scheduling.
