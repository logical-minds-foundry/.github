# `mq-resiliency-observability` Extraction — Implementation Plan

> **For agentic workers:** each task below is **PR-sized** and becomes its own
> GitHub issue (filed after `paad:alignment`, epic-create step 9). Each task's
> fine-grained red/green/refactor steps are produced at *implementation* time
> (via `issue-implement`), not here — this plan fixes task boundaries,
> interfaces, dependencies, and acceptance. Steps use checkbox (`- [ ]`) syntax.

**Goal:** Extract the lab's three non-MQI HA/DR state collectors and its portable
dashboards into a standalone, dual-packaged (`.rpm` + `.deb`) product,
`mq-resiliency-observability`, and have the lab dogfood the published artifact.

**Architecture:** A new public repo holds a pure-Python-stdlib package (collectors
+ systemd timers + a `render-dashboards` generator) plus a format-agnostic
packaging core with per-format adapters. Each collector is lifted from the member
repo, parameterized, and driven through the roadmap's six-step migration gate.
The lab becomes a pure consumer that installs the published package.

**Tech Stack:** Python 3 stdlib only (collectors); node_exporter textfile protocol;
systemd timers; RPM (`%post`/`%postun`) + Debian (`postinst`/`postrm`) packaging;
Grafana dashboard JSON; the repo's CI (`vrg-validate`, 100% branch coverage).

## Global Constraints

- **Language:** collectors are **stdlib-only** — no PyMQI, no compiled deps, no
  third-party runtime imports. (Verified for `nativehastate`/`rdqmstate`/
  `clusterstate`.)
- **License:** MIT. **Versioning:** semver from `0.x`; `1.0` only after the lab
  dogfoods through a cold rebuild.
- **Coverage:** pure logic to **100% branch** (repo gate). Validation is
  `vrg-container-run -- vrg-validate` only.
- **QM-name source of truth:** derive dashboard/profile QM names from
  `stacks.py` semantics (`{short}APP`/`{short}SVC`) — never a competing source.
- **Ownership boundary:** configure only what we own; declare-and-loudly-verify
  the rest; never edit `qm.ini`, never restart a QM, never mutate node_exporter's
  config. Fail loud; no stale panels; no silent failures.
- **Packaging:** `.rpm` **and** `.deb` from one source; format-agnostic core +
  adapters designed up front, extensible to a third format.
- **Publishing a release is human-gated** — no task performs a publish; the agent
  prepares it and stops.

---

## New-repo file structure (`mq-resiliency-observability`)

```
src/mqro/
  collectors/
    base.py          # atomic textfile write (temp+rename), `_m()` line formatter,
                     # timeout-bounded subprocess helper, *_last_write_timestamp
    drbd.py          # parse_drbd() — shared by rdqm + cluster (from clusterstate.py)
    nativeha.py      # cluster_nha_*   (from nativehastate.py)
    rdqm.py          # cluster_rdqm_*  (from rdqmstate.py)
    cluster.py       # cluster_*/cluster_drbd_* Pacemaker/DRBD (from clusterstate.py)
  detect.py          # precedence+exclusion activation; config override
  config.py          # profile + REQUIRED textfile-dir config
  contract.py        # emitted metric families + required scrape-side labels (versioned)
  dashboards/
    render.py        # render-dashboards CLI: profile -> Grafana JSON
    boards/          # de-hardcoded board builders (qm, messaging, cluster/HA)
packaging/
  core/              # format-agnostic package description (files, units, deps, meta)
  rpm/               # RPM adapter (%post gate, %postun cleanup)
  deb/               # DEB adapter (stubbed in slice 1: TODO(deb) at each divergence)
  units/             # *.service / *.timer templates
  scripts/           # install-time textfile-dir gate; runtime self-check
tests/
docs/README.md       # off-the-shelf wiring: scrape/relabel snippet, label contract,
                     # textfile boundary, node_exporter prerequisite
```

