# Site-docs-revamp Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. **Each task below becomes one GitHub issue filed under epic `logical-minds-foundry/.github#161`, closed by one PR** (except the operational tasks, which close on `Outcome: SUCCESS`).

**Goal:** Rethink the lab's site documentation from the ground up for the fall public release — consumer-first, matching the as-built lab — and exercise the consumer path end-to-end from a signed tarball on a clean x86 host.

**Architecture:** A journey-first MkDocs site with the Vergil-vs-consumer fork isolated to Getting Started. Docs tasks ship honestly and independently; a tooling task makes the never-run release pipeline produce a real artifact; operational tasks (deploy → validate) flip the consumer path from "documented" to "validated." A deferred methodology brainstorm gates the methodology doc.

**Tech Stack:** MkDocs Material (`docs/site/`), built via `vrg-container-docs`; Python `mqlab` orchestrator + Ansible; GitHub Actions release pipeline (`.github/workflows/release.yml`) + `git archive` curation (`.gitattributes export-ignore`); GPG-signed tarball.

## Global Constraints

- **Repo placement (placement law).** Every docs/tooling task's PR lands in the **member repo** `logical-minds-foundry/mq-resiliency-lab-for-linux` (the site docs live at `docs/site/`). Only `spec.md`/`plan.md` (this task) and `retrospective.md` live in `logical-minds-foundry/.github`. Cross-repo references are `Ref`, never `Closes`.
- **Anonymization.** All public artifacts stay generic; the methodology's example is a generic **"trading back-office gateway."** No employer/client-identifiable detail. Never name or link the private gateway repo.
- **Honest docs.** Documentation claims **only what has been proven**. Any path not yet live-validated says so explicitly and links its tracking issue.
- **Consumer-first.** The consumer path (generic x86, signed tarball, no Vergil) is the spine; Vergil is a thin aside. Architectural framing: strip Vergil → a standalone `mqlab` orchestrator + support tree.
- **Framing.** **Three HA mechanisms** (RDQM replicated-storage, Pacemaker/SAN shared-storage, Native HA log-replicated) across **four arms** (Native HA on both Ubuntu and RHEL); each a **3+3** HA/DR stack. Use this phrasing consistently everywhere.
- **MQ version pin.** Guides are pinned to **IBM MQ 9.4**.
- **Validation gate (docs).** The site must build strict with no broken links or orphan pages: `vrg-container-docs build --strict`. Repo validation: `vrg-container-run -- vrg-validate`. No remaining `mq-cluster-tooling` references anywhere.
- **Commits & git.** Use `vrg-commit` (conventional commits) and `vrg-git`/`vrg-gh` wrappers only. Work each task on its own `feature/<issue>-<slug>` branch in a worktree; the human runs `vrg-submit-pr`.
- **Stage the docs before building.** `vrg-container-run -- vrg-docs-stage --docs-dir docs/site/docs` (once per session or after `CHANGELOG.md`/`releases/` change) before a strict build.

## File structure (site, target state)

```
docs/site/mkdocs.yml                         # nav restructured to the journey-first spine
docs/site/docs/index.md                      # Home — consumer-first pitch
docs/site/docs/methodology.md                # NEW — parasite-lab case study (gated on 4a)
docs/site/docs/getting-started.md            # consumer path + Vergil aside (rebased on #894)
docs/site/docs/architecture/index.md         # four arms, current
docs/site/docs/architecture/diagrams/*.html  # existing diagrams, audited
docs/site/docs/operate/index.md              # NEW — Operate & Observe (failover/DR/e2e/Watcher)
docs/site/docs/guides/*.md                   # currency audit (no rewrite)
docs/site/docs/reference/index.md            # NEW — topology/endpoints/CLI verbs/gotchas
docs/site/docs/develop.md                    # NEW — thin Vergil dev workflow
docs/site/docs/design-and-specs.md           # stale mq-cluster-tooling links fixed
README.md                                     # consumer-first rewrite
.github/workflows/release.yml                 # hardened until it produces a real artifact
.gitattributes                                # export-ignore curation verified
docs/reports/<date>-getting-started-from-scratch.md   # NEW — validation report
```

---

## Task 1: Site IA + scaffolding

**Repo:** `mq-resiliency-lab-for-linux` · **Kind:** task (docs) · **Deps:** none (start early)

