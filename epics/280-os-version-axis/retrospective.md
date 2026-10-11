# Retrospective — Epic 280: multi-version OS axis

## 0. At a glance

**Set out to do:** go from one OS version per family to many. The lab should build any
MQ HA/DR stack on a chosen OS major (`nativeha-ubuntu` and `pcmk-ubuntu` on Ubuntu 24 or
26, `nativeha-rhel-crr` on RHEL 9 or 10, `rdqm-rhel` on RHEL 9). The newest version
should be the default where IBM supports it, the shared nodes should move to Ubuntu 26,
and stack identity should stay version-free. A catalog plus a resolver would replace
hard-coded version strings. The MQ version axis was left as a follow-on.

**What shipped:** all three phases. Every stack × supported OS row passed a cold
full-DR rebuild in one pass on MQ 10.0.0.5, with `qm e2e` and a human dashboard review,
on both hosts: macOS arm64 for the Ubuntu rows and cloud x86_64 for the RHEL rows.

- **Machinery:**
  - `lab/versions.yaml` is the catalog, with `src/mqlab/versions.py` as its resolver.
  - `mqlab bootstrap <stack> --config <file>` build files select a non-default OS.
  - Per-stack instance records hold a stack's resolved OS, and refusal rules reject unsupported, cross-family or mismatched selections.
  - Box names follow `<role>-<os><major>`.
  - A fail-loud per-OS vars include feeds Ansible.
  - A guardrail test keeps OS version tokens out of everything except the catalog and resolved box names.
- **Defaults:** `nativeha-rhel-crr` now defaults to **RHEL 10**. The Ubuntu stacks stay on **Ubuntu 24**, because IBM's MQ support matrix (SPCR) does not yet list Ubuntu 26.04. Ubuntu 26 ships as a lab-only selectable version with a warning. The default flip and the move of the MQ-bearing shared nodes are gated on IBM and carried into the follow-on epic (§5). `rdqm-rhel` stays RHEL 9 only; RDQM on RHEL 10 arrives with MQ 10.0.1.
- **Shared nodes:** split by MQ. Nodes without MQ (infra, obs, SANs) run Ubuntu 26. The MQ commons (svc-sim, app-client, mon-probe) stay on Ubuntu 24 under the same IBM gate.

