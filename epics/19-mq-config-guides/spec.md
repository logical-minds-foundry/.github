# MQ Configuration Guides — Epic Design

- **Date:** 2026-07-02
- **Status:** Active — approved 2026-07-02 (epic logical-minds-foundry/.github#19)
- **Type:** Epic (finite; accretes guide tasks over time, closes when the guide set is complete). Deliberately **not** a *standing* epic — see §1.
- **Related:** `2026-06-27-component-extraction-roadmap-design.md` (#368),
  `docs/reference/mq-json-logging-config.md` (prototype guide)

---

## 1. Purpose & scope

Extract **generic, product-level IBM MQ how-to guides** from the working lab and
publish them into the DocsTree. Each guide captures one coherent area of MQ
configuration — the collectors' required MQ settings, TCP tuning, security, JSON
logging, and more — written as **prose backed by declarative configuration
examples**, so the knowledge can be carried into an environment that will not
accept the lab's code.

The lab is the reference prototype: every guide describes a configuration we run
and can validate, generalized to the MQ product. This is a **normal, finite
epic** — it opens with a defined first cut and accretes new guide tasks as topics
surface, closing once the guide set is complete.

> **Deliberately not a *standing* epic.** In the vergil framework, "standing" is
> reserved for the single per-repo ad-hoc catch-all epic. There is intended to be
> exactly one standing epic per repository; designating a second, topical epic as
> standing would collide with that assumption (and with any framework code that
> relies on it). This epic is therefore ordinary and finite — it simply stays
> open while there are still guides worth writing.

### Distinct from the component-extraction roadmap (#368)

This epic is a **separate track** from the OSS component-extraction roadmap.

| | Component extraction (#368) | This epic |
|---|---|---|
| Deliverable | **Code** — standalone OSS packages | **Prose + config** — how-to documents |
| Consumer | Anyone who can install software | An environment that will **not** accept code |
| Form | Repos, MIT-licensed, versioned | Markdown guides in the DocsTree |
| Shared premise | Both use the lab as the reference prototype | |

The two are complementary. Where they touch the same subject (e.g. the
observability collectors), the roadmap harvests the *collector code*; this epic
documents the *MQ configuration the collectors require* — which is exactly the
part a colleague needs to replicate the setup without the code.

## 2. The deliverable: what a guide is

A guide is a **generic, self-contained, scan-safe MQ how-to document**:

- **Generic to the product.** No lab hostnames, no site-specific values, no
  code. If a value must appear, it is a placeholder or an illustrative default,
  never a lab secret.
- **Recommendation-bearing.** Where more than one approach exists, the guide
  ranks them **strongest → weakest recommendation** and states each option's
  compromise plainly. It recommends the best technical choice and supplies the
  evidence; the reader owns the tradeoff decision. (Archetypes: RHEL vs Ubuntu,
  Native HA vs RDQM — we make the recommendation and show the data, we do not
  dictate the choice.)
- **Two-layer.** The essential how-to is the front; supporting depth is
  appendices (see §4).
- **Living.** Each guide records the date it was last validated in the lab and
  is updated as we learn more.

## 3. Content boundary (scan-safe default)

To stay shareable into a scanned corporate environment, guides use **declarative
configuration only**:

**Allowed:**
- `qm.ini` / `mqs.ini` stanza blocks
- MQSC `DEFINE` / `ALTER` statements
- `setmqaut` authority statements
- Parameter = value tables (setting, value, default, rationale)

**Not allowed:**
- Shell, Python, Ansible, or any imperative script
- `runmqsc` pipelines or other command-orchestration snippets

Procedures are described in **prose** ("define a receiver channel with `SSLCIPH`
set to …"); the configuration artifacts themselves are shown verbatim because a
config example is far less likely to trip a scan than code. This is the exact
line the JSON-logging prototype already follows. The boundary can be **widened
later** per external pre-approval; the template enforces this safe default until
then.

## 4. Standard guide structure (two-layer, drill-down)

Every guide follows one template so the set reads as a coherent family. The
meat is the front (numbered sections); supporting material is appendices
(lettered) — the split is deliberate and visible, so a reader sees the forest
before the trees.

**Front matter (metadata block):** title · MQ version pinned (9.4) · status ·
*last validated in lab* date · related guides.

**Body — the how-to (target 2–3 pages):**
1. **Purpose & audience** — what this configures and who needs it
2. **Scope & version floor** — what is in/out; MQ version applicability
3. **Recommendation** — approaches ranked strongest → weakest, each with its
   compromise stated (omitted only where genuinely one way exists)
4. **How to configure it** — steps in prose, minimal inline declarative config
5. **Verify it worked** — the checks that prove success
6. **What stays / caveats** — what cannot be turned off, known sharp edges

**Appendices (A…N, supporting only):**
- **A — Full parameter reference.** Every setting, value, default, rationale.
- **B — Alternatives & tradeoffs in depth.** The evidence behind §3's ranking.
- **C — Complete configuration examples.** The exhaustive stanza / MQSC blocks.
- **D — Troubleshooting.** Symptom → cause → fix.
- **E — References.** Pinned IBM Docs links (9.4).

Not every guide needs every appendix; the letters are a menu, used where
warranted. The rule is structural: **numbered = essential, lettered =
supporting.** No essential material lives in an appendix, and no appendix
material pads the front.

## 5. Home & DocsTree integration

- **New tree: `docs/site/docs/guides/`** — under the published mkdocs `docs_dir`,
  so the guides render as a real site section. (Repo-root `docs/` is not part of
  the mkdocs build.) One file per guide, named `mq-<topic>-guide.md`.
- **Distinct from `docs/reference/`.** The reference tree holds lab-specific
  material (real hostnames, shell blocks — e.g. the `nativeha-crr-*` guides).
  The guides tree is 100% generic and shareable. Different audience, different
  home.
- **Published to the DocsTree.** A new **"Guides"** section is added to the
  `docs/site/mkdocs.yml` nav, with an index page that lists the guides and links
  the standard/template.

## 6. Epic task breakdown

### Task 0 — Guide standard & template (foundation)
Establish the mold before mass-producing:
- Write the template and authoring conventions: the two-layer structure, the
  content-boundary rule (§3), the recommendation-ranking convention (§2).
- Create the `docs/site/docs/guides/` tree and the DocsTree "Guides" index page +
  nav entry (§5).
- **Retrofit `mq-json-logging-config.md` to the template** as the worked example
  that proves the structure. This **is** the JSON diagnostic logging guide
  (formerly a separate Task 4, now folded in): promoting the prototype to the
  template both delivers the first real guide and validates the mold in one pass.

### Tasks 1–3 — Flagship guides
1. **Prometheus / MQ-metrics configuration.** The statistics and accounting
   settings — QM-wide and per-object — that must be coherent for the collectors
   and dashboards; the scrape-side label contract described in prose.
2. **TCP configuration.** The `TCP` stanza and related channel/listener
   parameters; what each changes and when to touch it.
3. **Security.** Two halves: (a) TLS/SSL setup and certificate management;
   (b) baseline security — `CHLAUTH`, `CONNAUTH`, object authorizations
   (`setmqaut`).

> JSON diagnostic logging is **not** a separate flagship task — it is delivered
> by Task 0 as the template's worked example. It pairs with Task 1's collectors
> story once both land.

### Backlog (seeded, not yet committed)
Seed as tasks so the epic has visible runway:
- HA-topology selection (Native HA vs RDQM vs Multi-instance) — flagship
  recommendation-ordered guide
- OS platform selection (RHEL vs Ubuntu)
- Queue-manager tuning & limits (MAXHANDS, MONQ, STAT*, `LimitNOFILE` / AMQ5657W)
- systemd lifecycle for QMs and mqweb
- mqweb / REST API configuration & hardening
- Client connection configuration (CCDT, auto-reconnect, safe SCO/plaintext)
- Log-shipping pipeline (MQ JSON → journal/syslog → SIEM), generic
- Certificate / PKI & cert-label management (addresses the open "are we
  over-using cert labels" question)
- MQ ↔ DNS interaction & dependencies — which addresses must be resolvable
  (forward and reverse), DNS's role in `CONNAME` / cluster-receiver CONNAMEs /
  `CHLAUTH`-by-hostname / HA-VIP / client (CCDT) resolution, and the costs, risks,
  and failure-domain implications of depending on DNS (resolve-by-IP vs
  by-hostname tradeoffs)

## 7. Non-goals

- **No code.** Per §3 — declarative config only.
- **No lab-specific values.** Hostnames, IPs, secrets, and site choices stay in
  `docs/reference/`, never in a guide.
- **Not the OSS roadmap.** Publishing the collector/dashboard *code* is #368's
  job, tracked separately.
- **No dictated tradeoffs.** Guides recommend and evidence; they do not decide
  for the reader.

## 8. Definition of done (per guide)

A guide is done when:
1. It follows the template (§4) — two-layer, front-loaded.
2. It respects the content boundary (§3) — no code.
3. It is generic — no lab-specific values.
4. Where alternatives exist, they are ranked with compromises stated (§2).
5. Its configuration has been **validated in the lab**, and the validation date
   is recorded in the front matter.
6. It is wired into the DocsTree nav and index (§5).

## 9. Maintenance model

Guides are living documents. As the lab evolves (notably once full observability
is up and the security posture is re-validated end-to-end), guides are revisited
and their *last-validated* dates advanced. The epic stays open while there are
still guides worth writing; new topics become new tasks under it. When the guide
set is complete and no further topics remain, the epic **closes** — it is finite,
not standing (§1).
