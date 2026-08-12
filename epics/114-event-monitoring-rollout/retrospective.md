# Retrospective — Event monitoring as a de-facto standard on every queue manager (epic #114)

Partners [`spec.md`](spec.md) and [`plan.md`](plan.md). Read spec → plan →
retrospective: what we set out to do, how we planned it, and honestly how it went.

## §0 — At a glance

Event monitoring started life as a **single-QM proof of concept** (only `SVCQM`).
This epic made it a **de-facto standard configured on every queue manager** — a
shared `mq-event-monitor` Ansible role wired into all four HA/DR arms plus the
shared `SVCQM` counterparty, collecting with `amqsevt -o json_compact` to journald
(`mq-events`) → Alloy → Loki → Grafana. The *mechanism* was unchanged; the epic
changed **which QMs run it**, and hardened the whole thing along the way.

### Work delivered

| PR | What it did |
|---|---|
| [#740](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/740) | T1 — consolidate the `mq-event-monitor` role (host-prep + MQSC halves); refactor `mq-qmgr` to include it |
| [#745](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/745) | T2 — wire it into the Native HA arms (nativeha-rhel + nativeha-ubuntu) |
| [#754](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/754) | T3 — wire it into the pcmk-ubuntu arm |
| [#747](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/747) | T4 — wire it into the rdqm-rhel arm (the disproportionate-effort task, last) |
| [#744](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/744) | T5 — label the stream `unit=mq-events` in Alloy regardless of systemd unit |
| [#739](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/739) | Docs — correct + re-verify the shipped how-to + site guide (the load-bearing LOGGEREV / `json_compact` fix) |
| [#771](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/771) | Docs sweep — reflect the every-QM standard across the site/report docs |
| [#126 (pending)](https://github.com/logical-minds-foundry/.github/issues/126) | Docs — update the `.github` epic-31 design spec to the shipped every-QM reality |

- **Repos touched:** `mq-resiliency-lab-for-linux` (role, four arms, docs);
  `.github` (spec/plan, epic-31 spec, this retrospective).
- **Tasks:** spec/plan (#115), T1–T5 (#721/#722/#723/#724/#742), docs review
  (#718), docs correction (#729), epic-31 spec (#126), two brainstorms
  (#116/#121, closed not-needed), retrospective (#135).
- **PRs:** 7 merged + #126 pending. **Releases:** none.
- **Span:** opened ≈2026-07-17 → closing **2026-07-26**.

## §1 — How the plan evolved

**The rollout itself was the easy part.** T1–T5 landed largely as planned: a
consolidated role, `mq-qmgr` refactored to include it, and the four arms wired one
by one (rdqm last, as its disproportionate-effort marking predicted). What
dominated the epic was the **chain of real defects the rollout flushed out** — most
of them latent long before this work touched them.

- **The swallowed-error / LOGGEREV correction (#729 → #739) was the load-bearing
  moment.** A how-to that had already been sent to the employer carried **untested
  `LOGGEREV(ENABLED)` syntax**. `LOGGEREV` is valid only on linear-logging QMs; on
  the lab's circular-logging QMs MQ rejects it (`AMQ8518E`) and — because
  `ALTER QMGR` is atomic — rejects the *entire* enable statement, so **no events at
  all**. Worse, the failure was **silently swallowed** by a
  `failed_when: rc not in [0, 10]`. We dropped `LOGGEREV`, made the conditions
  fail-loud, and **re-verified the whole document live in the lab** rather than
  trusting the docs.
- **The feed itself was being shredded.** `-o json` (pretty, multi-line) is torn
  apart by the line-oriented `logger`, and `logger`'s ~1 KiB default truncated long
  events. Fixed to **`-o json_compact`** (one event per line) piped through
  **`logger --size 32768`**.
- **Cold-building on x86 exposed portability bugs a macOS-only build had masked.**
  `fence_virsh` had a hardcoded `pmoore` hypervisor user (#751); `setmqaut` ran
  against a stopped QM (`AMQ7028E`, missing `strmqm`, #755). Each fix revealed the
  next.
- **The host wedged mid-rollout.** Repeated box re-bakes leaked libvirt base images
  and filled the ephemeral root disk (#759); we doubled the disk (#757) and pruned
  the orphans.
- **The collector's own resilience** (restart into the `MQRC_OBJECT_IN_USE` 2042
  exclusive-handle window) became the self-healing wrapper (#122 / #767).
- A small **Alloy relabel typo** (double-underscore journald field) was fixed as
  part of T5 (#744).

Live verification was **per distinct QM-creation mechanism** (SVCQM + one Native HA - pcmk + rdqm), not all four stacks, per the plan.

## §2 — Lessons learned

- **Fail loud on a must-succeed MQSC.** `failed_when: rc not in [0, N]` that hides a
  genuine rejection is exactly how a broken configuration ships looking green.
  Swallowing the error at the code layer became a hallucinated success at the
  human layer (a document that "worked" but enabled nothing).
- **Verify in the lab, not from the docs.** The `LOGGEREV` syntax reached the
  employer untested. The whole point of the lab is to validate before we publish —
  we re-verified the entire how-to live and re-dated it.
- **Cold-build in the *target* environment.** Building only on macOS masked
  host-user / hypervisor-access / QM-start-ordering assumptions; cold-building on
  x86 surfaced two portability bugs in quick succession.
- **`json_compact` + `logger --size` for line-oriented shipping.** A pretty JSON
  event and a line-based transport are fundamentally incompatible.

## §3 — Compromises & tradeoffs

- **Live proof was per-mechanism, not per-stack.** The Native HA twin
  (nativeha-ubuntu) was confirmed by config-inclusion rather than a separate live
  run — a deliberate acceptance-criteria choice, not an oversight.
- **rdqm was the disproportionate-effort task, sequenced last** — accepted up front
  in the spec; it still cost the most (and its cold-create surfaced separate DR
  bugs, below).
- **Both follow-on brainstorms (#116/#121) closed not-needed rather than run.** The
  rollout is stable through every expected real-world code path; deeper reliability
  numbers would need a long-run, high-volume study, which we judged not warranted.
  Reactive/operational monitoring covers it from here.

## §4 — New problems & opportunities

The rollout was a bug-flushing exercise; nearly everything it surfaced is filed:

- **Reliability / portability tail → reliability epic `.github#104`:** fence user
  (#750/#751), `setmqaut` ordering (#752/#755), disk headroom (#756/#757),
  box-image leak (#759), and the rdqm DR cold-create issues — **#746** (fixed) and
  **#768** (open: `crtmqm` mkfs I/O error).
- **Collector resilience → `.github#122` / #767** (built: the self-healing wrapper).
- **Python-3.14 toolchain instability** (surfaced later on the same infrastructure,
  during #110): `vergil-tooling#2460`/`#2461`/`#2468`, resolved in-repo by the #782
  pin. Adjacent, not owned by #114.

## §5 — What's next

- **Brainstorm dispositions (#116/#121):** a planned follow-on and a dedicated
  reliability initiative were both judged **not needed now** — the feed is stable,
  in production, and problems are better caught operationally. A long-run,
  high-message-volume reliability study is the only materially deeper option, held
  for if/when it's warranted.
- The **reliability tail (#768)** remains open under `.github#104`.
- The consume/generate reference work that #114 unblocked shipped as its own epic
  (**#110**); mechanising event generation from it was explicitly declined
  (#112, won't-do — a custom transactional handler is the better direction if we go
  deeper).