| PR | Task | Merged | What it did |
|---|---|---|---|
| .github#284 | .github#281 | 2026-10-02 | Spec and plan for the epic |
| .github#287 | .github#285 | 2026-10-03 | Spec/plan correction: split the shared nodes by MQ (`infra` vs `infra_mq`); gate the Ubuntu 26 default on an SPCR row |
| #1294 | #1271 | 2026-10-03 | OS/MQ support matrix research for Ubuntu 26.04 and RHEL 10 (the gate's evidence) |
| #1297 | #1272 | 2026-10-03 | Spike report: an Ubuntu 26.04 libvirt base box and x86-64-v3 guest exposure |
| #1293 | #1273 | 2026-10-03 | The OS version catalog and a pure version resolver |
| #1298 | #1274 | 2026-10-03 | Box builders take catalog inputs; `<role>-<os><major>` box names; the fleet comes from the catalog |
| #1323 | #1275 | 2026-10-03 | Topology names box roles; the resolved render goes through the version layer |
| #1325 | #1276 | 2026-10-03 | `bootstrap --config` build files, per-stack instance records, refusal rules |
| #1322 | #1277 | 2026-10-03 | Per-OS-version Ansible vars through a fail-loud include |
| #1326 | #1278 | 2026-10-03 | Baked the SAN targets into a `san` box and retired the SAN deb cache |
| #1327 | #1324 | 2026-10-03 | The box manifest hash now digests shared `ansible/` includes such as `os-vars.yml` |
| #1328 | #1279 | 2026-10-04 | Version-token guardrail test |
| #1330 | #1329 | 2026-10-04 | The guardrail scans only git-tracked files (it had tripped on `lab/.vagrant`) |
| #1334 | #1295 | 2026-10-04 | The IBM doc cache uses the shared cache bucket and per-version dirs; `build migrate` fails cleanly |
| #1335 | #1333 | 2026-10-04 | Restored a lost exec bit on `await-install.sh`, which had failed every RHEL base-box bake, and guarded it in the index |
| #1339 | #1336 | 2026-10-04 | Base-image GC tolerates a volume that disappears between list and dumpxml |
| #1341 | #1340 | 2026-10-04 | Build disks only grow; RHEL fat-box bakes had been shrinking the 20G base to 18G |
| #1345 | #1344 | 2026-10-04 | The event monitor resolves `qm.ini` from `dspmqinf`'s DataPath instead of a hard-coded path |
| #1347 | #1346 | 2026-10-04 | `.box` files are packaged as plain tar, dropping a redundant gzip pass |
| #1390 | #1342 | 2026-10-08 | Replaced the broken `e2e-test.sh` with `mqlab qm e2e <stack>` |
| #1393 | #1391 | 2026-10-08 | The cross-family OS refusal names the stack's supported versions |
| #1394 | #1392 | 2026-10-08 | `qm status` reports site B's Recovery group on a Native HA CRR stack |
| #1398 | #1395 | 2026-10-08 | Push and stage every catalog RHEL DVD (repeatable `--iso` and `--catalog`) |
| #1397 | #1286 | 2026-10-08 | RHEL 10: catalog entry, base box, vars, x86-64-v3 host gate |
| #1402 | #1282 | 2026-10-09 | Ubuntu 26: catalog entry, role fix-ups, infra and shared nodes on 26 |
| #1406 | #1403 | 2026-10-09 | RHEL bakes install `acl` only after the offline DVD repo exists (this had broken the RHEL 10 bake) |
| #1411 | #1407 | 2026-10-09 | Pinned MQ to 10.0.0.5, with media built and staged strictly from the pin |
| #1412 | #1408 | 2026-10-09 | RDQM picks its DRBD kmod with IBM's `modver` (kernel-family match), not an exact-release glob |
| #1416 | #1414 | 2026-10-09 | `motd-off` finds every `pam_motd` PAM file; Ubuntu 26.04's `/etc/pam.d/remote` had failed every `*-ubuntu26` bake |
| #1418 | #1417 | 2026-10-09 | Large artifacts are staged on disk under `lab_stage_dir`, never in `/tmp`, which is a RAM-sized tmpfs on Ubuntu 26.04 |
| #1421 | #1420 | 2026-10-10 | Every mqlab subprocess gets its own stdin, fixing a race that made `ansible-playbook` refuse to run |
| #1435 | #1289 | 2026-10-10 | `nativeha-rhel-crr` defaults to RHEL 10 (T11) |
| #1437 | #1399 | 2026-10-11 | `box gc --unreferenced` reclaims boxes that no catalog selection resolves to |
| #1438 | #1405 | 2026-10-11 | Support matrix: RDQM on RHEL 10 is supported from MQ 10.0.1, not in any 10.0.0.x |
| #1439 | #1436 | 2026-10-11 | Research write-up on the RDQM/Pacemaker strategic position, recording the keep-as-is decision (for .github#309) |
| #1445 | #1263 | 2026-10-11 | Docs review: the OS axis reflected across the site, dev and reference docs |