**Files:**
- Modify: `docs/site/mkdocs.yml` (the `nav:` block)
- Create: `docs/site/docs/methodology.md`, `docs/site/docs/operate/index.md`, `docs/site/docs/reference/index.md`, `docs/site/docs/develop.md` (as **stubs** with a one-line intro + a tracking note pointing at their owning task)
- Modify: `docs/site/docs/design-and-specs.md` (fix stale links)
- Grep target: every file under `docs/site/docs/` for `mq-cluster-tooling` and for HA-framing wording

**What to do:** Establish the journey-first nav and the section skeleton so downstream tasks fill focused pages, and clear the two mechanical drifts (stale repo name, HA framing).

- [ ] **Step 1 — Restructure the nav.** Edit `mkdocs.yml` `nav:` to: `Home` / `Methodology` / `Getting Started` / `Architecture` / `Operate & Observe` (→ `operate/index.md`) / `Guides` / `Reference` (→ `reference/index.md`) / `Develop the lab` (→ `develop.md`) / `Design & Specs`. Keep `Releases` where it is.
- [ ] **Step 2 — Create section stubs.** Each new page gets a title, one-sentence purpose, and an admonition: `!!! note "In progress" This section is authored in <epic #161 / task #NN>.` so the strict build has no orphan/broken nav entries.
- [ ] **Step 3 — Fix stale links.** In `design-and-specs.md`, replace every `mq-cluster-tooling` URL with the correct `mq-resiliency-lab-for-linux` path. Decide the link style once (open question: relative in-repo vs corrected GitHub URL) and apply uniformly. Grep the whole tree to confirm zero `mq-cluster-tooling` occurrences remain.
- [ ] **Step 4 — Settle the HA framing.** Grep for "three HA", "four arms", "3 HA", etc.; normalize every occurrence to the Global-Constraints phrasing (three mechanisms / four arms / 3+3).
- [ ] **Step 5 — Settle the stack-name drift.** Choose the canonical Ubuntu stack token (`pcmk-ubuntu` vs `nativeha-ubuntu`) and the naming pattern; reconcile `README.md` (`distributed-pcmk-ubuntu`) and `getting-started.md` to match. If the final choice depends on the validation run (Task 10), leave a single `<!-- canonical stack: TBD by #NN -->` marker in exactly one place and note it — do not scatter the ambiguity.
- [ ] **Step 6 — Strict build.** `vrg-container-run -- vrg-docs-stage --docs-dir docs/site/docs` then `vrg-container-docs build --strict`. Expected: PASS, no broken links, no orphan pages.
- [ ] **Step 7 — Commit.** `vrg-commit --type docs --scope site --message "restructure nav to journey-first spine + fix stale links (#NN)"`

**Acceptance:** Strict build green; nav matches the spine; zero `mq-cluster-tooling` references; HA framing consistent; stack-name drift resolved (or reduced to one marked TBD).

---

## Task 2: Home + Getting Started (consumer-first)

**Repo:** `mq-resiliency-lab-for-linux` · **Kind:** task (docs) · **Deps:** Task 1 (nav + stubs). Corrected later by Task 10.

**Files:**
- Modify: `docs/site/docs/index.md` (Home)
- Modify: `docs/site/docs/getting-started.md` — **rebase on the #894 (`240882c`) rewrite already on develop**; do not revert its `mqlab bootstrap <stack>` content.

**Interfaces (content contract):** Getting Started must present, in order: prerequisites (x86 host, `uv`, Python 3.12, libvirt/qemu/vagrant, gpg) → **download the signed tarball** → **verify** (fingerprint + out-of-band key + `sha256sum -c`) → **unpack** → `./scripts/setup` (`uv sync`) → `mqlab doctor` → `mqlab bootstrap <stack>` → send a first message → point at Operate & Observe. Vergil appears once, as a collapsed aside.

