# Bake the native-HA RHEL arm — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Eliminate the per-run MQ install on the native-HA RHEL arm by baking it into a new `mq-nativeha-rhel9` fat box, and prove a fast, reliable, one-shot cold rebuild of the arm **in the cloud on base x86** — reusing epic #70's proven RDQM-baking pipeline verbatim.

**Architecture:** Two implementation tasks landing in **`mq-resiliency-lab-for-linux`** (this plan lives in `.github`). Task 1 builds the box: a new `bake-nativeha-rhel.yml` that bakes only the native-HA install *adapter*, the `build-fatbox.sh` / `_manifest-hash.sh` / `mqlab` wiring, and — the one genuinely new refactor — the #659 skip-if-baked guard on `install-RedHat.yml` so a baked node stops re-copying the ~2 GiB MQ tarball. Task 2 repoints the six `nha-rhel-*` nodes at the box, applies phased startup, and proves via a cold rebuild that the baked installs no longer run. Two operational tasks (deployment = bake+cache; validation = the cloud/x86 cold rebuild + native-HA switchover/CRR) close the loop. Success is the elimination of the install work, **measured after** — not a target.

**Tech Stack:** Ansible (roles/plays), IBM MQ 9.4.5 Developer edition (base product, **no RDQM/DRBD**), Native HA (raft log replication — **no kernel module**) on RHEL 9.6 x86_64; alloy/node-exporter; `mqlab` (Python 3 + Typer, pytest @ 100% branch coverage); libvirt/Vagrant lab (TCG on the cloud x86 host); `build-fatbox.sh` provision-then-snapshot builder (already exists — #628).

## Global Constraints

- **Success = the per-run native-HA MQ install eliminated, measured after — not a target percentage.** Confirm from the bootstrap transcript that no MQ tar copy / unpack / `dnf install` runs on a baked node; record the observed before/after.
- **Reuse the #70 pipeline verbatim** — `build-fatbox.sh`, per-box bake playbook, `_manifest-hash.sh`, `mqlab` `_LOCAL_BOX_BUILDERS`, phased startup (#642), skip-if-baked (#648/#659), machine-id reset (#654). Add **no** parallel tooling.
- **Bake the install *adapter* only, never formation.** `bake-nativeha-rhel.yml` runs `mq-nativeha` with `tasks_from: install-RedHat`; `main.yml` (`crtmqm`, peer set, `mqmonitor@`) must **not** execute at bake.
- **`mq-install` is Ubuntu-only** (apt/deb, `UbuntuLinux` tar) — it cannot install on RHEL; the RHEL install stays on `install-RedHat.yml` (rpm/dnf, `LinuxX64` tar). The skip-if-baked fix is applied *to `install-RedHat.yml`*.
- **`mq-nativeha-rhel9` has no kernel pin, no `extra_disk`, no RDQM/DRBD/Pacemaker.** Native HA replicates in the raft log — no kmod. Cut 1 bakes from the stock `rhel/9.6-x86_64` base; leading-edge kernel is parked.
- **OS currency by box rebuild, not boot-time update** (#639/#78) — no `dnf update` at instance-build.
- **Run and validate entirely in the cloud on base x86.** The arm is already `rhel/9.6-x86_64` (x86 forced); the validation is a cloud/x86 cold rebuild. The human operates the lab.
- **Derive, never hardcode:** the box/platform come from `lab/topology.yaml`. No box-name or IP literal in shipped code.
- **Validation gate is exactly one command:** `vrg-container-run -- vrg-validate`. Dev-loop test command is `uv run pytest …` (build-tool use only; never embed `uv run` in shipped code/scripts).
- **Coverage floor: 100% branch** for `mqlab` — `uv run pytest --cov=src --cov-branch --cov-fail-under=100`; `# pragma: no cover` only for genuinely unreachable guards.
- **Cold-rebuild acceptance gate** applies to Task 2 and the validation: a full cloud/x86 cold rebuild must prove it one-pass; lint-green ≠ done.
- **Each task = one GitHub issue** in the lab repo, on `feature/<issue>-<slug>` off `develop`; commit with `vrg-commit`; PR into `develop`.

## Task dependency graph

```text
T1 (mq-nativeha-rhel9 box + install-RedHat.yml skip-if-baked guard + mqlab wiring)
      │
      ▼
T2 (repoint nha-rhel-* + phased startup + cold-rebuild proof)
      │
      ▼
Deployment (bake + cache mq-nativeha-rhel9)  ──▶  Validation #665
                                                  (cloud/x86 one-shot cold rebuild green
                                                   + measure; native-HA switchover + CRR/DR)
```

- **T1** — foundational, no blockers.
- **T2** — blocked-by **T1** (needs the guard + the buildable box).
- **Deployment** (operational) — blocked-by **T1** (the box must be bakeable/cacheable).
- **Validation `#665`** (operational) — blocked-by the **Deployment** and **T2**.

---

### Task 1: The `mq-nativeha-rhel9` box + the skip-if-baked seam

Additive foundation: make the box buildable and make the per-run RHEL MQ install skip when already baked — **without** yet repointing any node (un-baked `nha-rhel-*` nodes still install from the base box until Task 2). Modelled on #70 Tasks 2–3, minus the builder (which already exists).

**Files:**

- Create: `ansible/bake-nativeha-rhel.yml` (bake playbook — mirrors `ansible/bake-mq-rdqm.yml`)
- Modify: `ansible/roles/mq-nativeha/tasks/install-RedHat.yml` (add the #659 `stat cmqc.h` skip-if-baked guard)
- Modify: `lab/boxes/build-fatbox.sh:62-65` (add the `mq-nativeha-rhel9` `case` arm)
- Modify: `lab/boxes/_manifest-hash.sh:51-55` (add the `mq-nativeha-rhel9 → nativeha-rhel` stem mapping)
- Modify: `src/mqlab/cli.py:887` (add `mq-nativeha-rhel9` to `_LOCAL_BOX_BUILDERS`)
- Test: `tests/test_box_build.py` (resolution), and an `install-RedHat.yml`-guard assertion (see Step 6)

**Interfaces:**

- Consumes: the existing `build-fatbox.sh` builder (#628), `bake-host.ini`, the `mq-nativeha` role.
- Produces: `build/state/boxes/mq-nativeha-rhel9.box` buildable via `build-fatbox.sh --box mq-nativeha-rhel9` / `mqlab`; a per-run `install-RedHat.yml` that no-ops when `/opt/mqm/inc/cmqc.h` exists. `_manifest-hash.sh mq-nativeha-rhel9` digests `bake-nativeha-rhel.yml` + its role closure.

- [ ] **Step 1: Add the `install-RedHat.yml` skip-if-baked guard.** At the top of `ansible/roles/mq-nativeha/tasks/install-RedHat.yml`, add a `stat` of `/opt/mqm/inc/cmqc.h` registered as `mq_installed`, and gate the **DVD mount**, **repo drop**, **tar copy**, and **unpack** steps with `when: not mq_installed.stat.exists` (mirroring `ansible/roles/mq-install/tasks/main.yml:10-31`, the #648/#659 pattern). The `dnf install` / `setmqinst` steps keep their existing `creates:` guards (idempotent no-ops when baked). This is the fix for the currently-**unguarded ~2 GiB tar copy** (`install-RedHat.yml:29-33`).

```yaml
- name: check for an already-installed MQ (skip the baked install)
  ansible.builtin.stat:
    path: /opt/mqm/inc/cmqc.h
  register: mq_installed

# ...then on the DVD mount, repo drop, tar copy, and unpack tasks:
  when: not mq_installed.stat.exists
```

- [ ] **Step 2: Author `ansible/bake-nativeha-rhel.yml`.** Copy `ansible/bake-mq-rdqm.yml`'s structure; swap the install include to bake the **native-HA adapter only** and drop the RDQM `rdqm.service` benign-exception (there is no RDQM service). Bake set: `acl`, the native-HA MQ install adapter, the diagnostic default, `node-exporter` (full role, enabled), `alloy` install-half (inert).

```yaml
---
# Bake mq-nativeha-rhel9 (install-only; no QMs, no formation, no secrets). Native
# HA replicates in the raft log — NO RDQM/DRBD/Pacemaker, NO kernel pin.
- name: Bake mq-nativeha-rhel9 (install adapter only)
  hosts: bake
  become: true
  vars:
    mq_exporter_tls: false
  tasks:
    - name: acl package (unprivileged become prereq)
      ansible.builtin.dnf: { name: acl, state: present }

    # Base MQ (no RDQM) via the native-HA OS adapter — install body only; main.yml's
    # crtmqm/formation is a per-run concern and must NOT run at bake.
    - name: base IBM MQ (native-HA RHEL install adapter)
      ansible.builtin.include_role:
        name: mq-nativeha
        tasks_from: install-RedHat

    - name: node metrics agent (node-exporter, full role — static config, enabled)
      ansible.builtin.include_role: { name: node-exporter }

    - name: log shipper binary (alloy install half only)
      ansible.builtin.include_role: { name: alloy, tasks_from: install }
```

- [ ] **Step 3: Add the `build-fatbox.sh` case arm** at `lab/boxes/build-fatbox.sh:62-65`:

```bash
  mq-nativeha-rhel9) BASE_KIND=rhel;   BASE_BOX="rhel/9.6-x86_64";        BAKE=nativeha-rhel ;;
```

- [ ] **Step 4: Add the `_manifest-hash.sh` stem mapping** at `lab/boxes/_manifest-hash.sh:51-55`:

```bash
  mq-nativeha-rhel9) BAKE_STEM=nativeha-rhel ;;
```

  Then **verify the digest is not silently empty** (the #649-class regression): run
  `lab/boxes/_manifest-hash.sh mq-nativeha-rhel9` and confirm its output contains
  `bake=ansible/bake-nativeha-rhel.yml` **and** a non-empty role digest (i.e. the
  `[ -f "$BAKE" ]` branch fired against a real playbook, not a missing path). This
  requires Step 2's `bake-nativeha-rhel.yml` to exist first.

- [ ] **Step 5: Write the failing test for `mqlab` resolution.** In `tests/test_box_build.py`, add a test that `_needed_local_boxes` resolves `mq-nativeha-rhel9` to `build-fatbox.sh` when a node points at it (mirror the existing fat-box resolution tests).

```python
def test_needed_local_boxes_resolves_nativeha_rhel_box(monkeypatch):
    monkeypatch.setattr(cli, "_resolved_nodes", lambda: {
        "nha-rhel-a1": {"box": "mq-nativeha-rhel9"},
    })
    needed = cli._needed_local_boxes(["nha-rhel-a1"])
    assert needed["mq-nativeha-rhel9"].endswith("build-fatbox.sh")
```

- [ ] **Step 6: Run it, verify it fails** — `uv run pytest tests/test_box_build.py::test_needed_local_boxes_resolves_nativeha_rhel_box -v` → FAIL (`mq-nativeha-rhel9` not in `_LOCAL_BOX_BUILDERS`).
- [ ] **Step 7: Wire `mqlab`.** Add `"mq-nativeha-rhel9": "lab/boxes/build-fatbox.sh",` to `_LOCAL_BOX_BUILDERS` (`src/mqlab/cli.py:887`).
- [ ] **Step 8: Run tests + coverage** — `uv run pytest tests/test_box_build.py tests/test_cli_bootstrap.py --cov=src --cov-branch --cov-fail-under=100 -v` → PASS.
- [ ] **Step 9: Syntax-check + build the box** — `ansible-playbook --syntax-check ansible/bake-nativeha-rhel.yml`; then `build-fatbox.sh --box mq-nativeha-rhel9` (human-operated, cloud/x86) produces `build/state/boxes/mq-nativeha-rhel9.box`. Confirm the box has `/opt/mqm/bin/crtmqm` and `/opt/mqm/samp/mqmonitor@.service` (baked `MQSeriesSamples`), and that `rdqm.service`/DRBD are absent.
- [ ] **Step 10: Validate + commit** — `vrg-container-run -- vrg-validate`; `vrg-commit --type feat --scope boxes --message "bake mq-nativeha-rhel9 + skip-if-baked the native-HA RHEL install (#<issue>)" --body "…epic logical-minds-foundry/.github#88"`.

**Acceptance:** the box builds & caches; `_manifest-hash.sh mq-nativeha-rhel9` digests `bake-nativeha-rhel.yml` + its role closure; `install-RedHat.yml` no-ops on a node where `cmqc.h` is present (tar copy skipped); no bootstrap behavior change yet on the un-baked base box.

---

### Task 2: Repoint `nha-rhel-*` + phased startup + cold-rebuild proof

**Files:**

- Modify: `lab/topology.yaml` — add a `mq-nativeha-rhel9` `boxes:` entry (mirror the `mq-rdqm-rhel9` entry: **keeps the DVD** attach for the configure-half offline repo; **no** kernel-pin note, **no** `extra_disk`); repoint `nha-rhel-a1..3`, `nha-rhel-b1..3` `platform` from `rhel96-x86_64` to the new box.
- Modify (only if needed): `ansible/site-nativeha.yml` / the `mq-nativeha` play wiring — confirm the baked install is bypassed by Task 1's guard; apply phased startup (#642) so any baked service starts per-run bottom-up, with `node_exporter` the sole benign enabled-at-bake exception.

**Interfaces:**

- Consumes: the `mq-nativeha-rhel9` box + guard (Task 1).
- Produces: the six `nha-rhel-*` nodes boot from `mq-nativeha-rhel9`; the per-run path runs only formation + config.

- [ ] **Step 1: Add the `boxes:` entry** for `mq-nativeha-rhel9` (box + `arch: x86_64` + `dvd:` the RHEL DVD; **no** `extra_disk`, **no** kernel-pin comment — annotate that native HA is raft/kmod-free). Model the block on the existing `mq-rdqm-rhel9` entry, dropping the DRBD/kernel notes.
- [ ] **Step 2: Repoint the six `nha-rhel-*` nodes** `platform:` → `mq-nativeha-rhel9` (`lab/topology.yaml:234-263`).
- [ ] **Step 3: Apply phased startup (#642)** — ensure baked services are inert and started per-run; `node_exporter` stays benign-enabled. The per-QM `mqmonitor@{qm}` units are created per-run by `main.yml` (unaffected by baking).
- [ ] **Step 4: Cold rebuild the arm** — `mqlab rebuild nativeha-rhel` (human-operated, cloud/x86). Verify from the transcript: **no MQ tar copy / unpack / `dnf install` runs on the `nha-rhel-*` nodes**; the box's `cmqc.h` short-circuits `install-RedHat.yml`; the Native HA group forms (`dspmq -o nativeha -x` shows a live instance); the app QM (`NHARAPP`) comes up.
- [ ] **Step 5: Validate + commit** — `vrg-container-run -- vrg-validate`; `vrg-commit --type feat --scope nativeha --message "boot the native-HA RHEL nodes from the baked mq-nativeha-rhel9 box (#<issue>)" --body "…#88"`.

**Acceptance (cold-rebuild gate):** the native-HA RHEL arm comes up in one pass on the cloud x86 host with the heavy MQ install gone from bootstrap; transcript proves the baked install does not run per-run; the Native HA group is healthy.

---

## Operational tasks (filed under #88; run via issue-deploy / issue-validate — NOT PR-workable)

- **Deployment** — bake & cache `mq-nativeha-rhel9` into `build/state/boxes/` so a bootstrap can boot it (the "made usable" signal). Blocked-by **Task 1**. Run with `issue-deploy`.
- **Validation `#665`** — a **cloud/x86 one-shot cold rebuild** of the native-HA RHEL arm comes up green **and** the closing before/after measurement is recorded; **plus** native-HA function from the baked box: **HA switchover** (raft re-election of a live instance) and **CRR/DR cutover** (`site-nativeha-switchover.yml`; the site-B recovery group starting `nha_start=false` until `crr.yml` sets its GroupRole, #389), with native HA's own replication + TLS — the native-HA analogue of RDQM's #551. Blocked-by the **Deployment** and **Task 2**. Run with `issue-validate`.

## Bookend tasks (already created)

- **`.github#89` Documentation** — this spec + plan (first task; its PR publishes them).
- **`.github#90` Follow-on brainstorm** — the Ubuntu-arms (native-HA-Ubuntu + pcmk) + x86-on-Ubuntu vetting + Ubuntu-security-hardening-coordination successor epic + a what-shipped review (closing).
- **`mq-resiliency-lab-for-linux#664` Documentation review** — verify the site docs (box-model / bake-manifest pages) reflect the new `mq-nativeha-rhel9` box + the no-kernel-pin refinement (final close gate).

## Self-Review

- **Spec coverage:** §4.1 box contents → T1 (bake playbook) + T2 (repoint); §4.2 builder reuse (`build-fatbox.sh` case, `_manifest-hash.sh` map, `bake-nativeha-rhel.yml`, `mqlab` wiring) → T1 Steps 2-8; §4.3 skip-if-baked guard on `install-RedHat.yml` → T1 Step 1; §4.4 repoint + strip + phased startup → T2; §4.5 no kernel pin → T2 Step 1 (boxes entry) + Global Constraints; §7 verification (build/reuse, the manifest-change rebuild trigger → T1 Step 4 verify [#649-class guard], cold-rebuild, `mqmonitor@` template, switchover/CRR, measurement) → T1 Steps 4/9 + T2 Step 4 + Validation #665. All covered.
- **Placeholders:** none — exact file paths + line anchors, the guard YAML, the bake playbook body, the `case`/`_LOCAL_BOX_BUILDERS` diffs, and the pytest are concrete.
- **Type consistency:** `_LOCAL_BOX_BUILDERS`, `_needed_local_boxes`, `_resolved_nodes` match `cli.py`; the stem `nativeha-rhel` is identical across `build-fatbox.sh` (`BAKE=nativeha-rhel`), `_manifest-hash.sh` (`BAKE_STEM=nativeha-rhel`), and the playbook name `bake-nativeha-rhel.yml`.
