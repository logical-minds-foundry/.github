# Epic 275 — Retrospective: measurement-driven bootstrap staging

Spec: [`spec.md`](spec.md) · Plan: [`plan.md`](plan.md) · Epic: logical-minds-foundry/.github#275

## 0. At a glance

**Set out to:** make the cold bootstrap of the lab reliable and fast enough to be useful on
macOS/arm64, with an x86 cloud comparison. The method: instrument first, tune per-environment
levers from the data, and validate in parallel on both platforms. No upfront re-architecture.

**What shipped:** both platforms meet the bar for `nativeha-ubuntu --no-dr`.

| | before | after |
|---|---|---|
| macOS/arm64 | baseline failed after ~23 min; later runs never finished observe (40+ min) | **643 / 621 / 587 s** (huge-page guest RAM) |
| x86 cloud (GCE n2-standard-16) | 1018 / 1030 / 987 s; 1000 / 987 / **1367** s | **721 / 673 / 684 / 687 / 689 s**, 5/5 clean (SSD boot disk) |

The macOS bottleneck was **named from data**: second-level page-fault cost under arm64 nested
virtualization, about 1000× the Vergil VM's, triggered when any guest churns memory (#1240). The
fix was 2 MiB huge-page backing for guest RAM. Along the way a long tail of per-run fixed costs
and obs-pipeline defects was removed on both platforms.