- [ ] **Step 1 — Home.** Rewrite `index.md` to the consumer pitch: what the lab is, who it's for, the 30-second value, and the architectural truth (strip Vergil → standalone orchestrator). Link Methodology, Getting Started, Architecture. Remove any Vergil-first framing.
- [ ] **Step 2 — Getting Started prereqs + consumer preamble.** Add the prerequisites list and the download→verify→unpack preamble **above** the existing bring-up content (reuse the README's verify recipe; keep the fingerprint authoritative). Every command shown must be one a consumer actually runs from the extracted tarball.
- [ ] **Step 3 — Demote Vergil.** Replace the `vrg-vm create … --identity vergil-user` opening with a single `??? note "Developing with Vergil? (that's basically just the author)"` collapsed aside that points to `develop.md`.
- [ ] **Step 4 — Honesty markers.** Until Task 10 validates the path, add a one-line admonition linking the validation issue: "The consumer path is documented; end-to-end validation on a clean x86 host is tracked in #NN." (Task 10 removes it.)
- [ ] **Step 5 — Strict build + commit.** Strict build green; `vrg-commit --type docs --scope getting-started --message "consumer-first Home + Getting Started (#NN)"`.

**Acceptance:** Home reads consumer-first with no Vergil-first framing; Getting Started carries the full download→verify→unpack→bootstrap path with Vergil as a single aside; honesty marker present pending Task 10; strict build green.

---

## Task 3: Architecture + new Operate & Observe

**Repo:** `mq-resiliency-lab-for-linux` · **Kind:** task (docs) · **Deps:** Task 1

**Files:**
- Modify: `docs/site/docs/architecture/index.md`; audit `docs/site/docs/architecture/diagrams/*.html`
- Author: `docs/site/docs/operate/index.md` (replaces the Task 1 stub)

- [ ] **Step 1 — Architecture currency pass.** Verify the four arms (RDQM, Pacemaker/SAN, Native-HA-Ubuntu, Native-HA-RHEL) are each described as-built and match `lab/topology.yaml`; confirm the diagrams (`01`–`06`) still reflect reality (the pcmk-rhel arm was removed in #138 — ensure no stale references). Fix the arch-conditional TCG framing if any stale "TCG" labels remain.
- [ ] **Step 2 — Author Operate & Observe.** New page covering: driving failover drills; DR cutover/failback (`mqlab dr cutover|failback`); the end-to-end request/reply app + `svc-sim`; the Watcher (Prometheus/Grafana + MQ events → Alloy → Loki), including `mqlab obs` verbs and the `lab-watcher` board. Keep it operational (how to *run and watch*), linking Architecture for the *why*.
- [ ] **Step 3 — Strict build + commit.** Strict build green; `vrg-commit --type docs --scope operate --message "architecture currency + Operate & Observe (#NN)"`.

**Acceptance:** Architecture matches as-built (four arms, no removed-arm references); Operate & Observe covers failover/DR/e2e/Watcher with runnable verbs; strict build green.

---

## Task 4a: Methodology brainstorm (prerequisite)

**Repo:** `mq-resiliency-lab-for-linux` · **Kind:** task (design) · **Deps:** none · **Blocks:** Task 4b

**Files:**
- Create: `docs/specs/<date>-parasite-lab-methodology-design.md` (the brainstorm's converged output)

- [ ] **Step 1 — Run the dedicated brainstorm.** A `superpowers:brainstorming` session on the methodology, built on the corrected seed (mastering technology by standing an entire bespoke mini-enterprise up from the ground up, virtually — network + core services from code/distributions; a career-long, rare practice), NOT the "avoid the client environment" premise. Excavate: why / how / purpose / where-it's-going, and the anonymized generic-gateway first instance.
- [ ] **Step 2 — Capture the converged design** to `docs/specs/<date>-parasite-lab-methodology-design.md` (structure + argued content the site page will render). Anonymized throughout.
- [ ] **Step 3 — Commit.** `vrg-commit --type docs --scope methodology --message "parasite-lab methodology design from dedicated brainstorm (#NN)"`.

**Acceptance:** A committed methodology design doc reflecting the corrected thesis, ready to render as the site page. **No site prose is written before this lands.**

---

## Task 4b: Methodology / parasite-lab site page

**Repo:** `mq-resiliency-lab-for-linux` · **Kind:** task (docs) · **Deps:** Task 4a (Blocked-by), Task 1

**Files:**
- Author: `docs/site/docs/methodology.md` (replaces the Task 1 stub)

- [ ] **Step 1 — Render the case study** from the 4a design into the five-part page (thesis / why / how / first-instance = MQ via the generic trading back-office gateway / purpose + where-it's-going). Lab-scoped but authored with portability in mind.
- [ ] **Step 2 — Strict build + commit.** Strict build green; `vrg-commit --type docs --scope methodology --message "parasite-lab methodology case study page (#NN)"`.

**Acceptance:** Methodology page published, matching the 4a design, fully anonymized; strict build green.

---

## Task 5: README rewrite (consumer-first)

**Repo:** `mq-resiliency-lab-for-linux` · **Kind:** task (docs) · **Deps:** Task 2 (align copy with Getting Started)

**Files:**
- Modify: `README.md`

- [ ] **Step 1 — Restructure** to: What this is → **Getting Started (consumer)** → verify → run → a short **Developing with Vergil** aside → Development pointer. Lead with the consumer; the Vergil section is a few lines that say "this is the author's personal tooling; talk to me if you want it."
- [ ] **Step 2 — Reconcile** the stack token with Task 1's decision (`distributed-pcmk-ubuntu` → canonical) and the verify recipe with Getting Started (single source of the fingerprint).
- [ ] **Step 3 — Validate + commit.** `vrg-container-run -- vrg-validate`; `vrg-commit --type docs --scope readme --message "consumer-first README (#NN)"`.

**Acceptance:** README leads with the consumer path; Vergil is a short aside; stack token + verify recipe consistent with Getting Started; validation green.

---

## Task 6: Guides + Reference currency audit

**Repo:** `mq-resiliency-lab-for-linux` · **Kind:** task (docs) · **Deps:** Task 1

**Files:**
- Audit: `docs/site/docs/guides/*.md`
- Author: `docs/site/docs/reference/index.md` (replaces the Task 1 stub)

- [ ] **Step 1 — Guides currency audit (verify, don't rewrite).** For each of the five guides, confirm it matches the as-built lab and the MQ 9.4 pin; fix only drift. Record the audit outcome per guide in the PR description.
- [ ] **Step 2 — Author Reference.** Assemble the site Reference section: topology (from `lab/topology.yaml`), REST endpoints (`mqlab rest render`), the `mqlab` CLI verb map (net/vm/qm/obs/dr/bootstrap), and a curated pointer to the in-repo `docs/reference/` gotchas. Link, don't duplicate, the deep in-repo references.
- [ ] **Step 3 — Strict build + commit.** Strict build green; `vrg-commit --type docs --scope reference --message "guides currency audit + Reference section (#NN)"`.

**Acceptance:** Guides confirmed current (drift fixed, audit recorded); Reference section surfaces topology/endpoints/verbs/gotchas; strict build green.

---

## Task 7: Make the signed tarball actually work

**Repo:** `mq-resiliency-lab-for-linux` · **Kind:** task (tooling) · **Deps:** none (start early) · **Gates:** Tasks 9, 10

**Files:**
- Modify: `.github/workflows/release.yml`, `.gitattributes` (as needed)
- Reference: `docs/specs/2026-06-20-signed-tarball-release-design.md`, `docs/development/release-runbook.md`

**What to do:** First end-to-end exercise of the mostly-built pipeline. Expect breakage; iterate with fix PRs. **Do not burn `v1.0.0`** — iterate with a pre-release tag or `workflow_dispatch`.

- [ ] **Step 1 — Dry-run the archive locally.** `git archive --format=tar HEAD | tar -tf - | sort > /tmp/tree.txt` and confirm the curated tree: **includes** `src/mqlab/`, `ansible/`, `lab/`, `manifests/`, `docs/`, `scripts/`, `pyproject.toml`, `uv.lock`, `README.md`, `VERSION`, `RELEASE-KEY.asc`; **excludes** `.github/`, `.claude/`, `.vergil/`, `.worktrees/`, `.superpowers/`, `vergil.toml`, `tests/`, `.gitattributes`. Fix `.gitattributes` if anything is misfiled.
- [ ] **Step 2 — Add a safe iteration trigger.** Ensure `release.yml` can be exercised without a real `vX.Y.Z` tag (add `workflow_dispatch` and/or accept `v*-rc*` pre-release tags), so debugging never consumes the real coordinate. Confirm the version guard still ties tag ⇄ `pyproject.toml` ⇄ `VERSION`.
- [ ] **Step 3 — Exercise the pipeline (human triggers).** Human pushes a pre-release tag or dispatches via `! …`; agent reads the run logs and fixes: archive → version-guard → checksums → GPG sign (`RELEASE_GPG_PRIVATE_KEY` secret) → publish. Iterate until a pre-release Release appears with `<tarball>`, `SHA256SUMS`, `SHA256SUMS.asc`.
- [ ] **Step 4 — Verify the artifact out-of-band.** Download the pre-release tarball; `gpg --verify SHA256SUMS.asc SHA256SUMS`; `sha256sum -c SHA256SUMS`; extract and confirm the tree matches Step 1. 
- [ ] **Step 5 — Commit the fixes.** `vrg-commit --type ci --scope release --message "make the signed-tarball release pipeline produce a verifiable artifact (#NN)"`. (The real `v1.0.0` cut is Task 9, human-gated.)

**Acceptance:** A pre-release GitHub Release exists with a GPG-verifiable, checksum-matching tarball whose extracted tree is exactly the curated product tree. Workflow can be re-run without burning the real version. All fixes merged.

---

## Task 8: Provision the clean x86 host (setup)

**Repo:** `mq-resiliency-lab-for-linux` · **Kind:** setup (human-operated) · **Deps:** none · **Gates:** Task 10

> May collapse into Task 10's human-attested precondition rather than a standalone issue — decide at filing time. If standalone:

- [ ] **Step 1 — Allocate** a Vergil-free, nested-virt-capable Google Cloud x86 instance (sized per README: ~12 vCPU / 64 GiB, nested virtualization enabled).
- [ ] **Step 2 — Attest readiness.** Record host specs + that it has never had Vergil installed, as a comment on the task/precondition.

**Acceptance:** A clean x86 host exists and its Vergil-free provenance is attested.

---

## Task 9: Deployment — cut the first real release (operational, human-gated)

**Repo:** `mq-resiliency-lab-for-linux` · **Kind:** `deployment` · **Blocked-by:** Task 7

**Precondition (human-attested):** Task 7's pipeline produces a verifiable artifact; `pyproject.toml`/`VERSION`/tag agree on the release version.

- [ ] **Step 1 — Human cuts the release.** Human bumps version if needed, pushes the real `vX.Y.Z` tag via `! git tag … && git push origin …` (agent never cuts a release). The workflow publishes the signed Release.
- [ ] **Step 2 — Confirm downloadable + verifiable.** From a plain HTTPS client: fetch the tarball + `SHA256SUMS.asc`; `gpg --recv-keys <fingerprint>`; `gpg --verify`; `sha256sum -c`. Record `Outcome: SUCCESS` (or FAILURE + logs) as a comment.

**Acceptance (`Outcome: SUCCESS`):** A real signed release is published and verifies out-of-band over HTTPS.

---

## Task 10: From-scratch consumer validation (operational)

**Repo:** `mq-resiliency-lab-for-linux` · **Kind:** `validation` · **Blocked-by:** Task 9, Task 8, and enough of Task 2

**Precondition (human-attested):** clean x86 host ready (Task 8); real release published (Task 9).

**Files:**
- Create: `docs/reports/<date>-getting-started-from-scratch.md`

- [ ] **Step 1 — Follow Getting Started verbatim** on the clean host, agent asserting each step: install only the stated prereqs → fetch tarball over HTTPS → verify (fingerprint + key + `sha256sum -c`) → **unpack and run from the extracted tarball** → `./scripts/setup` → `mqlab doctor` → `mqlab bootstrap <ubuntu-stack>` → send a message via the e2e app → observe it in the Watcher.
- [ ] **Step 2 — Assert the export-ignore boundary.** Confirm `mqlab` ran with **no** dependency on `vergil.toml`, `.vergil/`, `tests/`, `.github/` (they are absent from the tarball). Any such dependency is a bug → fix in the owning task (release-blocker).
- [ ] **Step 3 — Triage findings.** Release/tarball-path failures → fix in-epic (loop back to Task 7/2). Other tooling failures → **file issues** (flag release-blockers); reflect honestly in the docs.
- [ ] **Step 4 — Write the report** to `docs/reports/<date>-getting-started-from-scratch.md` (the actual run, warts and all) and remove the Task 2 honesty marker if the path is green. Record `Outcome: SUCCESS/FAILURE` as a comment.

**Acceptance (`Outcome: SUCCESS`):** The canonical Ubuntu stack was stood up from the published tarball on a Vergil-free x86 host, message path proven, Watcher shows it; report committed; docs' "documented" markers flipped to "validated"; every gap fixed or filed.

---

## Closing bookends (already seeded — not filed from this plan)

- **Documentation review** — `mq-resiliency-lab-for-linux#896`. A sweep verifying the shipped changes are reflected in `docs/site/…`; spawns per-repo doc tasks if needed. Runs before the retrospective.
- **Retrospective** — `.github#163`, authored with `epic-retrospective`; its merge closes the epic.

## Self-review notes (spec coverage)

- Spec §3 IA → Tasks 1–6. §3.1 framing → Task 1. §4 methodology (deferred) → Tasks 4a/4b. §5 validation chain → Tasks 7/8/9/10. §2.4 non-goals (file, don't fix) → Task 10 Step 3. §7 placement/anonymization/honesty → Global Constraints + per-task markers. §8 DoD → per-task Acceptance + the two bookends. §10 open questions (stack name, link style) → Task 1 Steps 3 & 5.
- Dependency graph: 1,7,8 start early; 2←1; 3←1; 4b←4a; 5←2; 6←1; 9←7 (human-gated); 10←9,8,2. Docs tasks (1–6) never gated on the validation chain.
