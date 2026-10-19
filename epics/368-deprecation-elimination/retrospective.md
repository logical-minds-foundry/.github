# Deprecation-warning elimination and Ansible dependency gap — retrospective

- **Epic:** [logical-minds-foundry/.github#368](https://github.com/logical-minds-foundry/.github/issues/368). **Spec:** [spec.md](spec.md). **Plan:** [plan.md](plan.md), not amended. The one scope addition (lab#1654) and the one upstream task (vergil-actions#947) are recorded in the epic body.
- **Retrospective task:** [.github#370](https://github.com/logical-minds-foundry/.github/issues/370).
- **Opened → closed:** 2026-10-19 → 2026-10-19 (one day).
- **Outcome:** DELIVERED.
  - Ansible facts are read only through `ansible_facts[...]`, and a guard test enforces it.
  - `inject_facts_as_vars = False` is set.
  - Every deprecation in this repo's ansible-lint and CI output that the 2026-10-19 sweep found is gone. The MkDocs 2.0 banner was not exempted after all; it was silenced upstream.
  - The Ansible dependency-gap report is merged.
  - Runtime confirmation of the facts change waits for the next real rebuild (spec §3.3).

Repo shorthand: `lab#N` = `logical-minds-foundry/mq-resiliency-lab-for-linux#N`.

## §0 At a glance

A deprecation sweep on 2026-10-19 covered two sources: the CI run of lab PR #1637, and 663 lab run logs from the preceding two weeks. It found several deprecations running out the clock, and only one of them was tracked. The worst was Ansible's `INJECT_FACTS_AS_VARS`, which fired 1,438 times. Its removal in ansible-core 2.24 would break provisioning outright, because OS-family includes and per-distro var files key off top-level facts.

We set out to do two things: remove every deprecation warning this repo owns, and measure how far our Ansible dependencies lag their latest releases, without upgrading anything.

**What shipped:**

- **Facts namespace (F1).** All 70 top-level fact references in 43 files under `ansible/` now read `ansible_facts['<fact>']`. `inject_facts_as_vars = False` is set in `ansible/ansible.cfg`. `tests/test_ansible_facts_guard.py` rejects any top-level fact name outside its allowlist. It failed on the old tree, rejects an unknown future fact, and ignores fact names in comments.
- **Four small deprecation fixes (F2–F5, plus the added lab#1654):**
  - the `bool`-coercion warning in the `pdrbd-qm` DR-shape assert;
  - the `run-release` / `run-standards` inputs;
  - the no-op `container-suffix` on `ci-security`;
  - a `mkdocs<2` pin;
  - `release.yml` on an explicit `ubuntu-24.04` runner, with `checkout` and `setup-python` at v7.
- **The MkDocs 2.0 banner, silenced upstream.** Material documents an off switch, `NO_MKDOCS_2_WARNING=1`. A caller can't reach it, because the docs build runs inside vergil-actions' reusable workflows. vergil-actions#947 set it there, and it shipped in v2.1.43, which `v2.1` now resolves to.
- **The dependency-gap report (GAP):** `docs/reference/ansible-dependency-currency.md`. It covers the inventory, the gap, a management review and recommendations, with every claim tagged *data* or *judgment* and cited.
- **The docs sweep (lab#1643).** The developer guide states the `ansible_facts` rule, the guard test, the `inject_facts_as_vars` setting and the docs toolchain pin.

**Work delivered (merged PRs, queried from GitHub):**

| PR | Merged (UTC) | Task | What it did |
|---|---|---|---|
| .github#371 | 10-19 14:18 | .github#369 (DOC) | Spec and plan for the epic |
| lab#1653 | 10-19 16:41 | lab#136 (F1) | 70 fact references to `ansible_facts[...]`; injection off; guard test (+287/−100, 53 files) |
| lab#1657 | 10-19 17:08 | lab#1644 (F2) | Dropped the redundant `\| bool` on `pdrbd_dr_formed` |
| lab#1658 | 10-19 17:13 | lab#1646 (F4) | Pinned `mkdocs>=1.6,<2`; no lock churn (mkdocs stays 1.6.1) |
| lab#1659 | 10-19 17:18 | lab#1649 (GAP) | Ansible dependency currency and management review (+491) |
| vergil-actions#948 | 10-19 17:24 | vergil-actions#947 | `NO_MKDOCS_2_WARNING=1` in `ci-docs.yml` and `cd-docs.yml` |
| lab#1661 | 10-19 17:32 | lab#1645 (F3) | Dropped the deprecated `run-release` input and the dead `run-standards` wiring |
| lab#1662 | 10-19 17:36 | lab#1648 (F5) | `release.yml` on `ubuntu-24.04`, with `checkout@v7` and `setup-python@v7` |
| lab#1663 | 10-19 17:40 | lab#1654 | Dropped the no-op `container-suffix` from the `security` job |
| lab#1667 | 10-19 18:35 | lab#1643 (DR) | Developer docs: the `ansible_facts` rule, the guard test, the docs toolchain pin |

**Counts:**

- **Repos:** 3: the lab, `.github`, and `vergil-project/vergil-actions`. The last is referenced from the epic, not linked under it, because it is in another org.
- **Tasks:** 10 children, all closed at this retrospective's merge. One of them (lab#1654) was added during execution. One upstream task, vergil-actions#947, sits under vergil-actions' ad-hoc epic.
- **Merged PRs:** 10 (8 lab, 1 `.github`, 1 vergil-actions), plus this retrospective.
- **Releases cut:** one upstream, vergil-actions v2.1.43, which carried #948. None in the lab, which ships from `develop`.

## §1 How the plan evolved

The plan was not amended, and its shape held:

- Six independent tasks (F1–F5 and GAP) on disjoint files ran in parallel and merged within an hour of each other.
- The docs review followed.

Execution changed five things.

1. **F4's exemption condition didn't hold.** The plan allowed the MkDocs 2.0 banner to be exempt only if mkdocs-material offered no supported way to turn it off. F4 found that a supported switch exists, but this repo cannot reach it: the docs build runs in vergil-actions' reusable workflows, which take no environment input, and a caller's `env` does not reach a called workflow. The human ruled for an upstream fix over an exemption. vergil-actions#947 was filed under vergil-actions' ad-hoc epic, implemented, and released in v2.1.43. Nothing was exempted.
2. **F3 found one more deprecation.** `ci-security.yml@v2.1` marks `container-suffix` as deprecated and ignores it, and our `security` job still passed it. It had no warning to flag it, so the sweep missed it. It was added as lab#1654 and recorded in the epic body.
3. **F1's rescoping step was already done.** lab#136's body already carried the full scope from spec §2, so F1 went straight to the guard test.
4. **F5 resolved "current majors" to v7.** It jumped `checkout` from v4 and `setup-python` from v5 straight to v7, after checking that none of the skipped majors' breaking changes affect a tag-only workflow.
5. **The hand-off path broke twice** (§2). The cloud agents reported every branch ready, but none of them had pushed its branch. The four branches that touch `.github/workflows/` (lab#1645, #1648, #1654 and vergil-actions#947) could not be pushed from the cloud VM at all. Those four were re-done in a local session from patches posted on their issues, which is why their merged branch names differ from the cloud ones (for example `feature/1645-drop-run-release`).

## §2 Lessons learned

- **In a cloud session, route workflow-file changes to a local session from the start.** The cloud agent's GitHub App token cannot push a commit that touches `.github/workflows/`, and human credentials can't be used in the cloud. Four of this epic's ten PRs had to be re-done locally. Next time, decide at dispatch which tasks touch workflows and send them local before any work starts.
- **"Reported ready" must be checked on origin.** `report-ready` publishes only the relay ref, and its help says to push the branch first. The cloud agents skipped that step, and their claimed relay refs were not on origin either. The batch went to the human, and `vrg-submit-pr` failed on the second branch. Next time, the driver should check `vrg-git ls-remote origin` for both the branch and the relay ref before it presents a batch. The docs-review dispatch did, and its hand-off went cleanly.
- **Check a deprecation's annotation, not a text search of the log.** The `run-release` warning probably never fired on our PRs, because it fires only when the input is `false`. The string still appears in raw logs because the step prints its own script. A text search would have shown it gone, or present, for the wrong reason.
- **A green `vrg-validate` hides ansible-lint's warnings.** To show the `bool`-coercion warning on `develop` and its absence on the branch, F2 needed `vrg-validate --check common --output-format gha`. A plain run prints nothing when the check passes.
- **Owning a fix can mean filing it upstream.** The banner was "this repo's warning", but the only supported way to turn it off was in a dependency we consume. Filing it under vergil-actions' ad-hoc epic and referencing it from this epic, without a cross-org link, kept the work traceable without stretching the spec's exemption.
- **Look up the tool before saying a capability is missing.** The driver first told the human it could not create issues, without checking the `vrg-*` commands. `vrg-issue-create` and `vrg-adhoc-epic` were there; the human corrected it.

## §3 Compromises & tradeoffs

- **No runtime confirmation yet for F1** (spec §3.3, accepted risk in §6). With injection off, a fact reference the static guard can't see now fails the build instead of warning. The guard covers every `.yml`, `.yaml` and `.j2` file under `ansible/`. It is **unknown** at the time of writing whether a cold rebuild has run since lab#1653 merged; the next one is the real test, and a miss gets a small follow-up.
- **F5 is unexercised.** CI never runs `release.yml`; only a `v*` tag or a manual dispatch does. The v7 actions and the pinned runner first run at the next lab release.
- **The banner fix depends on a moving tag.** The lab picks up the fix because it calls vergil-actions at `@v2.1`, which now points at v2.1.43. **[judgment]** That is the intended consumption model, but it means the lab's docs output changed without a lab commit.
- **The `inject_facts_as_vars` comment overstates the upstream change.** It says 2.24 *removes* fact injection. The ansible-core 2.20 changelog says only that the default switches to `False` by 2.24. The docs review (lab#1643) found this after F1's branch was frozen, so it became a follow-up (lab#1669).
- **Upstream items stay out of scope**, as planned: vergil-actions#945 (`ubuntu-latest` moving to Ubuntu 26) and #946 (Node DEP0005 from `download-artifact@v7`) are both still open.
- **GAP's support-matrix data came from source, not the published page.** docs.ansible.com returned HTTP 429, so the report read the matrix's `.rst` source at a pinned commit. The 2.23 and 2.24 rows there are commented-out drafts, and the report labels them as such.

## §4 New problems & opportunities

| Surfaced | Where it went |
|---|---|
| `container-suffix` passed to `ci-security`, deprecated and ignored (F3) | lab#1654, fixed in this epic |
| The MkDocs 2.0 banner's switch can't be reached from the caller (F4) | vergil-actions#947, fixed and released in v2.1.43 |
| The `ansible.cfg` comment says 2.24 removes injection; the changelog says the default changes (DR) | lab#1669 (ad-hoc), open |
| Local `vrg-container-docs` builds still print the banner: the wrapper forwards only `[container].env-prefixes` variables (DR) | vergil-tooling#3342 (ad-hoc), open |
| vergil-actions' `ci-docs` and `cd-docs-deploy` pages don't mention `NO_MKDOCS_2_WARNING` (DR) | vergil-actions#953 (ad-hoc), open |
| The `ansible-core>=2.16` floor no longer works: 2.16 is end-of-life and doesn't run on the project's Python 3.14 (GAP) | Logged in the GAP report, not yet acted on |
| RHEL 9 guests run Python 3.9, which the draft 2.23 matrix row drops as a target (GAP) | Logged in the GAP report, not yet acted on: needs a newer Python, a core cap at 2.22, or retirement |
| Galaxy collections have `>=` floors and no lock, and the install step keeps whatever `build/cache` installed first (GAP) | Logged as an input to the later vergil-tooling dependency-management work |
| ansible-lint can't resolve the 14 `community.crypto` call sites, and `.ansible-lint.yml` can't fix that because `vrg-validate` reads only two of its keys (GAP) | Logged as an input to the later vergil-tooling dependency-management work |
| The box manifest hash leaves out the ansible-core and collection versions, so a bump doesn't force a box rebuild (GAP) | Logged in the GAP report, not yet acted on |
| Cloud sessions can't push `.github/workflows/` changes, and agents skipped the branch push before `report-ready` | Proposed to the human as a repo memory and a candidate vergil plugin issue; not yet decided |

## §5 What's next

- **The three follow-ups** lab#1669, vergil-tooling#3342 and vergil-actions#953 are filed and scheduled right after this retrospective is submitted.
- **Ansible dependency work.** The GAP report's recommendations are the input to a later dependency-management effort. Collection locking, drift detection, the lint-environment fix and a dependency-age check are the four marked for vergil-tooling. **[judgment]** The RHEL 9 / Python 3.9 decision is the most time-sensitive item: ansible-core 2.22 is due in November 2026, and 2.23 follows it.
- **Restore `inject_facts_as_vars` to its default** only in the PR that raises the ansible-core floor to `>=2.24` (GA November 2027 per the draft matrix), per GAP recommendation 10.
