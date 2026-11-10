# Trim the lab to develop-only and three stacks: implementation plan

**Status:** draft, in design review · **Epic:** logical-minds-foundry/.github#399 ·
**Planning issue:** logical-minds-foundry/.github#398 (review history is in its
comments)

> **For agentic workers:** each task below is one GitHub issue and one PR,
> implemented with `issue-implement` (operational tasks with `issue-deploy` or
> `issue-validate`). Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the release/tarball machinery and the two Ubuntu Pacemaker
stacks (`pcmk-ubuntu`, `pcmk-drbd-ubuntu`) from the live code, so `develop` is
the lab and exactly three stacks remain (`rdqm-rhel`, `nativeha-rhel-crr`,
`nativeha-ubuntu`). Then bring the open backlog in line.

**Architecture:** There are three tracks.

- **Track A** changes the repo profile first and removes the CI version gate in
  stages around a human ruleset `apply`.
- **Track B** first cuts the four couplings that tie retained stacks to
  Pacemaker code. It then cuts the stack model over in one PR (topology,
  catalog and Python consumers together, because the catalog loader and the
  dashboards fail loud on a missing group or role). Then it removes the dead
  Ansible, the collector and the mentions, area by area.
- **Track C** is a human-confirmed triage sweep that runs per track.

**Tech Stack:** Python 3.14 (`uv`, pytest), Ansible, GitHub Actions
(vergil-actions v2.1), vergil-tooling profile (`vergil.toml`), MkDocs.

**Spec:** [`spec.md`](spec.md) in this directory. Implementers read both.

## Global Constraints

- **Validation:** the only validation command is
  `vrg-container-run -- vrg-validate`, run on the committed branch. It must be
  green before `vrg-pr-workflow report-ready`.
- **Git and GitHub:** use `vrg-git`, `vrg-gh` and `vrg-commit` only. Work goes
  through one worktree per issue, `.worktrees/issue-<N>-<slug>/`.
- **Repository:** all implementation tasks land in
  `logical-minds-foundry/mq-resiliency-lab-for-linux` unless the delivery table
  says otherwise.
- **History stays:** no edits under `docs/specs/`, `docs/plans/`,
  `docs/reports/` or `docs/superpowers/`.
- **Retained stacks:** no behaviour change to `rdqm-rhel`, `nativeha-rhel-crr`
  or `nativeha-ubuntu`.
- **QM defaults:** where a QM default must exist, it is `NHAUAPP`. The
  `mq-exporter` QM, connection and port have no default at all.
- **No silent fallbacks:** a removed default is replaced by a retained value or
  by a required argument that fails loud.
- **The §4.6 boundary:** Pacemaker, Corosync and DRBD as RDQM's own technology
  stay, along with `docs/reference/rdqm-blueprint/**`.
- **Workflow pushes:** a delta touching `.github/workflows/**` makes GitHub
  refuse the agent push. That is expected. `report-ready` records the state,
  and the human's `vrg-submit-pr` pushes.

## Review Focus

These failure modes have no task test of their own yet. Each line names the
behaviour a person would expect, and its pinning test is added to the owning
task.

1. **A retained stack's bootstrap still renders its MQSC** after
   `mq-pcmk-qmgr` is gone. Every template path referenced from `site-rdqm.yml`,
   `site-nativeha.yml`, `site-nativeha-ubuntu.yml` and `roles/mq-qmgr` exists
   on disk. Task 5a adds `test_retained_mqsc_template_paths_exist`.
2. **The observability play parses on a retained-only topology.** No
   `observability.yml` role entry names a missing role. Task 6c adds
   `test_observability_roles_all_exist`.
3. **A missing exporter value fails loud, not wrong.** Running the
   `mq-exporter` role without `mq_exporter_qm`, `_conn` or `_port` stops with
   an assert that names the variable. Task 6c adds
   `test_mq_exporter_requires_qm_conn_port`.
4. **The cockpit refuses an unknown or removed arm.** Rendering it for
   `pcmk-ubuntu` raises `ValueError` naming the arm, never falling through to a
   default board. Task 6a adds
   `test_render_cluster_dashboard_rejects_removed_arm`.
5. **The version catalog loads with no `san` role.** `load_catalog()` on the
   committed `lab/versions.yaml` succeeds, and no role named `san`, `pcmk` or
   `pdrbd` appears in it. Task 6a adds `test_catalog_has_only_retained_roles`.

---

## Delivery table

| Task | Title | Repo | Kind | Blocked by |
|---|---|---|---|---|
| 1 | Profile to develop-only; un-version docs | lab | impl | — |
| 2 | Ruleset apply, delete `main` and release secrets | lab | deployment (human-run) | 1 |
| 3 | Remove release machinery and rewrite clone-and-run docs | lab | impl | 1 |
| 4 | Remove the CI `version` job | lab | impl | 2 |
| 5a | Move shared MQSC templates out of `mq-pcmk-qmgr` | lab | impl | — |
| 5b | Move `parse_drbd` into the RDQM collector | lab | impl | — |
| 6a | Stack-model cutover: topology, catalog and Python consumers | lab | impl | 5a |
| 6b | Remove Pacemaker-only CLI and model features; retire box names | lab | impl | 6a |
| 6c | Remove Pacemaker Ansible, lab scripts and SAN networks | lab | impl | 5a, 6a |
| 6d | Remove the PCMK collector and component defaults | lab | impl | 5b, 6c |
| 7a | Purge Pacemaker-stack mentions: Ansible and `lab/` | lab | impl | 5b, 6c |
| 7b | Purge Pacemaker-stack mentions: `src/mqlab` and `tests/` | lab | impl | 6a, 6b, 6c, 6d, 7a |
| 7c | Track B live docs | lab | impl | 3, 6c, 6d, 7a |
| 8 | Host cleanup on lab hosts | lab | deployment (human-run) | 6b, 6c |
| 9 | Cold rebuild of the three retained stacks | lab | validation | 6d, 7a, 7b, 8 |
| 10a | Triage: release-scope open work | lab | triage (human-confirmed) | 3 |
| 10b | Triage: Pacemaker-stack open work | lab | triage (human-confirmed) | 6c, 6d |
| — | Documentation review (`mq-resiliency-lab-for-linux#1712`) | lab | bookend | 4, 7c, 9, 10a, 10b |
| — | Retrospective (`.github#401`) | .github | bookend | all |

