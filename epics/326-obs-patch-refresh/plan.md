# Observability Patch Refresh — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bring the pinned observability components to their latest upstream releases and prove the result with cold builds on arm64 and x86.

**Architecture:** One PR moves every pin, updating the role default and the shared manifest together. Two validation tasks then cold-build the lab on each host platform.

**Tech Stack:** Ansible role defaults, `manifests/_shared/observability.yaml`, `ansible/group_vars/all/versions.yml`, `tests/test_obs_version_sync.py`, `mqlab` bootstrap.

**Spec:** `epics/326-obs-patch-refresh/spec.md`

## Global Constraints

- Prometheus stays on the 3.13 LTS line.
- Move the role default and the manifest together for every component.
- Verify the arm64 and amd64 artifacts before editing.
- The only validation command is `vrg-container-run -- vrg-validate`. A cold build is the acceptance gate.

---

### Task 1 — Bump all pins in one PR · issue #1446

**Files:**

- Modify: `ansible/roles/prometheus/defaults/main.yml` (`prometheus_version`)
- Modify: `ansible/roles/alloy/defaults/main.yml` (`alloy_version`)
- Modify: `ansible/roles/loki/defaults/main.yml` (`loki_version`, which also sets logcli)
- Modify: `ansible/group_vars/all/versions.yml` (`grafana_version`)
- Modify: the OpenSearch and OpenSearch Dashboards role defaults
- Modify: `manifests/_shared/observability.yaml` (every key above)
- Test: `tests/test_obs_version_sync.py` (existing), via `vrg-validate`

- [ ] **Step 1:** Re-check upstream latest for each component (GitHub Releases), taking any newer patch on the same line. Record the final targets in the PR description.
- [ ] **Step 2:** For each target, confirm the arm64 and amd64 artifacts return HTTP 200. Read the release notes and flag any breaking or config change. If a component needs a config migration, leave it out of this PR and file its own task.
- [ ] **Step 3:** Check that OpenSearch 3.9.0 still works with Data Prepper 2.16.0 (release notes and the opensearch sink's supported versions).
- [ ] **Step 4:** Edit every role default and its matching manifest key in a single change.
- [ ] **Step 5:** Run `vrg-container-run -- vrg-validate` and confirm the guardrail test and the full pipeline are green.
- [ ] **Step 6:** Commit with `vrg-commit`, then `report-ready`.

### Task 2 — VAL: arm64 macOS cold build · issue #1447 (blocked by #1446)

Operational task. Run a cold `nativeha-ubuntu` bootstrap on the arm64 macOS host. SUCCESS means all phases are green, Prometheus reports the new version and scrapes all targets, Grafana and Loki are up, and OpenSearch is green with a known log line arriving through Alloy and Data Prepper. Record the `Outcome:`.

### Task 3 — VAL: x86 cloud cold build · issue #1448 (blocked by #1446)

Operational task, run by a person in a cloud session. Same checks as Task 2, on x86, with no regression. Record the `Outcome:`.

## Terminal bookends

- **Docs review** (#1449): sweep the docs for version-pinned references, for example `docs/reference/data-prepper-dlq.md`.
- **Retrospective** (#328): terminal; merging it closes the epic.

## Self-review

Every spec item maps to a task: §2 and §3 to Task 1, §4 to Tasks 2 and 3, §5 to Task 1 Step 2, and the §6 OpenSearch Dashboards risk to Task 1 Steps 2–3. Nothing is left as a placeholder.
