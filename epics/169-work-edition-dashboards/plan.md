# Work-edition Grafana dashboards — implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: use `superpowers:subagent-driven-development`
> or `superpowers:executing-plans` to implement each task. Steps use checkbox (`- [ ]`) syntax.
> This is an **epic-framework** plan: each task below is filed as a GitHub issue under epic
> `logical-minds-foundry/.github#169` (`vrg-issue-create --epic …`), and lands its PR in the
> repo named in the task. The three closing bookends already exist (#170 docs, #931 docs-review,
> #171 retrospective) — do not re-file them.

**Goal:** Ship three portable, variable-driven Grafana boards — QM view, Queue/channel view,
Infra/HA-DR view — developed in the lab and carried to the author's work Grafana by JSON export.

**Architecture:** New work-edition generators (`workqmboard.py`, `workflowboard.py`,
`workinfraboard.py`) reuse the lab's tested panel primitives in `clusterboard.py` (`_ds`,
`_stat`, `_timeseries`, `_logs_panel`, `_state_timeline`, `_row`) but render a **portable**
shape: every datasource/QM/queue/channel is a Grafana **template variable**, never a hardcoded
UID or name. A `--portable` render mode emits import-ready JSON. Each board ships with a
prior-art citation doc; each non-identical datasource ships a **contract** doc.

**Tech Stack:** Python 3.12 board generators (mqlab), Grafana dashboard JSON, PromQL
(`ibmmq_*`), Lucene/Elasticsearch (Wave 1b), pytest (render-contract tests), the lab's existing
`mqlab obs dashboard` render path.

## Global Constraints

