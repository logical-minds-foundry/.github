# Epic 326 — Observability patch refresh

**Status:** spec · **Epic:** logical-minds-foundry/.github#326 · **Follows:** epic #236

## 1. Context

Epic #236 brought the observability stack current as of 2026-09-16. Its closing
version check on 2026-10-12 (GitHub Releases API) found that six of the nine
pinned components already had newer upstream releases. This epic is a single
patching pass that brings them current and proves the result with cold builds on
both host platforms.

## 2. Targets

**Data** (queried 2026-10-12):

| Component | Pinned | Target | Change |
|---|---|---|---|
| Prometheus | 3.13.3 | 3.13.4 | LTS patch |
| Grafana Alloy | 1.19.2 | 1.20.1 | minor |
| Loki + logcli | 3.7.7 | 3.7.8 | patch |
| Grafana | 13.2.2 | 13.2.3 | patch |
| OpenSearch | 3.8.0 | 3.9.0 | minor |
| OpenSearch Dashboards | 3.8.0 | 3.9.0 | minor |
| node_exporter | 1.12.1 | — | already latest |
| Data Prepper | 2.16.0 | — | already latest |
| mq-metric-samples | v6.0.0 | — | already latest |

**Decisions:**

- **Prometheus stays on the 3.13 LTS line** (#236's choice). 3.15.0 is newer but
  not LTS.
- **OpenSearch and Dashboards move together to 3.9.0.** #236 recorded this as the
  next coordinated step.
- **Targets are re-checked at implementation time.** A newer patch on the same
  line is taken.

## 3. Approach

- **One implementation task and one PR for all bumps.** In #236, five separate
  PRs edited adjacent lines of `manifests/_shared/observability.yaml` and had to
  be merged one at a time, each needing a hand-resolved conflict.
- **Each bump moves both sources of truth together.** That means the role default
  (for Grafana, `ansible/group_vars/all/versions.yml`) and the manifest, so the T1
  guardrail `tests/test_obs_version_sync.py` stays green. Grafana keeps its apt
  hold.
- **Verify before editing.** Each target's arm64 and amd64 artifacts must
  resolve upstream. Read each release's notes for breaking or config changes.
- **Check OpenSearch 3.9.0 against Data Prepper 2.16.0** (the log sink).

## 4. Validation

- **arm64 macOS cold build**, run locally.
- **x86 cloud cold build**, run in a cloud session, as a parity check.

Both must show: all bootstrap phases green; Prometheus on the new version and
scraping all its targets; Grafana and Loki up; OpenSearch green, with a known log
line arriving through Alloy and Data Prepper.

## 5. Out of scope

Prometheus 3.15 (non-LTS) and any upgrade that needs a config migration. If a
component turns out to need one, it moves to its own task and the rest proceed.

## 6. Risks

- **OpenSearch Dashboards 3.9.0 was released on 2026-10-12, the day this epic was
  planned.** If its arm64 artifact or its compatibility with the 3.9.0 engine
  isn't solid at implementation time, hold the OpenSearch pair at 3.8.0 and ship
  the rest.
- **The Alloy minor bump** (1.19 → 1.20) could change log-pipeline behavior. The
  log-arrival check in both validations covers it.
