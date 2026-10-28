# .github

Org-level configuration, community-health files, and profile for the
[logical-minds-foundry](https://github.com/logical-minds-foundry)
organization.

## Table of Contents

- [Status](#status)
- [Overview](#overview)
- [Getting Started](#getting-started)
- [License](#license)

## Status

Early development

## Overview

This repository is the planning home of the
[logical-minds-foundry](https://github.com/logical-minds-foundry)
organization. It holds no application code.

| Path | Contents |
| --- | --- |
| `epics/<N>-<slug>/` | Each epic's `spec.md`, `plan.md` and `retrospective.md`, plus any supporting research; `<N>` is the epic's issue number in this repository |
| `ideas/` | Parked ideas, captured but not yet scheduled as epics |
| `.github/workflows/ops.yml` | Daily GitHub config audit |
| `.github/workflows/epic-rollup.yml` | Closes an epic when its last task closes |
| `.github/workflows/epic-sweep.yml` | Daily reconciliation sweep that closes any epic or task the rollup missed |

The org's epics, its intake, planning and deferred queue epics, and each
repository's ad-hoc epic are issues in this repository.

## Getting Started

GitHub also uses an org's `.github` repository to supply default
community-health files and the organization profile to repositories that do
not define their own. This repository does not hold any of those yet.

## License

See [LICENSE](LICENSE).
