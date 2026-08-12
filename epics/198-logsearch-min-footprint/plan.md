# Minimal, reliable OpenSearch log-search tier — implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: use `superpowers:subagent-driven-development`
> or `superpowers:executing-plans` to implement each task. Steps use checkbox (`- [ ]`) syntax.
> This is an **epic-framework** plan: each task below is filed as a GitHub issue under epic
> `logical-minds-foundry/.github#198` (`vrg-issue-create --epic …`) and lands its PR in the
> **implementing repo** `logical-minds-foundry/mq-resiliency-lab-for-linux` (NOT `.github`).
> The bookends already exist — do **not** re-file them: #199 (docs, this spec + plan), #1021
> (docs-review sweep, lab repo), #200 (retrospective, `.github`), #1022 (cold-rebuild
> validation, lab repo).

**Goal:** Make the `logsearch` tier cold-boot reliably and fast by running OpenSearch
core-only (the "min" distribution), and prove it with repeated one-pass cold boots.

**Architecture:** Swap the OpenSearch download from the full bundle to the plugin-free min
distribution, remove the one config setting that only the (now-absent) security plugin
registers, fix a Data Prepper systemd unit bug, and add the missing Grafana OpenSearch
datasource. The lab's plaintext posture is already in place (#827), so the sink and Dashboards
need no change.

**Tech Stack:** Ansible roles (`opensearch`, `data-prepper`, `opensearch-dashboards`, `grafana`),
a baked Vagrant "fatbox" (`logsearch-ubuntu2404`), OpenSearch 3.8.0, `mqlab` bootstrap phases,
`vrg-validate` (pytest template-render tests + lint) as the per-task gate.

## Global Constraints

