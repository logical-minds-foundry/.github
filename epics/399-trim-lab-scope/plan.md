# Trim the lab to develop-only and three stacks: implementation plan

**Status:** draft, in design review · **Epic:** logical-minds-foundry/.github#399 ·
**Planning issue:** logical-minds-foundry/.github#398 (review history is in its
comments)

**Goal:** remove the release machinery and the `pcmk-ubuntu` and
`pcmk-drbd-ubuntu` stacks from the live code, so `develop` is the lab and three
stacks remain: `rdqm-rhel`, `nativeha-rhel-crr` and `nativeha-ubuntu`. Then
bring the open backlog in line.

**Spec:** [`spec.md`](spec.md). It holds the full inventory and the reasons;
this plan only orders the work into PRs.

## How the work is done

This is deletion. Each code task is: delete what the spec lists, fix what
breaks, and run `vrg-container-run -- vrg-validate` until it is green.

- **Tests.** Delete tests whose whole subject is gone. Update tests that cover
  retained behaviour so they run against a retained stack. Add a test only
  where retained code changes behaviour: the three tasks that do so name the
  test. Never add a test that checks a removed thing stays removed.
- **Mentions.** Each task rewrites the Pacemaker-stack mentions (spec §4.6) in
  the files it already touches, plus the files listed for it. The leftover
  check is a one-time grep in Task 12.
- **Component tests** run only in `mqlab component build`, not in
  `vrg-validate`. Fix the component tests the task names; the Task 12 rebake
  catches anything else.
- **Retained stacks** keep their behaviour. A removed default becomes a
  retained value or a required argument, never a silent fallback.
- **Workflow pushes.** A delta touching `.github/workflows/**` makes GitHub
  refuse the agent push. `report-ready` records the state anyway, and the
  human's `vrg-submit-pr` pushes.

## Delivery

All tasks are in `logical-minds-foundry/mq-resiliency-lab-for-linux`.

| Task | Title | Kind | Blocked by |
|---|---|---|---|
| 1 | Profile to develop-only | impl | — |
| 2 | Ruleset apply, delete `main` and release secrets | deployment (human) | 1 |
| 3 | Remove the release machinery; clone-and-run docs | impl | 1 |
| 4 | Remove the CI `version` job | impl | 2 |
| 5 | Tear down the removed stacks' guests | deployment (human) | — |
| 6 | Cut the couplings | impl | — |
| 7 | Stack model and `mqlab` | impl | 5, 6 |
| 8 | Ansible, `lab/` scripts and SAN networks | impl | 7 |
| 9 | Components | impl | 6, 8 |
| 10 | Live docs for three stacks | impl | 3, 8, 9 |
| 11 | Host cleanup | deployment (human) | 7, 8 |
| 12 | Cold rebuild of the three stacks | validation | 9, 10, 11 |
| 13 | Triage: release-scope open work | triage (human) | 3 |
| 14 | Triage: Pacemaker-stack open work | triage (human) | 8, 9 |
| — | Documentation review (`mq-resiliency-lab-for-linux#1712`) | bookend | 4, 12, 13, 14 |
| — | Retrospective (`.github#401`) | bookend | all |

The triage tasks make no PR. The human confirms every batch of closes.

## Track A: develop-only

### Task 1: Profile to develop-only

- `vergil.toml` `[project]`: `versioning-scheme = "none"`,
  `branching-model = "docs-single-branch"`, `release-model = "none"`.
- `.github/workflows/cd.yml`: trigger on `develop` only.
- `docs/site/mkdocs.yml`: remove `extra.version` (the `mike` provider).
- `pyproject.toml`: remove `mike` from the docs group, then `uv lock`.
- Keep the `ci.yml` `version` job; it passes as a no-op (spec §3.2 step 1).
- **Profile fitness first (spec §3.5).** Before the PR is ready, confirm and
  cite in the PR notes:
  - vergil-tooling `lib/config.py` accepts `docs-single-branch` with
    `repository-type = "tooling"`;
  - `vrg_commit.py` and `vrg_finalize_pr.py` accept `feature/*` with `develop`
    as the only eternal branch;
  - the vergil-actions version-bump job is a no-op under
    `release-model = "none"`;
  - the vergil-actions docs deploy (`actions/cd/docs/deploy/action.yml`) uses
    `mkdocs gh-deploy --force` when `versioning-scheme = "none"`;
  - vergil-tooling `lib/github_config.py` drops `version / version-bump` from
    the CI-gates ruleset when `release-model = "none"`.

  If any fails, stop and raise it with the human as a vergil-tooling issue.

### Task 2: Ruleset apply, delete `main` and release secrets (human)

No PR. The human runs it under their own GitHub login once Task 1 has merged
and the docs site has deployed without a version selector.

1. `vrg-github-repo-config apply` from the lab checkout, then run `ops.yml`;
   the audit reports no drift.
2. Delete the `main` branch. Delete only a ruleset or protection whose sole
   target is `main`; the shared rulesets stay (spec §3.4).
