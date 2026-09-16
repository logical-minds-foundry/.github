# Observability stack version-currency refresh — implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: use `vergil:issue-implement` per
> GitHub task (each task below maps to one issue under epic
> `logical-minds-foundry/.github#236`). Steps use checkbox (`- [ ]`) syntax.

**Goal:** Bring the observability stack to current stable — land the low-risk
version bumps, reconcile the drifted manifest behind a guardrail test, and gate
the one breaking change (Prometheus 2.x→3.x) behind a spike and decision.

**Architecture:** Each component is pinned in an Ansible role default (the
**bake-authoritative** literal `build-fatbox.sh` reads) mirrored in
`manifests/_shared/observability.yaml` (consumed at bring-up + `gather-versions`).
A bump edits **both**; a new guardrail test enforces their agreement. Acceptance
is a full cold rebuild — arm64 locally, x86_64 on the cloud host for RHEL-box
coverage — plus attested functional checks. The risky Prometheus major is
isolated: spike → go/no-go → its own bump → its own cold rebuild.

**Tech Stack:** Ansible role defaults, YAML manifest, `mqlab`/`build-fatbox.sh`
bake pipeline, `pytest` (guardrail test), `vrg-container-run -- vrg-validate`.

**Spec:** `epics/236-obs-version-currency/spec.md` (this directory). Read it
alongside this plan.

## Global Constraints

- **Role default is bake-authoritative; manifest mirrors it.** Every bump edits
  the role default (or `group_vars/all/versions.yml` for Grafana) **and**
  `manifests/_shared/observability.yaml`. Editing only the manifest bumps nothing
  baked. (spec §2)
- **arm64-native builds must work** on the Apple-silicon host; every target
  version's arm64 **and** x86_64/x64 artifact was verified present (spec §3).
- **No `uv run` in runtime.** `uv run` is dev-loop only; never in a role, script,
  or unit invoked at provision/runtime. Guardrail test runs under the existing
  `vrg-validate` pytest harness.
- **Do not touch** IBM MQ (`lab/mq-version` = 10.0.0.0) or the exporter
  (`mq_exporter_ref`/`MQ_EXPORTER_REF` = `v6.0.0`).
- **Validation is `vrg-container-run -- vrg-validate`** — the only validation
  command. Cold rebuild is the acceptance gate; lint-green ≠ done.
- **Commit with `vrg-commit --type <t> --scope <s> --message <m>`**; branch per
  issue off `develop` (`feature/<issue>-<slug>`); PRs into `develop`.

## Target versions (from spec §3, upstream-verified 2026-09-16)

| Component | From | To | Bake-authoritative file | Manifest key |
|---|---|---|---|---|
| node_exporter | 1.8.2 | 1.12.1 | `roles/node-exporter/defaults/main.yml` | `node_exporter` |
| Grafana Alloy | 1.3.1 | 1.19.2 | `roles/alloy/defaults/main.yml` | `alloy` |
| Loki (+logcli) | 3.1.1 | 3.7.7 | `roles/loki/defaults/main.yml` (`loki_version` drives both) | `loki` |
| Grafana | float | 13.2.2 | `group_vars/all/versions.yml` (`grafana_version`) | `grafana` |
| Prometheus | 2.53.2 | 3.x (gated) | `roles/prometheus/defaults/main.yml` | `prometheus` |
| OpenSearch trio | — | HOLD (already latest) | — | — |

## Dependency graph

```text
T1 (guardrail test + manifest hygiene)  ← must land first; enforces lockstep
   ├── T2 node_exporter ─┐
   ├── T3 alloy ─────────┤
   ├── T4 loki+logcli ───┼── VAL-A arm64 cold rebuild (blocked-by T1..T5)
   └── T5 grafana pin ───┘
   T2, T3 (all-boxes) ──────── VAL-B x86_64 cold rebuild on cloud host
   T6 prometheus spike → go/no-go
       └─(on go)─ T7 prometheus 3.x bump ── VAL-C prometheus cold rebuild
```

---

### Task 1: Version-sync guardrail test + manifest-drift reconciliation

Lands first: makes the "both sources of truth agree" doctrine executable, then
fixes the one existing divergence. This is TDD in the real sense — the test fails
on the live drift, and reconciling the manifest makes it pass.