Source of each lift (member repo `mq-resiliency-lab-for-linux`):
`src/mqlab/{nativehastate,rdqmstate,clusterstate}.py`;
`ansible/roles/{nativeha-state,rdqm-state,cluster-state}`;
`src/mqlab/{dashboard,clusterboard,messagingboard,qmboard}.py`; `stacks.py`.

---

## Phase 0 — Bootstrap

### Task 0: Create the `mq-resiliency-observability` repo *(= filed task #82)*

**Repo:** n/a (creates the repo). **Gate step:** precondition to all.

- **Deliverable:** the public repo exists, Vergil-scaffolded (MIT, `0.1.0`,
  Python), empty of product code.
- **Loose cross-org dependency:** `vergil-project/vergil-tooling#2382` (scripted
  `vrg-github-repo-init`). Prose reference, not a `Blocked-by` link. Sequence:
  land #2382 → create non-interactively; **fallback** = interactive wizard with a
  prepared answer table if #2382 isn't ready.
- **Includes:** the final naming pass (spec §11 open question) — resolve the
  placeholder name *before* creation.
- **Acceptance:** repo builds/lints green empty; branch protection + CI active.

---

## Phase 1 — Packaging framework (up front)

### Task 1: Format-agnostic packaging core + RPM adapter + Debian stub

**Repo:** `mq-resiliency-observability`. **Depends:** T0. **Gate step:** 2 (build).

- **Files:** `packaging/core/*`, `packaging/rpm/*`, `packaging/deb/*` (stub),
  `packaging/units/*`, `packaging/scripts/*`, CI build job.
- **Deliverable:** a declarative package description (files, install paths,
  systemd units, dependencies, metadata) that the **RPM adapter** turns into a
  `.rpm` for a trivial payload; the **Debian adapter is forked-and-stubbed** —
  present at every divergence with a `TODO(deb)` marker so slice-2 fills it in
  without reshaping the core.
- **Interfaces — Produces:** a `build(description, fmt)` entry point later tasks
  feed their collector/units into; `fmt ∈ {rpm, deb}` (deb raises
  `NotImplementedError("TODO(deb)")` for now).
- **Install/uninstall scripts (§3.6):** `%post`/`postinst` **fail loud** if the
  configured textfile directory is unspecified or not writable by the service
  user; `%postun`/`postrm` stop+disable timers and remove only our own artifacts.
  (Wired to a real payload in T4; here they exist and are unit-tested against a
  trivial payload.)
- **Tests:** building the trivial payload yields an installable `.rpm`; the
  `%post` gate errors on an unwritable/absent dir; the `deb` path raises the
  stubbed marker; adapter selection is covered.
- **Acceptance:** `.rpm` builds in CI; deb stub is explicit, not silent.

---

## Phase 2 — Slice 1: Native HA, end-to-end (six-step gate)

### Task 2: Extract & scrub the shared collector base + Native HA collector

**Repo:** `mq-resiliency-observability`. **Depends:** T0. **Gate step:** 1.

- **Files:** create `src/mqro/collectors/base.py`, `src/mqro/collectors/nativeha.py`,
  `src/mqro/config.py`, `tests/collectors/test_nativeha.py`,
  `tests/collectors/test_base.py`. Lift from member `src/mqlab/nativehastate.py`.
- **Scrub:** remove lab assumptions; the QM name and textfile directory become
  **profile/config inputs** (no hardcoded `NHARAPP`, no assumed path); keep
  stdlib-only; preserve `dspmq -o nativeha` parsing and the `cluster_nha_*`
  families verbatim (contract-preserving).
- **Interfaces — Produces:** `collect_nativeha(cfg) -> list[Metric]`;
  `base.write_textfile(dir, name, metrics)` (atomic temp+rename);
  `base.emit_last_write_timestamp(...)`. **Consumes:** `config.Profile`.
