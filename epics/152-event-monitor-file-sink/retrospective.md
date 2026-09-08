# File-sink variant of the resilient MQ event collector — retrospective

**Epic:** `logical-minds-foundry/.github#152`
**Status:** complete
**Span:** 2026-07-29 → 2026-07-30
**Partners:** [`spec.md`](spec.md) (what we set out to do) · [`plan.md`](plan.md) (how we planned it) · this (how it went)

---

## §0 At a glance

We set out to fix an anti-pattern in the two 2026-07-28 event-monitoring reports:
they shipped a fully tested **syslog** collector and then relegated the **file**
sink — the variant a real consumer runs — to a "`run.sh` is a one-line change"
footnote, so the *delivered* artifact was no longer the *tested* artifact. What
shipped: the file sink as a **first-class, render-time-selected, independently
tested** path in the `mq-event-monitor` role (`mq_event_sink: syslog|file`), proven
end-to-end on a live Native HA queue manager, and a report that documents the exact
script to install as written — the footnote retired.

**Work delivered:**

| PR | Merged | What it did |
|---|---|---|
| `.github#156` | 07-29 | Spec + plan (closed doc task `#153`) |
| `#840` | 07-29 | **Impl** — `mq_event_sink` selector; render-time Jinja branch in `run.sh.j2` (single-purpose delivered script; byte-identical syslog render guard via `tools/render-event-run.sh`); sink-aware `SERVICE`; sink-parameterized validation harness (closed `#836`) |
| `#842` | 07-29 | **Fix** — file-sink `DESCR` > MQ's 64-char limit (`AMQ8413E`); shortened + `assert` guard (closed `#841`) |
| `#843` | 07-30 | **Report** — `2026-07-29-mq-event-monitor-file-sink-resilient.md`; retire the one-line-change footnote in both 07-28 reports (closed `#839`) |
| `#845` | 07-30 | **Docs-review** — selectable/file sink reflected in the site guide + architecture doc (closed `#825`) |
| `.github#<this>` | — | Retrospective (closes `#154` and the epic) |

**Operational tasks (no PR — recorded `Outcome: SUCCESS`):**

| Task | What it proved |
|---|---|
| `#837` deploy | File sink provisioned + running on live `NHAUAPP` (file-variant `run.sh`, no `logger`; `.events.json` + `.error`) |
| `#838` validate | `== ALL SCENARIOS PASS ==` — A1 JSONL, A2 destructive drain, B2 clean stop/no orphan, B3 2042 crash recovery |

**Counts:** 8 children (7 + this retrospective) · 6 PRs · 2 operational tasks ·
**0 releases** · 2 repos (`mq-resiliency-lab-for-linux`, `logical-minds-foundry/.github`).

**Plan-vs-actual delta: moderate.** The engineering landed as designed; the delta
was all in *execution environment* — one unplanned fix task (`#841`) and a
validation-target substitution forced by host architecture (below).

---

## §1 How the plan evolved

The plan's spine — **impl → deploy → validate → report** — held exactly. Three
deviations, none in the design:

1. **Validation target: RHEL → Ubuntu Native HA (forced by host arch).** The plan
   named the Native HA **RHEL** arm (samples present, prior wrapper validation
   there). On bring-up, that arm's `mq-nativeha-rhel9` box is **x86_64** and cannot
   be built on the **aarch64** lab host — cross-arch emulated box builds are
   disabled by design (`#103 D11`). We diagnosed it to root cause (not a
   regression) and, with the human's call, substituted the **Ubuntu** Native HA arm
   (`NHAUAPP`): identical mechanism — same role, same `amqsevt`, same
   `setsid`/`SERVICE` model — so the corner cases proven are OS- and
   arch-independent. The report states this explicitly.

2. **One unplanned fix task (`#841`), surfaced by the live deploy.** The first file
   deploy failed with `AMQ8413E` (String Length Error): the file-sink `DESCR`
   embedded the data-file path (~103 chars) over MQ's 64-char `SERVICE` `DESCR`
   limit, aborting the whole `DEFINE`. A **runtime MQSC field limit** that neither
   `vrg-validate` nor the byte-identical render guard can see. Fixed with a
   path-free 59-char `DESCR` **and** a fail-loud `set_fact` + `assert` guard, then
   re-deployed clean.

3. **Lab bring-up friction (absorbed, not escalated).** The bootstrap needed a
   post-env-sync re-exec (`--from vms`), a `mq-nativeha-ubuntu` box rebake, and one
   idempotent retry after a transient `app-client` boot timeout. The `observe`
   phase failed on a transient SSH drop mid Grafana-dashboard copy — orthogonal to
   this epic (the file sink never touches the observability plane) and left as a
   retryable follow-up.

---

## §2 Lessons learned

