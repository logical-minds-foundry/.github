# HA-only (no-DR) bootstrap — design spec

- **Epic:** `logical-minds-foundry/.github#188`
- **Documentation task:** `logical-minds-foundry/.github#189` (this spec + the plan)
- **Documentation-review (closing bookend):** `logical-minds-foundry/mq-resiliency-lab-for-linux#993`
- **Retrospective (terminal bookend):** `logical-minds-foundry/.github#190`
- **Companion (separate):** `mq-resiliency-lab-for-linux#991` — double the base VM's vCPUs
  (12→24), the *bigger-footprint* half of the same performance strategy. **Not** part of this
  epic.
- **Status:** design approved (brainstorm complete); spec under review
- **Date:** 2026-08-11

## 1. Problem & motivation

Under a full lab load — the shared commons plus a Native HA stack, ~13 nested libvirt guests —
the base Lima VM runs at a **sustained ~65–71% CPU steal**, with a runqueue roughly twice the
core count and zero idle. Memory is not the constraint (tens of GiB free, no swap). The CPU
starvation is severe enough that IBM MQ **Native HA cannot hold quorum**: the group thrashes
`Active`↔`Replica`, the client-mode exporter never sees a stable active QM, and the dashboards
stay empty. (The slow-log-I/O warning `AMQ6729W` seen during this is CPU-starvation latency,
not disk.) This blocks essentially all lab development that depends on a live QM.

Most day-to-day work does **not** need the DR site. The DR (site-B) guests are needed only when
actively exercising DR failover behaviour; the rest of the time they are pure replication
overhead. A `nativeha-ubuntu` stack brings up **six** QM guests — site-A HA (`a1/a2/a3`) plus
site-B DR (`b1/b2/b3`) — and each runs a Native HA replication workload. Removing the DR site
halves that load and lets the primary stabilise.

This is the **lighter-footprint** half of a two-part performance strategy; the
**bigger-footprint** half (more base-VM CPU, `#991`) is separate and already report-ready.

## 2. Goals & non-goals

**Goals**

- A way to bring up **only the HA site** of a stack, skipping the DR site, to run a lighter
  footprint under load.
- A general mechanism, wired first for the stack under load (`nativeha-ubuntu`), extensible to
  the other HADR stacks in follow-on waves.
- Fail loud when asked to skip DR on a stack that has no DR site declared.

**Non-goals**

- **No remembered "HA-only" lab state.** `status`/`teardown`/`dr` are unchanged; a skipped
  site-B simply reads as "not created" (the existing state for a topology-declared but absent
  guest). Adding DR later is re-running `bootstrap` without the flag.
- **No net-phase change.** libvirt networks are CPU-free; there is no footprint reason to skip
  any, and doing so would add brittle per-site network logic for no gain.
- **The base-VM vCPU increase (`#991`)** — a separate change.
- **DR cutover/failback semantics** are untouched (they already require site-B up and fail loud
  without it).

## 3. Approach

A **stateless, flag-based** bring-up path.

### 3.1 Invocation

```
mqlab bootstrap <stack> --no-dr
```

A boolean **flag on the existing `bootstrap`** — not a new subcommand — so it reuses the whole
`net → vms → provision → observe` orchestration and composes with `--from`/`--only`. It is
**stateless**: the flag controls only what *that* bring-up starts; nothing is persisted.

Rationale for a flag over a subcommand: the bring-up is identical to a full bootstrap except
for (a) excluding the site-B guests from the guest-enumerating phases and (b) gating the DR
touch-points in provisioning. A subcommand would force either duplication of the phase
orchestration or a refactor to share it; a flag threads a single boolean through the existing
path.

### 3.2 Identifying the DR site — the `dr_groups` marker

Each stack block in `lab/topology.yaml` declares its DR (site-B) Ansible groups explicitly:

```yaml
stacks:
  nativeha-ubuntu:
    groups: [nha_ubuntu_a, nha_ubuntu_b]
    dr_groups: [nha_ubuntu_b]        # NEW
```

mirrored as a new field on the `Stack` dataclass:

```python
dr_groups: list[str]
```

The **presence of a non-empty `dr_groups`** is the signal that a stack supports `--no-dr`. This
is an explicit marker rather than an `_a`/`_b`-suffix heuristic: the suffix convention would
conflate a SAN target group (`san_b`) with a QM DR site and would silently break on any stack
that doesn't follow it. Explicit is per-stack-correct and self-documenting.

The **effective bring-up members** when `--no-dr` is set are the stack's members minus the
hosts contributed by its `dr_groups`:

```
effective_members = stack_members(stack) - hosts_of(dr_groups)
```

A small helper computes this; the guest-enumerating phases consume it.

### 3.3 Phase behaviour