(Every task's requirements implicitly include these — copied from the spec.)

- **Portability by contract:** no hardcoded datasource UID, QM name, queue/channel name, or
  undocumented field in any rendered board. Bind to template variables (`$datasource`, `$logs`,
  `$qmgr`, `$queue`, `$channel`, `$level`) and to a written contract. (spec §3.2, §5)
- **Do not disturb the lab's existing object-driven boards** (clusterboard/qmboard/etc.); the
  work edition is a separate generator family. (spec §2, §9)
- **Log/event panels are Elasticsearch-only** (Wave 1b) — no Loki placeholder in work-edition
  boards. (spec §4)
- **Signal-in-the-noise** on every board: status band → trend graphs (not stat tiles) →
  thresholds-encode-error → the Attention (query-filtered, empty=healthy) + collapsible
  Inventory pair → error-severity-default feed. (spec §6)
- **Never ship a silently-empty panel:** an empty panel must mean "healthy," achieved by pushing
  the filter into the query, not by hoping data exists. (spec §6)
- **Prior art is a shipped deliverable:** every non-obvious metric/panel choice cites checkable
  prior art, separating data (what a source says) from judgment. (spec §3.1)
- **Verified metric floor (Phase 0, done):** boards bind only to metrics confirmed live —
  238 `ibmmq_*` names incl. QSTATUS (`oldest_message_age`, `uncommitted_messages`,
  `input/output_handles`) and the EXTENDED queue class. Per-queue **backout** and **in-doubt**
  are NOT stock series → they come from the ES event feed (Wave 1b), never PromQL. (spec §3.2, §7.2)

---

## File structure

- `src/mqlab/workboards.py` (new) — shared work-edition helpers: the `$datasource/$qmgr/...`
  template-variable builders and the portable-render assembler. Kept separate from
  `clusterboard.py` (which stays lab-object-driven) but imports its panel primitives.
- `src/mqlab/workqmboard.py` (new) — QM-view generator.
- `src/mqlab/workflowboard.py` (new) — Queue/channel-view generator.
- `src/mqlab/workinfraboard.py` (new) — Infra/HA-DR generator (Wave 2).
- `src/mqlab/cli.py` (modify) — a `--portable` flag/target on the dashboard render path that
  emits the work editions to `build/work/grafana/work-edition/`.
- `tests/test_workboards.py` (new) — render-contract tests (portability + variable-driven +
  empty=healthy queries), one class per board.
- Contract + citation docs in the epic home repo (`.github`), under
  `epics/169-work-edition-dashboards/contracts/` and `.../citations/`.

**Repo placement (placement law):** generator code + its tests land in
`logical-minds-foundry/mq-resiliency-lab-for-linux` (tasks filed there). Contract/citation
**docs** land in `.github` under the epic dir (tasks filed there).

---

## Wave 1a — QM + Queue/channel metric boards (UNBLOCKED — do now)

### Task 1: Prometheus schema-note contract

**Repo:** `.github` · **Files:** Create `epics/169-work-edition-dashboards/contracts/prometheus-schema-note.md`

**Interfaces — Produces:** the authoritative list of `ibmmq_*` metric names + label keys the
boards bind to, the exporter version + config assumed, and the one-time work-import check. Tasks
3/4 cite it.

- [ ] **Step 1:** Extract the verified catalog from `phase0-observability-config.md` + the live
  exporter into a table: metric name, type (counter/gauge), labels, board(s) that use it.
- [ ] **Step 2:** Record the assumed exporter config (EXTENDED class, `useObjectStatus`,
  `monitoredQueues=*,SYSTEM.*`, `mq-metric-samples` version) and the import-time check:
  `import → confirm label_values(ibmmq_qmgr_status, qmgr) populates → done`.
- [ ] **Step 3:** Note explicitly which board signals are NOT here (backout, in-doubt → ES).
- [ ] **Step 4:** Commit (`vrg-commit --type docs --scope epic`).

**Acceptance:** every PromQL the Wave-1a boards use maps to a row in this note; nothing bound
that isn't verified live.

### Task 2: Portable render mode + shared work-edition helpers

**Repo:** lab · **Files:** Create `src/mqlab/workboards.py`, `tests/test_workboards.py`;
Modify `src/mqlab/cli.py` (dashboard render path)

**Interfaces — Consumes:** `clusterboard._ds/_stat/_timeseries/_row`. **Produces:**
`workboards.tmpl_var(name, query, ...)`, `workboards.portable_dashboard(title, uid, panels,
templating)`, and `workboards.write_work_dashboards(out_dir)`; a `mqlab obs dashboard --portable`
path writing to `build/work/grafana/work-edition/`.

- [ ] **Step 1: Write the failing test** — portability contract:

```python
def test_portable_dashboard_has_no_hardcoded_ds_or_qm():
    dash = workboards.portable_dashboard("QM", "work-qm", panels=[], templating=[
        workboards.tmpl_var("qmgr", "label_values(ibmmq_qmgr_status, qmgr)"),
    ])
    blob = json.dumps(dash)
    assert '"uid": "${datasource}"' in blob or '"datasource": {"uid": "${datasource}"}' in blob
    assert "NHAUAPP" not in blob and "SVCQM" not in blob  # no lab QM names leak
    assert any(v["name"] == "qmgr" for v in dash["templating"]["list"])
```

- [ ] **Step 2:** Run `uv run pytest tests/test_workboards.py -k portable -v` → FAIL.
- [ ] **Step 3:** Implement `tmpl_var` + `portable_dashboard` (datasource as `${datasource}`
  variable on every panel; templating list; no lab names).
- [ ] **Step 4:** Run the test → PASS.
- [ ] **Step 5:** Add the `--portable` render path in cli.py + a test that it writes files under
  `build/work/grafana/work-edition/`; run `vrg-container-run -- vrg-validate`.
- [ ] **Step 6:** Commit.

**Acceptance:** `vrg-validate` green; rendered JSON contains no lab-specific UID/QM/queue names;
`$datasource`/`$qmgr` variables present.

### Task 3: QM-view board generator (`workqmboard.py`)

**Repo:** lab · **Files:** Create `src/mqlab/workqmboard.py`; Test `tests/test_workboards.py`
(QM class); Create `epics/169-work-edition-dashboards/citations/qm-view.md` (in `.github`)

**Interfaces — Consumes:** `workboards.*` (Task 2), `clusterboard._stat/_timeseries`, Task 1
metric names. **Produces:** `workqmboard.work_qm_dashboard() -> dict` and its `--portable`
wiring.

Board **structure** (spec §7.1; metric floor from Task 1): ① status band (QM status · uptime ·
services · connections); ② trend band (message rate · recovery-log % · connections over time,
all `rate()` on counters per §7.2); ③ Attention (which service is down); ④ ES feed placeholder
seam (filled Wave 1b); drill-link carrying `$qmgr` → Queue/channel board.

- [ ] **Step 1: Write the failing test** — structure + binding contract:

```python
def test_work_qm_board_binds_only_verified_metrics_and_is_qm_scoped():
    dash = workqmboard.work_qm_dashboard()
    blob = json.dumps(dash)
    assert "ibmmq_qmgr_status" in blob and "$qmgr" in blob
    assert "ibmmq_queue_" not in blob  # per-queue detail belongs on the flow board
    assert "NHAUAPP" not in blob
```

- [ ] **Step 2:** Run → FAIL.
- [ ] **Step 3:** Implement the QM board from the §7.1 structure, reusing primitives; every
  panel `$datasource`; every query scoped by `$qmgr`.
- [ ] **Step 4:** Run → PASS; `vrg-container-run -- vrg-validate`.
- [ ] **Step 5:** Write `citations/qm-view.md` — for each non-obvious panel, the prior-art
  source (research brief S-refs), the claim, the link (data vs judgment).
- [ ] **Step 6:** Commit.
- [ ] **Step 7 (interactive, post-merge):** render `--portable`, import to the lab Grafana,
  tune v1 live, fold good tweaks back into the generator (spec §3.3). **Not scripted here** —
  this is the co-development loop; acceptance below gates "v1 done."

**Acceptance:** renders portably (Task 2 contract holds); binds only Task-1 metrics; QM-scoped;
citation doc complete; a human confirms v1 reads per the signal-in-the-noise model (§6).

### Task 4: Queue/channel-view board generator (`workflowboard.py`)

**Repo:** lab · **Files:** Create `src/mqlab/workflowboard.py`; Test `tests/test_workboards.py`
(flow class); Create `epics/169-work-edition-dashboards/citations/queue-channel-view.md`

**Interfaces — Consumes:** `workboards.*`, `clusterboard._stat/_timeseries`, Task 1 metrics.
**Produces:** `workflowboard.work_flow_dashboard() -> dict`.

Board **structure** (spec §7.2): ① status band (queues in trouble · channels not running · DLQ
depth red>0 · oldest-message-age); ② the **Attention + Inventory** pair (§6.4) for queues AND
channels — Attention query-filtered (`... > threshold`, empty=healthy) as hero, collapsible
Inventory (all, threshold-colored, worst-first); ③ trend band (depth over time · put-vs-get on
one graph · channel throughput · backout rate); ④ ES feed seam (Wave 1b). `$queue`/`$channel`
multi-select scoped to `$qmgr`.

- [ ] **Step 1: Write the failing test** — the empty=healthy contract is the crux:

```python
def test_attention_table_filters_in_query_not_by_rowcolor():
    dash = workflowboard.work_flow_dashboard()
    att = [p for p in dash["panels"] if p.get("title", "").lower().startswith("attention")]
    assert att, "no Attention panel"
    # empty=healthy is achieved by a threshold comparison in the PromQL itself
    assert any(">" in t["expr"] for p in att for t in p["targets"])
```

- [ ] **Step 2:** Run → FAIL.
- [ ] **Step 3:** Implement per §7.2; Attention queries carry the `> threshold` filter; Inventory
  is the full census; `$queue`/`$channel` scoped to `$qmgr`.
- [ ] **Step 4:** Run → PASS; `vrg-validate`.
- [ ] **Step 5:** Write `citations/queue-channel-view.md`.
- [ ] **Step 6:** Commit.
- [ ] **Step 7 (interactive, post-merge):** portable render → import → tune → fold back (§3.3).

**Acceptance:** Attention filters in-query (empty=healthy); Inventory complete + worst-first;
put-vs-get on one graph; binds only Task-1 metrics; citation doc complete; human confirms v1.

---

## Wave 1b — ES event/log feeds (BLOCKED-BY: LogSearch — imminent, high priority)

### Task 5: ES doc/field contract

**Repo:** `.github` · **Files:** Create `epics/169-work-edition-dashboards/contracts/es-doc-field-contract.md`
· **Blocked-by:** LogSearch project (defines the index/data-stream + fields)

**Interfaces — Produces:** index/data-stream name + field names the panels query (`@timestamp`,
`severity`, `qmgr`, `object`, `reason_code`, `message`). Tasks 6 bind to it.

- [ ] **Step 1:** From the delivered LogSearch ES mapping, record index/data-stream + each field
  (name, type, example). **Do not invent** — if a field is unconfirmed, mark it unknown and stop.
- [ ] **Step 2:** Map each board log/event panel need (QM event feed, error-log feed, channel &
  performance events) to fields; note backout + in-doubt sourced here (not PromQL).
- [ ] **Step 3:** Commit.

**Acceptance:** every Lucene query in Task 6 maps to a documented field; no invented fields.

### Task 6: ES event/log panels on the QM + Queue/channel boards

**Repo:** lab · **Files:** Modify `src/mqlab/workqmboard.py`, `src/mqlab/workflowboard.py`;
Test `tests/test_workboards.py` · **Blocked-by:** Task 5, `clusterboard._logs_panel`

- [ ] **Step 1: Write the failing test** — ES-only + severity-default:

```python
def test_es_feed_is_elasticsearch_typed_and_defaults_error():
    dash = workqmboard.work_qm_dashboard(with_es_feed=True)
    feed = next(p for p in dash["panels"] if "event" in p.get("title","").lower())
    assert feed["datasource"]["uid"] == "${logs}"        # ES var, not Loki
    assert "$level" in json.dumps(feed)                    # severity toggle, default error
```

- [ ] **Step 2:** Run → FAIL.
- [ ] **Step 3:** Implement the `④` ES feed on both boards, binding to Task-5 fields via
  `$logs`/`$level`; reuse `_logs_panel` adapted to the ES datasource type.
- [ ] **Step 4:** Run → PASS; `vrg-validate`.
- [ ] **Step 5:** Commit.
- [ ] **Step 6 (interactive, post-merge):** portable render → import against the lab ES → tune.

**Acceptance:** feeds are ES-typed (not Loki), severity-filtered (default error), bind only
Task-5 fields; boards now function end-to-end against Prometheus + ES.

---

## Wave 2 — Infra / HA-DR board (STARTABLE now against the embedded collector)

> Startable now: the embedded lab collector already publishes the Native HA / CRR metrics.
> **Completion** is gated on extraction (#79) + the metric-name contract (naming reconciliation),
> NOT on starting. Metric names are non-standard across implementations → bind to the lab
> collector's names and translate in-board initially (spec §3.2, §7.3).

### Task 7: Native HA CRR metric contract (with golden sample output)

**Repo:** `.github` · **Files:** Create `epics/169-work-edition-dashboards/contracts/nha-crr-metric-contract.md`
· **Blocked-by (completion, not start):** collector extraction #79

**Interfaces — Produces:** exact metric names, labels, source CLI (`dspmq -o nativeha` + CRR
status), parsing rules, output format for the CRR link·lag·role + quorum/in-sync signals, PLUS a
literal **golden `/metrics` block** captured from the lab, PLUS the naming-reconciliation note
(lab names ↔ work names).

- [ ] **Step 1:** Capture the embedded collector's live `/metrics` for the NHA/CRR series;
  record exact names/labels/sample values as the golden block.
- [ ] **Step 2:** Document source CLI + parsing so Claude-at-work can regenerate a faithful
  collector from prose; the golden block is the diff target.
- [ ] **Step 3:** Add the naming-reconciliation section: board binds to lab names now; converge
  via namespace agreement with work operators OR in-board translation.
- [ ] **Step 4:** Commit.

**Acceptance:** a regenerated collector can be diffed against the golden block; every Task-8
PromQL name appears in the contract.

### Task 8: Infra/HA-DR board generator (`workinfraboard.py`)

**Repo:** lab · **Files:** Create `src/mqlab/workinfraboard.py`; Test `tests/test_workboards.py`
(infra class); Citation `epics/169-work-edition-dashboards/citations/infra-view.md`
· **Consumes:** Task 7 names, `workboards.*`, `clusterboard._state_timeline/_stat`

Board **structure** (spec §7.3): quorum · in-sync replicas · CRR link/lag/role over time (lag =
RPO risk); replica/component state via the Attention + Inventory pair (§6.4). Native HA CRR
only; keyed by `$qmgr`.

- [ ] **Step 1: Write the failing test** — binds contract names, not stock ibmmq_, and shows lag
  as a trend:

```python
def test_infra_board_uses_contract_names_and_trends_crr_lag():
    dash = workinfraboard.work_infra_dashboard()
    blob = json.dumps(dash)
    assert "$qmgr" in blob
    assert any(p.get("type") == "timeseries" and "lag" in json.dumps(p).lower()
               for p in dash["panels"])   # CRR lag over time = RPO risk (§6.2)
```

- [ ] **Step 2:** Run → FAIL.
- [ ] **Step 3:** Implement per §7.3 against Task-7 names; lag/role/quorum trends; Attention +
  Inventory for replicas/components.
- [ ] **Step 4:** Run → PASS; `vrg-validate`.
- [ ] **Step 5:** Write `citations/infra-view.md`.
- [ ] **Step 6:** Commit.
- [ ] **Step 7 (interactive, post-merge):** portable render → import → tune; translate names as
  needed until the contract/namespace converges.

**Acceptance:** binds Task-7 contract names (translatable); CRR lag/role/quorum as trends;
Attention empty=healthy; citation doc complete; human confirms v1.

---

## Dependency graph

```text
Task 1 (Prom schema note) ─┬─▶ Task 3 (QM board) ──┐
                           └─▶ Task 4 (Flow board) ─┤
Task 2 (portable render) ──────▶ Task 3, Task 4 ────┤
LogSearch ─▶ Task 5 (ES contract) ─▶ Task 6 (ES feeds on 3+4)
embedded collector ─▶ Task 7 (NHA/CRR contract) ─▶ Task 8 (Infra board)
   (#79 extraction gates Task 7/8 *completion*, not start)
```

Runnable frontier now: **Tasks 1 → 2 → 3, 4** (Wave 1a) + **Task 7** (contract capture, startable).
Task 5/6 unblock when LogSearch lands (imminent). Task 8 runs against the embedded collector.

## Self-review

- **Spec coverage:** §3.1 → citation docs (T3/4/8); §3.2 → contracts (T1/5/7) + portability
  (T2); §5 variables → T2; §6 signal-in-the-noise → T3/4/8 acceptance; §7.1/7.2/7.3 → T3/4/8;
  §8 waves → task grouping + dep graph; §9 generators/`--portable` → T2 + generators; §10
  research → citation docs; §11 open questions → the interactive Step-7 co-development loop. No
  gaps.
- **No fabricated panel internals:** exact metrics/thresholds are deferred to the Step-7
  interactive loop per spec §7/§11 — the plan TDD's the *portability + binding + empty=healthy*
  contracts (genuinely testable) and gates panel design on human confirmation, rather than
  inventing threshold values the spec says are co-developed.
- **Type consistency:** `workboards.tmpl_var` / `portable_dashboard` / `write_work_dashboards`
  and `work_qm_dashboard` / `work_flow_dashboard` / `work_infra_dashboard` used consistently.