**Span:** opened 2026-09-30 → closed 2026-10-03 (≈3.5 days).
**Scope:** 42 child issues (this retrospective included): the spec/plan docs task, 31 code/docs
tasks, 7 validation tasks, 1 spike, 1 brainstorm. One more task (#1266) was moved to the ad-hoc
epic. **32 PRs** across 2 repos in this org, plus **3 PRs** in 2 platform repos
(vergil-project). **Releases:** vergil-vm `v2.1.42`; vergil-tooling `v2.1.223` and `v2.1.224`
(the second presumably carrying #3061; not verified).

### Work delivered (PRs; `mq-resiliency-lab-for-linux` unless noted)

| PR | issue | what it did |
|---|---|---|
| .github #278 | #276 | Epic spec + plan |
| #1206 | #1201 | `PerfRecord` model; per-phase/per-step timing in `run_steps` |
| #1207 | #1202 | `MQLAB_ENV` per-environment profile overrides (`env_profiles`) |
| #1208 | #1203 | Host-contention sampler (vCPU steal, host CPU/IO) |
| #1209 | #1204 | `mqlab perf diff` + parallel-validation procedure doc |
| #1210 | #1205 | Perf report emitted per bootstrap; OpenSearch/boot milestones |
| #1216 | #1211 | Documented `uv run mqlab` as the dev-checkout invocation |
| #1217 | #1212 | Skip the pre-flight SSH probe on a cold lab (−~4 min) |
| #1218 | #1214 | Dashboards post-green API calls: timeout + idempotent retries |
| #1219 | #1215 | Instrumentation gaps: failed-step time, preflight phase, guest busy/iowait/load, Vergil-VM steal |
| #1222 | #1220 | Data Prepper `parse_json` gated to JSON-looking bodies (error flood removed) |
| #1223 | #1221 | Sampler: one persistent SSH connection per guest |
| #1231 | #1225 | apt auto-update permanently off in baked Ubuntu boxes |
| #1232 | #1226 | No per-run `apt-get update` in nativeha acl tasks (−77 s) |
| #1233 | #1227 | Responder pymqi venv baked into `mq-ubuntu2404` |
| #1234 | #1228 | Sampler ssh-mux sockets moved to a local runtime dir (virtiofs fix) |
| #1235 | #1229 | Per-login dynamic MOTD off in baked Ubuntu boxes |
| #1236 | #1230 | OpenSearch `date_detection: false` + `ibm_commentInsert*` text mapping |
| #1242 | #1238 | Data Prepper sink metrics scraped; rejected-documents panel + alert |
| #1243 | #1239 | Bounded local-file DLQ for Data Prepper's OpenSearch sink |
| #1244 | #1241 | **macOS huge-page-backed guest RAM** (env lever + on-demand reservation) |
| #1246 | #1245 | `MQLAB_ENV` auto-detected from the platform (DMI) |
| #1254 | #1253 | net-state published on change; the lab-net-state probe retired |
| #1255 | #1252 | `mqlab` invoked only via `uv run` / an activated env |
| #1256 | #1248 | REUSE boxes stay registered; PKI once per run; `prereq:*` phases |
| #1257 | #1249 | **Cloud boot disk `pd-ssd`** (vergil.toml) |
| #1258 | #1250 | cloud-init trimmed, snapd off on the boot path |
| #1259 | #1251 | Checkout-bound lab-relay-heal timer retired |
| #1262 | #1261 | buildenv git failures print stderr; `obs net-state` needs no git |
| #1270 | #1265 | grafana drop-in dir kept through the snapd purge; bake-dirs guard |
| #1269 | #1199 | Docs review: epic's changes across site and dev docs |
| #1292 | #1268 | Docs: x86 cloud performance numbers filled in |
| vergil-vm #313 | vergil-vm#312 | `boot_disk_type` variable on the GCP (and Azure) vm modules |
| vergil-tooling #3057 | vergil-tooling#3056 | `boot_disk_type` in the VM spec, threaded to tofu with a module-support check |
| vergil-tooling #3062 | vergil-tooling#3061 | `vrg-finalize-pr` waits on required checks; a policy block means "not ready", not fatal |

**Validation and investigation (no PR):** cloud runs #1213, #1224, #1237; cloud streaks #1247
(FAILURE: over the 900 s target), #1260 (FAILURE: unrelated bake defect) and **#1267 (SUCCESS)**;
acceptance gate #1200; spike #1240; cross-stack brainstorm #1264.

## 1. How the plan evolved

`plan.md` defined five tasks (instrumentation → profiles → diff), then "tune levers one at a
time" under #1200. Its premise, inherited from the spec, was that the host was **oversubscribed**
and that staging levers (`boot_batch`, obs vCPU, phase overlap) would find the sweet spot. **The
instrumentation disproved that premise in the first run.** Host CPU averaged 15%, guest steal
stayed under 2%, and the Vergil VM's own steal was 0%. None of the planned levers was ever turned.
The `env_profiles` mechanism was used for a lever nobody had planned (`memory_backing`).

The rest of the epic followed the plan's *method* rather than its *task list*. Every run's data
named the next wall, and each wall became its own issue:

1. **Fixed costs the perf report exposed:**
   - a ~4 min pre-flight SSH timeout (#1212);
   - apt dpkg-lock waits, per-run `apt update`, and pymqi builds (#1225–#1227);
   - box re-adds and duplicate PKI (#1248);
   - cloud-init and snapd on every boot (#1250).
2. **obs-pipeline defects that blocked runs:**
   - the Dashboards API timeout (#1214);
   - a Data Prepper error flood (#1220);
   - an OpenSearch date-mapping rejection silently dropping ~688 docs/min (#1230);
   - their observability follow-ups, the alert (#1238) and DLQ (#1239).
3. **Measurement artifacts:** the sampler itself caused load. Per-tick SSH logins set off a MOTD
   pile-up (#1221), and the fix for that failed on virtiofs (#1228).
4. **The real macOS bottleneck,** found by a microbenchmark spike (#1240) that narrowed
   "everything is slow" down to memory churn in one guest degrading every other guest's page
   faults. The fix was huge pages (#1241), then the environment was auto-detected so it applies
   without configuration (#1245).
5. **The real cloud bottleneck,** found by the first cloud streak (#1247): guest disks on
   standard PD. That needed a **cross-org** change (vergil-vm, vergil-tooling releases) and a GCP
   SSD-quota raise from 500 to 1000 GB.

The acceptance gate was met differently from the plan's literal wording. Cloud passed a literal
5/5 streak (#1267). macOS was **accepted by the maintainer** on three huge-page runs (643/621/587
s), the third of which failed only at an unrelated, since-fixed step (#1261). The reasoning is
recorded on #1200.

The plan has **no "Evolution during execution" log**. This narrative is reconstructed from the
issue record and its comments.

## 2. Lessons learned

- **Instrument before tuning: it paid off immediately.** The spec's leading hypothesis
  (oversubscription) was wrong. Had we tuned levers first, we'd have optimised the wrong thing.
  The perf report also made every later regression legible.
- **"Slow everywhere" can be one shared resource.** The arm64 finding only fell out once tests
  were isolated one variable at a time: idle → CPU burn → memory churn. Sampled steal was 0 the
  whole time, and the cost was invisible to every in-guest metric.
- **The measurement tool can cause the problem it measures.** The sampler's per-tick SSH logins
  triggered `landscape-sysinfo` pile-ups on the very node being measured. Instruments need their
  own overhead budget, plus a check that they work on every platform (#1228's virtiofs failure).
- **Per-arch paths diverge in non-obvious ways.** The snapd purge ran only on x86 obs and,
  through `deb-systemd-helper purge`, deleted empty `/etc/systemd` directories (#1265). macOS
  passed with a box baked at the same commit. Box changes now carry a bake-time guard, and every
  box change needs both arches before acceptance.
- **Run the cross-platform twin early.** Cloud-specific walls (HDD boot disk, the SSD quota) only
  appeared once the cloud ran the same commit. Parallel tickets for the cloud agent worked well.
- **Ask, then verify against the code.** Several premises in issue bodies turned out to be wrong
  when checked:
  - "acl isn't baked": it was; the 77 s was `update_cache` (#1226);
  - "pymqi is offline-installed": it's from PyPI (#1227);
  - "snapd scripts are clean": debhelper injects the purge (#1265).

  Agents reading the code before acting saved rework each time.
- **Folding discovered issues into the epic kept the follow-up list short,** at the cost of a
  broader epic than the plan described. The maintainer chose this deliberately.

## 3. Compromises and tradeoffs

- **macOS acceptance used three runs, not five consecutive.** The maintainer judged the failing
  step (`publish net state`, a git call in a render-only path) to be unrelated to the performance
  being measured, and fixed it under #1261 instead of re-running the streak.
- **The planned staging levers (`boot_batch`, obs vCPU) were never tuned.** The data showed they
  weren't the bottleneck. `env_profiles` exists, but carries only `memory_backing`.
- **The huge-page reservation (~22 GiB for `nativeha-ubuntu`) is taken on demand at bootstrap**
  rather than permanently in the Vergil VM profile. That's cheaper at rest, but the bootstrap needs
  passwordless sudo and can fail loudly if memory is too fragmented to reserve.
- **The cloud SSD boot disk costs more** (PD-SSD vs standard; quota raised to 1000 GB). Accepted
  for performance. The libvirt-pool-on-`/vergil` alternative (#1249's first implementation) was
  built and then discarded to keep #386's rule and the "reprovision everything" principle.
- **#1266 (root-owned `__pycache__` from `become` localhost plays) was deferred to ad-hoc** to
  avoid overlap with epics #280/#288, although it affects both platforms. Its work-in-progress
  stays in a local worktree.
- **The `obs net-state` git failure** was made loud and decoupled (#1261), but its root cause
  (suspected intermittent virtiofs ownership reporting) remains unproven.
- **A data-loss incident:** an agent deleted the shared `build/state/runs` through a worktree
  symlink. September's records were restored from backup; the rest was accepted as purgeable. No
  guard was added, by maintainer decision (run records are purged periodically anyway).

## 4. New problems and opportunities surfaced

| finding | where it went |
|---|---|
| Other stacks (pcmk-ubuntu, rdqm-rhel, nativeha-rhel-crr) and every DR half were never measured; the desk audit found stack-specific gaps | **Epic .github#288** (cross-stack bootstrap parity), from #1264 |
| `vrg-finalize-pr` merged before required checks registered, then hard-failed | vergil-tooling#3061 (fixed, released) |
| `vrg-submit-pr` dropped a post-unfreeze commit | .github#279 (triage) |
| Root-owned `__pycache__` from `become` localhost plays, both platforms | #1266 (ad-hoc, deferred) |
| `fwupd-refresh` is now obs's slowest boot unit; app-client pymqi venv is built per run; Ansible `forks` = 5 | epic #288 tasks #1306, #1305, #1310 |
| obs box at 6.6 GiB dominates first-boot upload after a rebake | logged, not yet acted on |
| `teardown --commons` leaves libvirt networks up (`net` phase skipped) | logged, not yet acted on (harmless) |
| `vrg-validate` doesn't run `mkdocs build --strict`; host prerequisites in README / getting-started (12 vCPU) are stale | logged in #1199's report, not yet acted on |
| Intermittent git "dubious ownership" on the virtiofs checkout | logged; #1261 makes it visible if it recurs |

## 5. What's next

- **Epic logical-minds-foundry/.github#288, cross-stack bootstrap parity:** the forward follow-on.
  Spec, plan and 23 tasks are filed. It starts **after epic #280** (the multi-version OS axis),
  which it builds on.
- **Epic #280** is in progress in parallel and lands first.
- The deferred ad-hoc items above (#1266, .github#279) are picked up opportunistically.

## Appendix A — Operational notes

- **Cross-org change sequence for the SSD boot disk:**
  1. vergil-vm PR → `vrg-release` (v2.1.42);
  2. vergil-tooling PR → `vrg-release` (v2.1.223);
  3. this repo's `vergil.toml` PR;
  4. raise the GCP `SSD-TOTAL-GB-per-project-region` quota (us-east1) from 500 to 1000. The
     first rebuild failed with `Quota 'SSD_TOTAL_GB' exceeded`. The request needs `--email`, and
     the grant came back within minutes;
  5. `vrg-vm rebuild … --name cloud` on the macOS host.
- **Box-content changes need a rebake on both arches before any timed run.** Rebakes always
  happen outside the measured wall-clock (obs takes ~700 s on macOS and ~1240 s on cloud).
- **Measured runs** run detached (`setsid nohup`) so tool time limits can't kill them, with no
  validation containers on the macOS host during a run.

## Appendix B — Extended metrics

- **Microbenchmark, macOS (#1240):**

  | condition | first-touch / re-touch per page |
  |---|---|
  | Vergil VM | 0.78 µs / 0.16 µs |
  | nested guest, lab busy | 735 µs / 95 µs |
  | under another guest's memory churn, 4 KiB backing | 5.7–219 µs / 1.7–153 µs |
  | same churn, 2 MiB huge pages | 0.67–1.33 µs / 0.20–0.23 µs |

  obs's churn throughput went from 9–12 to 1115–1124 cycles per process.
- **Cloud streak #1267 phases** (run 2): `prereq:vms` 6.7 s, `vms` 272.2 s, `prereq:provision`
  57.0 s, `provision` 145.5 s, `observe` 190.2 s. The milestones were opensearch_green 17.6 s,
  data_prepper_ready 7.5 s, dashboards_ready 11.6 s.