"lab" is `logical-minds-foundry/mq-resiliency-lab-for-linux`.

The triage tasks change no code, so they have no PR. Each is run interactively
with the human, who confirms every batch of closes. The task closes with a
comment that lists each item and its outcome.

---

## Track A — develop-only

### Task 1: Profile to develop-only; un-version docs

**Files:**

- Modify: `vergil.toml` (`[project]`)
- Modify: `.github/workflows/cd.yml`
- Modify: `docs/site/mkdocs.yml`
- Modify: `pyproject.toml` (docs group)
- Modify: `uv.lock`
- Test: `tests/test_profile_develop_only.py` (new)

**Interfaces:**

- Produces: `[project]` `versioning-scheme = "none"`,
  `branching-model = "docs-single-branch"`, `release-model = "none"`. Later
  tasks and the Task 2 `apply` rely on these values.

- [ ] **Step 1: Profile-fitness check (spec §3.5).** Confirm two things in the
  installed vergil-tooling:
  - `lib/config.py` accepts `docs-single-branch` with
    `repository-type = "tooling"`.
  - `vrg_commit.py` and `vrg_finalize_pr.py` accept the lab's `feature/*`
    branches and the single `develop` eternal branch.

  Record the evidence in the PR notes. If either fails, stop and raise it with
  the human as a vergil-tooling issue.
- [ ] **Step 2: Write the failing test** in `tests/test_profile_develop_only.py`:
  - `test_profile_is_develop_only`: parse `vergil.toml` with `tomllib` and
    assert `project["versioning-scheme"] == "none"`,
    `project["branching-model"] == "docs-single-branch"` and
    `project["release-model"] == "none"`.
  - `test_cd_triggers_on_develop_only`: load `.github/workflows/cd.yml` and
    assert `on.push.branches == ["develop"]`.
  - `test_docs_not_versioned`: load `docs/site/mkdocs.yml` and assert
    `"version" not in config.get("extra", {})`.
  - `test_mike_not_a_dependency`: assert no entry in
    `pyproject.toml [dependency-groups].docs` starts with `mike`.
- [ ] **Step 3: Run it and confirm it fails** with
  `vrg-container-run -- vrg-validate`. Expected: the four new tests FAIL.
- [ ] **Step 4: Implement.**
  - Set the three `[project]` values.
  - Change the `cd.yml` trigger to `branches: [develop]`.
  - Remove `extra.version` (`provider: mike`) from `mkdocs.yml`.
  - Remove `mike` from the docs group and re-lock with `uv lock`.
  - Leave the `ci.yml` `version` job in place. Under `release-model = "none"`
    it passes as a no-op (spec §3.2 step 1).
- [ ] **Step 5: Validate.** Run `vrg-container-run -- vrg-validate`. Expected:
  green, with the `version / version-bump` job passing as a no-op on the PR.
- [ ] **Step 6: Commit and hand off.** Run
  `vrg-commit --type chore --scope profile --message "move the profile to develop-only, unversioned"`,
  then `report-ready`. The `cd.yml` change means the human pushes.

### Task 2: Ruleset apply, delete `main` and release secrets (deployment, human-run)

This task makes no PR. Create it with `vrg-issue-create --kind deployment`; it
is blocked by Task 1. The procedure is human-attested, because every step uses
the maintainer's own GitHub login.

- [ ] **Precondition:** Task 1 has merged, and CD on `develop` has deployed the
  docs site with no version selector. Confirm the site root loads without
  `/dev/` or `/latest/`.
- [ ] **Step 1:** From the lab checkout, run
  `vrg-github-repo-config apply`, so the required
  `version / version-bump` check drops from the CI-gates ruleset (spec §3.2
  step 2).
- [ ] **Step 2:** Run the `ops.yml` workflow by hand. Expected: the audit
  passes with no drift.
- [ ] **Step 3:** Delete the `main` branch on GitHub. Delete only a ruleset or
  classic protection whose sole target is `main`. The shared managed
  branch-protection and CI-gates rulesets stay untouched (spec §3.4).
- [ ] **Step 4:** Delete the repo secrets `RELEASE_GPG_PRIVATE_KEY` and
  `RELEASE_GPG_PASSPHRASE`.
- [ ] **Step 5:** Run `ops.yml` again. Expected: no drift. If it reports drift,
  run `vrg-github-repo-config apply` and audit again.
- [ ] **Result:** an `Outcome: SUCCESS` comment recording the audit-run URLs.
  Together with Task 4's green CI, this proves Track A's acceptance (spec §6).

### Task 3: Remove release machinery and rewrite clone-and-run docs

**Files:**

- Delete:
  - `.github/workflows/release.yml`
  - `RELEASE-KEY.asc`
  - `tools/release_version_guard.py` and `tests/test_release_version_guard.py`
  - `.gitattributes` and `tests/test_gitattributes_curation.py`
  - `VERSION`, `CHANGELOG.md` and `releases/`
  - `docs/development/release-runbook.md`
- Modify:
  - `docs/site/mkdocs.yml`: the Releases `nav` section (about lines 58-62),
    whose targets this task deletes. Otherwise `mkdocs build --strict` fails.
  - `vergil.toml`: the `[markdownlint]` ignores and their comment.
  - `.gitignore`: the changelog and releases lines, unless they sit in the
    vergil-managed block.
  - `pyproject.toml`: a comment that marks `version` as a fixed placeholder.
  - `src/mqlab/paths.py`: the `repo_root()` docstring.
  - `tests/test_no_pymqi_refs.py`
  - `README.md` and `CLAUDE.md`
  - `docs/site/docs/index.md`, `getting-started.md` and `develop.md`
  - `docs/development/operating-the-lab-from-a-dev-session.md`
- Test: `tests/test_no_release_machinery.py` (new)

**Interfaces:**

- Consumes: Task 1's profile.
- Produces: the docs say clone-and-run with `uv run mqlab` only. Task 7c
  changes the starting stack in those same files.

