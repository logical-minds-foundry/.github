# Minimal, reliable OpenSearch log-search tier — retrospective

## §0 At a glance

**Set out to do:** the `logsearch` tier ran the full OpenSearch 3.8.0 bundle
(~15 plugins) that churned 12+ minutes on 2 vCPUs and never bound `:9200`, blocking
all observability. Switch OpenSearch to the **min (core-only) distribution** so the
tier cold-boots reliably and fast — and *prove it with a repeated cold boot*.

**What shipped:** the min distribution is live; a cold
`mqlab bootstrap nativeha-ubuntu --no-dr` comes up one-pass with OpenSearch green on
`:9200`, Data Prepper on `:21892`, logs flowing end-to-end and searchable in
OpenSearch **and** a new pinned Grafana datasource, obs disk flat. The core switch
was easy; the epic's real cost was unearthing a stack of **latent, fleet-wide
reliability bugs that only a live cold boot exposed** — exactly what the "repeated
cold boot" acceptance was designed to catch.

Work delivered — 13 PRs; repos `mq-resiliency-lab-for-linux` and
`logical-minds-foundry/.github`; span 2026-08-12 → 2026-09-14:

| PR | Date | What it did |
|---|---|---|
| `.github#201` | 08-12 | spec + plan (#199) |
| #1026 | 08-12 | OpenSearch → **min** distro; drop min-invalid `plugins.security.disabled` (#1023) |
| #1027 | 08-12 | quote Data Prepper systemd `Environment=` so `-Xmx` survives (#1024) |
| #1028 | 08-12 | pinned Grafana **OpenSearch datasource**, plaintext `:9200` (#1025) |
| #1030 | 08-13 | Alloy: drop log-shippers' own journal entries — break the queue-full loop (#1029) |
| #1036 | 08-13 | OpenSearch cold-boot: cut JVM-bootstrap cost + widen readiness to 15m (#1034) |
| #1039 | 08-13 | restore the required `-javaagent` (#1034's removal crash-looped core) (#1038) |
| #1041 | 08-13 | widen Data Prepper + Dashboards readiness budgets to 15m (#1040) |
| #1064 | 09-09 | complete the **Python 3.14 migration** on the runtime/dev side (ad-hoc #1063) |
| #1066 | 09-10 | build `mq_prometheus` in the **Go container**, ship the artifact (ad-hoc #1065) |
| #1077 | 09-14 | **retry** the grafana port-forward verify — kill the relay-restart race (ad-hoc #1067) |
| #1078 | 09-14 | **River raw string** for the Alloy journal-drop regex — the log-shipping crash (#1068) |
| #1080 | 09-14 | docs-review sweep (#1021) |

Also: **#1022** validation closed SUCCESS (no PR — a live cold-boot check). Tasks
closed: 12 plus this retrospective. Releases cut: 0. One follow-up filed and open:
**#1079**.

## §1 How the plan evolved

The plan was small and correct on its face: four tasks — swap to min, fix the Data
Prepper `Environment=` quoting, create the Grafana datasource, then a cold-rebuild
validation — plus the note that the box rebake was automatic. Tasks 1–3 landed in a
single day (08-12) exactly as written.

The **delta was entirely on the validation axis, and it was large.** The plan
treated `#1022` as a confirmation step; it turned out to be the epic. The first cold
boots surfaced a cascade the plan never anticipated, each fixed in turn:

1. **OpenSearch min crash-loop** — stripping the bundle also stripped the
   `-javaagent` the core needs (`AgentPolicy NoClassDefFoundError`); restored it
   (#1038/#1039), and widened the still-too-tight readiness budgets (#1034/#1040)
   once real cold-boot timings were measured.
2. **A stale dev-VM toolchain** — the Python 3.14 migration had moved CI +
   `pyproject` but not the runtime venv (created on 3.12), so `mqlab` wouldn't even
   import; completing the migration properly (#1063/#1064) was a prerequisite to
   running the validation at all.
3. **obs-box bake ENOSPC** — the `mq_prometheus` cgo build auto-downloaded the whole
   Go toolchain into the fatbox build guest and overflowed it; moved the build into
   the Go container and shipped the artifact instead (#1065/#1066).
4. **Alloy dead fleet-wide** — the drop-rule added *in this epic* (#1029) used `\.`
   in a double-quoted River string, an invalid escape, so Alloy rejected its config
   and crash-looped everywhere — no logs to Loki or OpenSearch. A backtick raw string
   fixed it (#1068/#1078).
5. **A one-pass verify race** — the observe phase curled Grafana immediately after
   restarting its port-forward relay, racing the re-bind (exit 7); added
   `--retry-connrefused` (#1067/#1077).

The lesson the delta encodes: **the "repeat the cold boot" criterion was
vindicated.** Every one of these was a latent, reproducible failure that unit/render
tests were structurally blind to; only booting the whole tier from cold surfaced
them.

## §2 Lessons learned

- **Render/unit tests can't see runtime config-load failures.** Two of the worst
  bugs — the Alloy River-escape (#1068) and the min-invalid `plugins.security`
  setting (#1023) — rendered fine and passed `vrg-validate`, then killed the service
  at load time. Validate rendered configs against the actual engine (e.g. an
  `alloy fmt` / `opensearch`-parse check at CI), not just the template. The new
  import/escape guards added this cycle are a start.
- **A toolchain bump must move the runtime, not just CI + `pyproject`.** The
  half-applied 3.14 migration silently broke every existing dev VM. Migrations should
  update the venv/VM profile in the same stroke, or fail loudly on mismatch.
- **Acceptance criteria can pass for the wrong reason.** "obs disk flat / no
  sending-queue-full loop" was green while Alloy was *dead* — no loop because nothing
  ran. A criterion satisfiable by absence needs a positive companion (fresh docs
  actually landing).
- **A live cold-boot gate is worth its cost** on infra epics; it caught five real
  bugs a Gantt chart would have marked "done" a month earlier.

## §3 Compromises & tradeoffs

- **Relaxed the repeated cold boot (×2 → ×1).** The acceptance owner accepted one
  clean full boot in lieu of two, given the reliability bugs were root-caused and
  fixed *deterministically* (not heisenbugs) and a downstream DR epic was
  time-blocked. Recorded on #1022 with rationale; the DR bring-up doubles as a second
  full bring-up.
- **`mq_prometheus` still links MQ server bindings** (`libmqm_r`) — `mq-metric-samples`
  has no client-only build in this version — so the obs box keeps the full MQ
  runtime. Slimming obs to the MQ redistributable client was deferred, not attempted.
- **Box-hash over-sensitivity accepted for now.** Editing the Alloy role's *per-run*
  config template invalidated every Ubuntu fatbox's manifest-hash, forcing full
  rebakes on a change that isn't even baked in. Lived with; logged below.

## §4 New problems & opportunities surfaced

- **Data Prepper `parse_json` errors on every plain-text journal line** — noisy,
  non-fatal. → **Filed: #1079** (gate with `parse_when`).
- **Box manifest-hash is over-sensitive** — rebakes on a per-run config edit (§3).
  → Logged here; not yet filed.
- **CI-vs-runtime toolchain drift is a systemic risk** — the 3.14 gap will recur on
  the next bump. A guard (fail loud when the venv/VM interpreter ≠ declared) would
  prevent it. → Logged here; not yet filed.
- **Cold-boot staleness NOTICE false-positives** — `mqlab doctor` reported the VM
  "59 days old" from a write-once stamp on the *persistent* disk that survives
  `vrg-vm rebuild`, so it can never clear. → Logged here; not yet filed.

## §5 What's next

- **Unblocks #169 Wave 1b** — the Grafana logs/events dashboards now have a working
  OpenSearch datasource and a reliable tier to build on.
- **DR / Native HA on RHEL** is a separate epic (different substrate; this epic
  proved the Ubuntu HA side). Not gated on anything here.
- Open follow-up: **#1079**; the three logged-not-filed items in §4 if the team wants
  them tracked.