**Files:**
- Create: `tests/test_obs_version_sync.py`
- Modify: `manifests/_shared/observability.yaml`
- Reference: `src/mqlab/manifest.py:126` (`_OBS_VAR_MAP`),
  `ansible/roles/*/defaults/main.yml`, `ansible/group_vars/all/versions.yml`,
  `src/mqlab/mqexporter.py` (`MQ_EXPORTER_REF`)

**Interfaces:**
- Consumes: `_OBS_VAR_MAP` (manifest-key → ansible-var map), `MQ_EXPORTER_REF`.
- Produces: `test_role_default_matches_manifest` — the guardrail every later
  bump task relies on staying green.

- [ ] **Step 1: Write the failing test.** Assert each obs component's
  bake-authoritative role default equals its manifest value, and that the
  exporter ref agrees three ways.

```python
# tests/test_obs_version_sync.py
"""Guardrail: the bake-authoritative role defaults and the observability manifest
must agree. build-fatbox.sh bakes from the role defaults; the manifest is the
bring-up + gather-versions mirror. Drift here is a report-vs-reality lie (it let
mq_metric_samples_ref sit at 'master' while the box built v6.0.0). #236."""
import re
from pathlib import Path

import yaml

from mqlab import mqexporter
from mqlab.manifest import _OBS_VAR_MAP, repo_root

ROLE_DEFAULT_FILES = {
    "prometheus_version": "ansible/roles/prometheus/defaults/main.yml",
    "node_exporter_version": "ansible/roles/node-exporter/defaults/main.yml",
    "loki_version": "ansible/roles/loki/defaults/main.yml",
    "alloy_version": "ansible/roles/alloy/defaults/main.yml",
    "grafana_version": "ansible/group_vars/all/versions.yml",
}


def _yaml_scalar(path: Path, key: str) -> str:
    # Role defaults carry inline "# ..." comments after the value; strip them.
    for line in path.read_text().splitlines():
        m = re.match(rf"\s*{re.escape(key)}\s*:\s*(.*)", line)
        if m:
            val = m.group(1).split("#", 1)[0].strip()
            return val.strip('"').strip("'")
    raise AssertionError(f"{key} not found in {path}")


def _manifest() -> dict:
    root = repo_root()
    return yaml.safe_load((root / "manifests/_shared/observability.yaml").read_text())


def test_role_default_matches_manifest():
    root = repo_root()
    manifest = _manifest()
    for var, rel in ROLE_DEFAULT_FILES.items():
        manifest_key = _OBS_VAR_MAP[var]
        role_val = _yaml_scalar(root / rel, var)
        assert role_val == str(manifest[manifest_key]), (
            f"{var}={role_val!r} (role) != {manifest_key}={manifest[manifest_key]!r} (manifest)"
        )


def test_exporter_ref_agrees_three_ways():
    manifest = _manifest()
    role_ref = _yaml_scalar(
        repo_root() / "ansible/roles/mq-exporter/defaults/main.yml", "mq_exporter_ref"
    )
    assert role_ref == mqexporter.MQ_EXPORTER_REF, "role default != MQ_EXPORTER_REF"
    assert manifest["mq_metric_samples_ref"] == mqexporter.MQ_EXPORTER_REF, (
        "manifest mq_metric_samples_ref != MQ_EXPORTER_REF (the 'master' drift)"
    )
```

- [ ] **Step 2: Run it, confirm it fails on the live drift.**

Run: `vrg-container-run -- uv run pytest tests/test_obs_version_sync.py -v`
Expected: `test_exporter_ref_agrees_three_ways` FAILS —
`manifest mq_metric_samples_ref != MQ_EXPORTER_REF` (`master` != `v6.0.0`).
(`test_role_default_matches_manifest` passes: the other five already agree,
including `grafana_version` `""` == manifest `grafana` `""`.)

- [ ] **Step 3: Reconcile the manifest drift.** In
  `manifests/_shared/observability.yaml`: set `mq_metric_samples_ref: "v6.0.0"`
  (retire the `# float today; pin … #172` note — it is now pinned). Refresh the
  OpenSearch trio comment to record the review: `# reviewed 2026-09-16: 3.8.0 is
  latest stable; next coordinated move is 3.9.0 (unreleased, ~2026-09-29)`. Leave
  `grafana: ""` as-is (Task 5 pins it).

- [ ] **Step 4: Run tests, confirm green.**

Run: `vrg-container-run -- uv run pytest tests/test_obs_version_sync.py -v`
Expected: both tests PASS.

