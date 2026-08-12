# mqlab box-lifecycle CLI — manage the baked-image layer — design spec

- **Epic:** `logical-minds-foundry/.github#91`
- **Design task:** `logical-minds-foundry/.github#92`
- **Origin:** closing brainstorm of epic #70 → idea `logical-minds-foundry/.github#87`
  (folded in from `.github#86`)
- **Pattern epic:** `logical-minds-foundry/.github#70` (baked boxes made real)
- **Depends on:** correct manifest-hash (`mq-resiliency-lab-for-linux#649`, merged);
  a `vrg-vm` post-build hook (cross-org — see §9)
- **Status:** design (brainstorm output), pending review
- **Date:** 2026-07-16

## 1. Problem & motivation

Epic #70 made the lab's baked-box model real: the slow, deterministic install work
is baked once into a golden per-role box, and a bring-up only does the fast,
instance-specific *configure* work (RDQM bootstrap 85 → 33 min). Now that we will
rebuild and re-bake boxes **frequently**, the CLI needs to *manage* that layer — and
today it can't.

`mqlab` has `bootstrap`/`teardown` (stacks) and `build` (the `build/` buckets), but
baking a box image is either:

- **implicit** — a side-effect of `bootstrap`'s `_ensure_local_boxes()`, which is
  all-or-nothing and keyed off which guests a bring-up happens to need; or
- **manual** — `bash lab/boxes/build-fatbox.sh --box <name> --rebuild-box`.

The three-tier rebuild model documented under #70 (`docs/development/box-model.md`)
has two clean tiers — **nuclear** (wipe the `/vergil` data disk → re-bake every box)
and **VM rebuild** (`vrg-vm rebuild` wipes the boot disk, the `.box` cache on `/vergil`
survives, boxes are just re-registered). The **box-rebake-in-place** tier — refresh
one box's image selectively, in place, without nuking the data disk — has no
first-class command. That is the gap.

The strategic point: the full nuclear rebuild is expensive (it re-bakes everything
*and* needs the one credentialed artifact re-staged). This epic's tooling exists to
make that rebuild **rare and deliberate** by giving the operator selective control of
the baked layer, so the everyday answer to "this box is stale / a pin changed" is a
targeted `mqlab box rebuild`, not a scorched-earth data-disk wipe.

## 2. Goals & non-goals

### Goals

1. First-class `mqlab box` verbs to inspect and manage the baked-image layer per-box
   and `--all`, surfacing the builders' REUSE/BUILD/STALE decisions as first-class
   output. This is the missing box-rebake-in-place tier.
2. Make the RHEL DVD prerequisite — the one artifact a cold boot cannot fetch
   automatically — a fast, verifiable check plus a credential-free host-side
   auto-stage, instead of a silent manual chore.
3. A cold-boot staleness nudge so the `/vergil` data disk does not silently drift for
   months between clean-slate rebuilds.

### Non-goals

- Baking **new stacks/arms** — that is the arm-baking line (`.github#88` native-HA
  RHEL, then the Ubuntu arms) and `.github#72`.
- Rewriting the box builders (`build-fatbox.sh` / `build-box.sh`). This epic *manages*
  them; it does not reimplement their bake or staleness logic.
- Automating the **credentialed** RHEL download (option C). Explored and deferred to a
  closing-brainstorm follow-on (§9).
- Any external box registry / publishing. That is the closing-brainstorm follow-on
  `.github#93` (CI/automated box-library builds).

## 3. The managed fleet

`mqlab box` manages the **five locally-built boxes** — the boxes the lab *builds*, not
the ones it merely downloads:

| Box | Builder | Cache artifact | Staleness model |
|-----|---------|----------------|-----------------|
| `rhel/9.6-x86_64` (base) | `rhel96/build-box.sh` | `build/state/boxes/rhel-9.6-x86_64-libvirt.box` | presence + **30-day non-blocking NOTICE**; **no** manifest-hash, **no** hard refusal; **DVD-gated** (sole consumer of the RHEL ISO) |
| `mq-rdqm-rhel9` (fat) | `build-fatbox.sh` | `build/state/boxes/mq-rdqm-rhel9.box` (+ `.manifest-hash`) | manifest-hash + graduated age (7d NOTICE / 14d REFUSE) |
| `obs-ubuntu2404` (fat) | `build-fatbox.sh` | `…/obs-ubuntu2404.box` (+ `.manifest-hash`) | manifest-hash + graduated age |
| `infra-ubuntu2404` (fat) | `build-fatbox.sh` | `…/infra-ubuntu2404.box` (+ `.manifest-hash`) | manifest-hash + graduated age |
| `mq-ubuntu2404` (fat) | `build-fatbox.sh` | `…/mq-ubuntu2404.box` (+ `.manifest-hash`) | manifest-hash + graduated age |