- **Tests:** parse fixtures (captured `dspmq -o nativeha -x/-g` text, incl. the
  `Unknown`/non-numeric tolerance from member #390); atomic write; stale-write
  never republishes; **100% branch**.
- **Acceptance:** `cluster_nha_*` families byte-compatible with the lab's output
  for the same input.

### Task 3: Detection framework (precedence + exclusion) — Native HA arm

**Repo:** `mq-resiliency-observability`. **Depends:** T2. **Gate step:** 1.

- **Files:** `src/mqro/detect.py`, `tests/test_detect.py`.
- **Deliverable:** the activation resolver (spec §3.2): explicit config is the
  backbone; auto-detect is a convenience. Resolution order **RDQM → Native HA →
  standalone Pacemaker**, with RDQM suppressing generic Pacemaker. This task lands
  the **framework + the Native HA resolver + config override**; the RDQM and
  Pacemaker resolvers arrive with their collectors (T8, T9).
- **Interfaces — Produces:** `resolve(probes, config) -> set[Collector]`;
  probe seams `probe_nativeha()`, `probe_rdqm()`, `probe_pacemaker()` (RDQM/pcmk
  return stubs until T8/T9).
- **Tests:** Native HA env activates `nativeha`; config override wins; an
  unresolvable/unknown combination fails loud; the RDQM-both-probes case is
  asserted here as a **table stub** and completed in T8.
- **Acceptance:** no path silently guesses; config always overrides.

### Task 4: Package the Native HA collector — timer, textfile boundary, RPM

**Repo:** `mq-resiliency-observability`. **Depends:** T1, T2, T3. **Gate step:** 2.

- **Files:** `packaging/units/mqro-nativeha.{service,timer}`, wire the real
  payload (T2 collector + T3 detect) into the T1 core; `tests/packaging/*`.
- **Deliverable:** a `.rpm` that installs the Native HA collector, its ~5s timer,
  the install-time textfile-dir gate, the runtime self-check, and clean uninstall.
- **Tests:** install into a container/VM fixture → timer active, `.prom` appears
  in the configured dir; `%post` fails loud on a bad dir; `%postun` removes our
  units + our `.prom`, leaves the dir + foreign files intact.
- **Acceptance:** `rpm -i`/`rpm -e` round-trips clean; metrics scrape.

### Task 5: Metrics contract module + consistency test (Native HA subset)

**Repo:** `mq-resiliency-observability`. **Depends:** T2. **Gate step:** 2.

- **Files:** `src/mqro/contract.py`, `tests/test_contract.py`, `docs/README.md`
  (contract + example scrape/relabel snippet + the **required scrape-side labels**
  and **textfile-boundary** prerequisites).
- **Deliverable:** the versioned `cluster_nha_*` emitted-families half of the
  contract, plus the CI **consistency test harness** (every metric a board queries
  is one a collector emits — exercised fully once boards land in T10).
- **Tests:** the contract lists exactly what `collect_nativeha` emits; the harness
  fails if a listed family is not emitted.
- **Acceptance:** README leads with the label + textfile-dir traps.

### Task 6: CI in the new repo (build + test + contract)

**Repo:** `mq-resiliency-observability`. **Depends:** T4, T5. **Gate step:** 2.

- **Deliverable:** CI runs the test suite (100% branch), builds the `.rpm`, runs
  the contract-consistency test, on every PR.
- **Acceptance:** green CI on the Native HA slice.

### Task 7: Publish `0.1.0` (human-gated) *(operational — release)*

**Repo:** `mq-resiliency-observability`. **Depends:** T6. **Gate step:** 3.

- **Deliverable:** the first published `.rpm` on the chosen channel (spec §11 open
  question — resolved in the packaging sub-brainstorm; GitHub Releases is the
  likely first step).
- **Human-gated:** the agent prepares the release and **stops**; a human tags and
  publishes. Not PR-workable.

### Task 8: Dogfood — lab installs the published Native HA collector

**Repo:** `mq-resiliency-lab-for-linux` (member). **Depends:** T7. **Gate step:** 4.
*(deployment-kind operational task)*

- **Files:** modify `ansible/roles/nativeha-state` (and/or a new install role) to
  **install the published `.rpm`** on the RHEL Native HA node instead of deploying
  the in-repo script; feed the profile (QM name, textfile dir) from `stacks.py`.
- **Note:** slice 1 dogfoods the **RHEL** Native HA arm (`.rpm`); the **Ubuntu**
  Native HA arm waits for the `.deb` adapter (Task 11).
- **Acceptance:** the RHEL Native HA node runs the packaged collector; dashboards
  read live.

### Task 9: Delete the in-lab Native HA collector copy

**Repo:** `mq-resiliency-lab-for-linux` (member). **Depends:** T8. **Gate step:** 5.

- **Files:** remove `src/mqlab/nativehastate.py` and the in-repo deploy path of
  `ansible/roles/nativeha-state`; the deletion proves no silent fallback.
- **Acceptance:** lab has no local copy; provisioning uses only the published
  artifact.

### Task 10 *(= filed validation #644)*: Cold-rebuild proof — `.rpm` on RHEL

**Repo:** `mq-resiliency-lab-for-linux` (member). **Depends:** T8, T9.
**Gate step:** 6. *(validation-kind)*

- **Deliverable:** a full VM cold rebuild installs and runs the published `.rpm`
  Native HA collector **one-pass** on the RHEL node; metrics live; `Outcome:
  SUCCESS` recorded as a comment.

---

## Phase 3 — Slice 2+: Debian, RDQM, Pacemaker, dashboards

### Task 11: Complete the Debian adapter (fill the `TODO(deb)` stubs)

**Repo:** `mq-resiliency-observability`. **Depends:** T4. **Gate step:** 2.

- **Deliverable:** the `deb` adapter builds a `.deb` from the same core
  description; `postinst`/`postrm` mirror the RPM gate + cleanup.
- **Tests:** `.deb` builds in CI; install/uninstall round-trips on Ubuntu.
- **Acceptance:** both formats build from one source; "done" now reachable.

### Task 12: RDQM collector — extract, detection precedence, package, dogfood

**Repo:** both. **Depends:** T2 (base/drbd), T3, T11. **Gate step:** 1–6.

- **Files:** `src/mqro/collectors/drbd.py` (extract `parse_drbd`),
  `src/mqro/collectors/rdqm.py` (from `rdqmstate.py`); complete `probe_rdqm()` +
  the **RDQM-suppresses-Pacemaker** precedence (spec §3.2); package (`.rpm` for
  RHEL RDQM); dogfood + delete in-lab copy (`src/mqlab/rdqmstate.py`,
  `ansible/roles/rdqm-state`).
- **Tests:** RDQM env activates `rdqm` and **not** `cluster` even though `crm_mon`
  answers; `cluster_rdqm_*`/`cluster_drbd_*` contract-preserving; 100% branch.
- **Acceptance:** RDQM node scrapes the packaged collector; no double emission.

### Task 13: Pacemaker/DRBD collector — extract, detection, package, dogfood

**Repo:** both. **Depends:** T2, T3, T12 (shared `drbd.py`). **Gate step:** 1–6.

- **Files:** `src/mqro/collectors/cluster.py` (from `clusterstate.py`); complete
  `probe_pacemaker()` (standalone Pacemaker, not-RDQM); package; dogfood; delete
  `src/mqlab/clusterstate.py` + `ansible/roles/cluster-state`.
- **Tests:** standalone Pacemaker activates `cluster`; `cluster_*`/`cluster_drbd_*`
  contract-preserving; 100% branch.
- **Acceptance:** Pacemaker arm scrapes the packaged collector.

### Task 14: `render-dashboards` generator + portable dashboards

**Repo:** both. **Depends:** T5, T12, T13. **Gate step:** 1–2 (+dogfood).

- **Files:** `src/mqro/dashboards/render.py`, `src/mqro/dashboards/boards/*`;
  de-hardcode the member builders `dashboard.py`/`clusterboard.py`/
  `messagingboard.py`/`qmboard.py`. Profile from `stacks.py` semantics.
- **Order:** stock-only boards first (`qmboard`/`messagingboard` — no collector
  dependency), then the cluster/HA boards on the contract.
- **Tests:** the **full** contract-consistency test (every panel metric is
  emitted); rendering is deterministic; no hardcoded QM/resource names.
- **Acceptance:** boards render from a profile; lab repoints to the published
  generator; in-lab builders deleted.

### Task 15 *(= filed validation #645)*: Cold-rebuild proof — `.deb` on Ubuntu

**Repo:** `mq-resiliency-lab-for-linux` (member). **Depends:** T11, T12/T13 as
applicable. **Gate step:** 6. *(validation-kind)*

- **Deliverable:** a full cold rebuild installs and runs the published `.deb`
  one-pass on the Ubuntu node(s); `Outcome: SUCCESS`.

---

## Phase 4 — Discovery, packaging sub-brainstorm, closeout

### Task 16: Discovery — CLI-only / filesystem-only MQ metrics (discovery-only)

**Repo:** `.github` (a report doc) or member `docs/`. **Depends:** none.

- **Deliverable:** a written, prioritized candidate list (FFST `/var/mqm/errors`
  **count + oldest-age** at the top; `dspmqtrn`, `dspmqver`/`dspmqinst`,
  `dmpmqcfg`, `dspmqspl`). **No collector is built here** — building the top
  candidate is spun into the follow-on brainstorm (#81).

### Task 17: Packaging sub-brainstorm — dual-format build+publish, Vergil-ready

**Repo:** `.github`. **Depends:** T1 (learnings), before epic close.

- **Deliverable:** resolve spec §11 open questions — builder (`nfpm` vs `fpm` vs
  native), publish channel, and the **shape of the Vergil-toolkit extraction**
  (forward-engineered here, extracted by a follow-on epic).

### Task 18: Adopter playbook (out-of-band, disposable)

**Repo:** member `docs/` or handed off. **Depends:** T14. Gates nothing.

- **Deliverable:** a one-off markdown: manual, no-AI, UI-first dashboard rebuild
  in a from-scratch Grafana, incl. a **JSON-import availability test** + UI-only
  fallback. Not a product doc; may be pulled into its own brainstorm.

### Bookends *(already filed)*

- **#643** documentation review (multi-repo: lab `docs/site` + org `docs` repo if
  implicated; one PR per repo).
- **#81** follow-on brainstorm (successor epics: **packaging-tooling extraction to
  Vergil**, generic event handler for FFST-as-event, `mq-resiliency-logging`, the
  §6 metric builds).