- [ ] **Step 1: Write the failing test** `tests/test_no_release_machinery.py`:
  - `test_release_files_absent`: none of `.github/workflows/release.yml`,
    `RELEASE-KEY.asc`, `tools/release_version_guard.py`, `.gitattributes`,
    `VERSION`, `CHANGELOG.md`, `releases` or
    `docs/development/release-runbook.md` exists.
  - `test_live_docs_have_no_release_path`: for `README.md`, `CLAUDE.md`,
    `docs/site/docs/index.md`, `getting-started.md` and `develop.md`, the text
    matches none of `gpg --verify`, `RELEASE-KEY`, `releases/download`, `tar -x`
    or `signed release`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expected: both FAIL.
- [ ] **Step 3: Implement** the deletions and modifications listed above.
  - **Rewritten docs:** they describe clone the repo, run `./scripts/setup`,
    then `uv run mqlab …`. Remove the getting-started "consumer path" warning
    and its download section. CLAUDE.md's overview says the lab runs from a
    checkout of `develop`, and its branching line names `docs-single-branch`.
  - **`paths.py`:** the docstring says `MQLAB_REPO_ROOT` is a test hook for
    pointing the repo at a temporary tree. The behaviour is unchanged.
- [ ] **Step 4: Validate.** Expected: green, including the strict docs build.
  The kept `version` job passes as a no-op.
- [ ] **Step 5: Commit and hand off.** Run
  `vrg-commit --type chore --scope release --message "remove the release tarball machinery"`.
  The human pushes, because the delta touches `.github/workflows/`.

### Task 4: Remove the CI `version` job

**Files:**

- Modify: `.github/workflows/ci.yml` (the `version` job and the
  changelog-staging comment on the `docs` job)
- Test: `tests/test_profile_develop_only.py`

- [ ] **Step 1: Add the failing test**
  `test_ci_has_no_version_job`: load `.github/workflows/ci.yml` and assert
  `"version" not in workflow["jobs"]`.
- [ ] **Step 2: Implement** the removal.
- [ ] **Step 3: Validate.** Expected: green. The PR's required checks no longer
  include `version / version-bump`, because Task 2's `apply` removed it.
- [ ] **Step 4: Commit and hand off.** Run
  `vrg-commit --type ci --scope ci --message "drop the version-bump gate"`.
  The human pushes.

---

## Track B — three stacks

### Task 5a: Move shared MQSC templates out of `mq-pcmk-qmgr`

**Files:**

- Move: `ansible/roles/mq-pcmk-qmgr/templates/mqoperator.mqsc.j2` →
  `ansible/templates/mq/mqoperator.mqsc.j2`
- Modify:
  - `ansible/roles/mq-qmgr/tasks/main.yml:116`
  - `ansible/site-rdqm.yml` (the `mqoperator` include at about line 438, and
    the `authz.mqsc.j2` include at about line 393)
  - `ansible/site-nativeha.yml` (about line 140)
  - `ansible/site-nativeha-ubuntu.yml` (about lines 164 and 169)
  - `ansible/group_vars/all/authz.yml:118` (comment)
- Modify tests:
  - `tests/test_mqmon_authz.py:23`
  - `tests/test_mqweb_operator_authz.py:20,171`
  - `tests/test_channel_failover_qmini.py:125`
- Test: `tests/test_retained_mqsc_templates.py` (new)

**Interfaces:**

- Produces: `ansible/templates/mq/{authz,mqoperator}.mqsc.j2` are the only
  copies retained code reads. Task 6c deletes `roles/mq-pcmk-qmgr/` and
  `ansible/templates/mq/inter-qm.mqsc.j2` outright.

- [ ] **Step 1: Write the failing tests** in
  `tests/test_retained_mqsc_templates.py`:
  - `test_retained_mqsc_template_paths_exist` (Review Focus 1): scan
    `site-rdqm.yml`, `site-nativeha.yml`, `site-nativeha-ubuntu.yml` and
    `roles/mq-qmgr/tasks/main.yml` for `*.mqsc.j2` references, and assert each
    resolves to an existing file.
  - `test_retained_code_does_not_read_mq_pcmk_qmgr`: none of those files, nor
    the three repointed tests, contains `mq-pcmk-qmgr`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expected: the second
  test FAILS.
- [ ] **Step 3: Implement.**
  - Move the template.
  - Repoint the four readers, and repoint the two `authz.mqsc.j2` readers to
    `ansible/templates/mq/authz.mqsc.j2`.
  - In the three tests, point the `authz` and `mqoperator` template paths at
    `ansible/templates/mq/`. The `test_channel_failover_qmini.py` row for the
    `inter-qm.mqsc.j2` symlink stays; Task 6c removes it with the role.
  - Leave `roles/mq-pcmk-qmgr` and its symlinks in place, for Task 6c to
    delete.
- [ ] **Step 4: Validate.** Expected: green.
- [ ] **Step 5: Commit and hand off.** Run
  `vrg-commit --type refactor --scope ansible --message "read retained MQSC templates from templates/mq"`.

### Task 5b: Move `parse_drbd` into the RDQM collector

**Files:**

- Modify:
  - `components/mq-resiliency-observability/src/mqro/rdqmstate.py` (line 27
    import; define `parse_drbd` here)
  - `components/mq-resiliency-observability/src/mqro/clusterstate.py` (import
    `parse_drbd` from `mqro.rdqmstate`, until Task 6d deletes the file)
  - `ansible/roles/rdqm-state/tasks/main.yml:5`: the comment cites
    `clusterstate.parse_drbd`; it becomes `mqro.rdqmstate.parse_drbd`.
- Move tests: the `parse_drbd` cases from `tests/test_clusterstate.py` to
  `tests/test_rdqmstate.py`, with the `drbd_status.txt` fixture copied
  alongside the RDQM fixtures.

**Interfaces:**

- Produces: `mqro.rdqmstate.parse_drbd(text: str) -> <unchanged return type>`.
  The signature and behaviour are identical to today's. Task 6d deletes
  `clusterstate.py` and relies on this.