| Phase | With `--no-dr` |
|---|---|
| `net` | **unchanged** — all lab networks up (CPU-free) |
| `vms` | bring up **effective (site-A) members only** → 6 QM guests become 3 |
| `provision` | pass `-e dr_enabled=false` to the stack's provision playbook |
| `observe` | instrument **effective members only** |

### 3.4 Provision gating — `dr_enabled`

The stack's provision playbook gates its DR touch-points on `dr_enabled | default(true)`. The
default preserves today's full-HADR behaviour exactly; only an explicit `dr_enabled=false`
(passed by `--no-dr`) changes anything.

For `nativeha-ubuntu` (`ansible/site-nativeha-ubuntu.yml`) — whose provision already cleanly
separates its HA half (`_nativeha-ubuntu-cluster-ha.yml`) from its DR half
(`_nativeha-ubuntu-dr-replication.yml`) — there are exactly **two** gates:

1. The **authz service-accounts play** targets `nha_ubuntu_a:nha_ubuntu_b`; drop `:nha_ubuntu_b`
   when DR is off, so it never tries to reach the absent site-B hosts.
2. The **`import_playbook: _nativeha-ubuntu-dr-replication.yml`** gains `when: dr_enabled` (an
   `import_playbook` `when` applies to all plays in the imported file).

The preflight probes only `nha_ubuntu_a[0]` (site-A), so it is untouched.

### 3.5 Fail-loud guard

`--no-dr` against a stack whose `dr_groups` is empty raises a clear error
(`"<stack> declares no DR site to skip"`) rather than passing `dr_enabled=false` to a playbook
that does not honour it — which would bring up site-A guests but then fail provisioning against
an absent site-B. The `dr_groups` marker and the playbook gate are declared together per stack;
their joint presence is what makes a stack `--no-dr`-capable.

## 4. Scope & waves

- **Wave 1 — the general mechanism, wired for `nativeha-ubuntu`** (the stack under load). The
  flag, the `dr_groups` field + topology marker, the effective-member helper, the `dr_enabled`
  provision variable, and the two `site-nativeha-ubuntu.yml` gates. Fully unit-tested.
- **Wave 2+ — the remaining HADR stacks**, one task each: `pcmk-ubuntu`, `rdqm-rhel`,
  `nativeha-rhel`. Each declares its `dr_groups` and gates its own provision playbook. The RHEL
  stacks are x86-only, so their acceptance runs where a RHEL-capable host exists.
- **Validation** — a **cold-rebuild acceptance** (per the cold-rebuild acceptance gate), run
  after Wave 1 merges: `bootstrap nativeha-ubuntu --no-dr` brings up three guests, forms HA
  quorum, provisions the HA MQSC, runs **no** DR replication — one pass.
- **Docs** — this spec + the plan (task #189); a documentation-review sweep of the versioned
  site docs (task #993), which spawns per-repo doc tasks as needed; and the retrospective
  (task #190).

## 5. Testing & acceptance

**Unit (Wave 1, in `mq-resiliency-lab-for-linux`):**

- The effective-member helper excludes exactly the `dr_groups` hosts when `--no-dr`, and equals
  `stack_members` when not.
- The `--no-dr` guard errors on a stack with empty `dr_groups`.
- The `provision` phase emits `-e dr_enabled=false` under `--no-dr` and omits it otherwise.
- `vms` and `observe` enumerate site-A members only under `--no-dr`.
- `Stack.dr_groups` parses from the topology block.

**Acceptance (validation task, cold rebuild):**

`mqlab bootstrap nativeha-ubuntu --no-dr` on a cold lab →
- exactly three QM guests up (`nha-ubuntu-a1/a2/a3`), no `b*` guests;
- Native HA quorum forms (`QUORUM(3/3)`, an `Active` instance);
- the HA + app/inter-QM MQSC is applied;
- **no** DR replication playbook runs (site-B untouched);
- one pass, no manual intervention.

## 6. Risks & mitigations

- **A stack passed `dr_enabled=false` whose playbook does not gate DR** → site-A guests up but
  provisioning fails against absent site-B. *Mitigation:* the fail-loud guard (§3.5) — a stack
  is `--no-dr`-capable only where `dr_groups` **and** the playbook gate are declared together.
- **`import_playbook` `when` semantics** — the conditional must apply cleanly to the whole DR
  import. Verified supported; covered by the acceptance run showing no DR play executes.
- **Later "add DR" on a stateless HA-only lab** — re-running `bootstrap <stack>` (no flag) must
  bring up site-B and run the DR provisioning idempotently over the live site-A. The provision
  is `REPLACE`-based/idempotent; the acceptance for the DR-add path is out of Wave 1 scope but
  noted for the retrospective.

## 7. Open questions

- Flag naming settled as `--no-dr` (over `--ha-only`) during brainstorming.
- Whether Wave 2 should be one task per stack or a single multi-stack task — deferred to the
  plan; per-stack is the current assumption (independent provision playbooks, independent
  acceptance, RHEL host constraint).