- [ ] **Step 5: Full validate.**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS (100% branch coverage — the new test's helpers must all execute;
the two test functions cover both branches of the helper).

- [ ] **Step 6: Commit.**

```bash
vrg-commit --type test --scope obs \
  --message "add obs version-sync guardrail; reconcile mq_metric_samples_ref drift (#236)"
```

---

### Task 2: node_exporter 1.8.2 → 1.12.1 (all 8 boxes)

**Files:**
- Modify: `ansible/roles/node-exporter/defaults/main.yml` (`node_exporter_version`)
- Modify: `manifests/_shared/observability.yaml` (`node_exporter`)

**Interfaces:**
- Consumes: the guardrail test from Task 1 (must stay green).
- Produces: nothing consumed by later code tasks; VAL-A/VAL-B validate the bake.

- [ ] **Step 1: Verify the target artifact resolves for both arches** (fail-loud
  before editing — a bad pin bricks the bake).

```bash
for a in arm64 amd64; do
  curl -sIL -o /dev/null -w "%{http_code} node_exporter %{url_effective}\n" \
    "https://github.com/prometheus/node_exporter/releases/download/v1.12.1/node_exporter-1.12.1.linux-$a.tar.gz"
done
```
Expected: `200` for both.

- [ ] **Step 2: Bump both sources of truth.** Set `node_exporter_version: "1.12.1"`
  in the role default and `node_exporter: "1.12.1"` in the manifest.

- [ ] **Step 3: Guardrail + validate.**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS — `test_role_default_matches_manifest` green (both edited).

- [ ] **Step 4: Commit.**

```bash
vrg-commit --type chore --scope node-exporter \
  --message "bump node_exporter 1.8.2 -> 1.12.1 (#236)"
```

> Bake + boot proof is deferred to VAL-A (arm64) and VAL-B (x86_64, RHEL boxes) —
> node_exporter is baked into all 8 boxes. Functional proof: a `node_*` series
> present in Prometheus + one node panel rendering (recorded on VAL-A).

---

### Task 3: Grafana Alloy 1.3.1 → 1.19.2 (all 8 boxes)

**Files:**
- Modify: `ansible/roles/alloy/defaults/main.yml` (`alloy_version`; retire the
  `# confirm current stable in Task 11` comment)
- Modify: `manifests/_shared/observability.yaml` (`alloy`)

- [ ] **Step 1: Verify the target artifact resolves for both arches.**

```bash
for a in arm64 amd64; do
  curl -sIL -o /dev/null -w "%{http_code} alloy %{url_effective}\n" \
    "https://github.com/grafana/alloy/releases/download/v1.19.2/alloy-linux-$a.zip"
done
```
Expected: `200` for both.

- [ ] **Step 2: Bump both sources of truth** and delete the stale comment.
  Set `alloy_version: "1.19.2"`; `alloy: "1.19.2"` in the manifest.

- [ ] **Step 3: Guardrail + validate.**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

- [ ] **Step 4: Commit.**

```bash
vrg-commit --type chore --scope alloy \
  --message "bump Grafana Alloy 1.3.1 -> 1.19.2; retire confirm-current-stable note (#236)"
```

> Spec §3: log-path components (`loki.source.*`, `loki.write`,
> `otelcol.exporter.otlp`) are unaffected; the removed `otelcol.exporter.logging`
> (v1.5) is a debug exporter we don't use. The 16-minor gap crosses the
> OTLP→Data Prepper fan-out, so VAL-A **must** attest a log line reaching
> OpenSearch/logcli. Alloy is baked into all 8 boxes → VAL-B covers RHEL.

---

### Task 4: Loki + logcli 3.1.1 → 3.7.7 (obs box)

`loki_version` drives **both** the Loki and logcli download URLs — one edit moves
the pair in lockstep (spec §3).

**Files:**
- Modify: `ansible/roles/loki/defaults/main.yml` (`loki_version`; retire the
  `# confirm current stable in Task 11` comment)
- Modify: `manifests/_shared/observability.yaml` (`loki`)

- [ ] **Step 1: Verify BOTH artifacts resolve for both arches** (loki + logcli).

```bash
for pkg in loki logcli; do for a in arm64 amd64; do
  curl -sIL -o /dev/null -w "%{http_code} $pkg %{url_effective}\n" \
    "https://github.com/grafana/loki/releases/download/v3.7.7/$pkg-linux-$a.zip"
done; done
```
Expected: `200` for all four.

