# Retrospective — Stage 2: Internal MQ Authorization

Epic: [logical-minds-foundry/.github#74](https://github.com/logical-minds-foundry/.github/issues/74) · Partners [`spec.md`](spec.md) and [`plan.md`](plan.md).

## §0 — At a glance

Stage 1 secured and cert-authenticated every connection across the lab's HA arms. **Stage 2 acted on those identities**: it retired `MCAUSER('mqm')`/`CHLAUTH(DISABLED)` on the instrumented app queue managers, mapped each authenticated certificate DN to a non-privileged service account (`mqapp`/`mqmon`/`mqsvc`) via `CHLAUTH SSLPEERMAP`, and locked the queues to a minimal `setmqaut` surface behind a deny-all back-stop — proven on `pcmk-ubuntu` first, then fanned out to `nativeha-ubuntu` and `rdqm-rhel`, each validated live through a real failover. What shipped matches the plan closely; the one substantive addition was a genuine authorization **bug the hardening caught and fixed** (the DLQ `+setall` gap).

### PRs delivered

| PR | Issue | What it did | Merged |
|---|---|---|---|
| #621 | #612 | Provision `mqapp`/`mqmon`/`mqsvc` OS principals + single-source `authz.yml` | 2026-07-14 |
| #622 | #613 | Research: `PUTAUT(CTX)` / `+setall` context authority | 2026-07-14 |
| #623 | #614 | Research: per-channel vs QM-level dead-letter queue | 2026-07-14 |
| #624 | #616 | `CHLAUTH` + deny-all back-stop + `SSLPEERMAP` + `setmqaut` on the pcmk app QM | 2026-07-14 |
| #633 | #630 | Induced-denial probe + N1/N3/positive/post-failover validation harness | 2026-07-14 |
| #755 | #752 | Fix: start the QM before `setmqaut` (AMQ7028E on cold build) | 2026-07-21 |
| #793 | #792 | Stage the `OU=ops` keystore on the app host for the N1 check | 2026-07-27 |
| #846 | #844 (validates #617) | N2/N4/N5 induced-check hardening **+ the DLQ `+setall` fix** | 2026-07-30 |
| #850 | #619 | Fan-out: internal authorization on the `nativeha-ubuntu` arm | 2026-07-30 |
| #851 | #620 | Fan-out: internal authorization on the `rdqm-rhel` arm | 2026-07-31 |
| #852 | #611 | Product-level IBM MQ authorization guide in the site docs | 2026-07-31 |

**Closed on outcome (no PR — validation/research tasks):** #615 (mqweb REST authorization posture), #617 (hardening validation → landed via #844), #618 (pcmk induced-denial + post-failover + cold-rebuild validation).

- **Sub-issues:** 15 / 15 closed (+ this retrospective).
- **Repos touched:** `mq-resiliency-lab-for-linux` (all code + docs), `.github` (spec/plan/retrospective).
- **Span:** first Stage-2 PR 2026-07-14 → last fan-out 2026-07-31.
- **Arms proven:** `pcmk-ubuntu`, `nativeha-ubuntu`, `rdqm-rhel` — each one-pass authorized bring-up + induced denials + native-mechanism failover.

## §1 — How the plan evolved

The plan held. The core sequence (single-source `authz.yml` → account provisioning → `CHLAUTH`/`setmqaut` on pcmk → validation → fan-out → docs) ran as written. Four deviations are worth recording:

1. **The §10 negatives split first-slice vs. hardening.** The plan's own alignment folded N1+N3 (+positives+post-failover) into the first validation slice and deferred N2/N4/N5 to a coupled hardening task, because each of the deferred negatives needs its research (N4↔`+setall`, N5↔DLQ) or a throwaway cert (N2). That split held and was the right call.
2. **A validation task turned out to need code.** #617 was labelled `validation` ("not PR-workable"), but N2/N4/N5 genuinely required harness code (a rogue-cert mint, an identity-context probe, a DLQ probe) plus an `authz.yml` fix. The tooling correctly refuses a PR on a validation issue, so the work was split into a PR-workable task (#844) that the validation (#617) references. Latent inconsistency in the plan's task taxonomy, resolved cleanly.
3. **The `mqsvc` DLQ grant was corrected mid-flight.** The interim was `+put`; N5 proved a `PUTAUT(DEF)` receiver MCA opens the DLQ with `MQOO_SET_ALL_CONTEXT` and therefore needs `+put +setall`, or an undeliverable reply fails and wedges the channel. Fixed in `authz.yml` and re-validated.
4. **`rdqm-rhel` was briefly mis-assessed as blocked.** The `mqlab parity` matrix marks rdqm verbs `NOT_YET`; that was wrongly read as "the arm can't come up." It only gates the convenience `mqlab qm`/`dr` verbs — bootstrap/provisioning and authorization work fine, and failover is inducible natively via `rdqmadm`. Corrected within the session and the fan-out completed.

## §2 — Lessons learned

- **Validation-under-adversity is the real test.** N5 didn't rubber-stamp the interim DLQ grant — it *broke* it, surfacing a real least-privilege gap that would have silently wedged a channel in production. Induced negatives that are designed to *fail the current config* earn their keep.
- **Reason codes don't tell you intent.** The `+setall`-on-DLQ finding is a specific instance of a general truth this epic hammered home: the same MQ reason code (`2035`) can mean "misconfigured, fix it" or "correctly blocked a breach, never retry." Authorization posture must be reasoned about per-object and per-context, not pattern-matched.
- **A capability matrix is a declaration, not a probe.** `NOT_YET` describes intended tooling surface, not what the arm can physically do. When a matrix says "no," verify against the live system before believing it — the cost of the wrong assumption here was nearly skipping a whole arm.
- **The arm-agnostic model paid off exactly as designed.** Each fan-out (`nativeha`, `rdqm`) was a ~2-file per-arm delta reusing `authz.yml` + `authz.mqsc.j2` verbatim; only channel names and node lists changed. The Stage-1 topology-variable discipline made Stage 2 portable.
- **Bootstrapping from the feature worktree gives one-pass proof cheaply.** Because `mqlab` symlinks a worktree's `cache`/`state` back to main, each fan-out could bring its arm up *with the authz code baked in*, proving a one-pass authorized bring-up rather than applying authz live after the fact.

## §3 — Compromises & tradeoffs

- **The DLQ is still a bare QM-wide `SYSTEM.DEAD.LETTER.QUEUE`.** The `+setall` fix makes dead-lettering *work*, but there is no per-counterparty DLQ, no monitoring, no reporting, no handling. Knowingly deferred — it became the DLQ follow-on epic (§5).
- **`mqmon`'s `+sub` grant is a to-narrow placeholder.** The monitoring identity subscribes at `SYSTEM.BASE.TOPIC` (the topic-tree root) rather than a scoped `$SYS/MQ` resource-monitoring subtree — an over-grant flagged in `authz.yml`, left as-is because narrowing it precisely wasn't load-bearing for Stage 2's thesis.
- **`CONNAUTH` stays disabled (cert is the credential).** A deliberate posture, not an oversight; the site-docs guide presents enabling `CONNAUTH` as a ranked defense-in-depth option.
- **The N5 `authz.yml` change wasn't re-cold-rebuilt on pcmk specifically.** It was validated live on pcmk and cold-rebuilt one-pass on nativeha and rdqm (same `authz.yml`), so the grant set is proven to cold-build; a pcmk-specific re-confirm remains an optional low-priority loose end.
- **Process fumbles during the rdqm bring-up.** A stray `virsh destroy` broke the box-verify DHCP step (cost one bootstrap retry), and detached-process hygiene was sloppy (self-matching `pgrep` false alarms, an almost-double bootstrap). No lasting damage, and the `setsid`-detached bootstrap pattern that emerged is the right tool for disconnect-prone long runs — but the path was messier than it should have been, and the honest lesson is *don't touch live infra to "help" a running orchestrator*.

## §4 — New problems & opportunities

- **DLQ is a total observability + handling blind spot** — surfaced hard by N5. → Spun off as a follow-on epic seed: [logical-minds-foundry/.github#158](https://github.com/logical-minds-foundry/.github/issues/158).
- **Per-node OS service accounts are an operational burden** — they must exist on every HA/DR node or authorization breaks after failover. LDAP (`idpwldap`) authorization would retire them. → Backlogged as an idea: [logical-minds-foundry/.github#157](https://github.com/logical-minds-foundry/.github/issues/157).
- **RDQM cold-create is timing-flaky** (`AMQ3812E`/rc=71 on the coordinated create; cleared by a `--from provision` resume). Not new to this epic, but re-confirmed — a candidate for a resiliency hardening pass on the rdqm bring-up.
- **SSH-disconnect fragility of long orchestrations.** Several disconnects killed sub-agents and harness-tracked background tasks mid-run. The recovery pattern (re-derive state from GitHub + live lab; run long bootstraps `setsid`-detached) worked but is manual; worth a tooling convention.

## §5 — What's next

- **DLQ observability & handling framework** — fully brainstormed as the closing bookend; seeded and ready for `epic-create`: [.github#158](https://github.com/logical-minds-foundry/.github/issues/158). Phase A = observability-first (exporter depth gauge + non-destructive `MQDLH` decoder → Loki → Grafana dashboards + a reproducible failure-scenario catalog; dashboards not alerting; no percentage depth-events). Phase B = deliberately thin, deferred handling (observe-only + quarantine; reason-code auto-classification rejected as unsafe).
- **LDAP authorization** — backlogged idea [.github#157](https://github.com/logical-minds-foundry/.github/issues/157); revisit via `triage-review` when the ROI improves.

## Appendix A — Operational notes

This was a validation-heavy epic; the live-lab sequence per arm was:

1. **Bring-up:** `mqlab bootstrap <arm>` from the feature worktree (authz baked in) → one-pass authorized bring-up. Verify: `CHLAUTH(ENABLED)`, deny-all back-stop, three `SSLPEERMAP` maps, `setmqaut` grants, accounts on every node.
2. **Induced denials (as the app host, over the arm's app CONNAME):** N1 `mqmon` MQPUT → `APP.REPLY` denied 2035; N3 `mqapp` MQGET ← `SVC.REQUEST` denied 2035. Positive: `app_requester --count 1` round-trip. The `OU=ops` (`mq_prometheus`) keystore is staged from mon-probe onto the app host for the N1 identity (#792 pattern).
3. **Post-failover** (per-arm mechanism — the key portability point): pcmk `pcs resource move`; nativeha GroupRole switchover; **rdqm native `rdqmadm -s`/`-r`** (the `mqlab` failover verb is `NOT_YET`). Re-assert one negative + a positive on the new active node.
4. **Cold-rebuild acceptance:** the one-pass bring-up in step 1 *is* the cold-rebuild proof.

Gotchas: the DLQ `+setall` grant is required for dead-lettering; RDQM cold-create may need a `--from provision` resume; RDQM runs under TCG emulation, so its bring-up is materially slower than the Ubuntu (KVM) arms.