**Why RHEL is special.** RHEL has a locally-built *base-box layer* Ubuntu does not:
RHEL goes DVD + kickstart → base `.box` (`build-box.sh`) → fat box (`build-fatbox.sh`),
two stages. Ubuntu's base (`cloud-image/ubuntu-24.04`) is a pre-built cloud image
pulled from Vagrant Cloud — nothing to *build*, so there is no analog to manage. The
tool surfaces the base box's softer, honest decision model (no hash column, 30-day
NOTICE) rather than papering over the asymmetry. Ubuntu's cloud base stays unmanaged.

## 4. Command surface — `mqlab box`

A new Typer sub-app (`app.add_typer(box_app, name="box")`), four verbs, positional box
selection or `--all`:

| Command | Behavior |
|---------|----------|
| `box status [<box>…]` (default: all) | Fleet inventory table — each box × {cached? age; manifest-hash match (N/A for base); registered in `vagrant box list`?; **decision** REUSE / BUILD / STALE / FORCE}. Header line shows the **cold-boot age** (§7). Read-only; obtains each box's authoritative decision by running its builder `--dry-run`. Absorbs `list` (one verb, no meaningful cost difference between an inventory and a decision view). |
| `box build <box>… \| --all` | Ensure-present: REUSE a valid cache, else bake. The selective, first-class form of today's implicit `_ensure_local_boxes()`. |
| `box rebuild <box>… \| --all` | Force a fresh bake (`--rebuild-box`), overwriting the cache. **This is the missing box-rebake-in-place tier.** |
| `box clean <box>… \| --all` | Make it **pristine**: remove the durable `.box` (and `.manifest-hash`) from `build/state/boxes/` and deregister it from `vagrant box list`. The next `build` re-bakes. `--all` is guarded behind an explicit confirm flag (it means paying the full re-bake of the whole fleet); a single named box just does it, loudly reporting what it removed. |

Friendly aliases (e.g. `rhel-base`, `rdqm`, `obs`, `infra`, `mq`) may be added if cheap,
since one canonical box name (`rhel/9.6-x86_64`) contains a slash; canonical names
remain valid.

**Verb-to-decision mapping (single source of truth).** `build` / `rebuild` map onto the
builders' existing exit paths: REUSE (register from cache), BUILD (bake), FORCE-BUILD
(`--rebuild-box`), and the base box's presence check. `clean` is the only verb that
manipulates the cache directly (a file removal + deregister), not through a builder.

## 5. Architecture

**Thin Python orchestration over the existing shell builders.** The builders stay the
single source of truth for all bake and staleness decisions; the CLI surfaces their
output and adds selection/`--all`/pretty tables.

- New `src/mqlab/box.py` — the fleet definition (the five boxes → builder + cache
  artifact + staleness model), plus the orchestration helpers behind monkeypatchable
  seams (mirroring cli.py's existing `_build_*` seams so tests never touch real
  git/fs/virsh).
- `cli.py` grows `box_app` and its four verb handlers, reusing the existing
  `_LOCAL_BOX_BUILDERS`, `_box_build_steps`, and the `CommandStep`/`run_steps` plumbing
  — no duplication.
- `_ensure_local_boxes()` (bootstrap) is refactored to call the same `box build` core,
  so bootstrap and the CLI share one code path (bootstrap remains all-or-nothing over
  the guests it needs; the CLI adds selective control).
- `status` runs each box's builder with `--dry-run` (cheap: compute the manifest hash +
  stat the cache) and reads `vagrant box list` + the cold-boot stamp; it renders, it
  does not decide.

This respects the layering principle: `mqlab`'s own messages name fully-qualified
`mqlab` commands; the underlying builders speak for themselves.

## 6. RHEL DVD — verify-and-guide + host-side auto-stage

The RHEL 9.6 DVD ISO (~12.7 GB) is the **one artifact a cold boot cannot fetch
automatically**: IBM MQ and all Ubuntu/OSS artifacts are anonymously downloadable, but
Red Hat requires authentication and no automated path was ever found. It is needed only
on a nuclear rebuild (wiped `/vergil`). Today it is fully operator-supplied
(`stage-rhel-iso.sh` resolves `MQLAB_RHEL_ISO`/`RHEL_ISO` or `build/state/…dvd.iso`, and
errors loudly if absent). This spec adopts **verify-and-guide (option A)** plus a
credential-free convenience:

**In-VM verify (mqlab).** At the RHEL base-box BUILD path (the sole DVD consumer),
`mqlab box` checks the ISO at its canonical `build/state/` location is present and
matches a **pinned SHA-256** (per RHEL version). On MISSING or checksum FAIL it emits
fail-loud guidance — the version, the Red Hat download URL, and the destination — and
stops. No credential handling enters the tool.

**Host-side auto-stage (this repo).** So that a wiped `/vergil` does not force a manual
re-copy every time, the operator does a **one-time manual download per RHEL version**
and keeps it in a **statically-configured local directory** (a stable archive, not an
ad-hoc `build/` copy). Because `vrg-vm create/rebuild` runs natively on the macOS host —
the one place with access to both that local archive and the cloud build volume — a
**VM-specific customization** auto-stages it:

- an **idempotent rsync script** (in this repo) that syncs the configured DVD *directory*
  (drop new DVDs in, they sync) into `build/state/`; and
