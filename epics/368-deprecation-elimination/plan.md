# Plan: Deprecation-warning elimination and Ansible dependency gap

> **For agentic workers:** each implementation task runs through `issue-implement`, which writes that task's own test-first steps. This plan fixes the order, the gates, the routing and the definition of done.

- **Epic:** [logical-minds-foundry/.github#368](https://github.com/logical-minds-foundry/.github/issues/368). **Spec:** [spec.md](spec.md).
- **Form:** compact. Each task's files and acceptance are in this plan's task notes
  and repeated in its issue body.

Repo shorthand: `lab#N` = `logical-minds-foundry/mq-resiliency-lab-for-linux#N`.

## Global constraints

- **One validation command.** Every PR runs `vrg-container-run -- vrg-validate` and
  must be green.
- **No dedicated validation build** (spec §3.3). Each task verifies in its own PR.
  Runtime confirmation comes from later rebuilds done for other work. A miss is
  patched as a small follow-up task under this epic while it is open, or as an
  ad-hoc task after it closes.
- **No dependency upgrades** in this epic (spec §4). Task GAP measures and
  recommends only.
- **Cite sources** for every version and support claim in GAP: PyPI, Galaxy,
  GitHub releases, the ansible-core support matrix. Label each claim *data* or
  *judgment*.

## Task table

| ID | Issue | Kind | Run via | Blocked by | Done when |
|---|---|---|---|---|---|
| DOC | .github#369 | docs | this PR | — | This spec and plan merged |
| F1 | lab#136 (rescoped) | code | `issue-implement` | DOC | No top-level fact reference in `ansible/`; allowlist guard test green; `inject_facts_as_vars = False` set (PR merged) |
| F2 | lab#1644 | code | `issue-implement` | DOC | ansible-lint shows no `bool` coercion warning (PR merged) |
| F3 | lab#1645 | code | `issue-implement` | DOC | CI's `version / version-bump` shows no `run-release` warning (PR merged) |
| F4 | lab#1646 | code | `issue-implement` | DOC | `docs` group pins `mkdocs>=1.6,<2`; lock unchanged at 1.6.1; banner turned off by a supported means, or recorded as exempt (PR merged) |
| F5 | lab#1648 | code | `issue-implement` | DOC | `release.yml` pins an explicit runner image and current `checkout` and `setup-python` majors (PR merged) |
| GAP | lab#1649 | code (docs) | `issue-implement` | DOC | `docs/reference/ansible-dependency-currency.md` merged |
| DR | lab#1643 | docs review | `issue-implement` (sweep) | F1–F5, GAP | Site and reference docs reflect the epic |
| RETRO | .github#370 | retrospective | `epic-retrospective` | every other child closed | `retrospective.md` merged; the epic closes |

**Ordering.** F1–F5 and GAP are independent and touch disjoint files, so they can
run in parallel:

- F1: `ansible/` except `roles/pdrbd-qm`, plus `ansible/ansible.cfg` and a new test
- F2: `roles/pdrbd-qm`
- F3: `ci.yml`
- F4: `pyproject.toml` and `uv.lock`
- F5: `release.yml`
- GAP: `docs/reference/`

F1 is the largest. Start it first.

**Upstream, referenced and not owned:** vergil-project/vergil-actions#945
(`ubuntu-latest` moving to Ubuntu 26) and #946 (`download-artifact` DEP0005), both
under vergil-actions' ad-hoc epic.

## Task notes

### F1 — INJECT_FACTS_AS_VARS (lab#136, rescoped)

- **Rescope #136's body first.** Replace the 3-site list with the full static scope
  in spec §2 (about 70 references in about 40 files). Record the runtime origin
  counts from the 2026-10-19 sweep.
- **Test first.** Add `tests/test_ansible_facts_guard.py`. It walks `ansible/`
  (`*.yml`, `*.yaml`, `*.j2`) and fails on any `ansible_[a-z0-9_]+` name that is not
  on the allowlist in spec §3.1. `ansible_facts` is allowed.
  - It must fail on today's tree.
  - Add a fixture case proving that an unknown future fact (for example
    `ansible_memtotal_mb`) is rejected.
  - Add a fixture case proving that a fact name inside a YAML comment is ignored.
    Comments are not evaluated, so the test must not force edits to explanatory
    prose.
- **Convert every site** to `ansible_facts['<fact>']`, without the `ansible_`
  prefix.
  - `ansible_selinux.status` becomes `ansible_facts['selinux'].status`.
  - `ansible_distribution_major_version` becomes
    `ansible_facts['distribution_major_version']`, and so on for the rest.
  - The `runtime-install` guard `ansible_architecture is not defined` becomes
    `ansible_facts['architecture'] is not defined`. Keep its gather-on-demand
    behavior.
- **Set** `inject_facts_as_vars = False` under `[defaults]` in `ansible/ansible.cfg`.
  Add a comment citing #136 and the 2.24 removal.
- **Leave alone** `docs/reports/assets/guest-runtime-spike/spike.yml`, a frozen
  artifact (spec §6).
- **Accepted risk** (spec §6): a missed site breaks a later build outright and is
  patched as a follow-up.

### F2 — `bool` coercion in the pdrbd-qm DR-shape assert

- In `ansible/roles/pdrbd-qm/tasks/main.yml` (around lines 79–88), drop `| bool` on
  `pdrbd_dr_formed` in both the `that:` and `fail_msg:` expressions. The variable is
  already a boolean comparison.
- Keep `tests/test_pdrbd_qm_role.py` green. Its assertion at line 476 checks that
  `pdrbd_dr_formed` appears in the assert's `that`.
- Verify that `vrg-validate`'s ansible-lint output no longer contains `The bool
  filter coerced`.

### F3 — `run-release` deprecation (vergil-actions#688)

- In `.github/workflows/ci.yml`, remove `run-release:` from the `version` job's
  `with:` (line 63).
- The `run-release` `workflow_call` input also feeds the `security` job's
  `run-standards` (line 44). Read vergil-actions#688 and the `ci-security.yml@v2.1`
  input docs to decide what `run-standards` should key off.
  - If nothing still needs the input, remove it.
  - Otherwise rename the input to say what it now controls.
  - Record the decision in the PR body.
- Verify on the PR's own CI run: `version / version-bump` emits no `run-release is
  deprecated` warning.

### F4 — MkDocs 2.0 guard

- In `pyproject.toml`, the `docs` group becomes `["mkdocs>=1.6,<2",
  "mkdocs-material>=9", "mike>=2"]`. Add a one-line comment citing the Material
  team's MkDocs 2.0 notice:
  `https://squidfunk.github.io/mkdocs-material/blog/2026/02/18/mkdocs-2.0/`.
- Re-lock with uv. `mkdocs` must stay at 1.6.1 in `uv.lock`, with no other lock
  churn.
- Look for a documented, supported way to turn off mkdocs-material's MkDocs 2.0
  banner. Check the Material docs and changelog, and the blog post above. If one
  exists, apply it where the docs build runs and cite the source.
- If none exists, record that in the PR body with the sources checked. The banner
  is then exempt under spec §5.
- Verify that the PR's `docs / docs` job builds strict and green.

### F5 — `release.yml` runner and action majors

- `runs-on: ubuntu-latest` becomes an explicit image. Prefer `ubuntu-24.04`, which
  is what the job has run on until now. Moving to 26.04 is a later, deliberate
  bump.
- Bump `actions/checkout@v4` and `actions/setup-python@v5` to their current majors.
  Check the latest release on each GitHub releases page and cite it in the PR.
- `release.yml` runs only on a tag, so verification is `actionlint` (inside
  `vrg-validate`) plus reading the diff. The next real release exercises it.

### GAP — Ansible dependency gap and management review

Write `docs/reference/ansible-dependency-currency.md`. Link it from the reference
index if one exists.

1. **Inventory.** List every Ansible dependency, with its declared pin, the version
   actually locked or installed, and where each is declared:
   - `ansible-core`
   - `ansible-lint`
   - `ansible-compat`
   - `community.crypto`
   - `ansible.posix`
   - any other collection or module the play tree uses

   To find the last group, grep for FQCNs outside `ansible.builtin`.
2. **Gap.** For each dependency, give the latest release, how far behind we are,
   whether our version is still supported, and any notable breaking changes or
   deprecations between ours and the latest. Every claim cites its source.
3. **Management review.**
   - Python tools are locked by `uv.lock`.
   - Galaxy collections use `>=` floors with no lock, and `mqlab`'s
     `_galaxy_install_step` skips a collection that is already present. Assess what
     that means for reproducibility and for drift across `build/cache` instances.
   - Assess the lint environment's unresolved `community.crypto` modules
     (`.ansible-lint.yml` excludes `build/`).
   - Assess how a version bump would be proven.
4. **Recommendations.** Cover upgrades, lock and pin strategy, the lint-environment
   fix, and when to drop `inject_facts_as_vars` back to its default after 2.24.
   Label each as data or judgment. Note which recommendations are inputs to the
   later vergil-tooling dependency-management work.

## Terminal bookends

- **Docs review** (lab#1643). Sweep:
  - the role-authoring guidance, which must state the `ansible_facts[...]` rule and
    the guard test
  - the `ansible.cfg` explanation
  - the docs toolchain pin
  - GAP's report placement and links

  If another repo's docs need changes, spawn a same-repo task there.
- **Retrospective** (.github#370). Terminal; merging it closes the epic.

## Self-review

- **Coverage.** Every spec item maps to a task:
  - §3.1: F1–F5
  - §3.2: GAP
  - §3.3: the global constraints and each task's verification line
  - §4: the global constraints
  - §5: the done-when column
  - §6: the F1 notes
- **Placement.** Each task is filed in the repo its PR lands in.
- **Placeholders.** None. Every task is filed, and the table carries its real issue
  number.
- **Alignment (2026-10-19).** F4 could not meet spec §5 if mkdocs-material prints
  its banner whatever we pin. Resolved by human decision (option A): F4 looks for a
  supported way to turn the banner off, and failing that, the banner is exempt.
