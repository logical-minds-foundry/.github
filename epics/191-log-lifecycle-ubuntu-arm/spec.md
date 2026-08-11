# Native HA log-lifecycle observability — Ubuntu arm — Design Spec

**Epic:** logical-minds-foundry/.github#191 · **Follow-up to:** #145

## Summary

Epic #145 delivered Native HA log-lifecycle observability and proved it end-to-end
on the **RHEL** arm (`nativeha-rhel` / `NHARAPP`). This epic **proves the same
observability on the Ubuntu arm** (`nativeha-ubuntu` / short `NHAU`) and
**generalizes** the one RHEL-hard-scoped artifact — the live-lab validation
playbook — so a single parameterized playbook validates either arm. It is
deliberately small: the core code is already OS-generic.

## Background

#145 shipped and merged: log-type-aware `LOGGEREV` declare-and-verify (#809), the
non-MQI `loglifecycle.py` collector deployed by the `nativeha-state` role (#810),
the time-series cockpit log-health band (#811), the live-lab induce-and-assert
playbook (#812, fixed by #985), and the operator runbook (#813). It was proven on
RHEL via a cold-rebuild gate (#814) and a live-lab run (#815).

A sanity check established this epic's premise — **the core is OS-generic and
already wired for the Ubuntu group:**

- `LOGGEREV` — `ansible/site-nativeha-ubuntu.yml` already includes `mq-event-monitor`
  with `mq_log_type: replicated`.
- Collector — `ansible/observability.yml` runs the `nativeha-state` role on
  `nha_ubuntu_a`/`nha_ubuntu_b`; `loglifecycle.py` uses only MQ/node_exporter paths
  (`/var/mqm/log`, `/var/lib/node_exporter/textfile`), no OS-specific assumptions.
- Cockpit band — `qmboard.py` gates the band on `mechanism == "native-ha"`, not OS.
- Runbook — OS-agnostic content.
- `nativeha-ubuntu` is a fully-wired, bootstrappable stack in `lab/topology.yaml`
  (`cluster_group: nha_ubuntu_a`, `groups`, `provision: site-nativeha-ubuntu.yml`).

## Goals

1. One **generalized** validation playbook validates either arm from a single source.
2. The **Ubuntu arm is proven**: on a cold-rebuilt `nativeha-ubuntu`, `LOGGEREV`
   enabled, the collector emitting on all three site-A instances, and the live-lab
   induce-and-assert (incl. the negative-path guardrail) all green.
3. **RHEL non-regression**: the generalized playbook run with its defaults is
   behavior-identical to the current RHEL invocation.

## Non-goals

- No new observability features — alerting/thresholds and a `media_image_age`
  collector metric remain deferred (recorded as #145 follow-ons).
- No changes to the `LOGGEREV` / collector / band code (already generic).
- No refactor of the per-OS `site-nativeha*.yml` provisioning plays (their OS
  differences are real; only the OS-neutral *validation* playbook is generalized).

## Design

### 1 · Generalize the validation playbook

`ansible/validate-nativeha-log-lifecycle.yml` today hard-codes `hosts: nha_rhel_a`,
`delegate_to: groups['nha_rhel_a'][0]`, and `qm: qm_app | default('NHARAPP')`.

- Introduce a **host-group variable** (e.g. `nha_group`, default `nha_rhel_a`) used
  by the play's `hosts:` and every `delegate_to: groups[nha_group][0]`, and keep the
  existing `qm_app` var (default `NHARAPP`).
- **Ansible feasibility note (load-bearing):** a play's `hosts:` is evaluated
  *before* play-level `vars:` are bound, so the default must be expressed **inline in
  the `hosts:` templating** — `hosts: "{{ nha_group | default('nha_rhel_a') }}"` — or
  supplied via extra-vars/inventory, **not** via a `vars:` block. With no extra-vars
  the RHEL invocation is therefore byte-for-byte unchanged. The `delegate_to` and any
  `groups[...]` lookups may read a normal play `var` (they resolve after host
  selection), but keep them consistent with the same `nha_group`.
- Ubuntu invocation: `-e nha_group=nha_ubuntu_a -e qm_app=<Ubuntu QM>`. The Ubuntu QM
  name derives from `short=NHAU` → expected **`NHAUAPP`** (the `NHAR`→`NHARAPP`
  pattern); **verify at runtime** and treat the derived name as authoritative only
  after `dspmq` confirms it.
- Everything else in the playbook is OS-neutral and unchanged (induce technique,
  the three assertions, the negative-path drift guardrail).

**Folded-in cleanups** (too small for their own PR):
- Correct the stale "`nativeha-ubuntu` is a reserved stack that cannot be
  bootstrapped" comments in `src/mqlab/stacks.py` and `src/mqlab/phases.py` — the
  stack is now fully wired.
- One-line note in `docs/site/docs/guides/nativeha-log-lifecycle-guide.md` that the
  guide applies to both Native HA arms (RHEL and Ubuntu).

### 2 · Cold-rebuild validation (Ubuntu) — the #814 equivalent

`mqlab teardown nativeha-ubuntu` → `mqlab bootstrap nativeha-ubuntu` (bake the
`mq-nativeha-ubuntu` box as needed). Assert on the fresh arm: QM up one-pass;
`LOGGEREV(ENABLED)` with `qm.ini LogType=REPLICATED`; `mqlab_log_*` present on all
three `nha_ubuntu_a` instances (`sample_stale=0`, active/replica divergence visible);
`lab-nativeha-state.timer` active. Record PASS/FAIL as a comment (`issue-validate`).

### 3 · Live-lab validation (Ubuntu) — the #815 equivalent

Run the generalized playbook against the cold-rebuilt Ubuntu arm with the Ubuntu
extra-vars. Expect: the induce rolls an extent, the logger-event / collector-moved /
all-fresh assertions pass, and the negative-path drift guardrail aborts loud and is
rescued. Record PASS/FAIL as a comment (`issue-validate`).

## Acceptance criteria

- Generalized playbook: `vrg-validate` green; runs against either arm via extra-vars;
  RHEL default path behavior-identical.
- Cold-rebuild gate: **PASS** on Ubuntu.
- Live-lab gate: **PASS** on Ubuntu.
- Runbook notes both arms; the doc-review sweep is clean.

## Risks & open questions

- **Ubuntu QM name** — expected `NHAUAPP` by the `short`→name derivation; confirmed
  at runtime, not assumed. *(Low.)*
- **Templated `hosts:` + `delegate_to`** — standard Ansible, but the `delegate_to:
  groups[nha_group][0]` path is verified by the live Ubuntu run. *(Low.)*
- **`mq-nativeha-ubuntu` box bake + any Ubuntu-MQ-install quirk** — surfaced by the
  cold-rebuild gate; that is precisely what the gate is for. *(Medium — the reason
  we prove it rather than assume it.)*
- **Touching the RHEL-proven playbook** — mitigated by RHEL-default preservation; a
  RHEL re-run of the generalized playbook is a cheap optional confirmation.
