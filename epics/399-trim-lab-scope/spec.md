# Epic 399 — Trim the lab to develop-only and three stacks

**Status:** spec · **Epic:** logical-minds-foundry/.github#399 ·
**Repo:** logical-minds-foundry/mq-resiliency-lab-for-linux ·
**Planning issue:** logical-minds-foundry/.github#398 (review history is in its
comments)

## 1. Context

The lab is a personal, experimental tool. Its only user is its author. Two large
bodies of live code exist for an audience that will never arrive, and both cost
maintenance on every change:

1. **Release packaging.** A tag-driven workflow builds a curated tarball, signs
   it with GPG and publishes a GitHub Release. The repo runs `library-release`
   branching (`develop` integrates, `main` releases), a semver `VERSION`, a CI
   version-bump gate, and mike-versioned site docs. The first real release
   (#909) was never cut: `origin/main` is 723 commits behind `develop`, and
   there are no tags and no GitHub Releases.
2. **Stacks beyond the three that matter.** `lab/topology.yaml` defines five
   stacks. Two of them, `pcmk-ubuntu` (Pacemaker with an iSCSI shared SAN) and
   `pcmk-drbd-ubuntu` (Pacemaker with DRBD), are open-source Pacemaker builds on
   Ubuntu. They showed that an RDQM-like topology can be assembled on Ubuntu.
   IBM does not support that, and no real deployment would choose it, so the
   work was academic.

This epic removes both from the live code. Afterwards the lab has one branch,
`develop`, which is cloned and run with `uv run mqlab`. It supports exactly three
stacks.

## 2. Goals, non-goals, anti-goals

**Goals**

- G1. `develop` is the lab. The only supported usage is cloning the repo and
  running `uv run mqlab` from the checkout. There are no versions, no tags, no
  releases and no `main` branch.
- G2. The supported stacks are exactly `rdqm-rhel`, `nativeha-rhel-crr` and
  `nativeha-ubuntu`. `pcmk-ubuntu` and `pcmk-drbd-ubuntu` are removed from the
  live code, along with everything that only they use.
- G3. The open-work backlog matches the new scope. Every open issue and epic
  that exists only for removed functionality is closed as not planned, with a
  reason.

**Non-goals**

- Rewriting history. Specs, plans, reports and retrospectives under `docs/specs`,
  `docs/plans`, `docs/reports` and `docs/superpowers`, and the epics in `.github`,
  stay as the record of how the lab evolved. They are not edited to remove
  mentions of the removed stacks or of releases.
- Any behaviour change to the three retained stacks.
- Closing the operator-verb gap on `rdqm-rhel`, which `parity.py` declares
  `NOT_YET`.
- Redesigning the dashboards. They are a work in progress with a planned
  rework of their own, which covers the hardwired QM panel among other things.
  This epic only removes the parts specific to the removed stacks. It does not
  add Native HA rows to the fleet board or rework its QM panel.

**Anti-goals** (each would count as a failure even if the removal technically
works)

- A retained stack that bootstraps differently, or not at all, because a shared
  piece was deleted along with a Pacemaker stack (§4.2).
- Dead code left behind: verbs no stack implements, branches keyed on a removed
  mechanism, defaults that name a removed QM.
- A silent fallback introduced in place of a removed default. Where a default
  named a removed stack or QM, the replacement is either a retained value
  chosen on purpose or a required argument that fails loud.

## 3. Track A — develop-only, no releases

### 3.1 Remove (whole files)

- `.github/workflows/release.yml`
- `RELEASE-KEY.asc`
- `tools/release_version_guard.py` and `tests/test_release_version_guard.py`
- `.gitattributes`: it holds only tarball-curation `export-ignore` lines. Also
  remove `tests/test_gitattributes_curation.py`.
- `VERSION`, `CHANGELOG.md`, `releases/`
- `docs/development/release-runbook.md`

### 3.2 Change

- **`vergil.toml` `[project]`:** set `versioning-scheme = "none"`,
  `branching-model = "docs-single-branch"` and `release-model = "none"`. The
  vergil-tooling `lib/config.py` accepts these values. `docs-single-branch`
  integrates on `develop` and has no other eternal branch. Remove the
  `[markdownlint]` ignores for the changelog and release pages.
- **`.github/workflows/ci.yml`:** remove the `version` job (`ci-version-bump`),
  which compares against `origin/main`, and the changelog-staging comment. This
  lands in the same PR as the `vergil.toml` change.
- **`.github/workflows/cd.yml`:** trigger on `develop` only.
- **`docs/site/mkdocs.yml`:** remove the `mike` version provider and the
  Releases nav section. Docs then deploy with `mkdocs gh-deploy --force`, which
  replaces `gh-pages`. The old `/latest/` and `/dev/` URLs stop resolving; that
  is accepted.
- **`.gitignore`:** remove the changelog and releases lines. If they sit in the
  vergil-managed block and `vrg-gitignore-sync` would restore them, leave them;
  they are harmless.
- **`pyproject.toml`:** hatchling needs a `version`. It stays as a fixed
  placeholder with no meaning, documented as such, and nothing reads it.
- **`src/mqlab/paths.py` `repo_root()`:** keep the `MQLAB_REPO_ROOT` override,
  which about 200 tests use as a test hook. Reword its docstring so it no longer
  describes an installed mode.
- **`tests/test_no_pymqi_refs.py`:** drop the `CHANGELOG.md` and `releases/`
  references.
- **Live docs** (rewritten for clone-and-run with `uv run mqlab`):
  - `README.md`: remove the "Download and verify the release" section, the gpg
    prerequisite, the "signed" framing and the runbook link.
  - `CLAUDE.md`: the project overview and branching-model lines.
  - `docs/site/docs/index.md`, `getting-started.md` and `develop.md`.
  - `docs/development/operating-the-lab-from-a-dev-session.md`.

No `mqlab` Python code exists only for the release path. Every verb already
needs a git checkout (`buildenv`, `component.tree_hash`, the box scripts), so a
tree unpacked from a tarball could never have run. Checkout-only behaviour after
this track is exactly what runs today.

### 3.3 Not release code (keep)

These look like release code but serve other purposes:

- `artifact.py` and `manifest.py`: IBM MQ media.
- `runtime.py`: the guest Python.
- `mqexporter.py`, `logsearch.py` and `box.py`.
- `component install --version` and the on-guest
  `/opt/logical-minds-foundry/<name>/releases/` layout.
- `components/*/pyproject.toml` versions.
- Third-party apt-repository GPG keys.

"Main checkout" and "main worktree" in the docs and scripts mean git's main
worktree, not the `main` branch.

### 3.4 Published state (human steps)

Once the profile change has merged and CI is green on `develop` without the
version gate:

- Delete the `main` branch on GitHub, and any ruleset or branch protection that
  targets it. `ops.yml` reconciles GitHub configuration from the profile daily;
  run it once by hand to confirm it no longer expects `main`.
- Delete the repo secrets `RELEASE_GPG_PRIVATE_KEY` and
  `RELEASE_GPG_PASSPHRASE`.
- No tags or GitHub Releases exist, so there is nothing else to delete.

These steps are public and irreversible, and the agent tools cannot perform
them. The human runs them.

### 3.5 Profile fitness (verify first)

`docs-single-branch` is named for documentation repos. Before the profile change
merges, confirm against vergil-tooling that the model is valid for
`repository-type = "tooling"` and that commit, finalize, CI, CD and the
ops-github-config ruleset reconciliation all behave correctly with it. If
vergil-tooling cannot express a develop-only code repo, stop and raise it with
the human; that becomes a vergil-tooling issue, not a lab workaround.

## 4. Track B — three stacks

### 4.1 Remove

The full inventory is in the plan. In outline:

- **Topology and catalog:**
  - `stacks.pcmk-ubuntu` and `stacks.pcmk-drbd-ubuntu`.
  - Nodes `pcmk-*`, `pdrbd-*`, `san-a` and `san-b`, and their groups.
  - Networks `net-san-a` and `net-san-b`.
  - The `pcmk`, `pdrbd` and `san` roles and stack entries in
    `lab/versions.yaml`.
  - `"san"` in `versions.py` `INFRA_ROLES`, in the same commit as the catalog
    change.
  - `lab/secret-names` entries for the removed stacks.
  - The `build-fatbox.sh` bake cases.
- **Ansible:**
  - `site-pcmk*`, `site-pdrbd*`, `_pcmk-*` and `_pdrbd-*`.
  - `bake-pcmk-ubuntu`, `bake-pdrbd-ubuntu` and `bake-san`.
  - The pdrbd-only tasks and vars.
  - Roles `pcmk-cluster`, `pcmk-stonith`, `drbd-san`, `drbd9`, `iscsi-target`,
    `iscsi-initiator`, `pdrbd-qm`, `pdrbd-replication-tls`, `cluster-state` and
    `mq-pcmk-qmgr`. Remove `mq-pcmk-qmgr` only after the move in §4.2.
- **`lab/` scripts:** `pcmk-dr-*.sh`, `pcmk-qm-create.sh`, `drbd-degrade.sh`
  and `dr-provision.sh`.
- **Python:**
  - Drop the `parity.py` rows and the `stacks.py` `_MECH_LABEL` entries.
  - In `clusterboard.py`, remove the PCMK board; the generic helpers stay.
  - In `dashboard.py`, remove the PCMK rows, the PCMK cockpit link and the
    net-san planes. Re-default the hardwired `QMS` to `NHAUAPP` and its
    `SVCQM` channels (§4.2).
  - In `watcherboard.py`, remove the Pacemaker branch.
  - In `cli.py`, remove the `pcs` verb kind.
  - Remove `QmConfig.vip_ext` and its `cli.py` and `dns.py` plumbing.
  - Remove `mqlab qm destroy`, which no retained stack implements (my call).
  - Remove the `refuse` verb shape if no retained stack declares it.
- **Components:**
  - `mq-resiliency-observability`: remove `mqro/clusterstate.py`, its
    `lab-cluster-state` entry point and its systemd units, after the move in
    §4.2.
  - `mq-resiliency-clients`: replace default values that name PCMK QMs or VIPs
    with retained values or required arguments.
- **Tests:** delete the pcmk-, pdrbd-, drbd9- and san-only tests. Update every
  fixture that seeds a removed stack, since `conftest.py` loads the real
  `versions.yaml`.
- **Live docs:**
  - `README.md`: the canonical starting stack.
  - `docs/site/**`: delete diagram `05-pacemaker-san.html` and edit `02`.
  - `docs/development/box-bake-manifest.md`, `box-model.md` and
    `perf-and-staging.md`.
  - `docs/reference/`:
    - Delete `drbd-operations.md` and `rdqm-vs-vanilla-deviations.md`, both of
      which describe only removed stacks (my call; git keeps them).
    - Edit the other references to the removed stacks.
  - The component READMEs.

### 4.2 Couplings to cut before deleting

Three couplings tie retained stacks to Pacemaker code. Each must be moved before
its source is deleted, in the same PR or an earlier one.

1. **`mqoperator.mqsc.j2`** exists only in `roles/mq-pcmk-qmgr/templates/`. The
   SVC counterparty QM (`roles/mq-qmgr`), `site-rdqm.yml`, `site-nativeha.yml`
   and `site-nativeha-ubuntu.yml` all load it. Move it to
   `ansible/templates/mq/` and repoint them. Also repoint the `authz.mqsc.j2`
   readers in `site-rdqm.yml` and `site-nativeha-ubuntu.yml` to the identical
   copy already in `templates/mq/`.
2. **`mqro/rdqmstate.py` imports `parse_drbd`** from `mqro/clusterstate.py`.
   Move `parse_drbd` and its tests into the RDQM collector, or into a shared
   module, before `clusterstate.py` goes.
3. **Defaults that name PCMK.** These need replacements:
   - `roles/mq-exporter` defaults to `PCMKAPP`;
   - `dashboard.py` `QMS` hardcodes the `PCMKAPP` channels;
   - `render_cluster_dashboard` defaults to `arm="pcmk-ubuntu"`;
   - `gather-versions.yml` targets `pcmk_a:rdqm_a`.

   Where a QM default must exist, it is `NHAUAPP`, the `nativeha-ubuntu` QM.
   That applies to the `mq-exporter` default and to `dashboard.py` `QMS` and its
   channels. The dashboard is re-defaulted, not redesigned; replacing the
   hardwired QM belongs to the later dashboard rework. The cockpit arm becomes a
   required argument, and an unknown arm raises. `gather-versions.yml` targets
   `rdqm_a`.

### 4.3 Keep

- RDQM's own Pacemaker and DRBD: roles `rdqm-install`, `rdqm-ha`, `rdqm-state`
  and `rdqm-replication-tls`; `bake-mq-rdqm.yml`; `capture-rdqm-blueprint.yml`;
  `docs/reference/rdqm-blueprint/**`, including its `pcmk-*.txt` captures.
- The `corosync`, `pacemaker` and `drbd` version probes in
  `gather-versions.yml`.
- `bake-mq-ubuntu.yml`, which bakes the `mq-client` box behind `svc-sim`,
  `app-client` and `mon-probe`.
- Networks `net-hb-*`, `net-wan`, `net-ext` and `net-data-*`, and `extra_disk`.
- `site-distributed-shared.yml` and the shared SVC QM.
- `retired_boxes.py`'s existing `pcmk-ubuntu` entry, which is a pre-rename box
  name.

### 4.4 New defaults (stack and docs)

- **Canonical starting stack in the README and getting-started:**
  `nativeha-ubuntu`, the only retained stack that needs no RHEL subscription and
  no RDQM entitlement (my call).
- **CLI help examples** name a retained stack.
- **`parity.py`** no longer describes a "reference backend". It declares each
  retained stack's capability.

### 4.5 Host cleanup

Add the orphaned box names (`pcmk-ubuntu24/26`, `pdrbd-ubuntu24/26` and
`san-ubuntu24/26`) to `RETIRED_BOX_NAMES`, so `mqlab box gc` deregisters them
from hosts that built them.

### 4.6 Purge Pacemaker-stack mentions from retained code

The removal is thorough. Outside the historical paths there are about 2,600
mentions of `pcmk`, `pdrbd`, `pacemaker-san` and `pacemaker-drbd` in about 200
files, and most sit in code that stays. Typical examples are comments such as
"mirrors `site-pcmk.yml`'s guard" in `site-rdqm.yml`, `site-nativeha*.yml`,
`site-distributed-shared.yml`, `roles/mq-qmgr` and `group_vars/all/authz.yml`,
and default values such as `PCMKAPP` in the components.

- Rewrite every comment, docstring, help text and example that explains a
  retained thing in terms of a removed stack, so it stands on its own.
- Remove every special case in retained code that exists for a Pacemaker
  stack: a branch, a guard, a mechanism check or a fallback.
- Replace default values that name a removed stack, QM or VIP (§4.2).
- **The boundary:** Pacemaker, Corosync and DRBD as RDQM's own underlying
  technology are real and stay. Examples are `crm_mon` and `drbdsetup` in the
  RDQM collector, the `corosync`/`pacemaker`/`drbd` version probes, and RDQM's
  blueprint captures. What goes is any reference to the open-source Pacemaker
  *stacks*.

This is mechanical but sizable work, and it is split into PR-sized tasks by
area: Ansible, `src/mqlab`, components, tests and live docs.

### 4.7 Rebuild cost

Component edits change the component tree hashes, which feed the box manifest
hash. The `mq-rdqm`, `mq-nativeha` (both families) and `mq-client` boxes will
rebake. That is accepted: §6 requires a cold rebuild of every retained stack
anyway.

## 5. Track C — open-work triage

Sweep every open issue and epic in `logical-minds-foundry/mq-resiliency-lab-for-linux`
and `logical-minds-foundry/.github`, including the intake and deferred queues.
Match on bodies as well as titles. Classify each item:

- **Close as not planned:** the item exists only for removed functionality.
  Each close carries a one-line reason citing this epic. Title matches alone
  include about 36 candidates. Examples:
  - the pdrbd defect and feature issues;
  - the pcmk W1/W4 and DR measurement validations;
  - #909 (first signed release) and #1160 (download path);
  - `pcmk-rhel` (#238, #118);
  - the pacemaker-drbd arm epic `.github#310` and its open children.

  #1432 ("pacemaker-san: retire, demote or extract") is closed as answered by
  this epic.
- **Rescope:** the item also covers retained stacks, for example #1627 (shared
  roles), #1123 (the `SET AUTHREC` migration) and #288 (cross-stack bootstrap
  parity). Edit the item to drop the removed scope, and comment with what
  changed.
