# Guest component packaging (M1) — retrospective

- **Epic:** `logical-minds-foundry/.github#294`
- **Retrospective task:** `logical-minds-foundry/.github#297`
- **Spec / plan:** [`spec.md`](spec.md) (with §0 as-built errata) · [`plan.md`](plan.md)
- **Outcome:** DONE — both guest components run as standalone uv projects on one pinned
  CPython, installed from source and baked into the boxes; validated by cold rebuilds on
  both arches.
- **Date:** 2026-10-12

## §0 At a glance

We set out to turn the lab's guest-side Python (the HA/DR state collectors and the MQ
client tooling, until then copied into guests as loose scripts against the system
Python) into **standalone uv projects under `components/`**. Each is tested on one pinned
CPython 3.14.8 (python-build-standalone), built into a hash-pinned offline artifact, and
installed into its own venv under `/opt/logical-minds-foundry/<name>/`, baked into the
fat boxes. That shipped: `mq-resiliency-observability` and `mq-resiliency-clients` run
from baked components on every stack, the dark collector dashboards came back, and both
cold-rebuild validations passed. This is milestone M1 of three; M2 (Vergil binary
packages) is done and M3 (the lab consumes a published package) is epic
`.github#319`.

**Work delivered (merged PRs):**

| PR | Task | What it did |
|---|---|---|
| .github#299 | #295 | Spec and plan |
| lab#1363 | #1349 (T2) | Spike report: GO — pbs 3.14.8, pymqi from sdist, offline installs |
| lab#1364 | #1350 (T3) | Component boundary, pinned guest runtime, `roles.components` catalog key |
| lab#1365 | #1351 (T4) | `mqlab component build` / `status`: pinned-runtime test gate, staged artifacts |
| lab#1367 | #1352 (T5) | `runtime-install` / `component-install` roles, `mqlab component install`, box-builder wiring |
| lab#1368 | #1353 (T6) | Carve `mq-resiliency-observability` out of `src/mqlab` |
| lab#1370 | #1354 (T7) | Collectors run from the baked observability component |
| lab#1369 | #1355 (T8) | Carve `mq-resiliency-clients` (clients, DR, header) into a component |
| lab#1371 | #1356 (T9) | Clients run from the baked clients component; retire `mqvenv`/`rvenv` |
| lab#1373 | #1372 | Versioned releases with an atomic `venv` symlink flip (non-relocatable venvs) |
| lab#1376 | #1374 | `component status --host` tolerates ansible's timing-callback output |
| lab#1382 | #1377 | DR clients connect with mutual TLS |
| lab#1383 | #1378 | Textfile drop zone stays `02775`; node_exporter's ACL read stays effective |
| lab#1384 | #1379 | Second-stack observe: restore only the snapshot indices not already held |
| lab#1385 | #1380 | `mq-bench` is Native HA only; report the real MQRC |
| lab#1387 | #1348 | Docs review: site, development and reference docs |
| .github#306 | #305 | As-built errata for the spec and plan |

