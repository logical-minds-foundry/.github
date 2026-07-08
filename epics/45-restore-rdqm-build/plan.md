# Restore RDQM build functionality — plan

- **Epic:** `logical-minds-foundry/.github#45`
- **Spec:** [`spec.md`](./spec.md)
- **Date:** 2026-07-08

This is a repair epic (see spec §0): the plan sequences investigation and repair,
not the build of a pre-designed solution. Some tasks are **investigate-then-fix**
— their fix design is produced by the task, not here.

## Task graph

```
#560 rdqm-ssh-access ──┐
#561 coordinated create ┼──► #565 validation (cold-rebuild green, gating)
#562 diag fix ─────────┘
#563 IBM bug report        (independent; diagnostics already gathered)
#564 pcmk audit ──────────► #47 follow-on brainstorm ──► (maybe) new pcmk epic
#48 doc-review (final gate)
```

## Sequence

1. **#560 — `rdqm-ssh-access` role.** Foundational; everything else that creates a
   QM depends on it. Deliver enable + disable paths. A validated enable playbook
   from the triage session is the starting point.
2. **#561 — rewrite `rdqm-qm-create.sh`** to the coordinated one-`crtmqm`-per-site
   flow. Depends on #560. Removes the secondaries-first workaround.
3. **#562 — fix `mq-diag-logging` (investigate-then-fix).** The `ClusterSyslog`
   conflict is now reproducible through the working create; diagnose the true
   cause and fix. Can proceed in parallel with #560/#561 for the diagnosis, but
   its green confirmation needs the working create. Supersedes #556.
4. **#565 — validation (gating).** Runs only after #560/#561/#562 merge and a
   clean `rdqm-rhel` cold rebuild. Via `issue-validate`; PASS closes it, FAIL
   files fix tasks and keeps the epic open.
5. **#563 — IBM bug report.** Independent of the repair; the diagnostics are
   already captured (this session + #559). Land the `docs/reports/` doc for the
   human to file with IBM.
6. **#564 — pcmk audit (probe).** Independent investigation. Its result feeds #47.

## Bookends

- **#46 (this task)** — publishes this spec + plan.
- **#47 — follow-on brainstorm** — after the repair lands, review outcomes and
  mint follow-on epic(s); notably a "Restore pcmk build" epic if #564 finds pcmk
  affected.
- **#48 — doc-review (final gate)** — verify the versioned site docs (`docs/site/…`)
  reflect the RDQM changes before the epic rolls up.

## Carried-in issues

- **#559** (RDQM root cause) — the definitive diagnosis; source material for #562.
- **#558** (tlshd logging) — PR in flight; the observability fix that unmasked the
  TLS-is-fine finding. Land its PR.
- **#556** (ClusterSyslog v1) — merged but incomplete; #562 supersedes it. No
  reopen; #562 references it.

## Notes

- The triage that produced this epic churned the live `rdqm-rhel` DRBD/mqm state;
  #565 requires a genuine cold rebuild regardless.
- IBM docs are cached under `build/refs/ibm-docs/ibm-mq/9.4.x/`: `availability-drha-rdqm-worked-example`, `availability-creating-drha-rdqms`, `solution-setting-up-passwordless-ssh-sudo-access`, `availability-requirements-rdqm-ha-solution`.