- **Reframe:** the item was justified by a public release, for example `.github`
  epics #161 (site docs for public release) and #351 (defect reporting batched
  for the public release). Either rescope it to the develop-only lab or close
  it, decided item by item.

Open epics are closed the way their own framework requires. A finite epic whose
remaining children are all closed as not planned is closed as not planned, with
a comment pointing to this epic. Its unwritten retrospective is not authored.

The task presents its classification to the human before it closes anything.
Closes are outward-facing, so the human confirms each batch.

## 6. Ordering and acceptance

- Track A's profile change (§3.2) lands before the `main` deletion (§3.4).
- Each §4.2 coupling is cut before or with the deletion it guards.
- Track C runs after Tracks A and B have merged, so each close can cite the
  removal that made it moot.
- **Acceptance:** `vrg-container-run -- vrg-validate` is green. A full cold
  rebuild then brings up each of the three retained stacks in one pass, and
  `qm-e2e` passes on each. That cold rebuild is a validation task.
  - **Where it runs:** the RHEL stacks build only on the cloud x86 VMs, so the
    rebuild runs there. `nativeha-ubuntu` may run on either host.
  - **Baseline:** `rdqm-rhel` last passed a cold rebuild on 2026-10-14 (#1475,
    #1497), so the gate is reachable today.
  - **Leftover-mention check:** a `vrg-git grep` for `pcmk`, `pdrbd`,
    `pacemaker-san`, `pacemaker-drbd`, `PCMK`, `PDRBD` and the SAN role names,
    excluding the historical paths and the RDQM-owned paths in §4.3, returns
    nothing beyond the §4.6 boundary. Each remaining hit is listed with its
    reason in the validation comment.

## 7. Left to the builder

- How to split each track into PRs, within the plan's task-sizing rule.
- Where `parse_drbd` lands: in `rdqmstate.py` or in a small shared module.
- Whether the `refuse` verb shape survives. It goes if no retained stack
  declares it.
- The wording of the rewritten live docs.