- **Repos touched:** `mq-resiliency-lab-for-linux`, `.github`.
- **Tasks:** 52 children:
  - 36 delivered by PR (+16,947 / −3,801 lines);
  - 13 operational tasks closed on recorded evidence (6 deployment: D1 ×3, D2/D3 via #1413, #1283, #1287; 7 validation: two spike-data runs, V1 ×3, V2, V3);
  - .github#309, closed by decision with its write-up delivered through #1436;
  - #1396 (RHEL 9.8 re-pin), closed as not planned;
  - this retrospective.
- **Moved out:** to the follow-on epic .github#315: #1285 (T9), #1409, #1410 and the follow-on brainstorm .github#282.
- **Releases cut:** none.
- **Span:** opened 2026-10-02, closed 2026-10-11 (9 days).
- **Related, not counted:** guest component packaging (.github#294) ran alongside as its own epic. Two ad-hoc fixes found during this epic's bake rounds landed outside it: #1404 (a failed bake left its build domain running) and #1415 (`box gc` on a host with no `default` pool).

## 1. How the plan evolved

`plan.md` has no "Evolution during execution" log. This narrative is reconstructed from
the task and validation record.

**Phase 0 and Phase 1 went almost exactly to plan.**

- The spikes (#1271, #1272 with per-host data #1290/#1291) and the whole resolver refactor (#1273–#1279) merged within two days.
- One early correction changed the design (.github#285, day two). IBM does not list Ubuntu 26.04, so the shared nodes were split. Nodes without MQ (`infra`) moved to 26 unconditionally. MQ-bearing nodes (`infra_mq`) stay on IBM-listed Ubuntu under the same gate as the stack defaults.
- The first bake round (D1) then surfaced a run of latent box-builder defects unrelated to OS versions: a lost exec bit (#1333), a disk shrink (#1340), a GC race (#1336), and a manifest hash that missed a shared include (#1324). None would have been caught by unit tests.

**V1 was the first real delay** (D1 closed 2026-10-04, V1 2026-10-08).

- Cold rebuilds exposed three things:
  - stale tooling the epic had to fix to validate at all: the broken `e2e-test.sh`, replaced by `mqlab qm e2e` (#1342);
  - a CRR blind spot: `qm status` couldn't see site B (#1392);
  - a hard-coded `qm.ini` path (#1344).
- They also exposed a guest Python runtime mismatch, resolved by the separate component-packaging epic (.github#294) before Phase 1 could be signed off.

**Phases 2 and 3 were reshaped by three decisions the plan didn't anticipate:**

1. **Pin MQ before baking (#1407).** The fleet was about to be rebaked for Ubuntu 26 and RHEL 10 anyway, so the MQ pin moved from 10.0.0.0 to 10.0.0.5 (IBM declares it both CD and LTS) first, so the fleet was baked once. One combined cloud bake round (#1413) then covered D2 and D3.
2. **RHEL 9 stays at 9.6 (#1396, not planned).** Re-pinning to 9.8 appeared impossible because "IBM ships no kmod for the 9.8 GA kernel". Research for #1405 then showed the blocker was the **lab's own** exact-release glob: IBM's documented `modver` picker matches by kernel family. #1408 switched to `modver`. Staying on 9.6 remains a decision, now revisitable.
3. **RDQM on RHEL 10 arrived from IBM mid-epic.** IBM announced it for MQ 10.0.1 (AD26-0859, GA 2026-10-13). That reversed the premise of the strategic-research task .github#309, which had framed RDQM's RHEL lag as a sign of deprecation. The human's direction became: Native HA is primary, and RDQM and Pacemaker stay as completeness counter-options while IBM supports them.

**The Ubuntu 26 bakes found real OS differences.**

- 26.04's new `/etc/pam.d/remote` loads `pam_motd` (#1414).
- 26.04's `/tmp` is a RAM-sized tmpfs (#1417). The human directed "fail forward": keep tmpfs and stop staging large files in `/tmp`.
- The RHEL bake ordering defect (#1403) blocked the first RHEL 10 bake.

**Validation found one race in mqlab itself.** V2 row 3 lost a race between the perf sampler's `ssh` and `ansible-playbook` over a shared stdin (#1420). It was fixed and the row re-run cold. V3 passed all three RHEL rows in one pass. T11 then flipped the RHEL default.

**The close was restructured on 2026-10-11.** The remaining work all waited on outside events: MQ 10.0.1 GA, IBM listing Ubuntu 26.04, and the MQ-axis brainstorm. The human chose to move it into a follow-on epic (.github#315) so #280 could close on what it shipped. They also chose to skip a final full-fleet rebuild: every row had been cold-rebuilt within two days, and #315's 10.0.1 matrix revalidates everything anyway.

## 2. Lessons learned

- **The cold-rebuild gate earned its keep again.**
  - Every bake and validation round found something unit tests could not: an exec bit, a disk shrink, PAM and tmpfs differences on a new OS, a package-ordering defect, a stdin race.
  - Planning separate operational tasks per host (D/V × macOS/cloud) made each failure attributable and each fix re-verifiable.
  - **Same next time.**
- **Check vendor "facts" at the source before planning around them.** "No kmod for RHEL 9.8" and "RDQM is being wound down" both turned out wrong: one was our own glob, the other was contradicted by IBM's own docs and its RHEL 10 announcement. Each cost a decision or a research task. Labelling data vs judgment in the research docs (#1271, #1405, #1436) is what exposed the errors.
- **Gate defaults, not support.** Shipping Ubuntu 26 as selectable but lab-only, with a cited warning, let the lab adopt the newest LTS without claiming IBM support it doesn't have. The same rule carried naturally to `infra_mq`.
- **Pin moving dependencies before a bake round.** Moving to MQ 10.0.0.5 before the Ubuntu 26 / RHEL 10 bakes meant the fleet was baked once, not twice.
- **Fail forward on new-OS behaviour.** Adapting the lab to 26.04's tmpfs `/tmp`, rather than disabling it, left a cleaner and more general design (`lab_stage_dir`, a pinned `remote_tmp`, a guard test).
- **Different next time:** keep a plan "Evolution during execution" log as decisions happen. This narrative had to be rebuilt from issues.

## 3. Compromises & tradeoffs

- **The Ubuntu defaults stay on 24.** Ubuntu 26 is lab-only, and the MQ commons stay on 24, so the lab runs a deliberate mixed-version layout (Ubuntu 26 infra/obs/SANs next to Ubuntu 24 MQ nodes). V2 proved that layout, including Ubuntu 24 pcmk nodes over Ubuntu 26 SANs (the in-kernel DRBD 8.4.11). The flip waits on IBM (§5).
- **RHEL 9 stays at 9.6.** Its GA kernel (570.12.1) is in IBM's RDQM known-issue range, so the `mq-rdqm-rhel9` bake emits a non-fatal WARNING by design. A re-pin is possible now that `modver` is in place, but was not taken.
- **The epic closes on MQ 10.0.0.5, not the latest MQ.** The human's bar ("not public until it runs on the latest MQ") now lives in .github#315 (#1409, #1442, #1443), not here.
- **Validation corners, each a recorded judgment:**
  - V2 rows 1–2 ran before the #1420 fix and weren't re-run; the fix only changes how child processes get their stdin.
  - V3 ran before #1420 and didn't hit the race.
  - The RHEL 10 *default path* was proven by unit tests and by V3's `--config` row, not by a default-path cold rebuild.
  - No final full-fleet rebuild was done after the closing tasks (#1399, #1405, #1436, #1263), which don't change bring-up.
- **The 10.0.1 announcement is cited from pasted text.** IBM's announcement page refuses automated fetches, so it is flagged for a human to verify in a browser.

## 4. New problems & opportunities

| Surfaced | Where it went |
|---|---|
| RDQM on RHEL 10 (MQ 10.0.1) | #1440 under .github#315 (conditional on el10 kmods matching the RHEL 10.2 kernel) |
| MQ 10.0.1 adoption, including the most secure TLS everywhere and a post-quantum key exchange demo | #1410 under .github#315; whether the hardening becomes its own epic is decided by that evaluation |
| Re-pin, rebake and revalidate on 10.0.1 | #1409 → #1442 (bake) → #1443 (matrix) under .github#315 |
| The Ubuntu 26 defaults and the `infra_mq` move, gated per stack on IBM's SPCR | #1285 under .github#315 |
| Superseded `*-ubuntu24` shared-node boxes piling up | Fixed: #1399 (`box gc --unreferenced`) |
| A failed bake leaving its build domain running; `box gc` on a fresh host | Fixed ad hoc: #1404, #1415 |
| RDQM/Pacemaker strategic future | Decided (keep while IBM supports them); written up in #1436, closing .github#309 |
| A one-off "dubious ownership" git error on the virtiofs-shared repo, right after a teardown (V2 row 3) | Logged, not acted on; file it as a defect if it recurs |
| Ansible noise in bootstrap logs (the `INJECT_FACTS_AS_VARS` deprecation, a collections-path warning) and vagrant-libvirt's `libvirt_ip_command` warning | Logged, not acted on |
| IBM announcement pages can't be fetched by tools | Documented in CLAUDE.md (#1263); cite as human-verify |

## 5. What's next

- **.github#315 — Adopt IBM MQ 10.0.1 and the support-gated Ubuntu 26 default.** It carries the work this epic moved out: #1409, #1440, #1442, #1443, #1410, #1285, and the follow-on brainstorm **.github#282** (a selectable MQ major version axis, applying this epic's lessons). Its 10.0.1 validation matrix is the lab's public-release gate.
- **.github#309's outcome** (keep RDQM and Pacemaker as completeness counter-options while IBM supports them; Native HA is primary) is recorded in the lab's `docs/reference/rdqm-pacemaker-strategic-position.md`, and stated on the site's architecture page.

## Appendix A — Operational notes

- **Host split.** The RHEL arms need x86_64, and RHEL 10 needs an x86-64-v3 CPU, so RHEL bakes and rows ran on the cloud x86_64 VM. The Ubuntu rows ran on macOS arm64, with the cloud also baking its x86 Ubuntu fleet. The cloud half was handed to a cloud agent as `issue-deploy` / `issue-validate` tasks.
- **Per-row validation procedure** (V1–V3):
  1. a refusal check before any lab I/O;
  2. a detached cold full-DR bootstrap with stdin from `/dev/null` and a log-mtime stall watch;
  3. the instance record;
  4. `mqlab status`;
  5. two-site `qm status`;
  6. `qm e2e --count 20`;
  7. Prometheus exporter and collector metrics, then a human dashboard attestation;
  8. teardown (`--commons` after the last row).
- **Gotchas:**
  - Long bakes outlive a 2-hour background-command limit; re-run, since finished boxes REUSE.
  - ansible-core refuses non-blocking stdin; give background jobs stdin from `/dev/null`, and see #1420 for the in-tool fix.
  - Leftover `fatbox-*-build` domains after a failed bake were cleaned by hand until #1404.
  - The RHEL DVDs are pushed to the cloud host by the human with `scripts/push-rhel-iso.sh --catalog`.