*(Every task's requirements implicitly include these.)*

- **Plaintext is already the posture (#827):** `opensearch.yml` sets
  `plugins.security.disabled: true`; Data Prepper's sink writes `insecure: true` plain http;
  Dashboards targets `http://localhost:9200` with `ssl.verificationMode: none`. Do **not**
  re-implement plaintext — only what min changes.
- **Min removes the security plugin entirely.** Any `plugins.security.*` setting left in
  `opensearch.yml` becomes an **unknown setting** and OpenSearch refuses to start. Removing
  `plugins.security.disabled: true` is mandatory, not cosmetic.
- **No TLS/auth, no retention/ISM** (spec §7 — deferred). Do not add them.
- **Validation is `vrg-container-run -- vrg-validate`** — the only validation command; run it
  from inside the task's worktree. The live acceptance is task #1022, not a per-task step.
- **The min tarball's extracted top-level directory must reconcile** with the install/symlink
  references to `opensearch-{{ opensearch_version }}` (`install.yml` `creates:`/`src:`).

## File structure

- `ansible/roles/opensearch/defaults/main.yml` — the download coordinates (`opensearch_pkg`,
  `opensearch_url`) move from bundle to min.
- `ansible/roles/opensearch/templates/opensearch.yml.j2` — drop the `plugins.security.disabled`
  line (min has no security plugin to register it).
- `ansible/roles/opensearch/tasks/{install,configure}.yml` — reconcile the extracted-dir name;
  neutralize the now-dead admin-password drop-in.
- `ansible/site-logsearch.yml` (+ role readiness waits) — re-tune to the fast reality, keep
  fail-loud.
- `ansible/roles/data-prepper/tasks/install.yml` — quote the systemd `Environment=` line.
- `ansible/roles/grafana/templates/datasource.yml.j2` — add the OpenSearch datasource.
- `tests/…` — template-render assertions for the datasource and the data-prepper unit.

---

## Task 1: OpenSearch → min (core-only) distribution

**Repo:** lab · **Files:** Modify `ansible/roles/opensearch/defaults/main.yml`,
`ansible/roles/opensearch/templates/opensearch.yml.j2`,
`ansible/roles/opensearch/tasks/install.yml`, `ansible/roles/opensearch/tasks/configure.yml`,
`ansible/site-logsearch.yml`; Test `tests/` (opensearch render/defaults)

**Interfaces — Produces:** an `opensearch` role that installs the min distribution and renders
an `opensearch.yml` with **no** `plugins.security.*` settings. Consumed by the box bake and
`site-logsearch.yml`.

- [ ] **Step 1:** In `defaults/main.yml`, change the download to min. Set
  `opensearch_pkg: "opensearch-min-{{ opensearch_version }}-linux-{{ opensearch_arch }}"` and
  `opensearch_url: "https://artifacts.opensearch.org/releases/core/opensearch/{{ opensearch_version }}/{{ opensearch_pkg }}.tar.gz"`
  (note `/releases/core/`, not `/releases/bundle/`).

- [ ] **Step 2:** Verify the extracted top-level directory. Download + unpack the min tarball
  once (`curl -sL <url> | tar tzf - | head -1`) and confirm it extracts to
  `opensearch-{{ opensearch_version }}/`. If it differs (e.g. `opensearch-min-…`), update the
  `creates:` and symlink `src:` in `install.yml` to match; otherwise leave them.

- [ ] **Step 3:** In `templates/opensearch.yml.j2`, **remove** the
  `plugins.security.disabled: true` line and its comment block (min has no security plugin to
  register it → unknown-setting startup failure). Keep the core settings (`cluster.name`,
  `node.name`, `network.host`, `http.port`, `discovery.type: single-node`, `path.data`,
  `path.repo`).

- [ ] **Step 4:** In `tasks/configure.yml`, neutralize the now-dead admin-password path: the
  `OPENSEARCH_INITIAL_ADMIN_PASSWORD` drop-in and the `lab-secret.sh opensearch_admin_password`
  materialization exist only for the security plugin's bootstrap. With min there is no admin
  user. Remove those tasks (or guard them off); do not leave a fail-loud assert on a credential
  the engine no longer consumes.

- [ ] **Step 5:** In `site-logsearch.yml`, re-tune the OpenSearch readiness wait to the min
  reality — core OpenSearch should bind `:9200` in ~1–2 min; set the `wait_for`/`until` timeout
  and retry budget accordingly (e.g. ~180 s), and **keep it fail-loud** (a tier that does not
  come up must still abort observe).

- [ ] **Step 6:** Update/add the opensearch template-render test so it asserts the rendered
  `opensearch.yml` contains **no** `plugins.security` string and still carries
  `discovery.type: single-node` + `http.port`.

- [ ] **Step 7:** `vrg-container-run -- vrg-validate` → green. Commit
  (`vrg-commit --type fix --scope obs`).

**Acceptance:** role installs min; rendered `opensearch.yml` has zero `plugins.security.*`;
extracted-dir references reconcile; `vrg-validate` green. (Live proof is #1022.)

---

## Task 2: Data Prepper systemd `Environment=` quoting fix

**Repo:** lab · **Files:** Modify `ansible/roles/data-prepper/tasks/install.yml`; Test `tests/`
(data-prepper unit render)

**Interfaces — Consumes:** nothing from Task 1. **Produces:** a valid `data-prepper.service`
unit whose JVM heap flags both survive.

- [ ] **Step 1:** In `install.yml` (~line 95), the unit renders
  `Environment=JAVA_OPTS=-Xms{{ data_prepper_heap }} -Xmx{{ data_prepper_heap }}` **unquoted**,
  so systemd splits on the space into `JAVA_OPTS=-Xms…` and a bogus `-Xmx…`
  ("Invalid environment assignment, ignoring: -Xmx1g"). Quote the value:
  `Environment="JAVA_OPTS=-Xms{{ data_prepper_heap }} -Xmx{{ data_prepper_heap }}"`.

- [ ] **Step 2:** Add/extend a template-render test asserting the rendered unit line is a
  single quoted assignment — `Environment="JAVA_OPTS=` present, and no bare `-Xmx` token
  outside the quotes.

- [ ] **Step 3:** `vrg-container-run -- vrg-validate` → green. Commit
  (`vrg-commit --type fix --scope obs`).

**Acceptance:** rendered unit has one quoted `Environment="JAVA_OPTS=…"`; `-Xmx` preserved;
`vrg-validate` green. (The sink is already plaintext #827 — no sink change.)

---

## Task 3: Grafana OpenSearch datasource (create)

**Repo:** lab · **Files:** Modify `ansible/roles/grafana/templates/datasource.yml.j2` (+ the
role defaults/vars supplying the logsearch mgmt IP if not already available); Test `tests/`
(grafana datasource render)

**Interfaces — Consumes:** the logsearch node's mgmt IP from topology. **Produces:** a pinned
OpenSearch datasource `{uid: opensearch}` for #169 Wave 1b to reference.

- [ ] **Step 1:** Decide the datasource type. Prefer the dedicated **`grafana-opensearch-datasource`**
  plugin (correct OpenSearch support); if that plugin is not installed in the obs Grafana,
  fall back to the built-in **`elasticsearch`** type (OpenSearch is ES-API compatible for
  logs/events search). Record which in the task and, if the plugin route, add its install to
  the grafana role. Default to the fallback (`elasticsearch`, no plugin) unless the plugin is
  already present — the leaner path.

- [ ] **Step 2:** In `datasource.yml.j2`, add a third `datasources:` entry — `name: OpenSearch`,
  the chosen `type`, `uid: opensearch` (pinned, mirroring `prometheus`/`loki`),
  `access: proxy`, `url: http://{{ logsearch_mgmt_ip }}:9200`, no TLS, no credentials, an index
  pattern matching the logs index (`logs-*`), and the time field (`@timestamp`). Add
  `name: OpenSearch` to the `deleteDatasources:` list so a re-provision recreates it with the
  pinned uid (the #279 idempotence pattern).

- [ ] **Step 3:** Add a template-render test: rendered datasource YAML contains a datasource
  with `uid: opensearch` and `url: http://…:9200`, and `OpenSearch` is in `deleteDatasources`.

- [ ] **Step 4:** `vrg-container-run -- vrg-validate` → green. Commit
  (`vrg-commit --type feat --scope obs`).

**Acceptance:** Grafana provisions a pinned OpenSearch datasource on plaintext `:9200`;
render test asserts it; `vrg-validate` green. (Live query proof is #1022.)

---

## Task 4 (pre-existing #1022): cold-rebuild validation — the acceptance

**Repo:** lab · **Kind:** validation · **Blocked-by:** Tasks 1, 2, 3
· This task already exists (#1022) — do not re-file; add the `Blocked-by:` links and run it via
`issue-validate` after 1–3 merge.

**Precondition (automatic):** changing the `opensearch` role's download changes the
`logsearch-ubuntu2404` box hash, so `mqlab bootstrap`'s box-ensure **rebuilds the box with min
baked in** before the vms phase — no separate rebake task. Confirm `mqlab box status` shows the
logsearch box `BUILD` (mismatch) or a fresh min build.

- [ ] Tear down; `mqlab bootstrap nativeha-ubuntu --no-dr`; assert **one-pass**:
  OpenSearch binds `:9200` in ~1–2 min · Data Prepper binds `:21892` · a probe log is
  searchable in OpenSearch (REST) **and** via the new Grafana datasource · obs disk stays flat
  (no Alloy "sending queue is full" loop) · `bootstrap` exits 0.
- [ ] **Repeat** the cold boot and assert again (reliability — the missing-repeated-cold-boots
  gap).
- [ ] Record `Outcome: SUCCESS` (or FAILURE with evidence) as a comment. On SUCCESS the task
  closes.

**Conditional follow-up (only if this validation shows the node still can't fit):** bump the
`logsearch` node vCPU in `lab/topology.yaml` (the host has ample cores) and re-run. File this as
a new task **only if** the validation demands it — do not change sizing up front (spec §4.7).

---

## Dependency graph

```text
Task 1 (OpenSearch min) ──┐
Task 2 (Data Prepper env) ─┼─▶ #1022 cold-rebuild validation ─▶ #1021 docs-review ─▶ #200 retrospective
Task 3 (Grafana datasource)┘        (box rebake is automatic)        (multi-repo sweep)   (terminal)
```

Tasks 1–3 are independent of each other (different roles/files) and can be implemented in
parallel. #1022 is blocked-by all three. The terminal bookends (#1021 then #200) follow.

## Self-review

- **Spec coverage:** §3.1/§4.1 min distro → Task 1; §4.1 strip security + admin drop-in →
  Task 1 (already plaintext #827, so only the min-invalid `plugins.security.disabled` line +
  dead admin path); §4.2 data-prepper env fix → Task 2 (sink already plaintext); §4.3 Dashboards
  → **no change needed** (already plaintext #827) — called out, not a task; §4.4 Grafana
  datasource → Task 3; §4.5 box rebake → automatic precondition of #1022; §4.6 readiness →
  Task 1 Step 5; §4.7 sizing → #1022 conditional follow-up; §5 acceptance → #1022; §7 non-goals
  respected. No gaps.
- **Reality reconciled:** the spec assumed the sink/Dashboards needed repointing to plaintext;
  the code already runs security-disabled/plaintext (#827), so those shrink to no-ops — surfaced
  in Global Constraints and Task acceptance. The substantive security change is removing the
  min-invalid `plugins.security.disabled` setting.
- **No placeholders:** each task names exact files and the exact edit; the min-dir and
  datasource-type unknowns are written as explicit verify-and-branch steps, not TBDs.
