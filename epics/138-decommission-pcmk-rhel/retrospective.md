# Decommission the pcmk-rhel arm — retrospective

- **Epic:** `logical-minds-foundry/.github#138`
- **Partners:** `spec.md`, `plan.md` (read spec → plan → retrospective)
- **Retrospective task:** `logical-minds-foundry/.github#140`
- **Date:** 2026-07-27

## §0 At a glance

We set out to remove the dead `pcmk-rhel` proof-of-concept HA/DR arm from the
lab — surgically, touching only assets exclusive to the RHEL Pacemaker arm and
retaining everything the kept `pcmk-ubuntu` arm shares. That shipped exactly, and
the epic also swept up one piece of dead code the removal exposed. The result is
strongly subtractive: **+17 / −267 lines** across four merged PRs, with the lab
tree proving the arm gone by its own topology + parity tests. `pcmk-ubuntu` and
the RDQM / Native-HA arms are untouched.

**Work delivered**

| PR | Task | Repo | What it did |
|---|---|---|---|
| [.github#141](https://github.com/logical-minds-foundry/.github/pull/141) | #139 | `.github` | Spec + plan for the decommission (the opening bookend) |
| [#797](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/797) | #794 | lab | Code removal — topology nodes/groups/`.7x`, `site-pcmk-rhel.yml`, the exclusive `rhel-ha-repo` role, dead RHEL role-adapters, the parity matrix row (+8/−251) |
| [#799](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/799) | #795 | lab | Living-docs sweep — `README.md`, `box-model.md`, `box-bake-manifest.md` (+7/−10) |
| [#800](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/800) | #798 | lab | Removed the orphaned `rhel-ha` build-state classifier surfaced by #794 (+2/−6) |

- **Repos touched:** 2 — `logical-minds-foundry/.github` (spec/plan/retrospective docs), `mq-resiliency-lab-for-linux` (code + living docs).
- **Tasks:** 5 (#139 spec+plan, #794 code, #795 docs, #798 cleanup, #140 retrospective). **Merged PRs:** 4 + this retrospective.
- **Releases cut:** none (subtractive lab change integrated to `develop`; the `.github` repo is release-model `none`).
- **Span:** opened → all work merged on a single day, 2026-07-27 (epic created 14:19Z, last code PR merged 19:17Z).

## §1 How the plan evolved

`plan.md` (PR #141) scoped **two** work tasks — T1 code decommission (#794) and
T2 living-docs sweep (#795) — plus the retrospective bookend, with a clean
`T1 → T2 → retrospective` dependency chain. Execution followed that chain
faithfully; the one deviation is worth recording because it is the delta the plan
did not foresee.

While implementing T1, the code removal deleted the `rhel-ha-repo` Ansible role —
the **sole populator** of the `build/state/rhel-ha/` bucket. That left the
bucket's build-layout plumbing (`buildenv.py`'s `MIGRATION` entry, two test
lines, and `build-layout.md` prose) as orphaned dead code. The plan had not
anticipated this — it is visible only once you are inside the removal. Rather
than silently widen T1's frozen branch, T1's PR notes **flagged it as a
scoped-out follow-up candidate** with the evidence for its deadness. The human's
call was that confirmed-dead code exposed by a removal *is* part of the cleanup,
so a third code task (**#798**) was minted mid-flight under the epic and driven in
parallel with T2, then merged (PR #800).

**Planned vs. actual:** 2 work tasks → 3. A small, well-understood delta — an
in-scope cleanup made visible only by opening the code, captured as its own
tracked issue rather than a mutation of an already-merged branch. The planning
discipline held; the epic simply learned about one adjacent dead-code artifact
the design-time inventory could not see.

## §2 Lessons learned

- **Evidence beats intent for shared-vs-exclusive.** The origin design's "shared
  roles + OS-adapter seam" meant `pcmk-cluster` / `pcmk-stonith` / `drbd-san` were
  *shared* with `pcmk-ubuntu`; only `rhel-ha-repo` and the dead `install-RedHat.yml`
  adapters were truly exclusive. The safe removal came from confirming
  *who-includes-what in the current tree* (e.g. `drbd-san` is used only by the
  Ubuntu chain, never by the RHEL arm) — not from trusting the design's stated
  intent. That who-uses-what inventory is what kept `pcmk-ubuntu` intact.
- **TDD works for deletion, not just addition.** Flipping the parity tests to the
  post-removal state first (red: the `pcmk-rhel` column still present), then
  deleting the `MATRIX` entry (green), gave a mechanical proof the arm was gone —
  cheaper and more honest than eyeballing a diff.
- **Dead code exposed by a removal is its own issue.** Handling #798 as a new
  tracked task (not a fold-in to #794's frozen branch) kept one-task-one-PR intact
  and made the deadness argument auditable in its own PR.

## §3 Compromises & tradeoffs

- **Debian-only shared roles, on purpose.** Removing the `install-RedHat.yml`
  variants from the retained `pcmk-cluster` / `pcmk-stonith` roles leaves them
  Debian-only, while `main.yml` still dispatches
  `install-{{ ansible_os_family }}.yml`. A future RHEL re-entry therefore *fails
  loud* rather than silently mis-running — an intended, documented consequence
  (spec D1), not latent debt.
- **Historical docs left carrying the arm.** `docs/specs|plans|reports/*` still
  reference `pcmk-rhel` by design — they are immutable point-in-time records. A
  repo-wide grep will always show those historical hits; the acceptance sweeps
  filter them deliberately. The proof-of-concept's value is preserved there
  (notably `docs/specs/2026-06-17-pcmk-rhel-third-arm-design.md`).
- **No live validation.** Uniquely for this epic, the cold-rebuild acceptance gate
  did not apply: the change is purely subtractive and fully covered by the
  topology + parity unit tests, so there was nothing new to prove on a running
  lab. Accepted per spec §2.

## §4 New problems & opportunities

- **Orphaned `rhel-ha` build-state classifier** — surfaced by #794's removal of
  its sole populator. **Disposition:** logged as issue #798 under this epic and
  closed within it (PR #800). This is the model working — the removal exposed
  adjacent dead code, which was captured and cleared rather than lost.
- No other loose ends surfaced. The dead-code sweep (`pcmk[-_]rhel` /
  `san[-_]a[-_]rhel` and `rhel[-_]ha`) returns zero across code, tests, ansible,
  lab, and living docs on `develop`.

## §5 What's next

Nothing. This is a terminal, subtractive epic with **no follow-on brainstorm**
(spec §2 non-goal) — there is no forward axis to open once the arm is gone. The
arm originated from the closing brainstorm of epic #103 (`.github#107`); its
removal closes that thread. Any future desire to re-prove the OSS-Pacemaker-on-RHEL
substrate would start fresh from the preserved origin design, not from reviving
this code.
