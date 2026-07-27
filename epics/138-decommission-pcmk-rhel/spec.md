# Decommission the pcmk-rhel arm — design spec

- **Epic:** `logical-minds-foundry/.github#138`
- **Design task:** `logical-minds-foundry/.github#139`
- **Origin:** the epic #103 closing brainstorm `logical-minds-foundry/.github#107`.
- **Arm origin (what is being removed):** issues #238 / #244;
  `docs/specs/2026-06-17-pcmk-rhel-third-arm-design.md`; `ansible/site-pcmk-rhel.yml`.
- **Builds on (does not touch):** the kept `pcmk-ubuntu` arm and the shared
  Pacemaker/DRBD roles it depends on; the kept RDQM-RHEL arm (`mq-rdqm-rhel9`).
- **Status:** design (brainstorm output 2026-07-27; owner-approved scope), pending human review
- **Date:** 2026-07-27

## 1. Problem & motivation

The `pcmk-rhel` arm (nodes `san-a-rhel` + `pcmk-rhel-a1..3`,
`ansible/site-pcmk-rhel.yml`) was a **proof-of-concept**: it demonstrated that the
full RDQM-equivalent HA/DR substrate can be built from **OSS Pacemaker/DRBD on
RHEL** without IBM's bundled RDQM packages. That point is proven. Nobody runs it
in practice, which is exactly why epic #103 **dropped it from Part B** (baking)
rather than invest in a fat box for it. Carrying a dead arm costs topology
surface, network allocations, an unbaked-and-untested Ansible path, and reader
confusion. It earns removal.

This is a deliberately **tiny, subtractive** epic: one code-decommission task and
one docs-decommission task, no follow-on brainstorm, and no live validation (the
removal is fully covered by the existing topology + parity unit tests).

## 2. Decisions

### D1 — Surgical, pcmk-rhel-exclusive removal (retain everything shared)
Remove **only** assets exclusive to the RHEL Pacemaker arm. The kept `pcmk-ubuntu`
arm, the kept RDQM-RHEL arm, and the Ubuntu SAN targets share several roles;
none of those may regress. The verified determination (grep-traced, 2026-07-27):

