# Lab bootstrap performance — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Eliminate the identified I/O-bound inefficiencies in the `rdqm-rhel` bootstrap by baking static software into minimal per-role boxes (`mq-rdqm-rhel9`, `obs-ubuntu2404`, `infra-ubuntu2404`) instead of installing it on every node on every run.

**Architecture:** Six implementation tasks landing in **`mq-resiliency-lab-for-linux`** (this plan lives in `.github`). Task 1 splits the existing Ansible roles along a bake/configure line and proves each bake role runs standalone. Task 2 generalizes the RHEL-only `build-box.sh` into a **provision-then-snapshot** fat-box builder (boot base box → run a bake playbook → `qemu-img convert` → host-durable `build/state/boxes/` cache → `vagrant box add`) and wires it into `mqlab`. Tasks 3–5 build the three RDQM-lab boxes and repoint topology at them. Task 6 is the standalone eliminations. Two operational tasks (deployment, validation) close the loop. Success is the elimination of each inefficiency, **measured after** with the existing instrumentation — not a target.

**Tech Stack:** Ansible (roles/plays), IBM MQ 9.4.5 Developer edition, RDQM/DRBD 9.2/pacemaker on RHEL 9 (kernel-pinned), alloy/node-exporter/Prometheus/Grafana/Loki, BIND; `mqlab` (Python 3 + Typer, pytest @ 100% branch coverage); libvirt/Vagrant lab; `build-box.sh` (bash, `--dry-run` decision surface).

## Global Constraints

- **Success = each identified inefficiency eliminated, measured after — not a target percentage.** Re-run the instrumented bootstrap (`profile_tasks` + `tools/sample-host-resources.sh`) and record the observed before/after as evidence.
- **Bake/configure disjoint, explicit, reuse existing role logic** — no reimplementation of install steps.
- **Single-host bakeability:** every bake role must run green against a lone single-host inventory; inventory/group/`run_once`/delegation-coupled install roles are refactored into clean `*-install` sub-roles.
- **Provision-then-snapshot only** — not kickstart `%post`, not Packer. Host-durable `build/state/boxes/` cache; **no external registry**.
- **No new credential surface:** MQ Developer edition + all OSS fetched anonymously; the RHEL DVD is the only credentialed prefetch, already staged by `build-box.sh` and never committed. No entitlement artifact enters a committed box or shared cache.
- **`mq-rdqm-rhel9` kernel is pinned** for DRBD kmod compatibility.
- **Staleness:** base-OS update (`dnf`/`apt update`) at instance-build + manifest-hash rebuild trigger + graduated age policy (**warn 7d, refuse 14d**).
- **Derive, never hardcode:** boxes/platforms come from `lab/topology.yaml`. No box-name or IP literal in shipped code.
- **Validation gate is exactly one command:** `vrg-container-run -- vrg-validate`. Dev-loop test command is `uv run pytest …` (build-tool use only; never embed `uv run` in shipped code/scripts).
- **Coverage floor: 100% branch** for `mqlab` — `uv run pytest --cov=src --cov-branch --cov-fail-under=100`; `# pragma: no cover` only for genuinely unreachable guards.
- **Cold-rebuild acceptance gate** applies to every box task (2–6): a full VM cold rebuild must prove it one-pass; lint-green ≠ done. The human operates the lab.
- **Each task = one GitHub issue** in the lab repo, on `feature/<issue>-<slug>` off `develop`; commit with `vrg-commit`; PR into `develop`.

## Task dependency graph

```text
T1 (bake/configure split) ─┐
T2 (fat-box builder)      ─┴─▶ T3 (mq-rdqm-rhel9) ─┐
                               T4 (obs-ubuntu2404) ─┼─▶ Deployment ─▶ Validation
                               T5 (infra-ubuntu2404)┘   (bake+cache)    (cold-rebuild green
                                                                          + measure; run #551/#565)
T6a (acl + DRBD shrink) ── no blockers (parallel)
T6b (concurrency tuning) ── blocked-by T3,T4,T5 (after baking)
```

- **T1, T2** — foundational, no blockers, parallel-safe.
- **T3, T4, T5** — each blocked-by **T1 + T2**.
- **Deployment** (operational) — blocked-by **T3, T4, T5**.
- **Validation** (operational) — blocked-by **Deployment**.
- **T6a** — no blockers. **T6b** — blocked-by **T3, T4, T5**.

---