- a **`vergil.toml` post-build-hook declaration** (in this repo) that runs the script at
  the end of the VM build.

The rsync's idempotency keeps re-runs cheap despite the blob size (it rarely changes); it
is a build-time optimization triggered by the hook. Both the script and the `vergil.toml`
customization live in this repo. The **only** external piece is the `vrg-vm` post-build-hook
capability itself, which does not exist today (`vergil.toml` supports `packages`,
`vagrant_plugins`, `apt_repos`, `port_forwards`, sizing — no script hook). That is a
cross-org dependency (§9).

## 7. Cold-boot staleness nudge

Surface how long since the last full nuclear cold boot so the `/vergil` data disk does
not silently drift.

- **Anchor:** a stamp file `build/state/.cold-boot-stamp`, written **once** when `state/`
  is first initialized on a freshly-wiped `/vergil`. Its age = time since the last
  scorched-earth rebuild — a direct, honest signal, with no need to infer a volume
  creation time the OS does not readily expose.
- **Surfaces (a couple of places, so it whines loudly):** the `box status` header,
  `mqlab doctor`, and the `bootstrap` preflight.
- **Hardness:** **NOTICE-only, never blocking** — an age nudge, since the whole point of
  this tooling is to make the full rebuild rare and *deliberate*, not to nag the operator
  into one. Graduated (quiet → louder) via **tunable constants** with sane placeholder
  defaults.
- **Cadence deliberately unsettled.** The right nudge cadence — and whether it is
  eventually replaced by scheduled *proactive* rebuilds — is a higher-order problem to
  solve once a usage pattern emerges. That automation is exactly the closing-brainstorm
  `.github#93` (CI/automated box-library builds); the manual nudge is the bridge until it
  lands.

## 8. Error handling & testing

- **Fail-loud throughout** — no swallowed failures; a failed bake, a missing / checksum-mismatched
  DVD, or a builder non-zero exit — surfaces with the underlying tool's own message and a
  non-zero exit.
- **Unit tests** via monkeypatched seams, mirroring `test_cli_build.py` /
  `test_box_build.py`: verb → builder-invocation mapping; `status` table rendering from
  parsed builder decisions; `clean` guard on `--all`; DVD verify PASS / FAIL / MISSING;
  cold-boot stamp write-once + age banding; the `_ensure_local_boxes` shared-core
  refactor.
- **Coverage floor: 100% branch** for `mqlab`; `vrg-validate` (`vrg-container-run --
  vrg-validate`) is the single gate.
- **Cold-rebuild acceptance gate.** Box-layer changes are accepted only after a full VM
  cold rebuild proves them one-pass; lint-green ≠ done. The human operates the lab.

## 9. Dependencies, follow-ons, and cross-org handling

Most of this epic lands in `mq-resiliency-lab-for-linux`. One dependency and two
follow-ons live in **other orgs**, and GitHub epics cannot hold cross-org sub-issues.
The convention: create the task as a proper issue in the target repo's **standing
ad-hoc epic** in that org's `.github`, and reference it from epic #91 **by a comment**
(not a formal sub-issue).

1. **Cross-org dependency — `vrg-vm` post-build hook (Vergil tooling).** The host-side
   auto-stage (§6) needs `vrg-vm` to run a post-build customization script at the end of
   a VM build. Filed as an issue in the Vergil-tooling standing ad-hoc epic; referenced
   in #91 by comment. Until it exists, the rsync script is run manually and `mqlab box`
   verify-and-guide still covers the gap.

2. **Closing-brainstorm follow-on — automate the credentialed RHEL fetch.** Option C is
   not impossible; it just needs credentials, and this recurs (RHEL 9.7 / 9.8 / 10, and
   tracking arbitrary versions). A closing-brainstorm task explores automating the
   credentialed acquisition as a follow-on epic/challenge. Distinct topic from #93.

3. **Strategic idea (Vergil/plugin tooling) — first-class cross-org tasks.** The
   "dependency in another org" situation has recurred; the epic/task framework should
   support it formally (teach the `epic-create` skill to create a task in a remote repo
   and reference it from an epic in another org). Filed as an idea/brainstorm issue in
   the plugin standing repo.

## 10. Success criteria

- `mqlab box status` shows the five-box fleet with each box's cache state, age,
  manifest-hash match (or N/A), vagrant registration, and REUSE/BUILD/STALE/FORCE
  decision, plus the cold-boot age.
- `mqlab box build`, `rebuild`, and `clean` operate per-box and `--all`, with `rebuild`
  delivering the box-rebake-in-place tier and `clean` restoring a pristine (re-bakeable)
  state; `clean --all` is confirm-guarded.
- `bootstrap` uses the shared `box build` core (no behavior regression).
- A missing or checksum-mismatched RHEL DVD is caught early with actionable, fail-loud
  guidance; with the configured local archive + host hook in place, a rebuilt VM
  auto-stages the ISO with no manual copy.
- The cold-boot nudge appears in `box status`, `doctor`, and `bootstrap` preflight,
  loud but never blocking.
- `vrg-validate` green at 100% branch coverage, and the change is proven by a full VM
  cold rebuild.