3. Delete the secrets `RELEASE_GPG_PRIVATE_KEY` and `RELEASE_GPG_PASSPHRASE`.
4. Run `ops.yml` again: no drift.

Closes with `Outcome: SUCCESS` and the audit-run URLs.

### Task 3: Remove the release machinery; clone-and-run docs

- Delete the files in spec §3.1.
- `docs/site/mkdocs.yml`: remove the Releases `nav` section (about lines
  58–62), or the strict docs build fails.
- The other changes in spec §3.2: the `[markdownlint]` ignores, `.gitignore`,
  the `pyproject.toml` version comment, the `paths.py` docstring and
  `tests/test_no_pymqi_refs.py`.
- Rewrite `README.md`, `CLAUDE.md`, `docs/site/docs/index.md`,
  `getting-started.md`, `develop.md` and
  `docs/development/operating-the-lab-from-a-dev-session.md` for clone, then
  `./scripts/setup`, then `uv run mqlab …`. `CLAUDE.md` names
  `docs-single-branch`.

### Task 4: Remove the CI `version` job

Remove the `version` job and the changelog-staging comment from
`.github/workflows/ci.yml`, after Task 2's `apply` has dropped the required
check.

## Track B: three stacks

### Task 5: Tear down the removed stacks' guests (human)

No PR. On each host that has built the lab (the macOS Lima VM and the cloud x86
VM), before Task 7 merges: `uv run mqlab teardown pcmk-ubuntu` and
`uv run mqlab teardown pcmk-drbd-ubuntu`. Afterwards `mqlab` no longer knows
these stacks. Closes with `Outcome: SUCCESS` per host.

### Task 6: Cut the couplings

Spec §4.2. Nothing is deleted yet, and every stack still works afterwards.

- Move `roles/mq-pcmk-qmgr/templates/mqoperator.mqsc.j2` to
  `ansible/templates/mq/`, leaving a symlink at the old path for the Pacemaker
  roles until Task 8. Repoint `roles/mq-qmgr`, `site-rdqm.yml`,
  `site-nativeha.yml` and `site-nativeha-ubuntu.yml`, and their `authz.mqsc.j2`
  readers. Repoint the `authz` and `mqoperator` rows of `test_mqmon_authz.py`,
  `test_mqweb_operator_authz.py` and `test_channel_failover_qmini.py`.
- Move `parse_drbd` into `mqro/rdqmstate.py`. `clusterstate.py` imports it
  from there until Task 9. Move its tests to
  `components/mq-resiliency-observability/tests/test_rdqmstate.py`, and fix
  that file's existing `from mqro.clusterstate import parse_drbd` (line 276).
  Reword the `rdqmstate.py` docstring and the
  `ansible/roles/rdqm-state/tasks/main.yml:5` comment, which cite
  `clusterstate`.
- `ansible/observability.yml`: drop the `cluster-state` role entry and the
  `pcmk_a`/`pcmk_b` terms in `alloy_tail_mqweb`.
- `ansible/gather-versions.yml`: `hosts: rdqm_a`.

### Task 7: Stack model and `mqlab`

One PR, because `versions.load_catalog` and `dashboard.py` fail loud on a
missing role or group.

- `lab/topology.yaml`, `lab/versions.yaml`, `INFRA_ROLES`, `lab/secret-names`
  and the `build-fatbox.sh` case: drop the two stacks, their nodes, the SAN
  nodes, their groups and their roles.
- `parity.py`, `stacks.py` `_MECH_LABEL`, `watcherboard.py`: drop the removed
  rows and branches.
- `clusterboard.py`: `render_cluster_dashboard(topo, arm, ds_uid="prometheus")`
  with `arm` required; an unknown arm raises
  `ValueError(f"no cockpit board for arm {arm!r}")`. Remove the PCMK board and
  the PCMK-only helpers (spec §4.1); keep the named panel helpers.
  **New test:** an unknown arm raises `ValueError`.
- `dashboard.py`: drop the PCMK rows, the cockpit link and the net-san planes;
  `QMS` uses `NHAUAPP`.
- Dead features: `qm destroy`, the `pcs` verb kind, the `refuse` shape (only
  `pcmk-ubuntu` declares it) and `QmConfig.vip_ext` with its `cli.py` and
  `dns.py` plumbing.
- `retired_boxes.py`: add `pcmk-ubuntu24/26`, `pdrbd-ubuntu24/26` and
  `san-ubuntu24/26`.
- Rewrite the Pacemaker-stack mentions in `src/mqlab/`, `tests/` and
  `scripts/setup`. Help examples name `nativeha-ubuntu`.
- Tests: delete `test_topology_pcmk.py`, `test_topology_pdrbd.py` and the
  tests of the removed features. Rewrite every fixture that seeds a removed
  stack against a retained one.

### Task 8: Ansible, `lab/` scripts and SAN networks

- Delete the playbooks, tasks, vars and roles in spec §4.1, including
  `mq-pcmk-qmgr` with Task 6's symlink, and
  `ansible/templates/mq/inter-qm.mqsc.j2`.
