# Plan: Adopt IBM MQ 10.0.1 and the support-gated Ubuntu 26 default

- **Epic:** [logical-minds-foundry/.github#315](https://github.com/logical-minds-foundry/.github/issues/315). **Spec:** [spec.md](spec.md).
- **Form:** compact by decision (2026-10-11). Each task's full steps, files and acceptance live in its issue body (or its procedure comment, for the operational tasks). This plan fixes order, gates, routing and done-ness. It does not restate them.

Repo shorthand: `lab#N` = `logical-minds-foundry/mq-resiliency-lab-for-linux#N`.

## Task table

| ID | Issue | Kind | Run via | Blocked by | Done when |
|---|---|---|---|---|---|
| DOC | .github#316 | docs | this PR | — | This spec and plan merged |
| A | lab#1409 | code | `issue-implement` | 10.0.1 GA and all three Developer tarballs | Re-pin PR merged; kmod findings recorded on lab#1440 |
| B | lab#1440 | code | `issue-implement` | A, plus its own el10/`modver` gate | `rdqm-rhel` offers `rhel:10` (PR merged), **or** closed as not done with the gap in the support matrix §2.4 |
| D | lab#1442 | deployment | `issue-deploy` (macOS half locally; cloud half by the cloud agent) | A, B | SUCCESS comment for both hosts: all boxes REUSE on 10.0.1 |
| V | lab#1443 | validation | `issue-validate` (per host) | D | SUCCESS comment for both hosts: every row passes steps 1–8 in one pass, with `dspmqver` evidence |
| E | lab#1410 | code (docs) | `issue-implement` | 10.0.1 docs published (docs part); V (live parts) | Cited `docs/reference/mq-10.0.1-adoption.md` merged; adoption tasks filed; TLS/PQC placement decided |
| T9 | lab#1285 | code | `issue-implement` | Per stack (spec §2): MQ server row → `pcmk-ubuntu` and `infra_mq`; MQ server + Native HA rows → `nativeha-ubuntu` | Each gated change flipped, with the `mq-client-ubuntu26` bake and the MQ-commons rows re-validated cold for `infra_mq` (PR merged), **or** dispositioned per §Gate policy |
| DR | lab#1441 | docs review | `issue-implement` (sweep) | A, B, D, V, E, T9 resolved, plus any adoption tasks E files in this epic | Site, dev and reference docs reflect the epic |
| RETRO | .github#317 | retrospective | `epic-retrospective` | every other child closed | `retrospective.md` merged; the epic closes |

## Order of work

1. **Now (before GA):** DOC. E may begin its docs-only research as soon as IBM publishes the 10.0.1 docs, which could be at GA.
2. **At GA (2026-10-13 or later):**
   - **A.** Check the three tarballs and the el10 kmods (fail loud), then re-pin.
   - **B,** right after A.
     - If the el10 `modver` gate fails, close B as not done with the evidence; don't let it delay D.
     - Per the spec §3, a failure affecting only RDQM or Pacemaker is narrowed or deferred and never blocks Native HA.
3. **D:** the bake round on both hosts, run in parallel (macOS locally, cloud via its agent).
4. **V:** the matrix on both hosts. Native HA rows must pass. A failure affecting only pcmk or RDQM is proposed for narrowing.
5. **E's live parts:** these run after V, on the validated 10.0.1 lab. E files its adoption work:
   - smaller items become tasks under this epic;
   - the TLS/PQC hardening becomes tasks here or its own epic, which is the human's decision.
6. **T9:** whenever its gate opens. It is independent of 2–5.
7. **DR**, then **RETRO**.

## Gate policy

- **10.0.1 media.** Nothing re-pins until GA and all three Developer tarballs exist (spec §2). A records any wait as a comment.
- **T9 (Ubuntu 26 defaults and `infra_mq`).** Each change flips only once its own SPCR rows exist, so partial flips are allowed. At the DR stage, for any change still gated, the human chooses one of:
  - **move** T9 to the lab's ad-hoc epic, or to a successor (recommended, so the epic can close);
  - **keep it waiting** (the epic stays open).

  Either way, record the SPCR status as of that date, with the `getTSROsSupportSummary` URL, on T9 and in the retrospective §5.
- **Cold-rebuild gate.** D and V close only on SUCCESS evidence, never on a merge. Any defect V finds is filed as a task under this epic (as #1420 was in #280). The affected rows are re-run cold after the fix merges.

## Defect handling

- File each defect with `vrg-issue-create --epic logical-minds-foundry/.github#315`. Implement it in its own worktree, and leave it at report-ready for the human to submit.
- Re-run only the rows the fix affects, cold. Record in V's result why the other rows didn't need a re-run (the #1284 precedent).

## Notes for implementers

- **Catalog lines stay single-line `{...}`.** `lab/scripts/rhel-catalog-isos.sh` (awk) depends on it.
- **Version tokens appear only in the catalog and in resolved box names** (`tests/test_version_token_guardrail.py`).
- **IBM Docs:** use `python3 tools/ibm_doc_cache.py` (pinned to 10.0.x). Support pages and SPCR JSON can be fetched with curl using a browser User-Agent. Label each claim as data or judgment and give full URLs.
- **Long lab jobs:** run them detached, with stdin from `/dev/null` and a log-mtime stall watch.
- **Validation:** `vrg-container-run -- vrg-validate` is the only gate.
