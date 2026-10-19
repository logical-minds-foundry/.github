# Epic 368 — Deprecation-warning elimination and Ansible dependency gap

**Status:** spec · **Epic:** logical-minds-foundry/.github#368 ·
**Repo:** logical-minds-foundry/mq-resiliency-lab-for-linux

## 1. Context

On 2026-10-19 we swept two sources for deprecation warnings:

- **CI:** the full log of mq-resiliency-lab-for-linux PR #1637 (CI run
  39254813839, all 23 jobs).
- **Runtime:** every lab run log under `build/state/runs/` and
  `build/state/dr-runs/` from 2026-10-05 through 2026-10-18. That is 663 logs: 72
  box builds, 55 bootstraps, 62 qm-e2e runs, DR cutover and failback, and the obs
  phases. The window includes RHEL stacks built in the cloud, which cannot be built
  on the local macOS VM.

Several deprecations are counting down to a removal, and only one of them (#136)
was tracked. Even that issue's scope was wrong. This epic has two goals:

1. **Eliminate every deprecation warning this repo owns.** Both CI and a cold build
   of every stack should run with none.
2. **Measure the gap** between our Ansible dependencies and their latest releases,
   and assess how we manage them, as input to later dependency-management work.

## 2. Findings

**Data:** the warning text and counts below are from the sources in §1.

| # | Warning | Source | Deadline |
|---|---|---|---|
| F1 | `INJECT_FACTS_AS_VARS default to True is deprecated` | runtime: 1,438 warnings, 51 distinct origin sites, in every bootstrap and box-build log that warned | ansible-core 2.24 |
| F2 | `` The `bool` filter coerced invalid value '{{ pdrbd_dr_owned.rc == 0 }}' (str) to False `` | ansible-lint (`quality / common`) | ansible-core 2.23 |
| F3 | `run-release is deprecated: the version-bump gate now follows release-model in vergil.toml` (vergil-actions#688) | CI `version / version-bump` | vergil-actions v2.x |
| F4 | MkDocs 2.0 notice: plugins and theme overrides break, with no migration path | CI `docs / docs` | when mkdocs 2 resolves |
| F5 | `ubuntu-latest` moves to Ubuntu 26 starting 2026-10-19 (runner-images#14748) | CI annotation on every job | 2026-10-19 |
| F6 | Node `[DEP0005] Buffer() is deprecated` | `actions/download-artifact@v7` in the evidence jobs | upstream |

**F1 scope, data.** `git grep` of `origin/develop` finds about 70 top-level fact
references in about 40 files under `ansible/`. They use `ansible_architecture`,
`ansible_os_family`, `ansible_kernel`, `ansible_distribution*` and
`ansible_selinux`. No file uses `ansible_facts[...]` yet. #136 lists only 3 sites.

The runtime origins are a strict subset of the static list. Some sites never
warned even though their roles ran. Examples are
`include_tasks: install-{{ ansible_os_family }}.yml` in `pcmk-cluster`,
`iscsi-target` and `mq-nativeha`, and the kernel references in `drbd9`. Other roles,
such as `mq-client`, `pcmk-stonith`, `pdrbd-replication-tls` and
`mq-nativeha-spike`, did not run in the window at all.

**F1 risk, judgment.** Several of these sites decide which files load:
`tasks/os-vars.yml` and the OS-family `include_tasks`. In 2.24 the variables become
undefined, so provisioning fails rather than warns. The **static list is the fix
scope**, not the runtime origins.

**F2 cause, judgment.** We have not verified this on a host. ansible-lint renders
the assert's `vars:` without running the registered probe, so `pdrbd_dr_owned` is
undefined. The template then stays a literal string, and `| bool` coerces that
string. At runtime the expression should be a native boolean. No run log in the
window reached this task, because the code merged on 2026-10-19.

**Ownership.** F1–F4 are this repo's. F5 is mostly in the vergil-actions reusable
workflows, but this repo's own `.github/workflows/release.yml` also uses
`runs-on: ubuntu-latest`, `actions/checkout@v4` and `actions/setup-python@v5`. F6
is wholly upstream.

**Not deprecations, but noted.**

- **Unresolved collection modules in lint.** ansible-lint logs `Invalid value
  (None) for resolved_fqcn` for `community.crypto` modules about 14 times per run.
  `.ansible-lint.yml` excludes `build/`, where the collections live, so lint cannot
  resolve them. This is input to §3.2.
- **Unit tests are clean.** 3,209 tests passed with no pytest warnings.

## 3. Scope

### 3.1 Deprecation fixes (this repo)

- **T-F1, #136, rescoped.**
  - Replace every top-level fact reference with `ansible_facts['<fact>']`. The
    scope is the full static list, not the runtime list.
  - Set `inject_facts_as_vars = False` in `ansible/ansible.cfg`. That is the 2.24
    default, set now, so a reference the grep missed fails immediately instead of
    on the 2.24 upgrade.
  - Add a guard test that fails on any `ansible_*` name in `ansible/` that is not
    on an explicit allowlist. The allowlist is Ansible's documented connection
    variables and magic variables: the tree uses `ansible_host`, `ansible_user`,
    `ansible_port`, `ansible_connection`, `ansible_ssh_*`,
    `ansible_python_interpreter`, `ansible_play_hosts`, `ansible_play_hosts_all`,
    `ansible_limit`, `ansible_failed_result`, `ansible_failed_task` and
    `ansible_managed`, plus `ansible_facts` itself. An allowlist rather than a
    denylist of today's fact names means a newly introduced fact reference fails
    the guard too.
- **T-F2.** Drop the redundant `| bool` on `pdrbd_dr_formed` in
  `roles/pdrbd-qm/tasks/main.yml`. Then confirm ansible-lint no longer reports the
  coercion. The existing role test (`tests/test_pdrbd_qm_role.py`) must stay green.
- **T-F3.** Stop passing `run-release` to `ci-version-bump.yml` in
  `.github/workflows/ci.yml` (line 63), as vergil-actions#688 directs. The
  version-bump gate then follows `release-model` in `vergil.toml`.
  - The same `workflow_call` input also drives the security job's `run-standards`
    (line 44), so the task also decides what `run-standards` keys off.
  - Removing the input entirely is the end state only if nothing still needs it.
    No in-repo workflow calls `ci.yml` with it.
- **T-F4.** Add an explicit `mkdocs>=1.6,<2` to the `docs` dependency group, so
  MkDocs 2.0 cannot arrive through a transitive bump. The lock stays on 1.6.1.
  If mkdocs-material documents a supported way to turn off its MkDocs 2.0 banner,
  use it and cite the source. If it does not, record that in the PR; the banner is
  then exempt (§5).
- **T-F5.** In `release.yml`, pin an explicit runner image and move
  `actions/checkout` and `actions/setup-python` to their current majors.

### 3.2 Ansible dependency gap and management review (this repo)

This task produces a reference report at
`docs/reference/ansible-dependency-currency.md`. It changes no pins.

- **Current against latest.** For each dependency, give the version we pin, the
  version we lock or install, and the latest upstream release:
  - `ansible-core`: pinned `>=2.16`, locked 2.21.1
  - `ansible-lint` and `ansible-compat`: unpinned, locked 26.6.0
  - `community.crypto`: `>=2.20.0`
  - `ansible.posix`: `>=1.5.0`
  - anything else the play tree actually uses

  For each, also give the support status of our version and the breaking or
  deprecation changes between ours and the latest.
- **How the dependencies are managed.** Python-side tools are locked by
  `uv.lock`. Galaxy collections are not: `requirements.yml` gives only `>=`
  floors, and `mqlab`'s galaxy install step skips a collection that is already
  present. So each `build/cache` keeps whatever version it first installed. The
  report assesses:
  - reproducibility
  - drift across hosts
  - the lint environment's missing collections (the unresolved `community.crypto`
    modules in §2)
  - how a version bump would be proven
- **Recommendations.** What to upgrade, what to lock, and how. Each
  recommendation is marked as *data* (what a cited source says) or *judgment*.
  The recommendations also feed the later dependency-management work in
  vergil-tooling.

Every version and support claim cites a checkable source: PyPI, Galaxy, GitHub
releases, or the ansible-core support matrix.

### 3.3 Verification (no dedicated validation build)

These are deprecation warnings, not defects, so the epic does not pay for a
dedicated cold rebuild of boxes and stacks to prove them gone.

- **Each task verifies in its own PR** with what its pipeline already runs:
  - the guard test and the unit suite
  - ansible-lint, which must not report the F2 coercion
  - CI, which must not show the F3 and F4 notices
- **Runtime confirmation is incidental.** The 2.24 behavior is confirmed during
  the next real rebuilds done for other work: box rebakes and stack cold builds.
  A planned recurring scan of recent build logs for deprecation warnings would
  also catch any miss; that scan is a separate initiative, not part of this epic.
  A miss is patched as a small follow-up task.

## 4. Out of scope

- **F5 in the reusable workflows** is vergil-project/vergil-actions#945, under
  vergil-actions' ad-hoc epic. This epic only refers to it.
- **F6** is vergil-project/vergil-actions#946, under the same ad-hoc epic. This
  epic only refers to it.
- **Upgrading any Ansible dependency.** §3.2 measures the gap and recommends; the
  upgrades are follow-on work.
- **Draining the `.ansible-lint.yml` `skip_list`** (cosmetic, tracked by #403).

## 5. Acceptance

- The F1 guard test is in place. No top-level fact reference remains in
  `ansible/`, and `inject_facts_as_vars = False` is set.
- ansible-lint and CI show no deprecation owned by this repo. Two exceptions may
  remain:
  - the upstream F5 and F6 items
  - the Material for MkDocs 2.0 banner (F4), and only if mkdocs-material offers no
    documented, supported way to turn it off. In that case the `mkdocs<2` pin is the
    remedy and the banner is treated as informational.
- `docs/reference/ansible-dependency-currency.md` is merged.
- `vrg-validate` is green on every PR.

## 6. Risks

- **Accepted risk: `inject_facts_as_vars = False` can break a later build
  outright.** A missed reference fails rather than warns, and with no dedicated
  validation build (§3.3) that failure can surface in the middle of an unrelated
  rebuild. We accept that risk: any such break is a missed `ansible_facts`
  conversion, small and quick to patch. The flag also moves the 2.24 break to now,
  under our control.
- **Sites in roles that rarely run** are fixed without ever being exercised in a
  warning log, for example `mq-nativeha-spike`. The guard test covers them
  statically.
- **The archived spike playbook stays as it is**, at
  `docs/reports/assets/guest-runtime-spike/spike.yml`. It is a frozen evidence
  artifact that nothing runs, and it lies outside `ansible/` and outside the guard
  test.