| Asset | Disposition | Why |
|---|---|---|
| `ansible/site-pcmk-rhel.yml` | **remove** (whole file) | The arm's only playbook. |
| `roles/rhel-ha-repo` | **remove** | pcmk-rhel-exclusive — RDQM explicitly does **not** use it (`bake-mq-rdqm.yml` comment, verified #602); its only other consumer is `pcmk-cluster/tasks/install-RedHat.yml`, which itself dies with the arm. |
| `roles/pcmk-cluster/tasks/install-RedHat.yml` | **remove** (dead variant) | The RHEL task variant of a **kept** role; its only caller is the RHEL pcmk arm (`pcmk-ubuntu` uses `install-Debian.yml`). |
| `roles/pcmk-stonith/tasks/install-RedHat.yml` | **remove** (dead variant) | Same reasoning — verify no remaining RHEL caller at plan time. |
| `roles/pcmk-cluster`, `roles/pcmk-stonith`, `roles/mq-pcmk-qmgr`, `roles/drbd-san` | **keep** | Shared with `bake-pcmk-ubuntu` and/or the Ubuntu SAN targets (`san_a`/`san_b`). |

The precise per-variant dead-code determination is pinned in the plan; the
governing rule is D1 plus a **dead-code sweep** (§5): before removing any RHEL
task variant, grep-confirm it has no live non-pcmk-rhel caller.

**Consequence — `pcmk-cluster` and `pcmk-stonith` become Debian-only.** Both
roles dispatch per-OS via `include_tasks: install-{{ ansible_os_family }}.yml`.
Removing their `install-RedHat.yml` leaves each role with a Debian variant only,
which is correct — post-removal no RHEL-family host routes through either role.
The dynamic include is a **fail-loud** seam: if a future RHEL Pacemaker arm is
ever re-introduced, the missing `install-RedHat.yml` surfaces as an obvious
file-not-found (reinstate the variant then) rather than silent wrong behaviour.
The plan records this Debian-only state as the intended outcome.

### D2 — `san-a-rhel` is removed here, not in the SAN epic
`san-a-rhel` is the RHEL arm's iSCSI SAN target and is meaningful only as part of
`pcmk-rhel`; it is decommissioned **with** this arm. It is **not** in scope for
the separate SAN-hosts epic #108, which covers the **Ubuntu**-arm SAN targets
`san-a`/`san-b`.

### D3 — Living docs only; historical records are immutable
The docs decommission updates only the **living** docs (`README.md`,
`docs/development/*`). The historical `docs/specs|plans|reports/*` are
point-in-time records of what was true when written and are **left untouched**.

## 3. Scope

### 3.1 Code decommission — `mq-resiliency-lab-for-linux` (task #794)
- `lab/topology.yaml`: remove the `san-a-rhel` + `pcmk-rhel-a1..3` node entries,
  the `san_a_rhel` / `pcmk_rhel_a` groups, and the `.7x` network allocations
  reserved for this arm (`172.16.1.71..73` heartbeat + the arm's other reserved
  subnets) — after confirming no kept arm uses them.
- `ansible/`: remove `site-pcmk-rhel.yml`; strip pcmk-rhel references from
  `site-nativeha.yml`, `bake-mq-rdqm.yml`, `roles/rhel-ha-repo` (removed wholesale),
  and `roles/alloy/tasks/install.yml`; remove `roles/rhel-ha-repo` and the dead
  RHEL task variants per D1.
- `src/mqlab/parity.py` + `tests/test_parity.py`: drop the pcmk-rhel arm from the
  capability matrix and its assertions; update any other test referencing the
  pcmk-rhel nodes/groups.

### 3.2 Docs decommission — `mq-resiliency-lab-for-linux` (task #795, closing)
- Update the living docs to drop the pcmk-rhel arm: `README.md`,
  `docs/development/box-model.md`, `docs/development/box-bake-manifest.md`.
- Verify `docs/site` carries **no** pcmk-rhel references (confirmed none at design
  time) — the review is a single-repo sweep. Spawn a per-repo doc task only if the
  sweep discovers another repo's docs reference the arm.

## 4. Non-goals

- Any change to the kept `pcmk-ubuntu` arm, the kept RDQM-RHEL arm, or the shared
  Pacemaker/DRBD roles and their non-RHEL task variants.
- The Ubuntu SAN targets `san-a`/`san-b` (separate epic #108).
- Rewriting historical specs/plans/reports.
- A follow-on brainstorm (none) and a live cold-rebuild validation (unneeded — the
  change is subtractive and unit-test-covered).

## 5. Testing & acceptance

- **Single gate:** `vrg-container-run -- vrg-validate` green (ruff + mypy strict +
  pytest @ 100% branch coverage). The topology tests and `test_parity.py` are the
  acceptance: they prove the arm's nodes/groups are gone and that nothing else
  references pcmk-rhel.
- **Dead-code sweep (one-time plan step):** after the removals, grep the tree for
  `pcmk[-_]rhel` / `san[-_]a[-_]rhel` (excluding `build/` and the immutable
  historical docs) and confirm only intentional historical-record matches remain.
  This is a **thorough one-time cleanup verification**, done at removal time.
- **No permanent guard test** is added. A standing "this token must never
  reappear" assertion would be accumulated cruft managing a non-risk — careful
  removal plus the one-time sweep is sufficient, and nothing re-introduces a
  removed arm by accident. (Deliberate decision, recorded so it is not re-litigated.)
- No live-lab / cold-rebuild validation task (D-, V- style) — see Non-goals.

## 6. Open items for the implementation plan

- Enumerate the exact `.7x` network allocations reserved for the arm in
  `lab/topology.yaml` and confirm each is unused by a kept arm before removal.
- Confirm the final list of dead RHEL task variants inside shared roles
  (`pcmk-cluster`, `pcmk-stonith`) via the dead-code sweep, and whether removing
  them leaves any now-unused handler/var files. Record that both roles are
  intentionally **Debian-only** afterward (D1 consequence).
- Confirm the incidental pcmk-rhel reference in `roles/alloy/tasks/install.yml`
  and `ansible/site-nativeha.yml` is a pure removal with no behavioural change to
  the kept arms.