### Task 1: Bake/configure split + single-host bake foundation

Additive foundation: introduce the bake playbooks and prove they run standalone, **without changing bootstrap behavior** (the per-run skips land per-box in Tasks 3–5).

**Files:**

- Create: `ansible/bake-mq-rdqm.yml`, `ansible/bake-obs.yml`, `ansible/bake-infra.yml`
- Create: `ansible/inventory/bake-host.ini` (minimal single-host inventory for the transient build VM)
- Create: `docs/development/box-bake-manifest.md` (authoritative per-role bake-vs-configure classification, from spec §4.1)
- Modify (only where coupling requires): extract `*-install` sub-roles from install roles that carry `run_once`/`delegate_to`/group coupling (candidates: `mq-exporter` if its Go build delegates, `alloy`, `node-exporter`, `mq-install`, `rdqm-install`, `bind-dns`, `grafana`/`prometheus`/`loki`)

**Interfaces:**

- Produces: three bake playbooks (each includes only bake-classified roles/sub-roles) + `bake-host.ini`, consumed by Task 2's builder. The classification doc other tasks cite.

- [ ] **Step 1: Author the classification doc** `docs/development/box-bake-manifest.md` — the per-role table from spec §4.1, naming for each role whether it is bake, configure, or split, and for split roles which tasks/tags go where.
- [ ] **Step 2: For each bake role, prove single-host runnability.** Run the role against `bake-host.ini` (one build VM) and record pass/fail. For each failure caused by `run_once`/`delegate_to: localhost`/group references, extract an `<role>-install` sub-role containing only the host-local install tasks (no delegation, no `run_once`, no cross-host facts).
- [ ] **Step 3: Write the three bake playbooks**, each `hosts: bake` running exactly the bake roles/sub-roles for that box (mq-rdqm: `mq-install`, `rdqm-install`, `rhel-ha-repo`, `node-exporter`(install), `alloy`(install), `mq-exporter`(build), build tools, `acl`, the #569 qm.ini default seed; obs: `grafana`/`prometheus`/`loki`/`alloy`(install), `node-exporter`; infra: `bind-dns`(install), `node-exporter`, `alloy`(install)).
- [ ] **Step 4: Syntax + lint.** `ansible-playbook --syntax-check ansible/bake-*.yml`; then `vrg-container-run -- vrg-validate` — Expected: PASS.
- [ ] **Step 5: Commit** — `vrg-commit --type refactor --scope ansible --message "split provisioning into bake vs configure; add per-box bake playbooks (#<issue>)" --body "…epic logical-minds-foundry/.github#70"`.

**Acceptance:** each bake playbook contains only bake-classified roles and runs green against `bake-host.ini`; no change to a normal bootstrap (still installs everything until Tasks 3–5). Full proof of correctness rides Task 3's box build.

---

### Task 2: Generalized provision-then-snapshot fat-box builder + `mqlab` wiring

**Files:**

- Create: `lab/boxes/build-fatbox.sh` (parameterized `--box <name> --domain-type <..> --cpu-mode <..> [--rebuild-box] [--dry-run]`)
- Create: `lab/boxes/_manifest-hash.sh` (compute a box's bake-manifest hash — the package/version pin set + bake playbook path)
- Modify: `src/mqlab/cli.py` — extend `_LOCAL_BOX_BUILDERS` to map each fat box → `build-fatbox.sh` (with its `--box`); keep `rhel/9.6-x86_64` → `build-box.sh` as the base-OS builder the fat RHEL build depends on
- Modify: `tests/test_cli_bootstrap.py` (or Create: `tests/test_box_build.py`) — TDD the Python resolution

**Interfaces:**

- Consumes: bake playbooks + `bake-host.ini` (Task 1).
- Produces: `build-fatbox.sh` that, per box, ensures the base box → boots a transient VM → runs the box's bake playbook against `bake-host.ini` → `qemu-img convert -c` → caches `build/state/boxes/<box>.box` → `vagrant box add <box>`. Decision surface via `--dry-run` (`REUSE`/`BUILD`/`FORCE-BUILD`/`STALE`). `mqlab`'s `_needed_local_boxes` now resolves fat boxes from per-node `box` fields.

- [ ] **Step 1: Write the failing test for fat-box resolution.** In `tests/test_box_build.py`:

```python
from mqlab import cli

def test_needed_local_boxes_resolves_fat_rdqm_box(monkeypatch, tmp_path):
    # resolved topology points the rdqm nodes at the fat box
    monkeypatch.setattr(cli, "_resolved_nodes", lambda: {
        "rdqm-a1": {"box": "mq-rdqm-rhel9"}, "obs": {"box": "obs-ubuntu2404"},
    })
    needed = cli._needed_local_boxes(["rdqm-a1", "obs"])
    assert "mq-rdqm-rhel9" in needed
    assert needed["mq-rdqm-rhel9"].endswith("build-fatbox.sh")
    assert "obs-ubuntu2404" in needed
```

- [ ] **Step 2: Run it, verify it fails** — `uv run pytest tests/test_box_build.py::test_needed_local_boxes_resolves_fat_rdqm_box -v` → FAIL (fat boxes not in `_LOCAL_BOX_BUILDERS`).
- [ ] **Step 3: Implement.** Extend `_LOCAL_BOX_BUILDERS` in `cli.py` to include the fat boxes mapped to `lab/boxes/build-fatbox.sh`, and (if `_box_build_steps` hardcodes a single script's args) generalize it to pass `--box <name>` for `build-fatbox.sh` entries. Keep `rhel/9.6-x86_64 → build-box.sh`.
- [ ] **Step 4: Run tests + coverage** — `uv run pytest tests/test_box_build.py tests/test_cli_bootstrap.py --cov=src --cov-branch --cov-fail-under=100 -v` → PASS.
- [ ] **Step 5: Write `build-fatbox.sh`** — mirror `build-box.sh`'s cache/`--dry-run`/`REUSE` structure (host-durable `build/state/boxes/<box>.box`, git-common-dir main-worktree resolution), but the BUILD path: ensure base box present (invoke `build-box.sh` for the RHEL base; `vagrant box add` the cloud Ubuntu base) → define a transient build domain from the base → `ansible-playbook -i ansible/inventory/bake-host.ini ansible/bake-<box>.yml` → poweroff → `qemu-img convert -c` → tar `.box` → cache → `vagrant box add --force <box>`. Staleness: compute `_manifest-hash.sh <box>`, store it beside the cached box; `REUSE` only if the stored hash matches AND age < 7d; **warn 7–14d, refuse ≥14d** (exit non-zero demanding `--rebuild-box`).
- [ ] **Step 6: Test the decision surface** — add a test asserting `build-fatbox.sh --box mq-rdqm-rhel9 --dry-run` prints `BUILD` on a clean cache and `REUSE` when a matching-hash box is cached (mirror `build-box.sh`'s dry-run test).
- [ ] **Step 7: Validate + commit** — `vrg-container-run -- vrg-validate`; `vrg-commit --type feat --scope mqlab --message "provision-then-snapshot fat-box builder + mqlab wiring (#<issue>)" --body "…#70"`.

**Acceptance:** the builder produces and caches a fat box from a bake playbook; `--dry-run` REUSE/BUILD/stale decisions are unit-tested; `mqlab` resolves and builds fat boxes; bake reuses the existing DVD staging + anonymous fetches (no entitlement artifact committed/shared). Full box correctness proven in Task 3.

---

### Task 3: `mq-rdqm-rhel9` box (the RDQM node box)

**Files:**

- Create: `lab/boxes/mq-rdqm-rhel9/manifest.yml` (base `rhel/9.6-x86_64`, bake `ansible/bake-mq-rdqm.yml`, MQ 9.4.5.0 + RDQM/DRBD + alloy/node-exporter/exporter/build-tool version pins)
- Modify: `ansible/bake-mq-rdqm.yml` (finalize the bake set incl. `acl` + the #569 `DiagnosticMessages` default seeded into the box image)
- Modify: `lab/topology.yaml` — add `mq-rdqm-rhel9` to `boxes:`; repoint `rdqm-a1..3`, `rdqm-b1..3` `platform`/`box` to it (keep the kernel-pin + `extra_disk`, DVD as needed)
- Modify: `ansible/site-rdqm.yml` + affected roles — **skip the now-baked installs at bootstrap** (the bake roles no longer run per-run); add the base-OS update (`dnf update`) to the configure path (cross-cutting to all boxes)

**Interfaces:**

- Consumes: `build-fatbox.sh` (Task 2), `bake-mq-rdqm.yml` (Task 1).
- Produces: the RDQM nodes boot from `mq-rdqm-rhel9`; the per-run path contains only configure roles.

- [ ] **Step 1: Write the box manifest** and finalize `bake-mq-rdqm.yml` (bake the #569 `DiagnosticMessages` default so `RDQMAPP` is born with it — eliminating the post-create `endmqm -w`/`strmqm` bounce).
- [ ] **Step 2: Build the box** — `mqlab` (or `build-fatbox.sh --box mq-rdqm-rhel9`) produces `build/state/boxes/mq-rdqm-rhel9.box`.
- [ ] **Step 3: Repoint topology** and strip the baked installs from `site-rdqm.yml`/roles; add `dnf update` to configure.
- [ ] **Step 4: Cold rebuild** — `mqlab rebuild rdqm-rhel` (human-operated). Verify from the transcript: **no `mq-install`/`rdqm-install`/`alloy`/`mq-exporter` install tasks run during bootstrap**; the QM is born with its diagnostic stanza (no bounce); the RDQM HA+DR replication comes up (`drbdadm status` Connected/UpToDate; DR link syncs).
- [ ] **Step 5: Validate + commit** — `vrg-container-run -- vrg-validate`; `vrg-commit --type feat --scope rdqm --message "boot RDQM nodes from the baked mq-rdqm-rhel9 box (#<issue>)" --body "…#70"`.

**Acceptance (cold-rebuild gate):** RDQM lab comes up with the heavy installs gone from bootstrap; transcript proves the bake roles do not run per-run; replication healthy.

---

### Task 4: `obs-ubuntu2404` box (the observability box)

**Files:**

- Create: `lab/boxes/obs-ubuntu2404/manifest.yml` (base `cloud-image/ubuntu-24.04`, bake `ansible/bake-obs.yml`, Grafana/Prometheus/Loki/alloy/node-exporter pins)
- Modify: `ansible/bake-obs.yml`; `lab/topology.yaml` (`obs` node `box` → `obs-ubuntu2404`); `ansible/observability.yml`/`host-obs.yml` (skip baked installs; keep dashboards/scrape config per-run)

**Interfaces:** consumes Task 2 builder + Task 1 `bake-obs.yml`; produces the `obs` node booting from `obs-ubuntu2404`.

- [ ] **Step 1: Manifest + finalize `bake-obs.yml`.**
- [ ] **Step 2: Build the box.**
- [ ] **Step 3: Repoint `obs` in topology; strip baked obs-stack installs from the per-run observability plays; keep dashboards/scrape/alert config per-run.**
- [ ] **Step 4: Cold rebuild** — verify obs comes up (targets green, dashboards present) with **no obs-stack install during bootstrap**.
- [ ] **Step 5: Validate + commit** — `vrg-commit --type feat --scope obs --message "boot obs from the baked obs-ubuntu2404 box (#<issue>)"`.

**Acceptance (cold-rebuild gate):** obs boots baked; dashboards/targets work; no obs-stack install at bootstrap.

---

### Task 5: `infra-ubuntu2404` box (the infrastructure box)

**Files:**

- Create: `lab/boxes/infra-ubuntu2404/manifest.yml` (base `cloud-image/ubuntu-24.04`, bake `ansible/bake-infra.yml`, BIND + node-exporter/alloy pins)
- Modify: `ansible/bake-infra.yml`; `lab/topology.yaml` (`infra-client`, `infra-svc` `box` → `infra-ubuntu2404`); `ansible/site-dns.yml` (skip baked BIND install; keep **zone data** per-run)

**Interfaces:** consumes Task 2 builder + Task 1 `bake-infra.yml`; produces the infra nodes booting from `infra-ubuntu2404`.

- [ ] **Step 1: Manifest + finalize `bake-infra.yml`** (BIND install baked; zones stay per-run — infra is the generic node-role, BIND is its first tenant per spec §4.3).
- [ ] **Step 2: Build the box.**
- [ ] **Step 3: Repoint `infra-client`/`infra-svc`; strip baked BIND install from `site-dns.yml`; keep zone generation per-run.**
- [ ] **Step 4: Cold rebuild** — verify DNS resolves (both authoritative sides) with **no BIND install during bootstrap**.
- [ ] **Step 5: Validate + commit** — `vrg-commit --type feat --scope dns --message "boot infra nodes from the baked infra-ubuntu2404 box (#<issue>)"`.

**Acceptance (cold-rebuild gate):** infra boots baked; DNS works; only zone data applied per-run.

---

### Task 6a: `acl` anomaly + DRBD volume shrink (no blockers — parallel)

**Files:**

- Investigate/Modify: the `acl`-install path (root-cause the dnf/DVD-repo metadata stall)
- Modify: `lab/topology.yaml` (`rdqm-*` `extra_disk` 10→ smaller if the PV drives it) + the DRBD/drbdpool volume config (3 GiB → ~1 GiB)

**Interfaces:** independent of the box tasks; lands in parallel with everything.

- [ ] **Step 1: `acl` root-cause.** Reproduce the 5.8-min stall, identify the dnf/DVD-repo cause (metadata refresh, repo priority, `fastestmirror`, or DVD I/O), and fix at root; `acl` is baked (present) either way — this removes the *install stall*, not the package.
- [ ] **Step 2: Shrink DRBD volumes 3→1 GiB.** Adjust the drbdpool PV / logical-volume sizing; **verify the lab QMs fit** ~1 GiB (queue depths + logs). Re-run a DR cutover to confirm shorter full resync and a shorter #591 DR-wait.
- [ ] **Step 3: Validate + commit each independently.**

**Acceptance:** the `acl` stall no longer appears in the transcript; DRBD volumes are ~1 GiB with shorter resyncs.

---

### Task 6b: Concurrency / fork tuning (blocked-by the boxes — sequenced last)

**Files:**

- Modify: `ansible/ansible.cfg` (`forks`) and/or provision/observe overlap — guided by a re-measured sampler run **after** baking has removed most of the disk I/O

**Interfaces:** blocked-by Tasks 3–5 (baking changes the I/O picture first). Measurement-driven; may legitimately be a no-op.

- [ ] **Step 1: Re-measure.** After the boxes land, re-run the instrumented bootstrap (`profile_tasks` + sampler) and read the new iowait/idle profile.
- [ ] **Step 2: Tune only if the disk is no longer saturated.** Raise `forks` and/or overlap observe with provision; re-measure; keep only what the sampler shows helps (more parallelism on a still-saturated disk backfires).
- [ ] **Step 3: Validate + commit** — with the before/after sampler reading in the commit body.

**Acceptance:** any concurrency change is backed by a before/after sampler reading; no change ships without measured benefit.

---

## Operational tasks (filed under #70; run via issue-deploy / issue-validate — NOT PR-workable)

- **Deployment** — bake & cache the three boxes (`mq-rdqm-rhel9`, `obs-ubuntu2404`, `infra-ubuntu2404`) into `build/state/boxes/` so a bootstrap can boot them. Blocked-by Tasks 3–5. Run with `issue-deploy`.
- **Validation** — cold rebuild `rdqm-rhel` comes up green **and** record the instrumented before/after (`profile_tasks` + sampler). Blocked-by the deployment. Supersedes the `#565`-blocked validation; while the lab is stable, also run the **#551** RDQM replication-TLS checklist (its close is gated on this epic). Run with `issue-validate`.

## Bookend tasks (already created)

- **`.github#71` Documentation** — this spec + plan (first task; its PR publishes them).
- **`.github#72` Follow-on brainstorm** — generalization epic (nativeha + pcmk boxes, CI box-library, cross-technology `obs`/`infra`, registry/hardware-if-needed) + what-shipped review (closing).
- **`mq-resiliency-lab-for-linux#601` Documentation review** — site docs reflect the model (`docs/development/build-layout.md` + the new box-taxonomy / bake-vs-configure page) — final close gate.

## Self-Review

- **Spec coverage:** §4.1 split → Task 1; §4.2 builder + sources → Task 2; §4.3 taxonomy/3 boxes → Tasks 3–5; §4.4 standalone → Task 6a (`acl` + DRBD) + Task 6b (concurrency); §4.5 staleness → Task 2 (Step 5) + base-OS-update Task 3 (Step 3); §7 verification → cold-rebuild gates + operational validation; §8 #551/#565 dependency → validation task. All covered.
- **Placeholders:** none — file paths, the TDD test, and commands are concrete; per-box steps name real playbooks/roles.
- **Type consistency:** `_LOCAL_BOX_BUILDERS`, `_needed_local_boxes`, `_box_build_steps`, `parse_box_list`, `_resolved_nodes` match `cli.py`; `build-fatbox.sh`/`_manifest-hash.sh`/`bake-host.ini`/`bake-<box>.yml` referenced consistently across tasks.
