# Ground-up site-documentation revamp for public consumer release — Design

**Epic:** logical-minds-foundry/.github#161
**Documentation task:** logical-minds-foundry/.github#162
**Status:** Design (brainstormed; pending pushback + plan)
**Slug:** `site-docs-revamp`

## 1. Why this exists

The lab's site documentation was written in the earliest days of the project and
has drifted **months** behind the as-built lab. Since then the lab has grown into
a complete HA/DR stack spanning **three HA mechanisms across four arms**, an
end-to-end request/reply application, a full observability plane ("the Watcher"),
DR cutover/failback, and a signed-tarball release mechanism — almost none of
which the top-level narrative conveys.

We are targeting a **public release this fall** (~1–2 months out): a downloadable,
signed artifact a stranger can stand up **without any of the author's personal
dev tooling (Vergil)**. The current site actively works against that:

- **Getting Started is Vergil-only.** It opens with `vrg-vm create …`; there is
  no download-and-build-without-Vergil walkthrough anywhere on the site, even
  though the `README` and the signed-tarball design (#299) describe exactly that
  consumer path.
- **Stale artifacts.** `docs/site/docs/design-and-specs.md` still links every
  reference at the **old `mq-cluster-tooling` repo name**. Framing wobbles between
  "three HA mechanisms" and "four arms."
- **The methodology is undocumented.** The single most transferable part of this
  work — the *approach* that produced the lab — appears nowhere.

This epic is a **ground-up rethink** of the site docs to match reality and lead
with the consumer, plus the extraction of a lab-scoped **methodology case study**.

## 2. Charter, scope & non-goals

### 2.1 Charter — docs-shaped, consumer-first, honestly validated

Revamp the published site so that (a) a stranger can understand what the lab is,
what it is for, and where it is going; (b) that stranger can actually stand it up
from a signed tarball on a generic host; and (c) the *method* behind the lab is
captured as a reusable case study. The docs may only **claim what the from-scratch
walkthrough actually proves** — no documenting a known-broken path, honoring the
"no silent failures" rule at the documentation layer.

### 2.2 The consumer-first reframe (audience model)

There are two audiences, and their needs diverge **only** at the
get-the-environment-up layer:

- **The consumer (the product).** A generic **x86 VM with zero Vergil** →
  download the **published signed tarball** → verify → unpack → `uv sync` →
  `mqlab doctor` → `mqlab bootstrap <setup>`. This path must be **bulletproof**.
- **The Vergil developer (the author).** Vergil is the author's personal
  solo-practitioner dev tooling; realistically no one else adopts it. It is
  demoted to a thin *"if you're a Vergil user (i.e. you're me)"* aside. Anyone who
  wants to hack on the lab with Vergil talks to the author.

**Architectural truth to state plainly in the docs:** strip the Vergil layer and
what remains is a standalone Python orchestrator (`mqlab`) plus its support tree
(Ansible roles, lab definitions, manifests). Vergil only ever provides the *dev*
VM; it is a thin layer, not part of the deliverable.

### 2.3 In scope

- A ground-up site information architecture (§3), consumer-first.
- Home + Getting Started rewritten around the consumer path; Vergil as an aside.
- Architecture brought current (four arms, diagrams) + a **new Operate & Observe**
  section surfacing failover, DR cutover/failback, the e2e app, and the Watcher.
- Guides + Reference **currency audit** (audit, not rewrite — the guides are
  strong; verify they match the as-built lab).
- A README rewrite that leads with the consumer path.
- Fixing stale links (the `mq-cluster-tooling` references) and settling the
  "3 mechanisms / 4 arms" framing consistently everywhere.
- A lab-scoped **methodology ("parasite lab") doc** — see §4.
- **The one deliberate tooling exception:** make the signed-tarball release
  pipeline (`.github/workflows/release.yml`) actually produce a real, verifiable
  artifact — see §5.
- A **from-scratch consumer validation** (§5) producing a validation report.

### 2.4 Non-goals (filed, not fixed here)

- Any tooling gap the walkthrough surfaces **other than** the release-pipeline
  hardening. These are **filed as issues** (release-blockers flagged); the docs
  reflect today's honestly-supported state until they close.
- The **full standalone career case study** of the methodology. This epic ships a
  lab-scoped version authored *with portability in mind*; the portable career
  piece is a seeded follow-on (§9).
- Reworking the guides' content beyond a currency audit.

## 3. Architecture — site information architecture

A **journey-first spine**, consumer-first, with the Vergil-vs-consumer fork
isolated to the one layer where it bites (Getting Started). Proposed nav:

| Section | Purpose |
|---|---|
| **Home** | What this is, who it's for, the 30-second pitch. |
| **Methodology** | The parasite-lab thesis + anonymized gateway case (why/how/purpose). Content deferred to §4. |
| **Getting Started** | THE consumer path, bulletproof: x86 VM prereqs → download+verify tarball → `uv sync` → `mqlab doctor` → `mqlab bootstrap <setup>` → first message. A short *"Developing with Vergil?"* aside box. |
| **Architecture** | Outside-in, as-built walkthrough; four arms + diagrams. |
| **Operate & Observe** *(new)* | Run it: failover / DR cutover-failback drills, the end-to-end app, the Watcher. |
| **Guides** | Product-level MQ how-tos (currency audit). |
| **Reference** | Topology, endpoints, CLI verbs, gotchas. |
| **Develop the lab** | Thin: the Vergil dev workflow, contributing, releasing. |
| **Design & Specs** | The engineering record; fix the stale `mq-cluster-tooling` links. |

Key moves vs. today: **Methodology** promoted near the top; **Getting Started**
becomes consumer-first (Vergil shrinks to an aside); a **new Operate & Observe**
section surfaces the failover/DR/e2e/Watcher work currently buried or missing;
**Develop the lab** absorbs the Vergil material as a thin section. The mkdocs nav
in `docs/site/mkdocs.yml` is restructured to match.

### 3.1 The "three mechanisms / four arms" framing

Settle it consistently: **three HA mechanisms** — RDQM (replicated-storage),
Pacemaker/SAN (shared-storage), Native HA (log-replicated) — realized as **four
arms** because Native HA runs on both an Ubuntu arm and a RHEL arm. Each arm is a
**3+3** HA/DR stack (a 3-node HA group in Site A with an asynchronous DR/CRR
relationship to a matching 3-node group in Site B).

## 4. The methodology / "parasite lab" doc — DEFERRED sub-brainstorm

This is the epic's **most important and most transferable** deliverable, and it
is **explicitly deferred**: its content must be produced by its **own dedicated
brainstorm** before any prose is drafted, so it is not written on the wrong
premise.

**Corrected concept seed (for the dedicated brainstorm — NOT to be written yet):**
the practice is fundamentally about **mastering technology by standing an entire
bespoke mini-enterprise up from the ground up, virtually** — building the network
fabric and all the core enterprise services from code and distributions
(historically Kerberos, AFS, NFS, DNS, DCE) to simulate the *bootstrapping* of an
enterprise technology combination and truly learn how a new technology works.
Staying out of a client's production environment is a *side benefit*, not the
point. It is a **career-long practice** (since the author's first hands on
VirtualBox + Vagrant), now core to how the author works, and rare (only two or
three known practitioners).

**Approved five-part skeleton** (structure only; content pending the brainstorm):
1. The thesis — what the practice is.
2. Why — why build the whole thing virtually to learn it.
3. How — the repeatable method (topology-as-code, glass-box orchestration,
   spike-first checkpointed plans, fail-loud instrumentation, cold-rebuild
   acceptance gate, disciplined anonymization, human-operates-the-lab).
4. The first instance — MQ resiliency, seeded by an anonymized **generic trading
   back-office gateway** problem.
5. Purpose & where it's going — the method as a template the next labs clone
   (the `…-for-<platform>` family; the component-extraction roadmap).

The doc is **lab-scoped** now but authored with **portability in mind**; the full
standalone career case study is a seeded follow-on (§9).

**Scheduling.** The dedicated brainstorm is an **explicit prerequisite task**, not
a floating dependency: the methodology-doc task is `Blocked-by` it, and no prose
is drafted until the brainstorm converges. It runs as its own
`superpowers:brainstorming` session and may emit its own short sub-spec.

## 5. Validation — the from-scratch consumer path (the crux)

Human-operated, agent-scripted-and-asserted. A dependency chain:

1. **Make the tarball real (the in-scope tooling exception).** Exercise
   `.github/workflows/release.yml`. The pipeline is **mostly built and validated
   in pieces**; this is simply its **first end-to-end run**, so expect a few
   fix PRs — a heads-up, not a risk. Iterate until it publishes a GitHub Release
   with a **signed tarball + `SHA256SUMS.asc`**. Critically, verify the
   `.gitattributes export-ignore` curation boundary yields the **right published
   tree** (orchestrator + Ansible + lab defs + manifests + docs + `scripts/`,
   dev-only paths excluded). **Iterate with pre-release / `workflow_dispatch`
   triggers, not by burning the real `vX.Y.Z` coordinate** on a broken run
   (`VERSION` is already `1.0.0`); the real tag is cut once, for the validated
   release. The human drives the raw `git tag`/dispatch via `! …`; the agent
   drives the debugging. Cutting the real release is human-gated.
2. **Clean generic x86 host, zero Vergil.** A trivial provisioning task — allocate
   a Google Cloud x86 instance (as with the existing cloud VM). Install **only**
   the stated prerequisites (`uv`, Python 3.12, libvirt/qemu/vagrant, gpg). This
   **validates the prerequisites list by the act** of following it.
3. **The consumer happy path, every step an assertion:** fetch tarball over HTTPS
   → verify signature out-of-band (fingerprint + keyserver) → `sha256sum -c` →
   **unpack, and run everything from the extracted tarball, not a git checkout**
   → `./scripts/setup` (`uv sync`) → `mqlab doctor` → `mqlab bootstrap <stack>` →
   send a message through the e2e app → observe it in the Watcher. Running from
   the extracted tarball is the load-bearing assertion: it proves `mqlab` has
   **no runtime dependency on export-ignored paths** (`vergil.toml`, `.vergil/`,
   `tests/`, `.github/`) — the classic way a curated tarball breaks.
4. **Canonical getting-started stack = a subscription-free, x86-native Ubuntu
   arm.** On x86 the RHEL arms are native KVM but need a **subscription**; the
   Ubuntu arms need nothing extra. Getting Started leads with the Ubuntu stack
   (`pcmk-ubuntu` or `nativeha-ubuntu` — resolved at validation by which stands up
   cleanest) as the bulletproof happy path; RHEL arms are documented but flagged
   "needs a subscription." We fully live-validate that one stack; others are
   documented, and any not live-run this epic say so honestly.
5. **Output:** a from-scratch validation report under `docs/reports/` — the actual
   run, warts and all. Failures in the tarball/release path → fixed in-epic;
   other tooling failures → filed (release-blockers flagged) and reflected
   honestly in the docs.

This is realized as the **operational tasks**: a small **VM-provisioning** setup,
a human-gated **deployment** task (cut the first real release; produces the
published, downloadable tarball), and a **validation** task (the from-scratch
run), with `impl → deploy → validate` dependency ordering.

## 6. Build order (task breakdown)

Implementation tasks are filed from the plan (epic-create step 9). Anticipated
shape, each ≈ one PR unless noted:

1. **Site IA + scaffolding** — restructure `mkdocs.yml` nav to the journey-first
   spine; create section landing pages; fix stale `mq-cluster-tooling` links;
   settle the 3-mechanism/4-arm framing; **settle the stack-name drift** (README
   `distributed-pcmk-ubuntu` vs getting-started `pcmk-ubuntu`/`nativeha-ubuntu` —
   pick one and reconcile everywhere).
2. **Home + Getting Started (consumer-first)** — the bulletproof path; Vergil
   aside. **Rebase on the freshly-rewritten getting-started (#894 `240882c`)**,
   which already documents `mqlab bootstrap <stack>` — the task adds the consumer
   preamble (download/verify/`uv sync`) and demotes the Vergil open. (Corrected by
   the validation task.)
3. **Architecture + new Operate & Observe** — four arms current, diagrams;
   failover/DR/e2e/Watcher.
4a. **Methodology brainstorm (prerequisite)** — the dedicated sub-brainstorm (§4);
   blocks 4b.
4b. **Methodology / parasite-lab doc** — `Blocked-by` 4a; doc authored after.
5. **README rewrite** — consumer-first, Vergil aside.
6. **Guides + Reference currency audit.**
7. **Make the signed tarball actually work** — the in-scope tooling exception;
   gates the deployment/validation.
8. **Provision the clean x86 host (setup)** — allocate the Vergil-free Google
   Cloud x86 instance; gates the validation run. May collapse into the
   validation task's **human-attested precondition** rather than a separate issue
   — the plan decides granularity.
9. **Deployment (operational, human-gated)** — cut the first real signed release;
   confirm the tarball is downloadable + verifiable.
10. **From-scratch consumer validation (operational)** — the clean-x86 run; emits
   the report; files gaps.

Ordering: (1), (7), and (8) can start early and run in parallel; (9) is
human-gated and blocked-by (7); (10) is blocked-by (9) + (8) + enough of (2).
(4b) is blocked-by (4a) and otherwise parallel. The docs tasks (1–6) are not
gated on the validation chain — they ship honestly and are flipped from
"documented" to "validated" when (10) lands.

## 7. Cross-cutting concerns

- **Anonymization.** All public artifacts (this epic, spec/plan, the methodology
  doc) stay generic — the gateway example is a generic "trading back-office
  gateway," per the standing rule.
- **Honest docs.** The docs claim only what the walkthrough proves; unvalidated
  paths say so and link the tracking issue.
- **Placement law.** The documentation and retrospective tasks land their PRs in
  `.github`; the documentation-review sweep and the per-arm doc work land in the
  member repo (`mq-resiliency-lab-for-linux`), where the site docs live.

## 8. Testing & definition of done

- The site builds strict (`vrg-container-docs build --strict`) with no broken
  links or orphan pages; no remaining `mq-cluster-tooling` references.
- The consumer Getting Started path is **live-validated end-to-end** on a clean
  x86 VM for the canonical Ubuntu setup, with a committed validation report.
- The signed-tarball release pipeline has produced a **real, verifiable** Release
  artifact at least once.
- Every gap the walkthrough surfaced is either fixed (release path) or filed
  (everything else), with release-blockers flagged.
- The methodology doc is published (after its dedicated brainstorm), lab-scoped
  and portability-minded.
- The closing bookends are satisfied: the documentation-review sweep confirms the
  site docs reflect the shipped state; the retrospective lands.

## 9. Follow-on (anticipated)

- **The standalone methodology case study** — the portable career piece the
  lab-scoped doc is authored to feed (potential home: the author's
  infrastructure-mindset writing pipeline).
- **Release-pipeline hardening beyond first-run** — anything the deployment task
  surfaces that is out of scope for "make it work once."
- **Any release-blocking tooling gap** filed by the validation run.

## 10. Open questions

- Which canonical **Ubuntu stack** Getting Started leads with — `pcmk-ubuntu` vs.
  `nativeha-ubuntu` (getting-started already uses both names post-#894; the README
  still says `distributed-pcmk-ubuntu`) — resolved during validation by which
  stands up cleanest, and reconciled everywhere by task 1.
- Whether **Design & Specs** links should become in-repo relative links or
  corrected GitHub URLs (both fix the staleness; pick one for consistency).

## 11. Cross-epic links

- **#299 signed-tarball release** (`docs/specs/2026-06-20-signed-tarball-release-design.md`)
  — the consumer-path and release-artifact design this epic finally exercises.
- **Docs site + architecture** (`docs/specs/2026-06-08-docs-site-and-architecture-design.md`)
  — the original site design this revamp supersedes at the narrative layer.
- **#38 live-lab-validation-framework** — the validation discipline the
  from-scratch run draws on.
- **#79 observability-extraction** / component-extraction roadmap — the
  "where it's going" the methodology doc points at.
