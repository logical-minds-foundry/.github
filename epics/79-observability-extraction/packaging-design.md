# Dual-format OS packaging (`.rpm` + `.deb`) — Design

> **Status:** design — sub-brainstorm of epic #79, 2026-07-15.
> **Date:** 2026-07-15
> **Author:** Phillip Moore (with Claude)
> **Epic:** [logical-minds-foundry/.github#79](https://github.com/logical-minds-foundry/.github/issues/79)
> **Task:** [logical-minds-foundry/.github#84](https://github.com/logical-minds-foundry/.github/issues/84)
> **Feeds:** plan Task **T1** (the format-agnostic packaging core + per-format
> adapters) — see [`plan.md`](./plan.md); resolves spec §3.5 / §11 packaging open
> questions — see [`spec.md`](./spec.md).

---

## 1. Purpose

Settle *how* `mq-resiliency-observability` ships as both a `.rpm` (RHEL) and a
`.deb` (Debian/Ubuntu) from one source base — the builder, the publish channel,
the file layout, the install/uninstall behaviour, the toolchain, and the seam
along which a follow-on epic lifts this into a generic Vergil capability. It does
**not** implement packaging; it fixes the decisions T1 builds against.

## 2. Decisions at a glance

| Question | Decision |
|---|---|
| Builder | **nfpm** — one config → `.rpm` + `.deb` (+ `apk`/arch for free) |
| Publish channel | **GitHub Releases** (signed assets on the tagged release); native yum/apt repos **deferred** → #81 |
| Build host | **CI on tag**, reusing the existing `release.yml` / `vrg-release` spine |
| Toolchain | **Containerized** — nfpm + linters in the container image, **never** local macOS |
| Extraction | **Extraction-shaped, built in-repo**; the generic capability is lifted by a follow-on, not built here |

## 3. Builder — nfpm

[nfpm](https://nfpm.goreleaser.com) builds `.rpm` **and** `.deb` from a single
declarative `nfpm.yaml` (`nfpm pkg --packager rpm|deb`). It is chosen over the
alternatives because:

- **One config, many formats.** The spec's "format-agnostic core + per-format
  adapters, extensible to a third format" is satisfied *by the tool*: the core is
  `nfpm.yaml`; enabling `deb`, `apk`, or `archlinux` is a packager flag, not new
  authoring. There is **no `.spec` file and no `debian/` directory.**
- **Zero host toolchain.** nfpm is a single static Go binary — no `rpmbuild`,
  no `dpkg-dev` on the runner. It runs identically in any container or CI. This
  is decisive for a *generic* capability that cannot assume a build host (§7).
- **Maintainer scripts + GPG signing** for both formats, so the §5 install-time
  gate / clean-uninstall scriptlets attach cleanly and signing reuses the
  existing `RELEASE_GPG_*` secrets.

Rejected: **fpm** (needs Ruby + native packaging tools on the host — heavier,
worse for a generic capability); **native `rpmbuild` + `dpkg-deb`** (two
toolchains, two authoring formats — maximal per-format divergence, the opposite
of a format-agnostic core, and `rpmbuild`/`.spec` is a decades-old, clumsy stack).

## 4. Publish — GitHub Releases (native repos deferred)

The pipeline reuses the spine Vergil already has. Today `release.yml` fires on a
`v*` tag → builds a source tarball → **GPG-signs** it + `SHA256SUMS` → publishes
an idempotent **GitHub Release**. We add **one build stage**: nfpm produces the
`.rpm` + `.deb`, and they are attached to that same Release. The new repo sets
`[publish].release = true` in its `vergil.toml`; the tag→version guard mirrors
`release_version_guard.py`.

**Verification assets** (uniform with the existing tarball flow): each Release
carries `*.rpm`, `*.deb`, detached `.asc` signatures for each, and `SHA256SUMS`
(+ its `.asc`) — plus nfpm's **embedded RPM GPG signature** so `rpm --checksig`
also works. Same `RELEASE_GPG_PRIVATE_KEY` / `RELEASE_GPG_PASSPHRASE` secrets.

Adopters download the asset and `dnf install ./*.rpm` / `apt install ./*.deb`.
**Native yum/apt repositories** (`dnf install mq-resiliency-observability` +
auto-update) are **deferred** — nfpm's output is already repo-ready, so standing
them up later is additive, not a rework. That work is the end-game (§10) and is
mapped by the #81 follow-on brainstorm.

## 5. File structure & the one real per-format divergence

```text
packaging/
  nfpm.yaml            # format-agnostic core: contents (files->install paths),
                       # deps, metadata (maintainer, MIT, version from the tag),
                       # and references to the maintainer scripts below
  scripts/
    postinstall.sh     # spec §3.6 textfile-dir gate: FAIL LOUD if the configured
                       # node_exporter textfile dir is unspecified/unwritable;
                       # then enable + start the collector timers
    preremove.sh       # stop + disable the timers
    postremove.sh      # remove ONLY our artifacts (our units, the .prom files we
                       # wrote); never touch the shared dir or foreign files
  units/               # *.service / *.timer templates
```

The **build** has no meaningful per-format divergence — nfpm emits both formats
from `nfpm.yaml`, so both ship from day one (this **collapses** the spec's
"RPM-first, Debian fork-and-stub" on the build side; see §8).

The **maintainer scripts are the single genuine divergence.** nfpm runs one
script per hook for both formats, but the *runtime arguments differ*:

- rpm `%postun`/`%preun` receive `$1` = a count — **`0` = final removal**,
  **`≥1` = upgrade**.
- deb `postrm`/`prerm` receive a verb — **`remove`** / **`upgrade`** / **`purge`**.

If removal ignores this it will wipe our files on every *upgrade*. So the
scripts must branch on "is this a final removal or an upgrade?" in both dialects.
**This is where "RPM-first, Debian fork-and-stub" actually lives:** write the rpm
arg handling first, add the deb-verb branch second (a small, well-marked
`# deb:` fork inside each removal script), not a separate build path.

## 6. Toolchain — containerized, never local macOS

All validation and package builds run in the **containerized toolchain**, per the
repo's contract that `vrg-container-run -- vrg-validate` is the only validation
path. Therefore:

- **nfpm** (and any nfpm linter we adopt) is **provisioned into the container
  image** the repo uses — not installed ad-hoc on macOS. Local macOS stays
  tool-free; it is only ever used *before* validation/CI, never as a build
  dependency.
- CI uses the **same** containerized toolchain, so a package that builds under
  `vrg-validate` builds identically in the release workflow — no drift between the
  dev-loop check and the tagged build.
- nfpm being a single static binary makes vendoring/fetching it into the image
  trivial. The exact provisioning mechanism (the new repo's Vergil container
  config) is settled at T1.

This keeps the whole packaging lifecycle inside the reproducible container, in
line with the lab's "no precious local state" ethos.

## 7. The extraction seam

The split is clean and mirrors #368's capability-vs-configuration model:

- **Configuration (stays in `mq-resiliency-observability`):** `packaging/nfpm.yaml`,
  `scripts/`, `units/`, and the payload.
- **Generic capability (a follow-on lifts into Vergil):** the *build-sign-attach
  step* — "given an `nfpm.yaml` + the `RELEASE_GPG_*` secrets, produce signed
  `.rpm`/`.deb` and attach them to the tagged Release." It is authored in this
  repo's `release.yml` as a **self-contained job that reads only
  `packaging/nfpm.yaml`**, so the follow-on can lift it into a reusable
  `vergil-actions` workflow (mirroring the existing
  `vergil-project/vergil-actions/.github/workflows/cd-docs.yml`) plus a
  `vrg-release` hook — nearly verbatim.

We **build it in-repo now, extraction-shaped; we do not extract in this epic**
(proving it on one repo before generalizing a pattern we have never run). The
Vergil lift is an explicitly-anticipated follow-on epic (spec §10, task #81).

## 8. Plan impact (for alignment)

nfpm **collapses plan T11** ("complete the Debian adapter"): there is no separate
`.deb` build to implement — nfpm emits it from the same config. T11 reduces to
"add the deb-verb branch in the removal scripts + enable the `deb` packager,"
which **folds into T1**. The plan/tasks should be trued up accordingly at
alignment: T1 delivers *both* formats' build, RPM-correct scripts, and the
Debian script fork; T11 is either removed or downgraded to "verify `.deb`
install/upgrade/remove on Ubuntu" (which the #645 install validation already
covers).

## 9. Testing

- **nfpm config** lints in the container.
- A CI job **builds both packages** and asserts the expected asset set +
  signatures (`.rpm`, `.deb`, `.asc`, `SHA256SUMS`; `rpm --checksig` passes).
- **install / uninstall / upgrade round-trips** on throwaway RHEL + Ubuntu VMs:
  the §3.6 gate **fails loud** on an unwritable/absent textfile dir; an **upgrade
  preserves** our `.prom` files (the arg-semantics branch, §5); **removal cleans
  only our artifacts**. These feed the #644 (`.rpm`) / #645 (`.deb`) install
  validations.

## 10. Forward-looking — the native-repo end-game

The trajectory beyond this epic: **Logical Minds Foundry-hosted, natively-served
signed yum/apt repositories**, so an adopter runs `dnf install
mq-resiliency-observability` and gets auto-updates — vendor-grade distribution
for everything the Foundry publishes. nfpm's output is already repo-ready, so
this is purely additive (repo metadata via `createrepo` / apt `Packages`+`Release`,
HTTP hosting, repo GPG keys). It is the standing distribution goal for the
toolkit and is mapped by the **#81** follow-on brainstorm (which may spawn its own
epic/tasks — a one-to-many mapping is exactly that bookend's job).

## 11. Open items (settled at T1 / deferred)

- Exact container-image provisioning mechanism for nfpm (T1).
- nfpm linter selection, if any (T1).
- Native-repo hosting, keys, and domain (→ #81 / follow-on epic).