- Delete the `lab/` scripts in spec §4.1, and `lab/networks/net-san-a.xml` and
  `net-san-b.xml`.
- `roles/mq-exporter`: remove the `mq_exporter_qm`, `_conn` and `_port`
  defaults. The first task of `tasks/instance.yml` asserts the three are
  defined, naming the missing one. Delete `tasks/main.yml` if nothing calls the
  role without `tasks_from`. **New test:** the assert is the first task of
  `instance.yml` and covers all three.
- `provision-env`: reword the `cli.py` docstring and the `lab/secret-names`
  comment.
- `group_vars/all/tls.yml`: drop the "verified live on the pcmk-ubuntu arm"
  claim, and state only the setting. Task 12 verifies it, and its follow-up
  restores a citation.
- Rewrite the Pacemaker-stack mentions in the rest of `ansible/` and `lab/`.
- Tests: delete the pcmk, pdrbd, drbd9 and SAN tests; drop the Pacemaker rows
  from the others, including the `inter-qm` row of
  `test_channel_failover_qmini.py` and the `pdrbd-qm` row of
  `test_mqweb_operator_authz.py`.

### Task 9: Components

- `mq-resiliency-observability`: delete `mqro/clusterstate.py`, its
  `lab-cluster-state` entry point, its systemd units, its test file and its
  fixtures. Update `tests/test_selfcheck.py`, which lists it.
- `mq-resiliency-clients`: replace each PCMK-named default by its role
  (spec §4.2):
  - application-side QM: `NHAUAPP`;
  - service-side `--qm`: `SVCQM`, or required if no retained stack provides one;
  - VIP defaults: removed, the argument becomes required where still used;
  - the `dr_forced` label: neutral, naming no stack.

  **New test:** for each argument that becomes required, omitting it fails.
- Rewrite the Pacemaker-stack mentions in both components, their READMEs and
  their systemd units, so they rebake once.

### Task 10: Live docs for three stacks

Spec §4.1 live docs and §4.4: `README.md` and `getting-started.md` start with
`nativeha-ubuntu`. Delete `05-pacemaker-san.html`, `drbd-operations.md` and
`rdqm-vs-vanilla-deviations.md`. Edit the rest of `docs/site/docs/` (including
`operate/guest-components.md`), `docs/development/` and `docs/reference/`
outside `rdqm-blueprint/`. Document the one-time
`lab/scripts/net-down.sh net-san-a net-san-b` host step. After merge, propose to
the human the update for the `dr-drbd-reference` memory, which points at the
deleted `drbd-operations.md`; write nothing without approval.

### Task 11: Host cleanup (human)

No PR. On each host, once Tasks 7 and 8 have merged:

1. `virsh list --all` shows no `lab_pcmk-*`, `lab_pdrbd-*` or `lab_san-*`
   guest. If one remains, report it instead of removing it by hand.
2. `lab/scripts/net-down.sh net-san-a net-san-b`.
3. `uv run mqlab box gc`.
4. `virsh net-list --all` and `vagrant box list` show no `net-san-*` network
   and no `pcmk-*`, `pdrbd-*` or `san-*` box.

Closes with `Outcome: SUCCESS` per host.

### Task 12: Cold rebuild of the three stacks (validation)

Run with `issue-validate`; the human operates the lab.

1. On cloud x86: cold-rebuild `rdqm-rhel` and `nativeha-rhel-crr`, each in one
   pass, then `uv run mqlab qm e2e` on each.
2. On either host: the same for `nativeha-ubuntu`.
3. On `nativeha-ubuntu`: record the TLS version and cipher a channel
   negotiates (`DISPLAY CHSTATUS(*) SSLCIPH`) and the CHLAUTH state. File a
   follow-up task that rewrites the `group_vars/all/tls.yml` comment from those
   observations, blocking the documentation review.
4. Once: run the spec §6 leftover-mention grep and list each hit with its
   reason in the result.

Closes with `Outcome: SUCCESS` and the run-log paths, or stays open on
`FAILURE`.

## Track C: triage

Both tasks follow spec §5. List the open issues in the lab repo and `.github`,
including the intake and deferred queues. Classify each match as close, rescope
or reframe, naming its repository. Show the batches to the human, and apply each
confirmed batch with `vrg-gh issue close|edit` or `vrg-intake-route`. Close the
task with a list of every item and its outcome.

- **Task 13 (release scope):** `mq-resiliency-lab-for-linux#909` and `#1160`
  close; `mq-resiliency-lab-for-linux#1157`, `.github#161` and `.github#351`
  are reframed.
- **Task 14 (Pacemaker stacks):** the items in spec §5, including `.github#310`
  and its children.

## Bookends

- **Documentation review** (`mq-resiliency-lab-for-linux#1712`): sweep the live
  docs against the merged result.
- **Retrospective** (`.github#401`): `epic-retrospective`, once every other
  child is closed.