- [ ] **Step 1: Move the `parse_drbd` tests** into `test_rdqmstate.py`. Import
  from `mqro.rdqmstate`, and keep the assertions unchanged.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expected: the moved
  tests FAIL (`ImportError`).
- [ ] **Step 3: Move the function body** unchanged into `rdqmstate.py`, and flip
  the dependency so `clusterstate.py` imports from `rdqmstate`.
- [ ] **Step 4: Validate.** Expected: green.
- [ ] **Step 5: Commit and hand off.** Run
  `vrg-commit --type refactor --scope observability --message "own parse_drbd in the RDQM collector"`.
  This changes a component tree hash, so the observability-baking boxes rebake
  on their next build. That is accepted (spec §4.7).

### Task 6a: Stack-model cutover: topology, catalog and Python consumers

This is one PR on purpose. `versions.load_catalog` requires every
`INFRA_ROLES` entry to exist in `versions.yaml`, and `dashboard.py` raises
`DashboardError` on an unknown group, so the catalog, the topology and their
Python consumers must change together.

**Precondition (human-attested, before this PR merges):** on each host that has
built the lab (the macOS Lima VM and the cloud x86 VM), run
`uv run mqlab teardown pcmk-ubuntu` and `uv run mqlab teardown pcmk-drbd-ubuntu`
while the code still knows those stacks. After this task, `mqlab` refuses them
and any guest still defined is an orphan. Record the attestation in the PR
notes.

**Files:**

- Modify:
  - `lab/topology.yaml`: drop `stacks.pcmk-ubuntu` and
    `stacks.pcmk-drbd-ubuntu`; the nodes `pcmk-a1..3`, `pcmk-b1..3`,
    `pdrbd-a1..3`, `pdrbd-b1..3`, `san-a` and `san-b`; and the groups
    `pcmk_a`, `pcmk_b`, `pdrbd_a`, `pdrbd_b`, `san_a` and `san_b`.
  - `lab/versions.yaml`: drop the `pcmk`, `pdrbd` and `san` roles and the two
    stack entries.
  - `src/mqlab/versions.py:41`: `INFRA_ROLES = ("infra", "obs")`.
  - `lab/secret-names`: drop `pcmk_hacluster_password`,
    `pki-keyrpwd-pcmkapp` and `pki-keyrpwd-pdrbdapp`.
  - `lab/boxes/build-fatbox.sh:425`: drop the `pcmk-ubuntu | pdrbd-ubuntu`
    case.
  - `src/mqlab/parity.py`: drop the two `MATRIX` rows. The retained rows are
    unchanged, including `rdqm-rhel`, and the comment loses "reference
    backend".
  - `src/mqlab/stacks.py`: drop the `pacemaker-san` and `pacemaker-drbd`
    `_MECH_LABEL` entries.
  - `src/mqlab/clusterboard.py`:
    - `render_cluster_dashboard(topo, arm: str, ds_uid: str = "prometheus")`
      makes `arm` required. An arm that is neither a Native HA arm nor
      `rdqm-rhel` raises `ValueError(f"no cockpit board for arm {arm!r}")`.
    - Remove the PCMK branch and the `pcmk-ubuntu` entries in `_ARM_NAMES` and
      `_ARM_KIND`.
    - Keep `hero_tiles`, `integrity_panel`, `timeline_band`, `_COMPUTE_COLS`,
      `_STORAGE_COLS` and `_PCMK_SEL`, with their tests.
    - Delete the PCMK-only `log_row`, `perf_section`, `net_section`,
      `_CLUSTER_GROUPS` and `_CLUSTER_PLANES`, with their tests. If one turns
      out to have a retained caller, keep it and record why in the PR notes.
  - Every caller of `render_cluster_dashboard` passes the arm.
  - `src/mqlab/dashboard.py`:
    - Drop the `PCMK · A` and `PCMK · B` `ROWS` entries, the `net-san-a/b`
      entries in `NET_SECTIONS`, and the `/d/lab-pcmk-ubuntu-cluster` link
      (line 319).
    - `QMS` uses `NHAUAPP` with `SVCQM.NHAUAPP` and `NHAUAPP.SVCQM`.
  - `src/mqlab/watcherboard.py`: drop `_PACEMAKER_MECHANISMS` and the `mq_qm`
    branch of `_owner_resource`.
- Delete tests: `test_topology_pcmk.py`, `test_topology_pdrbd.py`, and the
  `test_versions.py` cases `test_san_is_an_infra_role` and the san-node tests.
- Modify tests: every fixture that seeds a removed stack. Rewrite each one
  against a retained stack; never delete a test that covers retained
  behaviour. The fixture users:
  - `test_cli_bootstrap`, `test_phases`, `test_cli_box`,
    `test_box_components` (on `box.FLEET["pcmk-ubuntu24"]`), `test_versions`,
    `test_cli_status`, `test_cli_qm`, `test_stacks`,
    `test_stack_provision_env`;
  - `test_clusterboard`, `test_dashboard`, `test_watcherboard`,
    `test_messagingboard`, `test_scrape`, `test_parity`, `test_cli_obs`,
    `test_cli_teardown`, `test_cli_dr`, `test_cli_vm`, `test_cli_hugepages`,
    `test_cli_commons`, `test_cli_pki`;
  - `test_inventory`, `test_roster`, `test_fleet`, `test_guestsel`,
    `test_vmstatus`, `test_instances`, `test_mqweb_endpoints`,
    `test_topology_integrity`, `test_dns`, `test_e2e`, `test_box_build`,
    `test_platforms`, `test_netstate`, `test_perfrun`, `test_lifecycle`,
    `test_net_cut`, `test_lab_secret`.
- Test: `tests/test_three_stacks.py` (new)

**Interfaces:**

- Produces:
  - `lab_stacks().keys() == {"rdqm-rhel", "nativeha-rhel-crr", "nativeha-ubuntu"}`;
  - `render_cluster_dashboard(topo, arm, ds_uid="prometheus")`, with `arm`
    required;
  - `dashboard.QMS` naming `NHAUAPP`.

  Tasks 6b, 6c and 7b rely on these.