**Operational tasks:** the T2 spike re-run on x86 (#1359, cloud), V1 cold rebuild on arm64
(#1357), V2 cold rebuild on x86 cloud (#1358) — all `Outcome: SUCCESS`, V1 and V2 after
the fixes above.

- **Counts:** 23 child tasks (8 planned implementation tasks, 3 operational, 6 fixes
  found by validation, 1 errata task, 5 bookends); 17 merged PRs.
- **Size:** lab +8,794 / −1,141 lines across 15 PRs; `.github` +1,848 / −2 across 2.
- **Repos touched:** `mq-resiliency-lab-for-linux`, `.github`.
- **Releases:** none (the lab is consumed from `develop`).
- **Span:** opened 2026-10-05; implementation, validation and docs review complete
  2026-10-07; the follow-on brainstorm closed 2026-10-11 (promoted to `.github#319`).

## §1 How the plan evolved

The plan held its shape: T2–T9 landed in order inside two days, and the T6–T9 chain
landed without develop ever being left half-switched for long. What moved was the
**install mechanics**, and almost all of it was found by the cold rebuilds rather than by
the pipeline:

- **Venvs are not relocatable** (E2). The plan built `venv.new` and renamed it into
  place; pip writes absolute shebangs, so every entry point broke (203/EXEC) on V1's
  first run. The fix — build each install at its final path under
  `releases/<version>+<tree>-<stamp>/`, then flip a `venv` symlink atomically, with a
  selfcheck before and after the flip and an automatic flip back — turned out to be a
  better design than the original: installs became reversible.
- **"RHEL guests are offline" was false** (E1). The guarantee M1 actually makes is that
  installs never consult an index, not that the network is absent.
- **Toolchain and filesystem surprises** (E3, E4): python-build-standalone's sysconfig
  names `clang` (the install swaps in `gcc` with the same hardening flags), and the
  macOS virtiofs mount under `build/` is case-insensitive, so the interpreter unpacks
  under `$XDG_CACHE_HOME`.
- **Secondary-stack scope was narrowed on purpose** (E5): `mq-bench` is Native HA only,
  and the DR criterion became "the DR entry points complete a TLS MQCONNX", with the
  full `dr-run.sh` deferred (lab#1375). The human's rule, recorded during this epic:
  Native HA is primary; RDQM and Pacemaker failures are narrowed or deferred, never
  allowed to block it.
- **Old defects surfaced by the new dashboards** (#1378, #1379): with the collectors
  emitting again, two pre-existing observability bugs (drop-zone permissions reset by
  four roles; a snapshot-restore collision on a second stack) became visible and were
  fixed narrowly.

The spec's §0 errata table (.github#305) is the authoritative list of what changed.

## §2 Lessons learned

- **Spike first, with a go/no-go, paid for itself.** T2 settled the runtime, pymqi-from-
  sdist and the offline install before any carve-out started; nothing in T3–T9 had to
  revisit those choices. Re-running the spike on x86 in the cloud (#1359) closed the one
  arch gap the local box could not.
- **Cold rebuilds are the real acceptance gate.** Every substantive defect (E2, the TLS
  DR clients, the drop-zone ACL, the snapshot collision) was invisible to a green
  `vrg-validate` and obvious on the first cold rebuild. Lint-green is not done.
- **Parallel PRs need a simulated merge, not just green CI.** Independent branches that
  were each green conflicted semantically once merged (tests asserting each other's old
  behavior). Cherry-picking onto a scratch copy of develop before `report-ready` caught
  these cheaply; doing it routinely would have saved two `vrg-submit-pr` failures.
- **Cloud validation agents work.** Handing V2 and the x86 spike to a cloud agent via
  `issue-validate` ran the expensive x86 rows without the human in the loop, and its
  findings came back as clean, separately filed fix tasks (#1377–#1380).
- **Destructive commands need guard rails, always.** On 2026-10-07 a chained command
  continued after a failed `cd` into a worktree (the worktree add had been refused by
  the naming rule) and removed the main checkout's `build/` (cache, state and work).
  It was restored from the previous night's backup with no lasting loss. Since then:
  `set -euo pipefail` or `&&` only, absolute verified paths, no relative `rm -rf`,
  cleanup functions that refuse to run outside their worktree, and worktree names that
  match the branch.

## §3 Compromises & tradeoffs

- **Two milestones of throwaway-ish install machinery.** M1 built its own runtime
  staging (`runtime-install`, sha256-pinned tarballs) knowing M2/M3 would replace it with
  published packages. Accepted to get components real and dashboards back now; M3
  (`.github#319`) retires it.
- **Secondary stacks are best effort.** `mq-bench` on RDQM/Pacemaker and an end-to-end
  `dr-run.sh` were deferred (lab#1375, still open) rather than fixed, by decision.
- **pymqi stays, built from sdist on each guest.** Building our own wheels was considered
  (.github#298) and superseded by moving to IBM's `ibmmq` wheels (epic `.github#300`).
- **The in-lab install keeps a `releases/` directory and two selfchecks** — more moving
  parts than a rename, in exchange for reversibility.

## §4 New problems & opportunities

| Surfaced | Where it went |
|---|---|
| `dr-run.sh` is not operational (no lab Vagrant env, exits 0 on failure, no DR queues) | lab#1375 (open) |
| RDQM/Pacemaker's strategic future and Red Hat support | `.github#309` (closed: kept while IBM supports them) |
| Own binary wheels for pymqi | `.github#298` → superseded by epic `.github#300` (pymqi → `ibmmq`) |
| Consume a published package instead of building in-lab | `.github#296` → epic `.github#319` (M3) |
| A pre-release package channel for release-to-test | `vergil-project/.github#370` (deferred) |

## §5 What's next

- **M3 — the lab consumes published component packages:** epic `.github#319`. It
  extracts `mq-resiliency-observability` into its product repo, publishes it from a new
  logical-minds-foundry package repository, moves the in-lab tier onto the
  `vergil-python3.14.8` package, and retires `runtime-install`.
- **pymqi → `ibmmq`:** epic `.github#300`.
