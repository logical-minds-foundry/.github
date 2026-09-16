# Observability stack version-currency refresh — design spec

- **Epic:** `logical-minds-foundry/.github#236`
- **Design task:** `logical-minds-foundry/.github#237`
- **Doc-review bookend:** `logical-minds-foundry/mq-resiliency-lab-for-linux#1122`
- **Retrospective (terminal):** `logical-minds-foundry/.github#238`
- **Design seed:** the MQ 10 upgrade epic (`#219`) explicitly scoped
  "Observability/toolchain version drift" **out**, noted as a follow-on. This
  epic is that follow-on.
- **Immediate trigger:** the `mq-metric-samples` v5.7.1 → v6.0.0 bump (#1120) —
  the old exporter predated MQ 10 and hung on collection — which surfaced that
  several sibling components are pinned to old or placeholder versions.
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-09-16

## 1. Problem & motivation

The MQ-10.0.0.0 rebaseline is done and the Prometheus exporter is current, but
completing it exposed the rest of the observability stack: several components
are pinned to old or placeholder versions, two roles literally carry the comment
`confirm current stable in Task 11`, and the shared manifest has **already
drifted** from the role defaults it is meant to mirror. "What version of the obs
stack are we on, and is it current?" has no confident answer today.

This epic answers it: research every component against upstream (done — §3),
land the low-risk upgrades, reconcile the drifted manifest, and gate the one
breaking change (Prometheus 2.x → 3.x) behind an explicit decision.

**Out of scope, and deliberately so:** IBM MQ (10.0.0.0, current — `lab/mq-version`)
and the `mq-metric-samples` exporter (`v6.0.0`, just bumped). Do not touch either.

## 2. Doctrine & principles

- **Currency is hygiene, not novelty.** We move to *current stable*, not to
  bleeding edge. Where "current stable" and "current LTS" differ, LTS is the
  conservative target and the choice is called out, not defaulted silently.
- **Two sources of truth move together or not at all.** Every pinned component
  lives in **both** its Ansible role default *and*
  `manifests/_shared/observability.yaml`. A bump edits both in the same change;
  a bump that touches one is a bug. Reconciling the *existing* drift
  (`mq_metric_samples_ref: "master"` vs the role's `v6.0.0`; `grafana: ""`) is
  part of this epic's hygiene, not a separate concern.
  - **The role default is bake-authoritative.** `lab/boxes/build-fatbox.sh`
    bakes from the role defaults and does **not** read the manifest — the obs
    overlay (`_obs_manifest_args`) is injected only at *bring-up*
    (`site-obs`/`site-logsearch`, the configure half, which never reinstalls) and
    for `gather-versions` reporting. So editing only the manifest bumps nothing
    that ships in the box; **Grafana's bake-authoritative pin lives in
    `group_vars/all/versions.yml` (`grafana_version`), not the manifest.** A
    guardrail test (added by the plan) asserts role↔manifest agreement so the
    mirror can never silently lie about the baked version — the exact failure
    that let `mq_metric_samples_ref` drift to `master` while the box built
    `v6.0.0`.
- **Data vs. judgment stays separated.** Every version/date/asset fact in §3 is
  attributable to an upstream source; the upgrade/hold calls on top are labelled
  judgment. A reviewer can re-verify the data without trusting the judgment.
- **Cold rebuild is the acceptance gate** (per the repo's cold-rebuild acceptance
  doctrine): lint-green ≠ done. A bump is accepted only after a full cold rebuild
  proves the box bakes and boots one-pass on the target arch.
- **Isolate the risky change.** Low-risk same-major bumps ride one validation
  cycle; the one major-boundary change (Prometheus) is isolated behind its own
  spike, decision, and cold rebuild so a failure is unambiguously attributable.
- **arm64 is a first-class build target,** not an afterthought — the primary host
  is Apple silicon. Every target version's arm64 artifact was verified to exist
  (§3) before it was recommended.
- **No `uv run` in runtime.** `uv` stays a dev-loop build tool; nothing in the
  provisioning/runtime path invokes it. (Not expected to bite here — these are
  Ansible role-default edits — but it constrains any helper tooling.)

## 3. Research findings (2026-09-16)

All facts verified against upstream this date: GitHub Releases (pages + API),
`prometheus.io`, `grafana.com`, `apt.grafana.com`, `opensearch.org`, and live
artifact HEAD checks against the exact URL patterns the roles use. Every target
version's arm64 **and** x86_64/x64 artifact was confirmed present.

### 3.1 Summary matrix

| Component | Pin | Latest stable (date) | Boundary | arm64 / x86 | Recommendation |
|---|---|---|---|---|---|
| node_exporter | 1.8.2 | **1.12.1** (2026-07-14) | minor only | ✓ / ✓ | **Upgrade** — low risk |
| Grafana Alloy | 1.3.1 | **1.19.2** (2026-08-26) | 1.x→1.x (no 2.0) | ✓ / ✓ | **Upgrade** — log-path unaffected |
| Loki + logcli | 3.1.1 | **3.7.7** (2026-08-27) | 3.x→3.x | ✓ / ✓ | **Upgrade** (lockstep) |
| Grafana | unpinned | **13.2.2** (2026-09-15) | already on 13.x | ✓ / ✓ | **Pin** + `apt-mark hold` |
| Prometheus | 2.53.2 | **3.14.0** (2026-08-17); 3.13.3 LTS | **MAJOR 2→3** | ✓ / ✓ | **Gated** — breaking |
| OpenSearch | 3.8.0 | **3.8.0** (2026-08-05) | — | ✓ / ✓ | **HOLD** — already latest |
| OS Dashboards | 3.8.0 | **3.8.0** (2026-09-01) | — | ✓ / ✓ | **HOLD** — already latest |
| Data Prepper | 2.16.0 | **2.16.0** (2026-07-02) | — | ✓ / ✓ | **HOLD** — already latest |

### 3.2 Per-component detail

**node_exporter — 1.8.2 → 1.12.1.** *Data:* latest stable 1.12.1, 2026-07-14
([releases][ne-rel]); still a 1.x project, no 2.x exists — the pin→latest move is
minor/patch across 1.9–1.12. arm64 + amd64 tarballs present at the role's URL
pattern. *Judgment:* **upgrade, low risk.** Watch-item is default-collector /
metric-rename deltas across four minors — a smoke-test of the node dashboards
after the bump covers it. Exact collector deltas 1.8→1.12 not enumerated in this
pass ([CHANGELOG][ne-cl] if wanted).

**Grafana Alloy — 1.3.1 → 1.19.2.** *Data:* latest stable 1.19.2, 2026-08-26
([releases][al-rel], [release-notes][al-rn]); same major (no 2.0). Breaking items
since 1.3.1 are all off our path: v1.5 removed `otelcol.exporter.logging` (a
*debug* exporter, not our OTLP fan-out); v1.11 bumped Alloy's embedded Prometheus
engine v2→v3 (irrelevant to a log-shipper role); v1.6 kafka-topic and v1.12
exporter-label changes we don't use. River / Alloy config syntax unchanged across
1.x. arm64 + amd64 zips present (HTTP 200). *Judgment:* **upgrade.** Our use —
journald/file tail, `loki.write`, `otelcol.exporter.otlp` (gRPC) to Data Prepper —
is unaffected. Smoke-test the OTLP→Data Prepper fan-out once, since it crosses a
large version gap.

**Loki + logcli — 3.1.1 → 3.7.7.** *Data:* latest stable 3.7.7, 2026-08-27
([releases][lo-rel], [upgrade guide][lo-up]); same major. The big 3.0 break
(structured-metadata-by-default → TSDB + schema v13) is already behind our 3.1.1
pin. Post-3.1 items: 3.3.0 changed bloom-block format incompatibly (delete blocks
first — but bloom filters are off by default and unused here); 3.6.0 → AWS SDK v2
(S3 config unchanged, N/A to a filesystem lab); 3.5.8 removed busybox from the
Docker image (N/A to a linux-zip single-binary install). No push-API/LogQL wire
break, no logcli flag removal. arm64 + amd64 present for both loki and logcli.
*Judgment:* **upgrade, keep the pair lockstepped** as the role already does. No
forced schema migration from a 3.1 base.

**Grafana — unpinned → pin `grafana=13.2.2` + `apt-mark hold`.** *Data:* latest
stable 13.2.2, 2026-09-15 ([releases][gf-rel]); OSS apt package `grafana` served
from `deb https://apt.grafana.com stable main`; 13.2.2 confirmed present in the
repo's `binary-amd64` **and** `binary-arm64` Packages indices. The float already
tracks 13.x, so pinning to 13.2.2 freezes where the box already sits — no major
crossing. *Judgment:* **pin** to 13.2.2 and `apt-mark hold grafana` so a rebake
is reproducible and Grafana can't silently jump a major under the dashboards.
This closes the last floating-version reproducibility gap in the stack.

**Prometheus — 2.53.2 → 3.x (GATED).** *Data:* latest stable 3.14.0, 2026-08-17;
current LTS line 3.13.x, latest patch 3.13.3, 2026-09-07 ([3.14.0][pr-314],
[3.13.0 LTS][pr-313], [3.13.3][pr-3133]). The 2.x line is wound down — last patch
2.53.5, 2025-06-27 ([2.53.5][pr-2535]), no successor — so "hold on 2.x" is a dead
end, not a supported position. Breaking 2→3 ([migration guide][pr-mig]):
remote-write HTTP/2 default flips true→false (`enable_http2: true` to restore);
config keys renamed (`scrape_classic_histograms` → `always_scrape_classic_histograms`);
`le`/`quantile` labels float-normalized (`"1"`→`"1.0"`); PromQL regex `.` now
matches newline and range selectors go left-closed→left-open; scrape now *fails*
on missing/unparseable Content-Type (needs `fallback_scrape_protocol`);
Alertmanager v1 API removed. arm64 + amd64 present at 3.14.0. *Judgment:*
**upgrade is warranted but needs-testing** — the 2.x dead-end makes staying put
the worse option over time. Conservative target is **3.13.3 (LTS)**; current is
3.14.0. This does **not** land in the low-risk wave; it goes through a spike +
go/no-go (§6).

**OpenSearch / Dashboards / Data Prepper — HOLD (no bump).** *Data:* engine 3.8.0
(2026-08-05, [rel][os-rel]), Dashboards 3.8.0 (2026-09-01, [rel][osd-rel]), Data
Prepper 2.16.0 (2026-07-02, [rel][dp-rel]) — **each is already the latest stable
on its line.** The `-min-` core, dashboards bundle, and `-jdk-` Data Prepper
tarballs are all confirmed published for arm64 + x64. The compatibility triangle
(engine 3.8 ↔ dashboards 3.8 ↔ Data Prepper 2.16, current 2.x, writes to OS 3.x)
holds at our exact pins. *Judgment:* **hold.** There is nothing to upgrade *to*;
the next coordinated move is 3.9.0, not yet released (scheduled 2026-09-29), and
it must move as a unit *with* a matching Dashboards 3.9.0 tag. Note the observed
**release-timing skew**: engine 3.8.0 shipped ~4 weeks before Dashboards 3.8.0 —
future coordinated bumps must wait for the Dashboards tag before moving. This
epic records the trio as reviewed-and-current; it files no bump task for it.

## 4. Structural constraints (what shapes sequencing)

Two facts come from the repo, not upstream, and drive §5–§6:

1. **Blast radius is asymmetric.** `node_exporter` + `alloy` are baked into **all
   8 fat boxes**; Prometheus, Grafana, Loki live in `obs-ubuntu2404` only; the
   OpenSearch trio in `logsearch-ubuntu2404` only. *But* a cold rebuild is always
   full/all-boxes, so blast radius governs **change surface**, not rebuild cost —
   the batching question (§6) is only "how many cold-rebuild cycles, what rides
   each."
2. **The arm64 gate has a coverage gap.** The 6 Ubuntu fat boxes build
   arch-native (arm64 on the Apple-silicon host); the 2 RHEL fat boxes are
   **x86_64-only, refused on ARM** (design D11). Those RHEL boxes *also* bake
   `node_exporter` + `alloy`, so an arm64-only cold rebuild validates the
   all-boxes bumps on the **6 Ubuntu boxes only**. RHEL coverage requires an
   **x86_64 cold rebuild** — hence the added x86 validation for those two bumps
   (§6, decision 3a).

## 5. Scope & work breakdown

**In scope — three buckets:**

- **Wave 1 — low-risk currency bumps (execute).** Four bumps, each editing the
  role default **and** the manifest in lockstep:
  - W1a: `node_exporter` 1.8.2 → 1.12.1 *(all 8 boxes)*
  - W1b: Grafana Alloy 1.3.1 → 1.19.2 *(all 8 boxes)* — verify OTLP→Data Prepper
  - W1c: Loki + logcli 3.1.1 → 3.7.7 *(obs box)* — keep the pair equal
  - W1d: Grafana pin `13.2.2` + `apt-mark hold` *(obs box)*
- **Wave 2 — manifest hygiene (execute).** Reconcile the drift the research
  surfaced: `mq_metric_samples_ref: "master"` → `v6.0.0`; `grafana: ""` → `13.2.2`;
  retire the two `confirm current stable` comments now that they're confirmed.
  Folds naturally into the Wave-1 edits that touch the same manifest keys.
- **Wave 3 — Prometheus 3.x (gated).** Spike → decision → (gated) execute (§6).

**Out of scope:** IBM MQ; the `mq-metric-samples` exporter; the OpenSearch trio
(HOLD — recorded, not bumped); any dashboard/query re-authoring beyond what a
version bump forces; introducing new components.

## 6. Sequencing, validation & the Prometheus gate

**Sequencing.** Wave 1 + Wave 2 first (all low-risk, mutually independent, can be
one PR each or grouped by box); Wave 3 last, isolated. The OpenSearch HOLD is a
documentation line, not a work item.

**Validation batching (decision 2a).** Because a cold rebuild is always
all-boxes, running one per trivial bump is wasted wall-clock. Therefore:

- **One arm64 cold-rebuild** validates the **entire Wave 1 + Wave 2** low-risk
  set. Low-risk failures are cheap to attribute (which service misbehaves points
  at which bump); if bisect is ever needed the cost is bounded.
- **One x86_64 cold-rebuild on the cloud host** (`n2-standard-16`; decision 3a),
  filed as its **own validation task blocked on W1a/W1b**, exercises the RHEL
  boxes' bake of the all-boxes bumps and is what actually closes the §4.2
  coverage gap. It gates the epic alongside the arm64 rebuild: the RHEL boxes are
  x86_64-only (refused on ARM), so the cloud host is the only venue that can
  prove them — naming the venue and giving it an owning task is what stops "we'll
  rebuild the cloud lab sometime" from quietly dropping RHEL coverage.
- **A separate cold rebuild** isolates Prometheus 3.x if it is greenlit, so the
  one risky change never shares a validation cycle with the safe ones.

**The Prometheus gate (Wave 3).** Structured as spike → go/no-go → gated execute:

1. **Spike** the config migration against the lab's actual `prometheus.yml` and
   scrape rules — the remote-write HTTP/2 flip, renamed keys, PromQL
   range-selector/regex semantics, and the scrape Content-Type failure mode.
   Decide target: **3.13.3 (LTS, conservative)** vs **3.14.0 (current)**.
2. **Go/no-go** recorded on the epic: is 3.x worth it now, and at which target?
   (The 2.x dead-end is input, not a forcing function for *this cycle*.)
3. **On go:** execute the bump (role default + manifest), migrate the config, and
   validate on its own cold rebuild.
4. **On no-go:** record the decision and defer; the low-risk wave still ships.

## 7. Acceptance criteria

- All four Wave-1 bumps landed with **both** sources of truth updated in lockstep;
  the Wave-2 manifest drift reconciled; the two `confirm current stable` comments
  retired.
- One **arm64 cold rebuild** green across the affected boxes (local,
  Apple-silicon host), and a **distinct x86_64 cold-rebuild validation task**
  green on the cloud host (`n2-standard-16`), blocked on the all-boxes bumps
  (W1a/W1b) — this is what closes RHEL `node_exporter`/`alloy` coverage. Both
  gate the epic.
- **Functional attestation post-bump, recorded as a SUCCESS/FAILURE comment on
  the validation task (no silent success):** the OTLP→Data Prepper path proven by
  a known log line appearing in OpenSearch Discover / via `logcli`, and
  node_exporter proven by a `node_*` series present in Prometheus plus one node
  panel rendering. The plan's validation-task scaffold operationalizes the exact
  probes; a green cold-rebuild boot alone is **not** sufficient evidence for
  these two.
- Prometheus 3.x: a recorded go/no-go decision. On go, the bump landed and
  validated on its own cold rebuild; on no-go, the deferral recorded with
  rationale.
- OpenSearch trio recorded as reviewed-and-current (next move 3.9.0, post
  2026-09-29).
- Human-facing docs reconciled (bookend #1122): the box-model/bake docs and any
  version references reflect the new pins.

## 8. Risks & mitigations

- **Batched low-risk failure is ambiguous.** *Mitigation:* low prior probability
  (all same-major, arm64+x86 assets pre-verified); service-level symptoms map to
  the responsible bump; bisect cost is bounded to four candidates.
- **arm64/RHEL split hides an x86-only break.** *Mitigation:* the explicit x86_64
  validation (3a) for the all-boxes bumps.
- **Prometheus 3.x config break slips to runtime.** *Mitigation:* the change is
  gated behind a dedicated spike + its own cold rebuild; never batched.
- **Manifest/role drift reintroduced.** *Mitigation:* doctrine — every bump edits
  both; Wave 2 also removes the stale comments that invited drift.
- **Upstream moves during the epic** (e.g. OpenSearch 3.9.0 ships 2026-09-29).
  *Mitigation:* re-verify latest-stable at execution time; the HOLD is a
  point-in-time record, not a standing claim.

## 9. Follow-on

- OpenSearch 3.9.0 coordinated bump once it and a matching Dashboards 3.9.0 tag
  ship (post 2026-09-29) — a natural next currency pass.
- If Prometheus 3.x is deferred here, it returns as its own scoped work before
  the 2.x line's age becomes a security liability.

<!-- source links -->
[ne-rel]: https://github.com/prometheus/node_exporter/releases/tag/v1.12.1
[ne-cl]: https://github.com/prometheus/node_exporter/blob/master/CHANGELOG.md
[al-rel]: https://github.com/grafana/alloy/releases
[al-rn]: https://grafana.com/docs/alloy/latest/release-notes/
[lo-rel]: https://github.com/grafana/loki/releases
[lo-up]: https://grafana.com/docs/loki/latest/setup/upgrade/
[gf-rel]: https://github.com/grafana/grafana/releases
[pr-314]: https://github.com/prometheus/prometheus/releases/tag/v3.14.0
[pr-313]: https://github.com/prometheus/prometheus/releases/tag/v3.13.0
[pr-3133]: https://github.com/prometheus/prometheus/releases/tag/v3.13.3
[pr-2535]: https://github.com/prometheus/prometheus/releases/tag/v2.53.5
[pr-mig]: https://prometheus.io/docs/prometheus/latest/migration/
[os-rel]: https://github.com/opensearch-project/OpenSearch/releases/tag/3.8.0
[osd-rel]: https://github.com/opensearch-project/OpenSearch-Dashboards/releases/tag/3.8.0
[dp-rel]: https://github.com/opensearch-project/data-prepper/releases/tag/2.16.0