- [ ] **Step 1: Write the failing tests** in `tests/test_three_stacks.py`:
  - `test_lab_stacks_are_the_three`: assert `set(lab_stacks())` equals the
    three retained names.
  - `test_catalog_has_only_retained_roles` (Review Focus 5): `load_catalog()`
    succeeds, and none of `san`, `pcmk` or `pdrbd` is a role.
  - `test_render_cluster_dashboard_rejects_removed_arm` (Review Focus 4):
    `pytest.raises(ValueError, match="pcmk-ubuntu")` on
    `render_cluster_dashboard({}, "pcmk-ubuntu")`.
  - `test_fleet_board_default_qm_is_nhau`: every `QMS` entry names `NHAUAPP`,
    or a channel built from it.
  - `test_parity_rows_are_the_three`: `set(parity.MATRIX)` equals the three.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expected: the new
  tests FAIL.
- [ ] **Step 3: Implement** the modifications above, then update the fixtures
  until the suite is green. Leave the Ansible playbooks and roles in place;
  Task 6c removes them. A playbook that nothing references is harmless for one
  PR.
- [ ] **Step 4: Validate.** Expected: green, with 100% branch coverage held.
- [ ] **Step 5: Commit and hand off.** Run
  `vrg-commit --type refactor --scope stacks --message "cut the stack model over to the three retained stacks"`.

### Task 6b: Remove Pacemaker-only CLI and model features; retire box names

**Files:**

