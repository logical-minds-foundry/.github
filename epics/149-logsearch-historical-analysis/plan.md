# logsearch — historical log-analysis tier — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a single-node OpenSearch + OpenSearch Dashboards tier (`logsearch`) that ingests the lab's existing structured-log corpus via an Alloy fan-out sink, is usable by hand for historical full-text investigation, and persists across rebuilds via host-side snapshot/restore.

**Architecture:** A new dedicated `logsearch` node (baked fat box, mgmt plane) is a stack-agnostic sibling of `obs`. Alloy stays the sole collector and gains one fan-out sink to OpenSearch; the Loki path is untouched. The live corpus runs on the guest's ephemeral disk; `mqlab logsearch snapshot` copies it to `build/state/logsearch/` (host-durable) and bring-up auto-restores the latest snapshot.

**Tech Stack:** OpenSearch + OpenSearch Dashboards (Apache-2.0), Grafana Alloy, Ansible, vagrant-libvirt, Python/Typer (`mqlab`), pytest.

## Global Constraints

- **Epic:** `logical-minds-foundry/.github#149`. Spec: `epics/149-logsearch-historical-analysis/spec.md` (read it — this plan implements it).
- **Single node, instrumentation side.** No A/B, no store HA/DR, no cross-site replication. `number_of_replicas: 0`.
- **No MQ coupling.** The tier indexes whatever Alloy forwards; field mappings derive from the log envelope, not MQ semantics.
- **Persistence = host-side snapshot/restore only.** No keep-on-`destroy` volume, no synced folder (both ruled out by #386/#397 and `lab/Vagrantfile:26`). Live data on the guest ephemeral disk; durable copy under `build/state/logsearch/` resolved via `mqlab build path state` — never a hardcoded `build/<X>` path.
- **Security:** mgmt-plane-only; admin credential runtime-injected (`OPENSEARCH_ADMIN_*` / OpenSearch initial-admin-password), never committed. No TLS/RBAC in v1.
- **Ansible, not shell**, for provisioning. Roles split along the `loki` bake/configure line (`tasks/install.yml` + `tasks/configure.yml` + `tasks/main.yml`).
- **No `uv run` in any runtime path.** Companion tools invoked by bare name via `$PATH`.
- **Layered error vocabulary:** `mqlab` messages name fully-qualified `mqlab` commands; OpenSearch tooling speaks for itself.
- **Validation:** `vrg-container-run -- vrg-validate` is the only validation command. Provisioning changes are accepted only after a full cold rebuild (Task 12).
- **Commits:** conventional commits via `vrg-commit --type <t> --scope <s> --message <m>`; work on the task's feature branch.

---

### Task 1: Connector spike — Alloy → OpenSearch (GATING)

De-risk the one load-bearing assumption before building anything on top of it. This gates Tasks 9–10.

**Files:**

- Create: `docs/reports/2026-07-29-alloy-opensearch-connector-spike.md` (findings)

**Interfaces:**

- Produces: the chosen connector (`otelcol.exporter.*` native **or** Data Prepper fallback) and the exact Alloy config block that ships journald lines into OpenSearch — consumed by Task 9.

- [ ] **Step 1: Enumerate the native path.** On a scratch host with Alloy installed (or the `alloy` role's binary), check whether the build exposes an Elasticsearch/OpenSearch exporter: `alloy tools ...` / the component reference, and grep the embedded component list. Record present/absent.
- [ ] **Step 2: Stand up a throwaway OpenSearch.** `docker run` (or a scratch VM) a single-node OpenSearch with security disabled for the spike only. Confirm `curl -s localhost:9200/_cluster/health` returns.
- [ ] **Step 3: Prove the winning path end-to-end.** Either (a) native: an `otelcol.receiver.*` → `otelcol.exporter.elasticsearch` Alloy pipeline, or (b) fallback: Alloy OTLP → OpenSearch **Data Prepper** → OpenSearch. Feed it 3 synthetic journald-shaped JSON lines; confirm they are searchable: `curl -s localhost:9200/logs-*/_search?q=<token>` returns the docs.
- [ ] **Step 4: Record the decision.** Write the findings doc: chosen path, the exact config block, and whether a second component (Data Prepper/Fluent Bit) lands on the `logsearch` node. If native is absent, Data Prepper is the committed fallback (spec §6).
- [ ] **Step 5: Commit.** `vrg-commit --type docs --scope logsearch --message "connector spike: Alloy -> OpenSearch decision (#149)"`

---

### Task 2: Snapshot mechanism spike (GATING for Task 4/11)

Prove the host-side snapshot/restore round-trip before wiring it into roles and the CLI.

**Files:**

- Create: `docs/reports/2026-07-29-opensearch-snapshot-spike.md`

**Interfaces:**

- Produces: the chosen snapshot mechanism (native `_snapshot` filesystem repo **or** cold-copy of a stopped `path.data`) and the exact guest→host transport (Ansible `fetch`/`synchronize`) — consumed by Tasks 4 and 11.

- [ ] **Step 1: Register a filesystem snapshot repo** on the throwaway OpenSearch (`path.repo` in `opensearch.yml`, then `PUT _snapshot/logsearch-fs`). Index 5 docs.
- [ ] **Step 2: Snapshot + wipe + restore (native path).** `PUT _snapshot/logsearch-fs/snap-1?wait_for_completion=true`, delete the index, `POST _snapshot/logsearch-fs/snap-1/_restore`, confirm the 5 docs return.
- [ ] **Step 3: Cold-copy alternative.** Stop OpenSearch, `tar` `path.data`, start, delete index, stop, untar, start, confirm docs return. Compare effort/robustness with Step 2.
- [ ] **Step 4: Prove host transport.** From an Ansible ad-hoc run, `fetch` the repo/tarball from the guest to a scratch `build/state/logsearch/` path and copy it back — confirm the bytes round-trip. Confirm the target path comes from `mqlab build path state`.
- [ ] **Step 5: Record the decision** (which mechanism, exact steps, restore-on-bring-up sequence) and **commit** (`--type docs --scope logsearch`).

---

### Task 3: `opensearch` role — install half

**Files:**

- Create: `ansible/roles/opensearch/tasks/main.yml`, `tasks/install.yml`, `defaults/main.yml`, `templates/opensearch.yml.j2`, `handlers/main.yml`
- Reference pattern: `ansible/roles/loki/tasks/{main,install}.yml`

**Interfaces:**

- Produces: an installed-but-inert OpenSearch (`systemctl is-enabled opensearch` → disabled after bake) with `path.data` on the guest disk, `vm.max_map_count=262144`, and a static `opensearch.yml`. Consumed by Task 4 (configure), Task 7 (bake), Task 10 (site play).

- [ ] **Step 1: `main.yml`** mirrors loki — `import_tasks: install.yml` then `import_tasks: configure.yml` (configure added in Task 4; create the file now importing only install, add the configure import in Task 4).
- [ ] **Step 2: `defaults/main.yml`** — `opensearch_version` (pinned; confirm current stable in the spike), arch-resolved download/apt source, `opensearch_data_dir: /var/lib/opensearch`, `opensearch_heap: 2g` (tunable), `opensearch_http_port: 9200`.
- [ ] **Step 3: `install.yml`** — set `vm.max_map_count=262144` via `ansible.posix.sysctl` (persistent); install the OpenSearch package/binary at the pinned version; create the `opensearch` user + `path.data`; drop `templates/opensearch.yml.j2`; install the systemd unit **without** enabling/starting it (inert bake). Set `OPENSEARCH_INITIAL_ADMIN_PASSWORD` only in the per-run configure half, never here.
- [ ] **Step 4: `opensearch.yml.j2`** — `cluster.name`, `node.name`, `network.host` (mgmt IP), `discovery.type: single-node`, `path.data`, `path.repo: /var/lib/opensearch/snapshots` (the snapshot repo dir).
- [ ] **Step 5: Lint + commit.** `vrg-container-run -- vrg-validate`; `vrg-commit --type feat --scope logsearch --message "opensearch role: install half (#149)"`

---

### Task 4: `opensearch` role — configure half (start + index template + snapshot/restore)

**Files:**

- Create: `ansible/roles/opensearch/tasks/configure.yml`, `templates/index-template.json.j2`
- Modify: `ansible/roles/opensearch/tasks/main.yml` (add `import_tasks: configure.yml`)

**Interfaces:**

- Consumes: Task 2's snapshot mechanism decision.
- Produces: the `OPENSEARCH_ADMIN_*` credential seam (via `build/state/secrets/`); a running OpenSearch (green on single node); the `logs-*` index template with `number_of_replicas: 0` + daily-index pattern; a registered snapshot repository; the **restore-on-bring-up** behavior. Consumed by Task 5 (reuses the credential seam), Task 10, Task 11, Task 12.

- [ ] **Step 1: Establish the credential source (§10).** Source/generate the `OPENSEARCH_ADMIN_*` credential via the **existing** lab secret mechanism — `lab/scripts/lab-secret.sh` staging under `build/state/secrets/` (host-durable, gitignored, never committed) — the same runtime seam grafana's admin password uses. Fail loud if the secret cannot be materialized; never fall back to a default password.
- [ ] **Step 2: Enable + start** the `opensearch` unit with `OPENSEARCH_INITIAL_ADMIN_PASSWORD` from Step 1's seam. Wait for `_cluster/health` reachable (authenticated with the injected credential).
- [ ] **Step 3: `index-template.json.j2`** — `index_patterns: ["logs-*"]`, `settings.number_of_replicas: 0`, mappings for the log envelope (timestamp, level/severity, host, unit, message). `PUT _index_template/logs` (idempotent).
- [ ] **Step 4: Register the snapshot repo** (`PUT _snapshot/logsearch-fs` with `type: fs`, `settings.location: {{ path.repo }}`), per Task 2.
- [ ] **Step 5: Restore-on-bring-up.** If `build/state/logsearch/<latest>` exists on the control host: copy it to the guest repo dir (Ansible `copy`/`synchronize`), then `POST _snapshot/logsearch-fs/<latest>/_restore`. Guarded so a fresh build (no snapshot) is a clean no-op, logged clearly (not a silent skip).
- [ ] **Step 6: Lint + commit** (`--type feat --scope logsearch`).

---

### Task 5: `opensearch-dashboards` role (install + configure)

**Files:**

- Create: `ansible/roles/opensearch-dashboards/tasks/{main,install,configure}.yml`, `defaults/main.yml`, `templates/opensearch_dashboards.yml.j2`, `handlers/main.yml`

**Interfaces:**

- Consumes: the running OpenSearch from Task 4 (`opensearch.hosts`); the `OPENSEARCH_ADMIN_*` credential seam from Task 4 Step 1.
- Produces: OpenSearch Dashboards reachable on the mgmt plane at a known port (default 5601), pointed at the local OpenSearch, **with a default `logs-*` index pattern** so Discover works out of the box. Consumed by Task 11 (`open`), Task 12.

- [ ] **Step 1: `install.yml`** — install the Dashboards package at the pinned version (Task 6); drop `opensearch_dashboards.yml.j2` (`server.host` = mgmt IP, `opensearch.hosts` = local OpenSearch, `server.port`); install the unit inert.
- [ ] **Step 2: `configure.yml`** — enable + start; inject the OpenSearch credential from the Task 4 seam (`build/state/secrets/`); wait for `/api/status` green.
- [ ] **Step 3: Create the default `logs-*` index pattern** (PLUMBING, not seeded content — spec §2 non-goal covers seeded *queries/saved-searches/dashboards*, not the pattern that makes Discover display the corpus at all). Idempotent `POST` to the Dashboards saved-objects API for an `index-pattern` over `logs-*` with the timestamp field. Without this, Discover opens empty and criterion #3 fails.
- [ ] **Step 4: `main.yml`** imports install then configure.
- [ ] **Step 5: Lint + commit** (`--type feat --scope logsearch`).

---

### Task 6: Pin OpenSearch + Dashboards versions in the shared manifest

**Files:**

- Modify: `src/mqlab/manifest.py` (the shared observability version overlay), the manifest YAML it reads, `ansible/roles/opensearch/defaults/main.yml` + `opensearch-dashboards/defaults/main.yml` (consume the pin)
- Test: `tests/test_manifest.py`

**Interfaces:**

- Consumes: nothing.
- Produces: `opensearch_version` / `opensearch_dashboards_version` available to the roles from the single manifest source, so a re-bake never changes the version under an existing snapshot.

- [ ] **Step 1: Write the failing test.**

```python
def test_observability_manifest_pins_opensearch(tmp_path):
    overlay = load_observability_manifest()   # existing loader in manifest.py
    assert overlay["opensearch_version"]       # present + non-empty
    assert overlay["opensearch_dashboards_version"]
```

- [ ] **Step 2: Run it — expect FAIL** (`KeyError`/missing). `uv run pytest tests/test_manifest.py -k opensearch -v`
- [ ] **Step 3: Add the pins** to the observability manifest YAML and ensure the loader surfaces them.
- [ ] **Step 4: Run — expect PASS.**
- [ ] **Step 5: Wire the roles** to read the pin (defaults reference the manifest-provided value, matching how `loki_version` is threaded). Lint.
- [ ] **Step 6: Commit** (`--type feat --scope logsearch`).

---

### Task 7: Register + bake the `logsearch-ubuntu2404` box

**Files:**

- Modify: `src/mqlab/cli.py` (`_LOCAL_BOX_BUILDERS`, ~line 936)
- Create: `ansible/bake-logsearch.yml`
- Test: `tests/test_box_fleet.py` (or the existing box-fleet test module)

**Interfaces:**

- Consumes: Tasks 3–5 roles (install halves), Task 6 pins.
- Produces: a baked `logsearch-ubuntu2404` fat box (OpenSearch + Dashboards + node-exporter + alloy, inert). Consumed by Task 8 (topology platform).

- [ ] **Step 1: Failing test** — assert the derived `FLEET` includes `logsearch-ubuntu2404` mapped to `build-fatbox.sh`:

```python
def test_fleet_includes_logsearch_box():
    from mqlab.box import FLEET
    assert "logsearch-ubuntu2404" in FLEET
    assert FLEET["logsearch-ubuntu2404"].builder.endswith("build-fatbox.sh")
```

- [ ] **Step 2: Run — expect FAIL.**
- [ ] **Step 3: Add `"logsearch-ubuntu2404": "lab/boxes/build-fatbox.sh"`** to `_LOCAL_BOX_BUILDERS`.
- [ ] **Step 4: Run — expect PASS.**
- [ ] **Step 5: `ansible/bake-logsearch.yml`** — mirror `bake-obs.yml`: `hosts: bake`, include `node-exporter` (full), `alloy` (install), `opensearch` (install), `opensearch-dashboards` (install). No secrets, no per-run renders.
- [ ] **Step 6: Lint + commit** (`--type feat --scope logsearch`). *(Actual box bake is exercised in Task 12's cold rebuild.)*

---

### Task 8: Add the `logsearch` node to topology

**Files:**

- Modify: `lab/topology.yaml` (boxes block + `nodes:`)
- Test: `tests/test_topology_integrity.py` (or a new `tests/test_topology_logsearch.py`)

**Interfaces:**

- Consumes: Task 7's box.
- Produces: a `logsearch` node on the mgmt plane. Consumed by Tasks 9–12.

- [ ] **Step 1: Failing test:**

```python
def test_logsearch_node_present_and_mgmt_only():
    topo = load_topology()          # existing helper used by the other topology tests
    node = topo["nodes"]["logsearch"]
    assert node["platform"] == "logsearch-ubuntu2404"
    assert node["memory"] >= 6144
    assert set(node["nics"]) == {"net-mgmt"}   # instrumentation, mgmt plane only
```

- [ ] **Step 2: Run — expect FAIL (KeyError 'logsearch').**
- [ ] **Step 3: Add** the `logsearch-ubuntu2404` boxes-block entry (mirror the `obs-ubuntu2404` comment/entry) and the node: `platform: logsearch-ubuntu2404`, `cpus: 2`, `memory: 6144`, `nics: { net-mgmt: 10.50.0.4 }` (confirm a free mgmt IP).
- [ ] **Step 4: Run — expect PASS.** Re-run the full topology suite to catch integrity assertions (IP uniqueness, etc.).
- [ ] **Step 5: Commit** (`--type feat --scope logsearch`).

---

### Task 9: Alloy fan-out sink to OpenSearch

**Files:**

- Modify: `ansible/roles/alloy/templates/config.alloy.j2`, `ansible/roles/alloy/defaults/main.yml` (gating var)

**Interfaces:**

- Consumes: Task 1's connector decision; Task 4's running OpenSearch.
- Produces: the same journald/mqweb corpus Loki receives, also written to OpenSearch. Consumed by Task 12.

- [ ] **Step 1: Add a gating default** `alloy_fanout_opensearch: false` and an `opensearch_endpoint` var.
- [ ] **Step 2: Add the sink block** from Task 1 to `config.alloy.j2`, wrapped in `{% if alloy_fanout_opensearch %}` — it re-uses the existing journald/mqweb sources; it does not touch the `loki.write` block.
- [ ] **Step 3: Set the gate** in `site-logsearch.yml` / the alloy configure invocation so only the intended source set fans out (default: the same set Loki receives — spec §13).
- [ ] **Step 4: Lint + commit** (`--type feat --scope logsearch`). *(End-to-end flow asserted in Task 12.)*

---

### Task 10: `site-logsearch.yml` — the per-run configure play

**Files:**

- Create: `ansible/site-logsearch.yml`
- Reference: `ansible/site-obs.yml`

**Interfaces:**

- Consumes: Tasks 3–5, 9.
- Produces: `mqlab`-invokable bring-up of the `logsearch` node (configure halves + alloy fan-out + auto-restore). Consumed by Task 11 (CLI may shell to it) and Task 12.

- [ ] **Step 1:** `hosts: logsearch`, `become: true`, run the configure halves: `opensearch` (start + index template + snapshot repo + restore-on-bring-up), `opensearch-dashboards` (start), `alloy` (configure with `alloy_fanout_opensearch: true`), `node-exporter` (re-apply).
- [ ] **Step 2:** Confirm the play is wired into the lab bring-up ordering (after `obs`, since it is downstream of the same collection).
- [ ] **Step 3: Lint + commit** (`--type feat --scope logsearch`).

---

### Task 11: `mqlab logsearch` CLI

**Files:**

- Create: `src/mqlab/logsearch.py`
- Modify: `src/mqlab/cli.py` (add `logsearch_app`, `app.add_typer(..., name="logsearch")`)
- Test: `tests/test_logsearch_cli.py`

**Interfaces:**

- Consumes: Task 4 (OpenSearch health/API), Task 5 (Dashboards URL), Task 2 (snapshot transport).
- Produces: `mqlab logsearch {status,open,snapshot,restore}`.

- [ ] **Step 1: Failing test — health interpretation (pure).**

```python
def test_single_node_yellow_is_not_failure():
    # green and yellow are both healthy on a single-node replicas:0 cluster;
    # only red / unreachable is a failure.
    assert interpret_health({"status": "green"}) == "healthy"
    assert interpret_health({"status": "yellow"}) == "healthy"
    assert interpret_health({"status": "red"}) == "unhealthy"

def test_latest_snapshot_selection():
    assert latest_snapshot(["snap-1", "snap-3", "snap-2"]) == "snap-3"
```

- [ ] **Step 2: Run — expect FAIL** (`ImportError`). `uv run pytest tests/test_logsearch_cli.py -v`
- [ ] **Step 3: Implement `logsearch.py`** — pure helpers `interpret_health`, `latest_snapshot`, `dashboards_url(host, port)`, plus Typer commands: `status` (query `_cluster/health` + disk-used + read-only-index check → loud on full), `open` (print `dashboards_url`), `snapshot` (invoke the Task 2 mechanism, land under `mqlab build path state`/logsearch), `restore` (latest or `--snapshot`). mqlab-scoped error messages.
- [ ] **Step 4: Wire `cli.py`** — `from mqlab import logsearch` group, `app.add_typer(logsearch.app, name="logsearch")` (mirror `obs_app`).
- [ ] **Step 5: Run — expect PASS.**
- [ ] **Step 6: Commit** (`--type feat --scope logsearch`).

---

### Task 12: Cold-rebuild validation (operational — the epic's `validation` task)

This is the epic's `validation` operational task (filed on GitHub at plan close, blocked-by Tasks 1–11). Not PR-workable; run via `issue-validate`; closes only on `Outcome: SUCCESS`.

**Files:**

- Create: `docs/reference/logsearch-validation-runbook.md`

**Procedure & acceptance:**

- [ ] **Step 1: Cold rebuild** the box + lab from scratch (`vrg-vm rebuild` → box bake → `mqlab vm up` → bring-up incl. `site-logsearch.yml`). One-pass, no manual fixups.
- [ ] **Step 2: Baseline** — `mqlab logsearch status` healthy (green); Dashboards reachable via `mqlab logsearch open`; **doc count rising** (ingestion flowing); a full-text search returns; a generic `date_histogram` aggregation (events-per-hour by severity) returns non-empty.
- [ ] **Step 3: Snapshot round-trip** — `mqlab logsearch snapshot`; `vagrant destroy logsearch && mqlab vm up logsearch`; confirm bring-up **auto-restored** and the corpus is present. Separately confirm a destroy with **no** snapshot returns an empty store (documented tradeoff, not a failure).
- [ ] **Step 4: Loud-not-silent** — confirm `status` reports disk-used and would surface a read-only/full state.
- [ ] **Step 5: Record `Outcome: SUCCESS/FAILURE`** as a comment per the scaffold; write the runbook. Commit the runbook (`--type docs --scope logsearch`).

---

### Task 13: v1 documentation

**Files:**

- Modify: `docs/development/build-layout.md` (add `logsearch/` to the `state/` inventory)
- Create/modify: site docs for the `logsearch` tier + `mqlab logsearch` CLI reference

**Interfaces:**

- Consumes: all prior tasks.
- Produces: the human-facing v1 docs. *(The broader multi-repo sweep is the doc-review bookend `#818`.)*

- [ ] **Step 1:** Add `logsearch/` (host-side snapshot store) to the `state/` bucket row in `build-layout.md`.
- [ ] **Step 2:** Document the `logsearch` tier (architecture: fan-out, single-node, snapshot/restore, boundary tenet) and the `mqlab logsearch` commands in the site docs.
- [ ] **Step 3:** `vrg-container-run -- vrg-validate`; commit (`--type docs --scope logsearch`).

---

## Self-Review

**Spec coverage:**

- §2 criteria 1–6 → Tasks 3–5/7/8 (node+engine), 9 (fan-out), 5 (`logs-*` index pattern so Discover works) + 11 (Discover reachable via `open`; status), 4/11/12 (snapshot/restore), 8/11 (unit tests). ✔
- §4 fan-out / stack-agnostic → Task 9 (re-uses envelope sources). ✔
- §5 node (single, sized, max_map_count, version pin, node-exporter) → Tasks 3/6/7/8. ✔
- §6 connector spike-first + Data Prepper fallback; daily indices + replicas:0 → Tasks 1, 4. ✔
- §7 snapshot/restore, build/state/logsearch via `mqlab build path state`, disk safety → Tasks 2, 4, 11, 12. ✔
- §8 roles (bake/configure split), plays, Ansible transport → Tasks 3–5, 7, 10. ✔
- §9 CLI (status/open/snapshot/restore), layered errors → Task 11. ✔
- §10 mgmt-plane + runtime-injected cred → Task 4 Step 1 (credential seam via `build/state/secrets/`/`lab-secret.sh`), reused by Task 5; Task 8 (mgmt-only NIC). ✔
- §11 cold-rebuild gate + assertions → Task 12. ✔
- §12 follow-ons → out of scope (tracked in #819); §13 open questions → resolved by Tasks 1, 2, 8. ✔

**Placeholder scan:** spikes carry concrete exit criteria (commands + expected output), not "TBD"; code tasks carry real tests. ✔
**Type consistency:** `interpret_health`, `latest_snapshot`, `dashboards_url`, `FLEET`, `load_topology`, `load_observability_manifest` used consistently across tasks. ✔