- **Live deployment earns its place in the pipeline.** The single most valuable
  moment of the epic was the `AMQ8413E` catch — a real defect that every static gate
  (lint, render-diff, unit tests) passed clean, visible only when the `DEFINE SERVICE`
  hit a live queue manager. The `impl → deploy → validate` ordering is not
  ceremony; it is where runtime-only constraints get caught. Repeatable takeaway:
  for anything that emits MQSC (or any command a static tool can't execute), a live
  deploy step is load-bearing, not optional.
- **The epic's own thesis, proven on itself.** Had the file variant shipped as the
  original "change this one line" footnote, `AMQ8413E` would have shipped *to the
  consumer*, on their queue manager, at the point of use — exactly the failure the
  epic existed to prevent. Render-time selection + a single-purpose *tested*
  delivered artifact is the right pattern; the footnote was not a shortcut, it was a
  latent bug delivery mechanism.
- **Push variant selection up into the management layer.** Keeping the sink choice
  in Ansible (render-time Jinja) rather than the runtime script gave two wins at
  once: a clean single-purpose artifact *and* a maximal shared resilience core that
  can't drift. The byte-identical syslog-render guard let us refactor that shared
  core fearlessly.
- **Arch-gating is real and OS-independence is leverage.** x86 arms need x86 hosts.
  Because the collector's corner cases are OS-independent, the Ubuntu arm was a
  faithful stand-in — recognizing that invariant unblocked the whole run.

## §3 Compromises & tradeoffs

- **Validated on Ubuntu, not RHEL.** Deliberate, forced by `#103 D11`; acceptable
  because the mechanism is identical and OS-independent, and the substitution is
  documented in the report and #838. RHEL-specific validation would need an x86 host
  and is not blocked by anything in this epic.
- **File sink deployed as a temporary override.** We did not stand up a permanent
  file-sink lab arm; `NHAUAPP` reverts to the syslog default on its next normal
  provision, keeping the committed lab default and the journald→Alloy pipeline
  intact. The tradeoff: no standing file-sink demo, re-provision to reproduce.
- **`observe` phase left incomplete.** The bring-up's Grafana dashboard step failed
  on a transient and was not retried — out of scope for a file-sink epic that never
  touches observability, but it means the arm's dashboards are not up.
- **Non-goals stay non-goals.** Rotation (copy-truncate), checkpointed forwarding,
  and lossless/transactional delivery remain the consumer's responsibility, as
  designed — documented, not built.

## §4 New problems & opportunities

- **RHEL-on-aarch64 unbuildability is under-discoverable — now gated.** `#103 D11`
  is correct and intentional, but it surprised us mid-flight, surfacing only as a
  deep box-bake `StepFailedError`. *Disposition:* **filed and since shipped as
  `mq-resiliency-lab-for-linux#847` (closed), under the lab-reliability epic `.github#104`.**
  The fix formalizes the rule — on Apple Silicon (aarch64) only the Ubuntu
  stack is practical (Ubuntu-for-ARM + emulated MQ; RHEL will not ship for Apple
  Silicon), x86 runs everything natively — as a **fast preflight gate** that aborts
  RHEL-stack create/bootstrap on an aarch64 host before any box bake, with
  `mqlab doctor` reporting the same capability.
- **MQSC field-length limits are a bug class, not a one-off.** `#841` fixed the
  `DESCR` case with an assert; the same runtime limit exists for other fields
  (e.g. `STDOUT`/`STDERR` path lengths, max 256). *Disposition:* logged; the assert
  pattern generalizes if another field ever bites.
- **`observe`-phase transient.** A retryable SSH-drop during Grafana dashboard
  deploy. *Disposition:* logged; `mqlab bootstrap nativeha-ubuntu --from observe`
  when the dashboards are wanted.

## §5 What's next

- **No enabling chain was seeded**, so there is no follow-on epic to reference. The
  file-sink resilient collector is a clean candidate for the **reusable-component
  extraction roadmap** (`mq-resiliency-lab-for-linux#368`, closed design issue) —
  the collectors are already stdlib-only/non-MQI and now dual-sink — but harvesting
  it is that roadmap's call, not a debt this epic owes.
- The RHEL-on-aarch64 gate (§4) shipped as `mq-resiliency-lab-for-linux#847`
  (closed) under the lab-reliability epic `.github#104` — the next operator on Apple
  Silicon now gets a clean abort instead of a deep box-bake failure.

---

## Appendix A — Operational notes

This epic was deploy/validation-heavy; the mechanical sequence, for the next person
reproducing it:

1. **Bring up the arm (aarch64 host):** `mqlab bootstrap nativeha-ubuntu`. Expect a
   post-env-sync re-exec (`--from vms`), a `mq-nativeha-ubuntu` box rebake, and a
   possible idempotent retry on a transient guest boot timeout. The RHEL arm will
   **not** build here (`#103 D11`).
2. **Deploy the file sink** without the full site playbook (which needs
   bootstrap-injected `qm_app`): run only the `mq-event-monitor` role against
   `nha_ubuntu_a` with `qmgr_name=NHAUAPP mq_event_sink=file` (host-prep on all
   nodes → service on the active). `ANSIBLE_ROLES_PATH` must point at
   `ansible/roles` when the playbook lives outside `ansible/`.
3. **Cycle into file mode:** `STOP`/`START SERVICE(MQ.EVENT.MONITOR)` — `DEFINE …
   REPLACE` updates the definition but does not restart a running collector.
4. **Validate:** `tools/validate-event-monitor-wrapper.sh NHAUAPP MQ.EVENT.MONITOR
   file /var/mqm/event-monitor/NHAUAPP.events.json /var/mqm/event-monitor/NHAUAPP.error`
   → require `== ALL SCENARIOS PASS ==`. Evidence:
   `docs/reports/assets/mq-event-monitor-file-sink/`.
5. **Gotcha:** the `SERVICE DESCR` must be ≤ 64 chars (`AMQ8413E` otherwise) — the
   role now asserts this pre-flight.