- Modify:
  - `src/mqlab/cli.py`: remove the `qm destroy` command (line 2176), the
    `pcs` verb kind (lines 2159 and 2846), and the `refuse` verb shape if no
    retained stack declares it. Remove `qm_vip_ext=` (line 2027).
  - `src/mqlab/stacks.py`: remove `QmConfig.vip_ext` (line 72) and its
    parsing (line 194).
  - `src/mqlab/dns.py:111-112`.
  - `src/mqlab/retired_boxes.py`: add `pcmk-ubuntu24`, `pcmk-ubuntu26`,
    `pdrbd-ubuntu24`, `pdrbd-ubuntu26`, `san-ubuntu24` and `san-ubuntu26` to
    `RETIRED_BOX_NAMES`, and extend the module docstring to say these come
    from the stack removal (epic .github#399).
- Modify tests: the `qm destroy`, `pcs`-kind, `refuse` and `vip_ext` tests are
  deleted, and `test_retired_boxes` / `test_cli_box` gain the new names.

- [ ] **Step 1: Write the failing tests:**
  - `test_qm_destroy_not_a_command`: invoking `mqlab qm destroy` through the
    Typer test runner exits non-zero with "No such command".
  - `test_qmconfig_has_no_vip_ext`: `"vip_ext" not in QmConfig.__dataclass_fields__`.
  - `test_removed_boxes_are_retired`: the six names are in `RETIRED_BOX_NAMES`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expected: FAIL.
- [ ] **Step 3: Implement** the removals. The `mqlab vm provision-env` verb
  stays.
- [ ] **Step 4: Validate.** Expected: green.
- [ ] **Step 5: Commit and hand off.** Run
  `vrg-commit --type refactor --scope cli --message "drop Pacemaker-only verbs and fields; retire their boxes"`.

### Task 6c: Remove Pacemaker Ansible, lab scripts and SAN networks

**Files:**

- Delete playbooks under `ansible/`:
  - `site-pcmk.yml`, `site-pcmk-qm-down.yml`, `site-pcmk-authz-validate.yml`,
    `_pcmk-cluster-ha.yml`, `_pcmk-dr-replication.yml`;
  - `site-pdrbd.yml`, `site-pdrbd-qm-down.yml`, `site-pdrbd-dr-switch.yml`,
    `site-pdrbd-tls-verify.yml`, `_pdrbd-cluster-ha.yml`,
    `_pdrbd-dr-replication.yml`;
  - `bake-pcmk-ubuntu.yml`, `bake-pdrbd-ubuntu.yml`, `bake-san.yml`.
- Delete tasks and vars:
  - `ansible/tasks/pdrbd-dr-switch-source.yml`, `rpo0-drill-seed.yml` and
    `rpo0-drill-verify.yml`;
  - `ansible/vars/pdrbd-corosync.yml`, `pdrbd-crm-mon.yml` and
    `pdrbd-peers.yml`.
- Delete roles under `ansible/roles/`: `pcmk-cluster`, `pcmk-stonith`,
  `drbd-san`, `drbd9`, `iscsi-target`, `iscsi-initiator`, `pdrbd-qm`,
  `pdrbd-replication-tls`, `cluster-state` and `mq-pcmk-qmgr`.
- Delete `lab/` scripts and networks:
  - `pcmk-dr-cutover.sh`, `pcmk-dr-force.sh`, `pcmk-dr-seed-peer.sh`,
    `pcmk-qm-create.sh`, `drbd-degrade.sh` and `dr-provision.sh`;
  - `lab/networks/net-san-a.xml` and `net-san-b.xml`.
- Delete `ansible/templates/mq/inter-qm.mqsc.j2`: only `mq-pcmk-qmgr` (through
  its symlink) and `pdrbd-qm` read it.
- Modify:
  - `ansible/observability.yml`: drop the `cluster-state` role entry and the
    `pcmk_a`/`pcmk_b` terms in `alloy_tail_mqweb` (coupling 4).
  - `ansible/gather-versions.yml`: `hosts: rdqm_a`, with the
    corosync/pacemaker/drbd probes kept.
  - `ansible/roles/mq-exporter/defaults/main.yml`: remove the
    `mq_exporter_qm`, `mq_exporter_conn` and `mq_exporter_port` defaults.
    `mq_exporter_instance` stays derived from `mq_exporter_qm`.
  - `ansible/roles/mq-exporter/tasks/instance.yml`: as its first task, an
    `ansible.builtin.assert` that the three variables are defined, with
    `fail_msg` naming the missing one. It goes in `instance.yml`, the only
    file that reads them, so `build.yml` (which `site-obs.yml` runs alone) is
    not gated.
  - `src/mqlab/cli.py:1883-1887` and `lab/secret-names`: reword the comment
    and docstring for `provision-env`.
- Delete tests:
  - `test_pcmk_cluster_auth_gate.py`, `test_pcmk_cluster_totem.py`;
  - `test_pdrbd_dr_switch.py`, `test_pdrbd_dr_switch_run.py`,
    `test_pdrbd_qm_role.py`, `test_pdrbd_replication_tls.py`;
  - `test_drbd9_role.py`, `test_san_baked.py`.
- Modify tests:
  - `test_mqweb_always_on.py:38-41`;
  - `test_collector_roles_rewired.py:35,51-53`;
  - `test_textfile_dropzone.py:34`;
  - `test_ansible_os_vars.py:52,55`;
  - `test_version_token_guardrail.py:208`;
  - `test_event_monitor_dr_nodes.py`;
  - `test_channel_failover_qmini.py:125,128`: drop the
    `roles/mq-pcmk-qmgr/templates/inter-qm.mqsc.j2` row, and set the expected
    line count to the retained sites;
  - `test_acl_baked.py`, `test_client_roles_rewired.py` and
    `test_stage_off_tmpfs.py`, which read files this task deletes;
  - `test_mqweb_operator_authz.py:173`: drop the `pdrbd-qm` row.
- Any other test this removal breaks: update it until `vrg-validate` is green.
  Delete a test only when its whole subject is gone; never delete one that
  covers retained behaviour.
- Test: `tests/test_ansible_retained_only.py` (new)

**Interfaces:**

- Consumes: Task 5a's template paths and Task 6a's topology.
- Produces: no live Ansible file references a removed role or playbook.

- [ ] **Step 1: Write the failing tests** in `tests/test_ansible_retained_only.py`:
  - `test_observability_roles_all_exist` (Review Focus 2): every `role:` named
    in `ansible/observability.yml` exists under `ansible/roles/`.
  - `test_mq_exporter_requires_qm_conn_port` (Review Focus 3): the defaults
    file defines none of the three variables, the first task of
    `tasks/instance.yml` is an `assert` whose `that` covers all three, and
    `tasks/build.yml` has no such assert.
  - `test_no_removed_playbooks_or_roles`: none of the deleted paths exists.
  - `test_no_san_networks`: no `lab/networks/net-san-*.xml` exists.
  - `test_site_obs_passes_exporter_values`: every exporter item in
    `ansible/site-obs.yml` sets `qm`, `conn` and `port`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expected: FAIL.
- [ ] **Step 3: Implement** the deletions and modifications above.
- [ ] **Step 4: Validate.** Expected: green, and `ansible-lint` (run inside
  vrg-validate) is clean.
- [ ] **Step 5: Commit and hand off.** Run
  `vrg-commit --type refactor --scope ansible --message "remove the Pacemaker stacks' Ansible, scripts and SAN networks"`.

### Task 6d: Remove the PCMK collector and component defaults

**Files:**

- Delete:
  - `components/mq-resiliency-observability/src/mqro/clusterstate.py`;
  - `systemd/lab-cluster-state.service` and `.timer`;
  - the clusterstate-only fixtures: `iscsi_session.txt`,
    `stonith_history.txt`, `multipath.txt`, `crm_mon.xml`, and the original
    `drbd_status.txt` once Task 5b's copy exists;
  - `tests/test_clusterstate.py` (its `parse_drbd` cases already moved).
- Modify:
  - `components/mq-resiliency-observability/pyproject.toml`: drop the
    `lab-cluster-state` entry point.
  - `components/mq-resiliency-clients/src/mqrc/`: replace the PCMK-named
    defaults.
    - `app_requester.py` (lines 38 and 175): `pcmk-vip-*` and `PCMKAPP`.
    - `dr_flow.py` (lines 273-274) and `dr_responder.py` (line 195).
    - `svc_responder.py` (line 144): `PCMKSVC`.
    - `dr_forced.py` (line 62): `"pcmk-dr"`.
    - `reconnect_probe.py`.

    Each default is set by its role:
    - an application-side QM default becomes `NHAUAPP`;
    - a service-side `--qm` default becomes `SVCQM`, or the argument becomes
      required if no retained stack provides one;
    - a VIP default is removed, because no retained stack has a Pacemaker VIP,
      and the argument becomes required where it is still used;
    - `dr_forced`'s `"pcmk-dr"` label becomes a neutral label that names no
      stack;
    - where the systemd units always pass a value, the argument becomes
      required.
  - Purge the components' Pacemaker-stack mentions, in their code and READMEs
    (spec §4.6), so the rebake happens once.
- Modify tests: `test_dlq_probe`, `test_authz_probe` and
  `test_dr_entry_points`, using `NHAUAPP` strings.

- [ ] **Step 1: Write the failing tests:**
  - `test_no_cluster_state_entry_point`: the observability `pyproject.toml`
    `[project.scripts]` has no `lab-cluster-state`.
  - `test_clients_have_no_pcmk_defaults`: a text scan of `mqrc/*.py` finds no
    `PCMK`, `pcmk-` or `PDRBD`.
  - One test per changed default: the application-side default is `NHAUAPP`,
    the service-side default is `SVCQM` (or the argument is required), each
    removed VIP default leaves the argument required, and the `dr_forced`
    label names no stack.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expected: FAIL.
- [ ] **Step 3: Implement** the changes above.
- [ ] **Step 4: Validate.** Expected: green.
- [ ] **Step 5: Commit and hand off.** Run
  `vrg-commit --type refactor --scope components --message "remove the PCMK collector and PCMK-named client defaults"`.

### Task 7a: Purge Pacemaker-stack mentions: Ansible and `lab/`

**Files:** every live file under `ansible/` and `lab/` that `vrg-git grep -nE`
matches for `pcmk|pdrbd|pacemaker-san|pacemaker-drbd|PCMK|PDRBD|clusterstate`,
outside the §4.6 boundary. The known ones:

- `site-rdqm.yml`, `site-nativeha*.yml`, `site-distributed-shared.yml` and
  `_rdqm-cluster-ha.yml`;
- `site-rdqm-authz-validate.yml`, `site-nativeha-ubuntu-authz-validate.yml`,
  `site-qm-reload.yml`, `site-dns.yml:71` and `host-starvation.yml`;
- `group_vars/all/{apt,tls,authz}.yml`;
- `roles/{mq-install,mq-event-monitor,mq-diag-logging,mqweb,pki-distribute,rdqm-app-tls,entropy,app-requester,mq-qmgr}`;
- `lab/topology.yaml` comments (`boot_batch`, the nha-ubuntu block, the
  `stacks:` header, `svc.exporter_port`);
- `lab/scripts/lab-secret.sh` and `lab/boxes/_manifest-hash.sh:44`.

- Test: `tests/test_no_pcmk_mentions.py` (new; Task 7b extends it)

**Interfaces:**

- Produces: `tests/test_no_pcmk_mentions.py` with a parametrised scanner,
  `_hits(root: str) -> list[str]`, and an `ALLOW` set of
  `(path, reason)` boundary exceptions. Task 7b reuses both.

- [ ] **Step 1: Write the failing test** `test_ansible_and_lab_have_no_pcmk_mentions`.
  It scans `ansible/` and `lab/` for the pattern above,
  case-sensitively for `PCMK|PDRBD` and case-insensitively for the rest, and
  asserts no hit outside `ALLOW`. `ALLOW` holds only RDQM-boundary paths
  (`capture-rdqm-blueprint.yml` and the `rdqm-*` roles, where "pcmk" names
  RDQM's own Pacemaker) and `retired_boxes`-style history, each with its
  reason.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expected: FAIL,
  listing the hits.
- [ ] **Step 3: Rewrite each hit** so the comment stands on its own. Remove any
  special case that exists for a Pacemaker stack.
  - **`group_vars/all/tls.yml:4`** becomes "Verified live on the
    nativeha-ubuntu stack: channels run TLS 1.3". Task 9 performs that
    verification; if it fails, Task 9 fails.
- [ ] **Step 4: Validate.** Expected: green.
- [ ] **Step 5: Commit and hand off.** Run
  `vrg-commit --type docs --scope ansible --message "purge Pacemaker-stack mentions from retained Ansible"`.

### Task 7b: Purge Pacemaker-stack mentions: `src/mqlab` and `tests/`

**Files:** every live hit under `src/mqlab/` and `tests/`. The known ones:

- `cli.py`: help text at lines 3202 and 3432; comments at 592, 1763, 1788,
  1974, 2083-2085, 2826 and 3121.
- `dns.py`, `inventory.py`, `mqweb_endpoints.py` and `mqweb_probe.py`.
- `phases.py` (lines 188, 315 and 589), `perfrun.py:67`, `qmboard.py:561` and
  `box.py:1188`.
- `stacks.py` docstrings.
- `scripts/setup:43,47`, the stale `distributed-pcmk-ubuntu` example.
- Test fixtures that keep a `pcmk` string only as a name.

**Interfaces:**

- Consumes: Task 7a's `_hits` and `ALLOW`.

- [ ] **Step 1: Add the failing test**
  `test_src_tests_scripts_have_no_pcmk_mentions` over `src/mqlab`, `tests` and
  `scripts`.
  - **Extra `ALLOW` entries:**
    - `src/mqlab/retired_boxes.py` (historical box names);
    - the kept `clusterboard.py` panel helpers, `_PCMK_SEL` and the
      `cluster_iscsi_sessions` / `cluster_fence_count` queries (spec §4.1 and
      §6), each with its reason;
    - `tests/test_clusterboard.py`, which tests the kept panel helpers;
    - the guard tests, which name removed paths or stacks to assert they are
      gone: `tests/test_no_pcmk_mentions.py` itself,
      `tests/test_three_stacks.py`, `tests/test_ansible_retained_only.py`,
      `tests/test_retained_mqsc_templates.py` and
      `tests/test_retired_boxes.py`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expected: FAIL.
- [ ] **Step 3: Rewrite the hits.** Help examples name `nativeha-ubuntu`.
- [ ] **Step 4: Validate.** Expected: green.
- [ ] **Step 5: Commit and hand off.** Run
  `vrg-commit --type docs --scope mqlab --message "purge Pacemaker-stack mentions from mqlab and its tests"`.

### Task 7c: Track B live docs

**Files:**

- Modify `README.md`: the canonical starting stack becomes `nativeha-ubuntu`
  (lines 61 and 111-121).
- Modify under `docs/site/docs/`:
  - `index.md` (lines 11, 28 and 32) and `getting-started.md`;
  - `operate/index.md`, `architecture/index.md`, `reference/index.md` and
    `design-and-specs.md:31`;
  - `architecture/diagrams/02-inside-lab-vm.html` and `_diagram.css`
    (`.net-san`).
- Delete `docs/site/docs/architecture/diagrams/05-pacemaker-san.html`.
- Modify under `docs/development/`: `box-bake-manifest.md`, `box-model.md` and
  `perf-and-staging.md:168`.
- Delete `docs/reference/drbd-operations.md` and
  `rdqm-vs-vanilla-deviations.md`.
- Modify `docs/reference/rdqm-blueprint/README.md`: drop the row-by-row
  reading pointer.
- Modify the other `docs/reference/` files:
  - `rdqm-pacemaker-strategic-position.md`, `lab-bootstrap.md`,
    `dns-fqdn-inventory.md`;
  - `mqweb-endpoint-verification.md`, `mqweb-spike-findings.md`,
    `tls-coverage.md`;
  - `os-version-support-matrix.md`, `lab-gotchas.md`,
    `ansible-dependency-currency.md:66,307` and `rdqm-ha-cheatsheet.md:23`.
- Document the one-time host step `lab/scripts/net-down.sh net-san-a net-san-b`
  in `docs/development/operating-the-lab-from-a-dev-session.md`.
- Test: extend `tests/test_no_pcmk_mentions.py` with
  `test_live_docs_have_no_pcmk_stack_mentions`. It covers `README.md`,
  `docs/site/`, `docs/development/` (except `lab-bringup-capture.md`, which is
  already marked superseded) and `docs/reference/` (except `rdqm-blueprint/`).

- [ ] **Step 1: Add the failing test** and run `vrg-container-run -- vrg-validate`.
  Expected: FAIL.
- [ ] **Step 2: Edit and delete** as listed above. The getting-started stack
  walk-through uses `nativeha-ubuntu`.
- [ ] **Step 3: Validate.** Expected: green, with markdownlint clean and the
  site building.
- [ ] **Step 4: Commit and hand off.** Run
  `vrg-commit --type docs --scope docs --message "document the three retained stacks"`.
- [ ] **Step 5: Propose the memory update** to the human: the
  `dr-drbd-reference` memory entry points at the deleted `drbd-operations.md`.
  Write nothing without approval (CLAUDE.md "Memory").

### Task 8: Host cleanup on lab hosts (deployment, human-run)

This task makes no PR. Create it with `vrg-issue-create --kind deployment`; it
is blocked by Tasks 6b and 6c. The human runs it on each host that has built
the lab: the macOS Lima VM and the cloud x86 VM.

- [ ] **Precondition:** Tasks 6b and 6c are merged, and the host's checkout is
  on `develop` at or after both.
- [ ] **Step 0:** Run `virsh list --all`. Expected: no guest of a removed stack
  (`pcmk-*`, `pdrbd-*`, `san-*`). If one is listed, the Task 6a teardown
  precondition was missed; stop and report it, rather than removing it here.
- [ ] **Step 1:** Run `lab/scripts/net-down.sh net-san-a net-san-b`. Expected:
  each network is destroyed and undefined, or reported absent.
- [ ] **Step 2:** Run `uv run mqlab box gc`. Expected: the six retired box names
  are deregistered and their cache artifacts reclaimed.
- [ ] **Step 3:** Run `virsh net-list --all` and `vagrant box list`. Expected:
  no `net-san-*` network and no `pcmk-*`, `pdrbd-*` or `san-*` box.
- [ ] **Result:** an `Outcome: SUCCESS` comment per host, with the step 3
  output.

### Task 9: Cold rebuild of the three retained stacks (validation)

Create it with `vrg-issue-create --kind validation`; it is blocked by Tasks
6d, 7a, 7b and 8. Run it with `issue-validate`. The human operates the lab.

- [ ] **Precondition:** every blocker is closed, and both hosts are on
  `develop` at or after the last of them.
- [ ] **Step 1 (cloud x86):** cold-rebuild `rdqm-rhel` and
  `nativeha-rhel-crr`, each in one pass, with `uv run mqlab bootstrap <stack>`
  from a clean `build/` state for that stack. Then run `uv run mqlab qm e2e`
  on each. Expected: both pass.
- [ ] **Step 2 (either host):** do the same for `nativeha-ubuntu`.
- [ ] **Step 3:** on `nativeha-ubuntu`, confirm that a TLS channel negotiates
  TLS 1.3 (`DISPLAY CHSTATUS(*) SSLCIPH`, or the lab's TLS verify play).
  Expected: a TLS 1.3 cipher. This backs the Task 7a comment.
- [ ] **Step 4:** run the spec §6 leftover-mention grep,
  `vrg-git grep -nE 'pcmk|pdrbd|pacemaker-san|pacemaker-drbd|PCMK|PDRBD|iscsi|drbd-san|clusterstate'`,
  excluding the historical paths and the §4.3 RDQM paths. Expected: only the
  `ALLOW` entries from Tasks 7a and 7b, each listed with its reason in the
  result comment: the clusterboard panel helpers and their queries,
  `tests/test_clusterboard.py`, `src/mqlab/retired_boxes.py`, and the guard
  tests named in Task 7b.
- [ ] **Result:** `Outcome: SUCCESS` with the run-log paths for each stack, or
  `FAILURE`. On failure the task stays open.

---

## Track C — open-work triage

Both triage tasks follow the same procedure. They make no PR. Each is run with
the human, who confirms each batch before anything is closed or edited.

1. List the open issues in `logical-minds-foundry/mq-resiliency-lab-for-linux`
   and `logical-minds-foundry/.github`, including the intake and deferred
   queues (`vrg-queue intake`, `vrg-queue deferred`). Match titles and bodies.
2. Classify each match as **close** (not planned, with one line citing
   `.github#399` and the merged removal), **rescope** (edit it to drop the
   removed scope, and comment with what changed) or **reframe** (rescope to the
   develop-only lab, or close). Every reference names its repository.
3. Present the batches to the human, and apply each confirmed batch with
   `vrg-gh issue close <N> --repo <repo> --reason "not planned" --comment …`,
   `vrg-gh issue edit` or `vrg-intake-route … drop`, as fits the item's queue.
   An epic that closes as completed through the rollup is acceptable.
4. Close the triage task with a comment that lists each item and its outcome.

### Task 10a: Triage release-scope open work

Blocked by Task 3. Known items:

- `mq-resiliency-lab-for-linux#909` and `#1160`: close.
- `mq-resiliency-lab-for-linux#1157`: reframe.
- `.github#161` and `.github#351`: reframe.

### Task 10b: Triage Pacemaker-stack open work

Blocked by Tasks 6c and 6d. Known items, all in
`mq-resiliency-lab-for-linux` unless qualified:

- **Close:**
  - the pdrbd issues `#1684`, `#1668`, `#1664` and `#1660`;
  - the pcmk validations `#1583`, `#1581`, `#1430`, `#1318`, `#1315`,
    `#1314`, `#1313`, `#1300` and `#1299`;
  - `#1431`, `#1419`, `#1400`, `#1311`, `#1307`, `#1306`, `#1304` and `#175`;
  - `pcmk-rhel`: `#238` and `.github#118`;
  - the epic `.github#310` and its open children, including `.github#312`.
- **Close as answered by this epic:** `#1432`.
- **Rescope:** `#1627`, `#1123` and `.github#288`.

---

## Bookends

- **Documentation review** (`mq-resiliency-lab-for-linux#1712`): sweep
  `docs/site/` and the other live docs against the merged result. Spawn a
  per-repo doc task for any other repo that needs edits. Runs after Tasks 4,
  7c, 9, 10a and 10b.
- **Retrospective** (`.github#401`): run `epic-retrospective` once every other
  child is closed.