---

## Dependency graph (task → blocked-by)

```
T0(#82) ─┬─ T1 ─┬───────────────── T4 ─ T6 ─ T7 ─ T8 ─ T9 ─ T10(#644)
         │      │                  │
         ├─ T2 ─┼─ T3 ─────────────┘
         │      └─ T5 ──────────────────────────── (contract)
         └───────────────────────────
   T11(deb) ⟵ T4        T12(rdqm) ⟵ T2,T3,T11     T13(pcmk) ⟵ T2,T3,T12
   T14(dashboards) ⟵ T5,T12,T13      T15(#645) ⟵ T11,(T12/T13)
   T16 discovery — independent      T17 packaging sub-brainstorm — before close
   T18 playbook ⟵ T14 (out-of-band)
```

## Self-review — spec coverage

- §2 charter/scope → T2/T12/T13 (three collectors, stdlib); non-goals honored
  (no net/app collectors, no logging, no event handler, no MQI).
- §3.1 package shape → T1/T4. §3.2 detection precedence → T3/T12/T13.
- §3.3 contract (both halves) → T5/T14 + README. §3.4 dashboard generator → T14
  (profile from `stacks.py`).
- §3.5 dual-format packaging up front → T1 (core+rpm+deb stub) / T11 (deb).
- §3.6 textfile boundary + clean removal → T1 scripts, T4 wiring, tests in T4.
- §4 build order + six-step gate → T2–T10 (slice 1), T11–T15 (slice 2+); per-format
  cold rebuild → T10(#644)/T15(#645).
- §5 repo bootstrap + cross-org dep → T0(#82). §6 discovery-only → T16.
- §7 adopter playbook → T18. §9 testing/DoD → per-task tests + T6 CI + T10/T15.
- §10 follow-on → #81. §11 open questions → T0 (naming), T17 (packaging/publish).

No spec requirement is left without a task.