- [ ] **Step 2: Bump both sources of truth** and delete the stale comment.
  Set `loki_version: "3.7.7"`; `loki: "3.7.7"` in the manifest.

- [ ] **Step 3: Guardrail + validate.**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

- [ ] **Step 4: Commit.**

```bash
vrg-commit --type chore --scope loki \
  --message "bump Loki + logcli 3.1.1 -> 3.7.7; retire confirm-current-stable note (#236)"
```

> Spec §3: 3.0 schema break (TSDB/v13) is already behind the 3.1 pin; the 3.3.0
> bloom-block break is N/A (bloom off by default; lab is ephemeral/24h). Loki
> lives in the obs box only → covered by VAL-A (arm64), no RHEL exposure.

---

### Task 5: Pin Grafana to 13.2.2 (obs box)

Grafana's bake-authoritative pin is `group_vars/all/versions.yml`
(`grafana_version`), **not** the manifest — the grafana role installs
`grafana={{ grafana_version }}` (`roles/grafana/tasks/install.yml:26`) at bake
time, and the bake never reads the manifest. Pinning only the manifest would
leave the baked Grafana floating.

**Files:**
- Modify: `ansible/group_vars/all/versions.yml` (`grafana_version` `""` → `"13.2.2"`)
- Modify: `manifests/_shared/observability.yaml` (`grafana` `""` → `"13.2.2"`)
- Modify: `ansible/roles/grafana/tasks/install.yml` (add an `apt-mark hold` step —
  defense-in-depth so a later apt action can't drift the frozen box)

- [ ] **Step 1: Verify 13.2.2 is served for both arches in the Grafana apt repo.**

```bash
for a in amd64 arm64; do
  curl -s "https://apt.grafana.com/dists/stable/main/binary-$a/Packages" \
    | grep -q "^Version: 13.2.2" && echo "$a 13.2.2 present" || echo "$a MISSING"
done
```
Expected: both `present`.

- [ ] **Step 2: Pin in both sources of truth.** `grafana_version: "13.2.2"`;
  manifest `grafana: "13.2.2"` (retire the `# apt-float today` note).

- [ ] **Step 3: Add the hold** in `roles/grafana/tasks/install.yml`, immediately
  after the install task:

```yaml
- name: hold grafana at the pinned version (bake stays reproducible) (#236)
  ansible.builtin.dpkg_selections:
    name: grafana
    selection: hold
  when: grafana_version | default('') | length > 0
```

- [ ] **Step 4: Guardrail + validate.**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS — `test_role_default_matches_manifest` green (`grafana_version`
now `13.2.2` == manifest).

- [ ] **Step 5: Commit.**

```bash
vrg-commit --type chore --scope grafana \
  --message "pin Grafana 13.2.2 + apt hold; retire apt-float (#236)"
```

---

### Task 6 (SPIKE): Prometheus 3.x config-migration spike + go/no-go

Not PR-workable code — a spike. Deliverable is a report and a **recorded
decision** (comment on the epic): upgrade to **3.13.3 (LTS)** vs **3.14.0
(current)** vs **defer**. Execute Task 7 only on a "go".

**Files:**
- Read: `ansible/roles/prometheus/templates/prometheus.yml.j2`,
  `ansible/roles/prometheus/files/lab.rules.yml`,
  `ansible/roles/prometheus/tasks/{install,configure}.yml`
- Create: `docs/reports/2026-09-16-prometheus-3x-migration-spike.md`

- [ ] **Step 1: Audit the config against the 2→3 breaking set** (spec §3): in
  `prometheus.yml.j2` — any `remote_write` (HTTP/2 default flip), renamed keys
  (`scrape_classic_histograms`), scrape configs lacking Content-Type handling
  (`fallback_scrape_protocol`); in `lab.rules.yml` — PromQL touched by the regex
  `.`-matches-newline and left-open range-selector changes; check the
  Alertmanager version requirement (≥0.16, v1 API removed).
- [ ] **Step 2: Verify the chosen target's artifacts** resolve for both arches
  (repeat the curl check for 3.13.3 AND 3.14.0 — the research verified 3.14.0
  only; 3.13.3 assets were not individually confirmed).
- [ ] **Step 3: Write the report** — findings, the config diff required, the
  target recommendation with rationale (data vs judgment, sourced).
- [ ] **Step 4: Record the go/no-go** as a comment on
  `logical-minds-foundry/.github#236` with the chosen target (or defer +
  rationale). On defer, Task 7 and VAL-C are not filed; the epic ships the
  low-risk wave and the retrospective records the deferral.

---

### Task 7 (GATED — only on Task 6 = go): Prometheus 2.53.2 → 3.x (obs box)

**Blocked-by:** Task 6 decision = go. **Files:**
- Modify: `ansible/roles/prometheus/defaults/main.yml` (`prometheus_version`)
- Modify: `manifests/_shared/observability.yaml` (`prometheus`)
- Modify: `ansible/roles/prometheus/templates/prometheus.yml.j2` and/or
  `files/lab.rules.yml` per the Task 6 migration diff

- [ ] **Step 1: Verify the chosen target's artifacts** resolve for both arches.
- [ ] **Step 2: Apply the config migration** from Task 6's report.
- [ ] **Step 3: Bump both sources of truth** to the chosen target.
- [ ] **Step 4: Guardrail + validate.** `vrg-container-run -- vrg-validate` → PASS.
- [ ] **Step 5: Commit.**

```bash
vrg-commit --type feat --scope prometheus \
  --message "upgrade Prometheus 2.53.2 -> <target>; migrate config for 3.x (#236)"
```

> Isolated: never batched with the low-risk wave. VAL-C is its dedicated cold
> rebuild. Prometheus lives in the obs box only.

---

## Validation (operational) tasks

Created with `vrg-issue-create --kind validation --blocked-by …` (not
PR-workable; close on an attested `Outcome: SUCCESS` comment). File these when
the blocking impl tasks are filed (epic-create step 9).

- **VAL-A — arm64 cold rebuild (low-risk wave).** Blocked-by T1,T2,T3,T4,T5.
  Full arm64 cold rebuild on the Apple-silicon host (per the cold-rebuild
  acceptance doctrine). **Attested checks (spec §7, no silent success):** (1) the
  OTLP→Data Prepper path — a known log line appears in OpenSearch Discover / via
  `logcli`; (2) node_exporter — a `node_*` series present in Prometheus + one
  node panel renders; (3) Loki/Grafana up on the obs box. SUCCESS only if all
  attested.
- **VAL-B — x86_64 cold rebuild on the cloud host (RHEL coverage).** Blocked-by
  T2,T3 (the all-boxes bumps). Cold rebuild on `n2-standard-16`; the RHEL boxes
  (`mq-rdqm-rhel9`, `mq-nativeha-rhel9`) are x86_64-only and bake node_exporter +
  alloy, so this is the only venue that proves their bake. SUCCESS = both RHEL
  boxes bake + boot with the bumped agents.
- **VAL-C — Prometheus cold rebuild.** Blocked-by T7 (only if Task 6 = go).
  Dedicated cold rebuild after the Prometheus major; attest Prometheus starts,
  scrapes, and the migrated rules load without error.

## Self-review

**Spec coverage:**
- §3 low-risk bumps → T2 (node_exporter), T3 (alloy), T4 (loki+logcli),
  T5 (grafana). ✅
- §5 Wave 2 manifest hygiene → T1 (exporter-ref reconcile + comment refresh). ✅
- §2 lockstep doctrine made enforceable → T1 guardrail test. ✅
- §3/§6 gated Prometheus → T6 spike + go/no-go, T7 gated bump. ✅
- §3 OpenSearch HOLD/record → T1 manifest comment refresh (review recorded). ✅
- §6/§7 validation batching + x86 RHEL coverage + attested checks → VAL-A/B/C. ✅
- §2 Grafana bake-authoritative in versions.yml → T5 (explicit). ✅

**Placeholder scan:** no TBD/TODO; every code/step block is concrete. The one
deliberately-open value is Task 7's `<target>`, resolved by the Task 6 gate — by
design, not a placeholder.

**Type/name consistency:** `test_role_default_matches_manifest` and
`test_exporter_ref_agrees_three_ways` (T1) are the names VAL/impl tasks refer to;
`_OBS_VAR_MAP`, `MQ_EXPORTER_REF`, `repo_root` match `src/mqlab/manifest.py` and
`src/mqlab/mqexporter.py` as read this session. Bake-authoritative file paths
match the spec §3 table and the repo.
