# Decommission the pcmk-rhel arm — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the dead `pcmk-rhel` arm (nodes `san-a-rhel` + `pcmk-rhel-a1..3`,
`ansible/site-pcmk-rhel.yml`, `roles/rhel-ha-repo`, the parity entry) surgically —
retaining everything the kept `pcmk-ubuntu`, RDQM-RHEL, and Ubuntu-SAN arms share.

**Architecture:** Two tasks. **T1** (code, `#794`) is a pure subtractive removal
proven by `vrg-validate` (the topology + `test_parity` suites are the acceptance —
they assert the arm's nodes/groups/matrix are gone with nothing dangling). **T2**
(docs-review closing bookend, `#795`) removes the arm from the three *living* docs.
The historical `docs/specs|plans|reports/*` are immutable point-in-time records
and are left untouched. No follow-on, no live validation, no permanent guard test.

**Tech Stack:** Python 3.12 + Typer (`mqlab`), pytest @ 100% branch coverage;
YAML lab topology; Ansible roles/playbooks; libvirt/Vagrant lab.

**Design spec:** `epics/138-decommission-pcmk-rhel/spec.md` (Epic `.github#138`).

## Global Constraints

- **Validation is one command:** `vrg-container-run -- vrg-validate` (ruff + mypy
  strict + pytest @ **100% branch coverage**). Authoritative per-task gate before
  every commit.
- **Git/GitHub:** `vrg-git` / `vrg-commit` (conventional commits). Raw `git`/`gh`
  denied. All work on the task's own worktree/branch.
- **Surgical removal (spec D1):** remove only pcmk-rhel-**exclusive** assets;
  retain `pcmk-cluster`, `pcmk-stonith`, `drbd-san`, `mq-pcmk-qmgr` (shared with
  `bake-pcmk-ubuntu` / the Ubuntu SAN targets / RDQM). The two shared roles
  become **Debian-only** once their dead `install-RedHat.yml` variants are removed.
- **Immutable history (spec D3):** never edit `docs/specs|plans|reports/*`.
- **Derive, never hardcode:** `net-san-rhel-a` has no standalone declaration — it
  is derived from node `nics`, so removing the nodes removes the plane.

## Task dependency graph

```text
  T1 code decommission (#794, mq-resiliency-lab-for-linux)
       │  merged
       ▼
  T2 docs-review decommission (#795, closing bookend)
       │  merged
       ▼
  Retrospective (#140, .github) — terminal (epic-retrospective)
```

---

## Task 1: Code decommission — topology, ansible, parity (`#794`)

**Platform:** agnostic-code · **Repo:** `mq-resiliency-lab-for-linux`

**Files:**

- Modify: `src/mqlab/parity.py` (drop the `pcmk-rhel` MATRIX entry + its comment)
- Modify: `tests/test_parity.py` (drop the pcmk-rhel test + header/ARMS assertions)
- Modify: `lab/topology.yaml` (remove the arm block L255–276, the groups L367–368,
  and the `.7x`/`net-san-rhel-a` allocation comments L257/L282/L318)
- Delete: `ansible/site-pcmk-rhel.yml`
- Delete: `ansible/roles/rhel-ha-repo/` (whole role: `tasks/main.yml` + `README.md`)
- Delete: `ansible/roles/pcmk-cluster/tasks/install-RedHat.yml` (dead RHEL variant)
- Delete: `ansible/roles/pcmk-stonith/tasks/install-RedHat.yml` (dead RHEL variant)
- Modify (comment cleanups): `ansible/site-nativeha.yml:113`,
  `ansible/bake-mq-rdqm.yml:25-27`, `ansible/roles/alloy/tasks/install.yml:23`

**Interfaces:**

- Consumes: nothing from other tasks.
- Produces: a lab with no `pcmk-rhel` arm; `parity.MATRIX` has five arms → four;
  `parity.render_markdown()` header auto-drops the column (it is `list(MATRIX)`-driven).

- [ ] **Step 1: Update the parity tests to the post-removal state (red)** — in
  `tests/test_parity.py`: delete `test_pcmk_rhel_starts_not_yet_everywhere`
  (the `def` and its body, ~L24–27); in the render test change the expected header
  assertion from
  `assert "| verb | pcmk-ubuntu | pcmk-rhel | rdqm-rhel |" in text` to
  `assert "| verb | pcmk-ubuntu | rdqm-rhel |" in text`; and remove the
  `"pcmk-rhel",` element from the arms list (~L49).

- [ ] **Step 2: Run to verify it fails** — `uv run pytest tests/test_parity.py -v`
  → FAIL (the header still contains `pcmk-rhel` because `MATRIX` still has it).

- [ ] **Step 3: Remove pcmk-rhel from the parity matrix** — in `src/mqlab/parity.py`
  delete the line `"pcmk-rhel": dict.fromkeys(VERBS, Support.NOT_YET),` from
  `MATRIX`, and drop the now-stale `pcmk-rhel (issue #238) is …` clause from the
  preceding comment block (keep the `rdqm-rhel` / `nativeha-rhel` notes intact).

- [ ] **Step 4: Run to verify it passes** — `uv run pytest tests/test_parity.py -v`
  → PASS.

- [ ] **Step 5: Remove the topology arm** — in `lab/topology.yaml` delete the
  `pcmk-rhel` arm node block (the `# --- pcmk-rhel arm …` banner + `san-a-rhel`,
  `pcmk-rhel-a1`, `pcmk-rhel-a2`, `pcmk-rhel-a3` entries, L255–276, including the
  `net-san-rhel-a (10.40.3.0/24)` plane comment L257) and the two groups
  `san_a_rhel:  [san-a-rhel]` and `pcmk_rhel_a: [pcmk-rhel-a1, …]` (L367–368).

- [ ] **Step 6: Reclaim the `.7x` allocation comments** — update the two
  allocation-map comments that reserved `.7x` for this arm: L282
  (`# Host octets .9x — kept clear of the pcmk-rhel arm's .7x block …`) and L318
  (`# arm (rdqm .3x/.4x, pcmk .5x/.6x, pcmk-rhel .7x, nha-rhel .9x) …`) — drop the
  `pcmk-rhel .7x` reservation so the map reflects that `.7x` is now free.

- [ ] **Step 7: Delete the pcmk-rhel-exclusive Ansible assets** —
  `vrg-git rm ansible/site-pcmk-rhel.yml`;
  `vrg-git rm -r ansible/roles/rhel-ha-repo`;
  `vrg-git rm ansible/roles/pcmk-cluster/tasks/install-RedHat.yml ansible/roles/pcmk-stonith/tasks/install-RedHat.yml`.
  These two roles are now **Debian-only** (their `main.yml` `include_tasks:
  install-{{ ansible_os_family }}.yml` fails loud if a future RHEL arm re-enters —
  intended, per spec D1).

- [ ] **Step 8: Clean the three incidental comments** — remove the pcmk-rhel
  mention from `ansible/site-nativeha.yml:113` (the `.7x is pcmk-rhel` aside),
  `ansible/roles/alloy/tasks/install.yml:23` (drop `pcmk-rhel` from the RHEL-arm
  list, leaving `nativeha-rhel / rdqm-rhel`), and simplify the
  `ansible/bake-mq-rdqm.yml:25-27` note (the `rhel-ha-repo` it contrasts against no
  longer exists — keep the point that RDQM's Pacemaker/DRBD comes from the MQ
  Advanced tar PreReqs, drop the `that repo is the pcmk-rhel arm's` clause).

- [ ] **Step 9: Dead-code sweep (one-time)** — run
  `grep -rInE 'pcmk[-_]rhel|san[-_]a[-_]rhel' . | grep -vE '^\./build/|^\./docs/(specs|plans|reports)/'`
  and confirm **zero** matches remain (every legitimate remaining match lives only
  in the immutable historical docs, which the filter excludes). No permanent guard
  test is added (spec §5).

- [ ] **Step 10: Full gate** — `vrg-container-run -- vrg-validate` → PASS.

- [ ] **Step 11: Commit** — `vrg-commit --type refactor --scope lab --message "decommission the pcmk-rhel arm — topology, ansible, parity (#138)"`

---

## Task 2: Docs decommission (closing bookend, `#795`)

**Platform:** agnostic-code · **Repo:** `mq-resiliency-lab-for-linux` · **Runs after T1 merges.**

**Files:**

- Modify: `README.md:34` (the RHEL box/subscription note)
- Modify: `docs/development/box-model.md:65` (the not-yet-baked RHEL arms note)
- Modify: `docs/development/box-bake-manifest.md:219` (the pcmk-rhel bake note)

**Interfaces:**

- Consumes: the merged T1 removal (the arm no longer exists in code).
- Produces: living docs with no `pcmk-rhel` arm; the epic's closing bookend.

- [ ] **Step 1: README** — at `README.md:34` remove `pcmk-rhel` from the list of
  RHEL-based arms that need a RHEL box/subscription (leave `RDQM` and the
  Native-HA-RHEL arm; the sentence still reads correctly).

- [ ] **Step 2: box-model.md** — at `docs/development/box-model.md:65` drop
  `pcmk-rhel-*` and `san-a-rhel` from the "RHEL arms that are not yet baked"
  example so it names only the arms that still exist (RDQM); reword if the
  sentence now lists a single arm.

- [ ] **Step 3: box-bake-manifest.md** — at `docs/development/box-bake-manifest.md:219`
  remove the per-role bake row/paragraph that describes the **pcmk-rhel** arm
  (`site-pcmk-rhel.yml`, its use of `pcmk-cluster`/`rhel-ha-repo`); keep the RDQM
  and Ubuntu-arm rows intact.

- [ ] **Step 4: Verify `docs/site` is clean** — run
  `grep -rIn 'pcmk[-_]rhel\|san[-_]a[-_]rhel' docs/site` → **zero** matches
  (confirmed none at design time). If any appear, fix them here; if they turn out
  to live in another repo's docs, spawn a per-repo doc task under `#138` rather
  than reaching across the repo boundary.

- [ ] **Step 5: Living-docs sweep** — run
  `grep -rInE 'pcmk[-_]rhel|san[-_]a[-_]rhel' README.md docs/development docs/reference docs/site`
  → zero matches. (The immutable `docs/specs|plans|reports/*` are intentionally
  excluded and untouched.)

- [ ] **Step 6: Full gate** — `vrg-container-run -- vrg-validate` → PASS (doc-lint
  clean).

- [ ] **Step 7: Commit** — `vrg-commit --type docs --scope lab --message "remove the pcmk-rhel arm from the living docs (#138)"`

---

## Terminal: Retrospective (`#140`, `.github`)

Once T1 and T2 have merged (every other child of `#138` closed), author the
retrospective with **`epic-retrospective`** — its preflight refuses to run until
then. Its docs PR publishes `epics/138-decommission-pcmk-rhel/retrospective.md`
and closes the epic. The human reviews and submits that PR.

---

## Self-Review

**Spec coverage:**

- D1 surgical/pcmk-rhel-exclusive removal → T1 Steps 5–8 (topology/ansible),
  Steps 1–4 (parity); the shared roles retained, RHEL variants removed
  (Debian-only) → Step 7. ✓
- D1 Debian-only consequence recorded → Step 7 note. ✓
- D2 `san-a-rhel` removed here → T1 Step 5 (in the arm block). ✓
- D3 living-docs-only / history immutable → T2 (only README + docs/development),
  sweep filters exclude `docs/specs|plans|reports`. ✓
- §5 acceptance (`vrg-validate` + one-time sweep, no guard test) → T1 Steps 9–10,
  T2 Steps 5–6. ✓
- §6 open items (enumerate `.7x` nets; confirm dead RHEL variants; incidental
  comments) → T1 Steps 5–8. ✓

**Placeholder scan:** every step names exact files/lines and the concrete edit;
the parity change ships the literal before/after assertion strings. No TBDs.

**Type consistency:** `parity.MATRIX` / `parity.render_markdown()` /
`parity.supported()` are unchanged in signature; only a dict entry is removed, and
`render_markdown()` is already `list(MATRIX)`-driven so the header follows.
