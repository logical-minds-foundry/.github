# M3: the lab consumes published component packages — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `mq-resiliency-observability` becomes a Vergil-managed product that releases signed
`.deb`/`.rpm` into a new logical-minds-foundry package repository. The lab installs it at an
exact pinned version, with the repository keys verified against per-user trust anchors, while
`mq-resiliency-clients` stays in-lab on the same published runtime package.

**Architecture:**

- **Org infrastructure** (OP1, KR/PUB, REG) mirrors vergil's M2 stand-up one for one:
  - a key ceremony;
  - `logical-minds-foundry/packages` (the index plus the `logical-minds-foundry-archive-keyring`
    product);
  - an interim `orgs.py` entry in vergil-tooling.
- **Trust anchors** live in `~/.config/vergil/package-trust.toml`.
  - They are authored on the host with `vrg-package trust add` and mirrored into VMs by
    `vrg-vm` (TRUST-SYNC).
  - The lab reads them (`mqlab package trust list`) and hands them to Ansible as
    `package_trust`.
- **Ansible roles.** A `package-repos` role trusts each org with the standard third-party
  recipe: fetch the key, verify the primary fingerprint, add the source. A `package-install`
  role installs exact pins, and the pinned runtime is always added.
  - Pins and repos reach Ansible from `lab/versions.yaml` through a group_vars `lookup`, the
    same way `mq_version` does today.
  - Only the per-user anchors travel by `-e @trust-vars.json`.
- **In-lab tier.** In-lab components keep `releases/` and the atomic `venv` flip, but build on
  `/opt/vergil/python/3.14.8/bin/python3.14` from the `vergil-python3.14.8` package.
  `runtime-install` and the tarball pin are deleted.

**Tech Stack:**

- `mqlab`: Python 3.14 + Typer, pytest at 100% branch coverage, `uv run pytest`.
- Ansible (`ansible.builtin` only), bash box builders.
- vergil-tooling: argparse, pytest, the `Transport` protocol.
- nFPM through M2's python builder; GitHub Actions through `vergil-actions@v2.1`.

**Spec:** `epics/319-lab-consumes-packages/spec.md` (this directory). Executors read both.

**Repos:**

| Task(s) | Repo |
|---|---|
| OP1, D, retrospective, #321, #324 | `logical-minds-foundry/.github` |
| KR/PUB | `logical-minds-foundry/packages` (created by OP1) |
| X1, X2, X3 (product half) | `logical-minds-foundry/mq-resiliency-observability` |
| L1, L2, L3, V1, V2, X3 (lab half), doc review #1444 | `logical-minds-foundry/mq-resiliency-lab-for-linux` |
| REG, TRUST-SYNC (cross-org; filed there, listed in the epic body) | `vergil-project/vergil-tooling` |

## Global Constraints

- **Repositories (verbatim from the spec):**
  - vergil `https://vergil-project.github.io/packages`
  - logical-minds-foundry `https://logical-minds-foundry.github.io/packages`
  - The key is `<url>/keys/<org>.asc`.
  - apt: `URIs: <url>/deb`, `Suites: <codename>` (`noble`, `resolute`), `Components: main`.
  - dnf: `baseurl=<url>/rpm/el<major>/$basearch` — **the major is written literally, never
    `$releasever`** (RHEL guests are minor-pinned, so `$releasever` would expand to e.g.
    `9.6`).
- **Guest file paths** (one set per org):
  - `/etc/apt/keyrings/<org>.asc`
  - `/etc/apt/sources.list.d/<org>.sources`
  - `/etc/pki/rpm-gpg/RPM-GPG-KEY-<org>`
  - `/etc/yum.repos.d/<org>.repo`
  - The lab never installs `<org>-archive-keyring`.
- **Pins** (`lab/versions.yaml` `packages.pins`):
  - `vergil-python3.14.8: "3.14.8+20261003-1"`, verified on the live vergil index on
    2026-10-12;
  - `mq-resiliency-observability: "0.2.0-1"`, which becomes `"1.0.0-1"` at X3.
  - A pin is the full `<version>-<revision>` string: dpkg `${Version}`, or rpm
    `%{VERSION}-%{RELEASE}`.
  - Revisions are integers. Native per-target revisions (`1~noble`, `1.el9`) are refused.
- **The runtime pin** is the single `packages.pins` key matching `^vergil-python(\d+\.\d+\.\d+)$`.
  - Its interpreter is `/opt/vergil/python/<X.Y.Z>/bin/python<X.Y>`.
  - It is added to every install, so it is never left to resolve as a floating dependency.
- **Anchor file:** `~/.config/vergil/package-trust.toml`. Exact format, shared by
  vergil-tooling and `mqlab`:

  ```toml
  # Vergil package-repository trust anchors: one primary-key fingerprint per org.
  # Written by `vrg-package trust add` on the host; synced into VMs by vrg-vm. Never commit.
  [orgs.vergil]
  fingerprint = "B3A1D804DC03AB036566A4E367861822166ECABE"
  ```

  - Org slugs match `^[a-z0-9][a-z0-9-]*$`.
  - Fingerprints are exactly 40 uppercase hex characters. Input is normalized by removing
    spaces and upper-casing.
- **Paths visible from outside (D4):**
  - `/opt/logical-minds-foundry/<name>/venv` (for packaged products this is M2's
    `/opt/<vendor>/<name>/venv`);
  - config in `/etc/logical-minds-foundry/<name>/<unit>.env`;
  - units in `/usr/lib/systemd/system/`.
  - Nothing new uses `/etc/opt/` or `/opt/vergil/cpython-*`.
- **M1 → M3 is a cold rebuild.** `package-install` refuses to install a package over
  `/opt/logical-minds-foundry/<name>/releases` or a symlinked `.../venv`, naming
  `mqlab box build`. No role migrates an old install in place.
- **No fallbacks:** none to an unsigned repo, an unpinned version, the keyring package or the
  tarball runtime. mqlab errors name fully qualified commands (`mqlab …`, `vrg-package …`,
  `vrg-vm session`). Ansible and apt/dnf speak for themselves.
- **Agent boundaries:** the agent never handles key material. Releases, PR submission and
  merges are human-gated. Raw `git`/`gh` are denied: use `vrg-git`/`vrg-gh` and
  `vrg-commit --type <t> --scope <s> --message <m>`.
- **Validation:** `vrg-container-run -- vrg-validate` in every repo. It is the only
  validation command.
- **`build/` paths** go through `mqlab.paths` only.
- **Ansible** uses `ansible.builtin` modules only, and its interpreter stays
  `/usr/bin/python3`.

## Review Focus

1. **A RHEL guest pinned to a minor release** (`point: "9.6"`) must still resolve the dnf
   repository. `baseurl` uses the literal major (`el9`/`el10` from
   `ansible_distribution_major_version`), never `$releasever`. Pinned by
   `test_package_repos_rpm_baseurl_uses_literal_major` (L1).
2. **Lowering a pin** (rolling back to an older release) must install the older version rather
   than silently keep the newer one. Every pinned install sets `allow_downgrade: true`, and a
   post-install assert compares the installed version with the pin. Pinned by
   `test_package_install_allows_downgrade_and_asserts_pin` (L1).
3. **A missing anchor discovered mid-bake** would waste a long bake and leave scratch behind.
   `mqlab box build` resolves the anchors **before** the first bake starts, and the bake
   fails before any Ansible runs. Pinned by `test_build_plan_requires_anchors_before_any_bake`
   (L2).
4. **An anchor deleted on the host** must stop being trusted in the VM. The sync mirrors the
   host file, so a missing host file removes the VM copy. Pinned by
   `test_sync_package_trust_removes_guest_copy_when_host_has_none` (TRUST-SYNC).
5. **An app built against a newer runtime than the lab's runtime pin** makes apt/dnf unable
   to satisfy both exact pins. The install fails with the package tool's own error, and the
   role adds one line naming both pins in `lab/versions.yaml`. Pinned by
   `test_package_install_failure_names_the_pins` (L1).

## Task dependency graph

```text
D        docs: spec + plan (.github#320)
OP1      key ceremony + trust roots (deployment, human-attested)          [D]
KR/PUB   logical-minds-foundry/packages: keyring product + publish-index  [OP1]
REG ⟂    vergil-tooling orgs.py interim entry + release (human-gated)     [OP1]
TRUST-SYNC ⟂ vrg-package trust + vrg-vm mirror + release (human-gated)    [D]
X1       product repo: import (overwrite) + Vergil scaffolding           [D]
X2       release 0.2.0, confirmed on the LMF index (deployment)          [X1, REG, KR/PUB]
L1       lab: packages catalog, trust reader, package-repos/-install     [D]
L2       lab: in-lab tier on vergil-python3.14.8; runtime-install retired [L1]
L3       lab: observability from the package; in-lab copy removed       [L2, X2]
V1       validate: cold rebuild, macOS arm64                             [L3, TRUST-SYNC]
V2       validate: cold rebuild, x86 cloud                               [L3, TRUST-SYNC]
X3       release 1.0.0; bump the lab pin; reprovision check              [V1, V2]
Closing  doc review (lab#1444) · follow-ons (#321, #324) · retrospective (#322)
```

L2 depends on L1 because it uses L1's `package-install` role and catalog block. L3 depends on L2
because it uses L2's `--packages` manifest flag and `BoxEntry.packages` (the spec's table lists
L3 ← L1, X2; the L2 edge is added here).

---

## Operational OP1 (deployment, human-attested): Key ceremony and trust roots

Created with `--kind deployment` in `logical-minds-foundry/.github`, `--blocked-by` #320.
It mirrors vergil-project/vergil-tooling#3073 step for step. **The agent never touches key
material.** The human performs every step and attests to it.

- **Precondition (attested):** an offline workstation with GnuPG, and a fresh `GNUPGHOME`
  (e.g. `~/lmf-signing-ceremony/gnupg`).
- **Procedure (human):**
  1. Generate the keys:
     - `gpg --quick-generate-key "logical-minds-foundry packages <packages@logical-minds-foundry.invalid>" rsa4096 cert 10y`
     - `gpg --quick-add-key <PRIMARY_FPR> rsa4096 sign 2y`

     RSA 4096 matches vergil's choice for RHEL compatibility.
  2. Export three things: the public key (`gpg --armor --export <FPR> > logical-minds-foundry.asc`),
     the secret backup and the revocation certificate. For CI, export **the subkey only**:
     `gpg --armor --export-secret-subkeys '<SUBKEY_FPR>!'`.
  3. Verify that `gpg -K` shows `sec#` plus `ssb` in the CI export, and that a test
     `--clearsign` with the subkey verifies.
  4. Create the repository with `vrg-github-repo-init logical-minds-foundry/packages --no-publish-docs`.
     The flag is load-bearing: without it the tool claims the Pages site. Then set Pages to
     build from Actions:
     `gh api -X POST repos/logical-minds-foundry/packages/pages -f build_type=workflow`.
  5. Create the environments. Each holds `PACKAGE_SIGNING_KEY` (the subkey export) and
     `PACKAGE_SIGNING_PASSPHRASE`:
     - `package-signing`, deployment branch `main` only, in `packages` **and**
       `mq-resiliency-observability`;
     - `index-signing`, deployment branch `develop` only, in `packages`.
  6. Confirm the org's GitHub App is installed with `contents: write` on `packages`, and that
     `APP_CLIENT_ID` and `APP_PRIVATE_KEY` are available to both repos. The observability
     `ops.yml` already uses them daily, but confirm they cover `packages`.
  7. Record the rotation date: renew the subkey before its 2-year expiry. Then cross-reference
     #324.
- **Acceptance:** the SUCCESS comment carries:
  - the 40-hex **primary** fingerprint;
  - the armored public key, in a fenced block;
  - the environment names created per repo;
  - the App confirmation;
  - `Outcome: SUCCESS`.

  REG and KR/PUB copy the fingerprint and key from this comment.

---

## Task KR/PUB: `logical-minds-foundry/packages` — keyring product and index

**Repo:** `logical-minds-foundry/packages`, on a feature branch off `develop` (the repo was
created by OP1). Reference: `vergil-project/packages`, checked out read-only at
`/Users/pmoore/dev/projects/vergil-project/packages`.

**Files** (each mirrors the vergil file of the same path; only the vendor tokens change):

- Create: `keys/logical-minds-foundry.asc` (the armored public key from OP1's comment).
- Create: `vergil.toml`:

  ```toml
  [project]
  repository-type = "infrastructure"
  # copy the remaining [project]/[ci]/[publish]/[dependencies] keys from
  # vergil-project/packages/vergil.toml verbatim (ci versions ["latest"], publish release=true docs=false)

  [package]
  builder = "staged"
  vendor = "logical-minds-foundry"
  name = "logical-minds-foundry-archive-keyring"
  summary = "Archive keyring and repository configuration for logical-minds-foundry packages"
  smoke = "test -s /usr/share/keyrings/logical-minds-foundry-archive-keyring.asc"
  noarch = true
  ```

- Create: `packages.toml`:

  ```toml
  vendor = "logical-minds-foundry"
  products = [
    "logical-minds-foundry/packages",
    "logical-minds-foundry/mq-resiliency-observability",
  ]

  [retention]
  keep = 3
  lines = 2
  ```

- Create: the `packaging/` files, copied from vergil with `vergil` replaced by
  `logical-minds-foundry` throughout, in file names, paths and URLs:
  - `nfpm.overlay.yaml`
  - `postinst.sh`
  - `postrm.sh`
  - `logical-minds-foundry.repo`
- Create: `.github/workflows/{ci,cd,publish-index}.yml`, copied verbatim from vergil. `cd.yml`
  and `publish-index.yml` keep `secrets: inherit` (packages#6). `publish-index` triggers on
  `repository_dispatch: [package-released]`, `workflow_dispatch` and the weekly cron.
- Create: `README.md` (what the repo serves, with the primary fingerprint), `CHANGELOG.md`,
  `VERSION` (`1.0.0`).

- [ ] **Step 1: Diff against vergil.** Run
  `diff -r /Users/pmoore/dev/projects/vergil-project/packages <this worktree>` and confirm
  that every remaining difference is a vendor token, the key, the products list or the README
  prose.
- [ ] **Step 2: Grep for stray tokens.** Run `grep -rn "vergil-project\|vergil-archive\|/vergil\.\|RPM-GPG-KEY-vergil" .`
  The only expected hits are the `vergil-actions` workflow references.
- [ ] **Step 3: Check the key.** Run
  `gpg --show-keys --with-colons keys/logical-minds-foundry.asc | grep -m1 '^fpr:'`. Field
  10 must equal OP1's primary fingerprint.
- [ ] **Step 4: Validate.** Run `vrg-container-run -- vrg-validate`.
- [ ] **Step 5: Commit and report ready.** Commit with
  `vrg-commit --type feat --scope packages --message "stand up the logical-minds-foundry package repository"`,
  then run `vrg-pr-workflow report-ready`.
- **After merge (human-gated release):** the human releases `v1.0.0` from `main`. The release
  dispatches `publish-index`.
- **Acceptance:** after the release, all three succeed:
  - `curl -fsSL https://logical-minds-foundry.github.io/packages/keys/logical-minds-foundry.asc`
  - `.../deb/dists/noble/InRelease`
  - `.../rpm/el9/x86_64/repodata/repomd.xml.asc`

  Record the output on the task.

---

## Task REG ⟂ (vergil-project/vergil-tooling): interim `orgs.py` entry

**Files:**

- Modify: `src/vergil_tooling/lib/package/orgs.py:27-36` (`ORGS`).
- Modify: `tests/vergil_tooling/test_package_orgs.py`. The existing test that asserts an
  unknown vendor raises must use a still-unknown vendor (e.g. `"nobody"`).

**Interfaces:** `orgs.for_vendor("logical-minds-foundry") -> OrgRepo`. The release
`package-index` stage (`lib/release/package_index.py:164`) is the consumer.

- [ ] **Step 1: Write the failing test.**

  ```python
  def test_logical_minds_foundry_is_registered() -> None:
      org = orgs.for_vendor("logical-minds-foundry")
      assert org.github_org == "logical-minds-foundry"
      assert org.base_url == "https://logical-minds-foundry.github.io/packages"
      assert org.keyring_package == "logical-minds-foundry-archive-keyring"
      assert re.fullmatch(r"[0-9A-F]{40}", org.fingerprint)
  ```

- [ ] **Step 2: Run it and see it fail.**
  `uv run pytest tests/vergil_tooling/test_package_orgs.py -v` fails with
  `unknown package vendor`.
- [ ] **Step 3: Add the entry.** Copy the fingerprint from OP1's SUCCESS comment.

  ```python
  # INTERIM (logical-minds-foundry/.github#319, D8): moves to ~/.config/vergil
  # configuration in the follow-on registry brainstorm (logical-minds-foundry/.github#321).
  "logical-minds-foundry": OrgRepo(
      vendor="logical-minds-foundry",
      github_org="logical-minds-foundry",
      base_url="https://logical-minds-foundry.github.io/packages",
      fingerprint="<40-HEX FROM OP1>",  # attested in OP1, logical-minds-foundry/.github#<OP1>
      keyring_package="logical-minds-foundry-archive-keyring",
  ),
  ```

- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Validate, commit, report ready.** Run `vrg-container-run -- vrg-validate`,
  commit with `vrg-commit --type feat --scope package --message "register the logical-minds-foundry package org (interim)"`,
  then run `report-ready`.
- **After merge:** a vergil-tooling release (human-gated). The task records the release
  version, which X2 checks.

---

## Task TRUST-SYNC ⟂ (vergil-project/vergil-tooling): host anchors and VM mirror

**Files:**

- Create: `src/vergil_tooling/lib/package/trust.py`
- Modify: `src/vergil_tooling/bin/vrg_package.py`, adding a `trust` subparser with `add` and
  `list`
- Modify: `src/vergil_tooling/lib/vm_guest.py`, adding `sync_package_trust`
- Modify: `src/vergil_tooling/bin/vrg_vm.py`. Call `sync_package_trust` in four places:
  - `_st_credentials` (`:629`), `_cs_credentials` (`:935`) and `_cmd_restart` (`:1192`),
    each after `inject_credentials`, **unconditionally**: a credential-less identity still
    syncs trust;
  - `_cmd_session` (`:2570`), before `copy_claude_config`.
- Create: `tests/vergil_tooling/test_package_trust.py`. Extend `test_vm_guest.py`,
  `test_vrg_vm.py` and the `vrg_package` CLI tests.
- Modify: `docs/site/docs/reference/` (the package-config or consumer page): an "Anchoring
  trust in an org's packages" section.

**Interfaces (produced; the lab reads the same file format):**

```python
TRUST_FILE = Path("~/.config/vergil/package-trust.toml")
GUEST_TRUST_FILE = "~/.config/vergil/package-trust.toml"

class TrustError(PackageError): ...

def trust_path() -> Path: ...                                  # TRUST_FILE.expanduser()
def normalize_fingerprint(raw: str) -> str: ...                # strip spaces, upper; 40 hex or TrustError
def load_anchors(path: Path | None = None) -> dict[str, str]:  # {} if absent; TrustError if malformed
def render_anchors(anchors: dict[str, str]) -> str: ...        # the Global Constraints format, orgs sorted
def add_anchor(org: str, fingerprint: str, *, replace: bool = False, path: Path | None = None) -> bool: ...
    # True if written; False if identical; TrustError if a DIFFERENT anchor exists and not replace
def sync_package_trust(transport: Transport, host_path: Path | None = None) -> None: ...  # in vm_guest
```

- [ ] **Step 1: Write the failing tests** (`test_package_trust.py`):

  ```python
  def test_add_anchor_writes_canonical_file(tmp_path: Path) -> None:
      p = tmp_path / "package-trust.toml"
      assert trust.add_anchor("logical-minds-foundry", "aaaa " * 10, path=p)
      assert trust.load_anchors(p) == {"logical-minds-foundry": "A" * 40}
      assert p.read_text().startswith("# Vergil package-repository trust anchors")

  def test_add_anchor_refuses_silent_replacement(tmp_path: Path) -> None:
      p = tmp_path / "t.toml"
      trust.add_anchor("vergil", "A" * 40, path=p)
      with pytest.raises(trust.TrustError, match="--replace"):
          trust.add_anchor("vergil", "B" * 40, path=p)
      assert trust.add_anchor("vergil", "B" * 40, replace=True, path=p)

  def test_add_anchor_identical_is_noop(tmp_path: Path) -> None:
      p = tmp_path / "t.toml"
      trust.add_anchor("vergil", "A" * 40, path=p)
      assert trust.add_anchor("vergil", "a" * 40, path=p) is False

  @pytest.mark.parametrize("bad", ["", "XYZ", "A" * 39, "A" * 41])
  def test_normalize_fingerprint_rejects(bad: str) -> None:
      with pytest.raises(trust.TrustError):
          trust.normalize_fingerprint(bad)

  @pytest.mark.parametrize("org", ["", "Vergil", "-x", "a_b"])
  def test_add_anchor_rejects_bad_org(tmp_path: Path, org: str) -> None:
      with pytest.raises(trust.TrustError):
          trust.add_anchor(org, "A" * 40, path=tmp_path / "t.toml")

  def test_load_anchors_absent_is_empty(tmp_path: Path) -> None:
      assert trust.load_anchors(tmp_path / "missing.toml") == {}

  def test_load_anchors_malformed_names_file(tmp_path: Path) -> None:
      p = tmp_path / "t.toml"
      p.write_text("[orgs.vergil]\nfingerprint = 7\n")
      with pytest.raises(trust.TrustError, match=str(p)):
          trust.load_anchors(p)
  ```

  In `test_vm_guest.py`, using the existing fake transport (which records `run`/`pipe`):

  ```python
  def test_sync_package_trust_pipes_host_file(tmp_path, fake_transport) -> None:
      host = tmp_path / "package-trust.toml"
      trust.add_anchor("vergil", "A" * 40, path=host)
      vm_guest.sync_package_trust(fake_transport, host)
      cmd, content = fake_transport.pipes[-1]
      assert "~/.config/vergil/package-trust.toml" in cmd and "chmod 644" in cmd
      assert content == host.read_text()

  def test_sync_package_trust_removes_guest_copy_when_host_has_none(tmp_path, fake_transport) -> None:
      vm_guest.sync_package_trust(fake_transport, tmp_path / "absent.toml")
      assert any("rm -f ~/.config/vergil/package-trust.toml" in " ".join(c) for c in fake_transport.runs)

  def test_sync_package_trust_refuses_malformed_host_file(tmp_path, fake_transport) -> None:
      host = tmp_path / "t.toml"
      host.write_text("not toml [")
      with pytest.raises(SystemExit):
          vm_guest.sync_package_trust(fake_transport, host)
      assert fake_transport.pipes == []
  ```

  In `test_vrg_vm.py`, assert that `_st_credentials`, `_cs_credentials`, `_cmd_restart` and
  `_cmd_session` each call `sync_package_trust`. Patch it, following the existing
  stage-assertion pattern at `:3499-3543`, and include a credential-less identity case.

  In the CLI tests:
  - `vrg-package trust add vergil <fpr>` exits 0;
  - adding a different fingerprint exits 1 with a message naming `--replace`;
  - `vrg-package trust list` prints `ORG  FINGERPRINT  FILE` rows.

- [ ] **Step 2: Run the tests and see them fail.** Use `uv run pytest` on the four test
  files.
- [ ] **Step 3: Implement `trust.py`.** Read with `tomllib`; validate the top-level shape
  (`orgs` table only; each org table holds exactly `fingerprint`); write atomically (temp file
  in the same directory, then `os.replace`, mode `0644`, parent created `0700` if absent).
  `render_anchors` emits exactly the Global Constraints format, with orgs sorted.
- [ ] **Step 4: Implement `sync_package_trust`.**
  - If the host file is absent: `transport.run("bash", "-c", "rm -f ~/.config/vergil/package-trust.toml")`
    and print `package trust: no host file; removed any VM copy` (indented two spaces,
    like the other stage messages).
  - Otherwise: call `load_anchors(host)`. On `TrustError`, print `ERROR: <message>` to stderr
    and `raise SystemExit(1)` (the existing `inject_credentials` error idiom). Then
    `transport.pipe("mkdir -p ~/.config/vergil && cat > ~/.config/vergil/package-trust.toml && chmod 644 ~/.config/vergil/package-trust.toml", text)`
    and print `package trust: synced N org(s)` (indented the same way).
- [ ] **Step 5: Wire the four call sites, and add `trust add|list` to `vrg_package.py`.**
  The handlers are `_cmd_trust_add` and `_cmd_trust_list`.
  - `TrustError` makes the command exit 1 with the message.
  - On success `add` prints `trusted <org> <fpr> in <file>; re-enter the VM (vrg-vm session) to sync it`.
- [ ] **Step 6: Run the tests and see them pass. Then validate.** Run
  `vrg-container-run -- vrg-validate` (100% branch coverage).
- [ ] **Step 7: Commit and report ready.** Commit with
  `vrg-commit --type feat --scope package --message "add vrg-package trust and mirror package-trust.toml into VMs"`,
  then run `report-ready`.
- **After merge:** a vergil-tooling release (human-gated). Its version is recorded on the
  task. The human then runs `vrg-package trust add vergil B3A1D804DC03AB036566A4E367861822166ECABE`
  and `vrg-package trust add logical-minds-foundry <OP1 FPR>` on the macOS host and on the
  cloud host's workstation.

---

## Task X1: `mq-resiliency-observability` — import (overwrite) and Vergil scaffolding

**Repo:** `logical-minds-foundry/mq-resiliency-observability`, on a feature branch off
`develop`. It can run in parallel with L1/L2: the source tree to import is the lab's
`components/mq-resiliency-observability` at the develop commit when X1 starts. Record that
commit hash.

**Files:**

- Delete everything except `.git`, `.github/`, `.claude/`, `CLAUDE.md`, `LICENSE`,
  `vergil.toml` and `docs/site/`.
- Copy in from the lab component:
  - `pyproject.toml` (name `mq-resiliency-observability`; version `0.2.0`)
  - `uv.lock`
  - `src/mqro/`
  - `tests/`
  - `systemd/` (6 unit files)
  - `README.md`, extended as below
- Create: `VERSION` (`0.2.0`), `CHANGELOG.md`, `releases/v0.2.0.md`.
- Create: `packaging/nfpm.overlay.yaml`.
- Modify: `vergil.toml`. Add `[package]`, and set `[ci] versions = ["3.14"]` (the component
  is `requires-python = "==3.14.*"`).
- Modify: `.github/workflows/ci.yml`, adding the `package` job.
- Modify: `.github/workflows/cd.yml`. The release job gets `secrets: inherit` with the
  `# nosemgrep: yaml.github-actions.security.secrets-inherit.secrets-inherit` comment, as in
  vergil-tooling's `cd.yml`. Without it `PACKAGE_SIGNING_KEY` arrives empty
  (vergil-project/packages#6).
- Modify: every unit file. Change `EnvironmentFile=/etc/opt/logical-minds-foundry/...` to
  `EnvironmentFile=/etc/logical-minds-foundry/mq-resiliency-observability/<unit>.env`.
- Modify: `docs/site/` pages. Describe the product, how to install it from the LMF
  repository (trust both orgs; `apt install mq-resiliency-observability`), the units and env
  keys, and its provenance.

**`vergil.toml` `[package]`:**

```toml
[package]
builder = "python"
vendor = "logical-minds-foundry"
summary = "Stdlib-only HA/DR state collectors for IBM MQ (Native HA, RDQM, Pacemaker) emitting node_exporter textfile metrics"
smoke = "mq-resiliency-observability-selfcheck"

[package.python]
runtime = "3.14.8"
```

**`packaging/nfpm.overlay.yaml`:** units only. There are no maintainer scripts: the deployer
enables units, and nFPM rejects raw `systemctl` anyway.

```yaml
contents:
  - { src: systemd/lab-cluster-state.service,  dst: /usr/lib/systemd/system/lab-cluster-state.service }
  - { src: systemd/lab-cluster-state.timer,    dst: /usr/lib/systemd/system/lab-cluster-state.timer }
  - { src: systemd/lab-nativeha-state.service, dst: /usr/lib/systemd/system/lab-nativeha-state.service }
  - { src: systemd/lab-nativeha-state.timer,   dst: /usr/lib/systemd/system/lab-nativeha-state.timer }
  - { src: systemd/lab-rdqm-state.service,     dst: /usr/lib/systemd/system/lab-rdqm-state.service }
  - { src: systemd/lab-rdqm-state.timer,       dst: /usr/lib/systemd/system/lab-rdqm-state.timer }
```

**ci.yml addition:**

```yaml
  package:
    uses: vergil-project/vergil-actions/.github/workflows/ci-package.yml@v2.1
```

- [ ] **Step 1: Import.** Copy the tree from the recorded lab commit (via a read-only
  checkout of the lab at that commit). Run
  `git -C <lab> rev-parse <commit>:components/mq-resiliency-observability` for the tree hash.
- [ ] **Step 2: Add the README "Provenance" section.** It records three things:
  - lineage: first attempt #79 (a failed iteration, overwritten), then in-lab prototype
    epic `.github#294`, then extraction epic `.github#319`;
  - the lab repo, commit and component tree hash;
  - the fact that the history is not carried.
- [ ] **Step 3: Apply the scaffolding edits listed above.**
- [ ] **Step 4: Check the package build locally.** In the dev VM, run
  `vrg-package build` for one target, then
  `vrg-package install-test --target ubuntu24-arm64` (or the matrix cell name
  `vrg-package matrix` prints).

  Expected: the install test reports `systemd-analyze verify` passing for the six units,
  `smoke: pass`, and no residue after purge. Fix any unit `verify` complaint in the unit
  file.
- [ ] **Step 5: Validate.** Run `vrg-container-run -- vrg-validate`. The bar is 100% branch
  coverage: add tests where the component's own tests fall short, and never lower the bar.
- [ ] **Step 6: Commit and report ready.** Commit with
  `vrg-commit --type feat --scope package --message "import mq-resiliency-observability from the lab and package it"`.
  The body carries the provenance line (lab commit, tree hash, lineage). Then run
  `report-ready`.
  - The PR notes that open PR #37 and issues #28/#36/#42 refer to the overwritten tree. The
    human closes them as superseded.
- **Acceptance:** CI's `package / evidence` gate is green on all 8 target cells.

---

## Operational X2 (deployment; release human-gated): `0.2.0` on the LMF index

Created with `--kind deployment` in the product repo, `--blocked-by` X1 and KR/PUB, and a
dependency note on REG (cross-org).

- **Preconditions (machine-checkable):**
  1. REG's vergil-tooling release is the one the release container uses. Check that the
     `cd-release` run log's `vrg-version` line is ≥ REG's release.
  2. The LMF key URL returns 200.
- **Human-gated:** the human cuts `v0.2.0` from `main`. The agent never does.
- **Procedure (agent):** wait for the `package-index` stage. Then confirm the package is on
  the index:
  - `curl -fsSL https://logical-minds-foundry.github.io/packages/deb/dists/noble/main/binary-arm64/Packages | grep -A3 '^Package: mq-resiliency-observability$'`
    shows `Version: 0.2.0-1` and
    `Depends: vergil-python3.14.8 (>= 3.14.8+20261003)`;
  - the rpm `repodata` for `el9/x86_64` and `el10/x86_64` lists `0.2.0-1`.
- **Acceptance:** record `Outcome: SUCCESS` with the three index excerpts.

---

## Task L1: Lab — packages catalog, trust reader, `package-repos` and `package-install`

**Files:**

- Modify: `lab/versions.yaml`. Add the `packages:` block (spec §5.3) after `runtime:`.
  `runtime:` stays until L2.
- Modify: `src/mqlab/versions.py`:
  - `_TOP_KEYS` += `"packages"`;
  - new `_PACKAGES_KEYS = ("repos", "pins")`;
  - `PackageRepo`, `RuntimePackage`, `PackagesCatalog` dataclasses;
  - `_packages(raw) -> PackagesCatalog`;
  - `Catalog.packages`.
- Create: `src/mqlab/packages.py`, holding the trust reader and the trust-vars writer.
- Modify: `src/mqlab/cli.py`. Add `package_app` (`mqlab package`) with a `trust` sub-app and
  a `list` command.
- Create: `ansible/group_vars/all/packages.yml`.
- Create: `ansible/roles/package-repos/{tasks/main.yml,defaults/main.yml}`.
- Create: `ansible/roles/package-install/{tasks/main.yml,defaults/main.yml}`.
- Create: `ansible/package-install.yml`, a play: `hosts: all`, an `ansible_limit` pre-check
  (copied from `component-install.yml`), `roles: [package-install]`.
- Create: `tests/test_packages.py`, `tests/test_cli_package.py`,
  `tests/test_package_roles.py`. Extend `tests/test_versions.py`.
- Modify: `docs/site/docs/operate/guest-components.md`. Add a "Package trust" section: the
  host commands, the sync, and `mqlab package trust list`.

**Interfaces (produced):**

```python
# versions.py
@dataclass(frozen=True)
class PackageRepo:
    org: str
    url: str                                   # https://…/packages, no trailing slash

@dataclass(frozen=True)
class RuntimePackage:
    name: str                                  # "vergil-python3.14.8"
    pin: str                                   # "3.14.8+20261003-1"
    @property
    def python(self) -> str: ...               # "3.14.8"
    @property
    def minor(self) -> str: ...                # "3.14"
    @property
    def interpreter(self) -> str: ...          # "/opt/vergil/python/3.14.8/bin/python3.14"

@dataclass(frozen=True)
class PackagesCatalog:
    repos: dict[str, PackageRepo]
    pins: dict[str, str]
    @property
    def runtime(self) -> RuntimePackage: ...

# packages.py
class TrustError(RuntimeError): ...
TRUST_FILE: Path                               # Path.home() / ".config/vergil/package-trust.toml"
def trust_path() -> Path: ...
def load_anchors(path: Path | None = None) -> dict[str, str]: ...   # format per Global Constraints
def require_anchors(orgs: Iterable[str], path: Path | None = None) -> dict[str, str]: ...
def trust_vars_path() -> Path: ...             # paths.work("packages", "trust-vars.json")
def write_trust_vars(catalog: PackagesCatalog) -> Path: ...  # {"package_trust": {org: fpr}} for EVERY catalog repo
def trust_args(catalog: PackagesCatalog) -> list[str]: ...  # ["-e", f"@{write_trust_vars(catalog)}"]
def render_trust(catalog: PackagesCatalog) -> str: ...      # the `mqlab package trust list` table
```

The `require_anchors` error text, exactly:

```text
mqlab: no package trust anchor for org '<org>' in <file>.
  On the HOST (not in this VM): vrg-package trust add <org> <PRIMARY-FINGERPRINT>
  then re-enter the VM (vrg-vm session) to sync it. The fingerprint comes from the org's
  key-ceremony record; never copy it from the repository you are about to trust.
```

**Ansible variables (produced):**

- `package_catalog` (group_vars):
  `"{{ (lookup('file', playbook_dir + '/../lab/versions.yaml') | from_yaml).packages }}"`.
  It has `repos: {org: {url}}` and `pins: {name: pin}`.
- `package_runtime_name` (group_vars): the single pins key matching `^vergil-python\d`.
- `package_trust` (`-e @trust-vars.json`): `{org: fingerprint}`.
- `package_names` (a `package-install` input, default `[]`): the names to install. The role
  always adds `package_runtime_name`.

- [ ] **Step 1: Write the failing catalog tests** (`tests/test_versions.py`):

  ```python
  def test_committed_packages_block() -> None:
      cat = load_catalog()
      assert cat.packages.repos["vergil"].url == "https://vergil-project.github.io/packages"
      assert cat.packages.repos["logical-minds-foundry"].url == "https://logical-minds-foundry.github.io/packages"
      rt = cat.packages.runtime
      assert (rt.name, rt.pin, rt.interpreter) == (
          "vergil-python3.14.8", "3.14.8+20261003-1", "/opt/vergil/python/3.14.8/bin/python3.14")

  @pytest.mark.parametrize(("keys", "value", "match"), [
      (("packages", "repos", "vergil", "url"), "http://x/packages", "https"),
      (("packages", "repos", "vergil", "fingerprint"), "A" * 40, "fingerprint"),   # D7: never in the catalog
      (("packages", "pins", "mq-resiliency-observability"), "0.2.0", "revision"),
      (("packages", "pins", "mq-resiliency-observability"), "0.2.0-1~noble", "revision"),
      (("packages", "pins", "vergil-python3.14.8"), "3.14.9+20261101-1", "vergil-python3.14.8"),
      (("packages", "pins", "vergil-python3.14.9"), "3.14.9+20261101-1", "exactly one"),
  ])
  def test_packages_block_refuses(tmp_path, keys, value, match) -> None:
      with pytest.raises(VersionError, match=match):
          load_catalog(_write(tmp_path, _mutated(keys, value)))
  ```

  - The `fingerprint` row means any key other than `url` is refused, and the message names
    D7's rule: "fingerprints live in ~/.config/vergil/package-trust.toml".
  - The second `vergil-python` row adds a second runtime pin.
  - The `"vergil-python3.14.8"` row checks that the pin's version starts with the name's
    `X.Y.Z+`.

- [ ] **Step 2: Write the failing trust tests** (`tests/test_packages.py`). Use
  `monkeypatch.setenv("HOME", str(tmp_path))`:

  ```python
  def test_require_anchors_names_host_command(home) -> None:
      with pytest.raises(packages.TrustError) as e:
          packages.require_anchors(["logical-minds-foundry"])
      msg = str(e.value)
      assert "vrg-package trust add logical-minds-foundry" in msg and "vrg-vm session" in msg
      assert str(home / ".config/vergil/package-trust.toml") in msg

  def test_write_trust_vars_covers_every_catalog_org(home, anchors_file) -> None:
      p = packages.write_trust_vars(load_catalog().packages)
      assert json.loads(p.read_text()) == {"package_trust": {
          "logical-minds-foundry": "C" * 40, "vergil": "B3A1D804DC03AB036566A4E367861822166ECABE"}}

  def test_load_anchors_refuses_bad_fingerprint(home) -> None:
      _anchors(home, {"vergil": "nothex"})
      with pytest.raises(packages.TrustError, match="40"):
          packages.load_anchors()

  def test_render_trust_marks_missing(home, anchors_file_vergil_only) -> None:
      out = packages.render_trust(load_catalog().packages)
      assert "logical-minds-foundry" in out and "MISSING" in out
  ```

  In `tests/test_cli_package.py`:
  - `mqlab package trust list` exits 0 and prints `ORG  FINGERPRINT  STATUS`, even when an
    anchor is missing (`list` reports; it does not fail);
  - `mqlab package trust add` does not exist (exit 2), because authoring is host-side.

- [ ] **Step 3: Write the failing structural role tests** (`tests/test_package_roles.py`).
  They reuse the `_load`, `_flat` and `_tasks(role)` helpers from `tests/test_install_roles.py`;
  import them, or move them to `tests/roles.py` if the import is awkward.

  ```python
  def test_package_repos_verifies_before_any_source_is_written() -> None:
      t = _flat(_tasks("package-repos"))
      verify = _index(t, "assert the primary fingerprint equals the anchor")
      for writer in ("install the verified apt key", "write the apt source",
                     "install the verified rpm key", "write the dnf repository"):
          assert _index(t, writer) > verify

  def test_package_repos_rpm_baseurl_uses_literal_major() -> None:
      repo = _task(_tasks("package-repos"), "write the dnf repository")["ansible.builtin.yum_repository"]
      assert "ansible_distribution_major_version" in repo["baseurl"]
      assert "releasever" not in repo["baseurl"]
      assert (repo["gpgcheck"], repo["repo_gpgcheck"]) == (True, True)

  def test_package_repos_requires_every_anchor_first() -> None:
      first = _flat(_tasks("package-repos"))[0]
      assert "ansible.builtin.assert" in first and "package_trust" in str(first)
      assert "vrg-package trust add" in str(first)

  def test_package_repos_never_installs_a_keyring_package() -> None:
      assert "archive-keyring" not in yaml.safe_dump(_tasks("package-repos"))

  def test_package_install_refuses_in_lab_layout_first() -> None:
      t = _flat(_tasks("package-install"))
      refuse = _index(t, "refuse to install over an in-lab layout")
      assert refuse < _index(t, "install the pinned packages (apt)")
      assert "mqlab box build" in str(t[refuse])

  def test_package_install_allows_downgrade_and_asserts_pin() -> None:
      t = _flat(_tasks("package-install"))
      for name in ("install the pinned packages (apt)", "install the pinned packages (dnf)"):
          mod = _task(t, name)
          body = mod.get("ansible.builtin.apt") or mod.get("ansible.builtin.dnf")
          assert body["allow_downgrade"] is True
      assert _index(t, "assert each installed version equals its pin") > _index(t, "install the pinned packages (dnf)")

  def test_package_install_failure_names_the_pins() -> None:
      blk = _task(_tasks("package-install"), "install the pinned packages")
      assert "lab/versions.yaml" in str(blk["rescue"])

  def test_package_install_always_adds_the_runtime() -> None:
      assert "package_runtime_name" in str(_task(_tasks("package-install"), "the names to install"))
  ```

  - `_task(tasks, name)` returns the task dict with that `name`.
  - `_index(tasks, name)` returns its position in the flattened list (blocks expanded).

- [ ] **Step 4: Run the tests and see them fail.** Run
  `uv run pytest tests/test_versions.py tests/test_packages.py tests/test_cli_package.py tests/test_package_roles.py -v`.

- [ ] **Step 5: Add the catalog block** to `lab/versions.yaml`. Put a comment block above it
  stating: pins are full `<version>-<revision>`; the single `vergil-python<X.Y.Z>` pin is the
  runtime; there are no fingerprints (D7).

  ```yaml
  packages:
    repos:                         # org -> repository URL (NO fingerprints: D7)
      vergil:                { url: "https://vergil-project.github.io/packages" }
      logical-minds-foundry: { url: "https://logical-minds-foundry.github.io/packages" }
    pins:
      vergil-python3.14.8:         "3.14.8+20261003-1"
      mq-resiliency-observability: "0.2.0-1"
  ```

- [ ] **Step 6: Implement the `versions.py` parsing.**
  - `_packages(raw)` uses `_mapping` and `_check_keys`.
  - Repo org slugs match `^[a-z0-9][a-z0-9-]*$`. The URL matches
    `^https://[^\s]+[^/]$`, and is the only allowed key.
  - Pin names match `^[a-z0-9][a-z0-9.+-]*$`. Pin values match
    `^[0-9][0-9A-Za-z.+~]*-[0-9]+$`, with the revision an integer; the message names
    "revision".
  - Exactly one name matches `^vergil-python(\d+\.\d+\.\d+)$`, and its pin starts with
    `<X.Y.Z>+`.
  - Every error ends with `— {_CATALOG_FIX}`.

- [ ] **Step 7: Implement `packages.py` and the CLI.**
  - `render_trust` prints `ORG  FINGERPRINT  STATUS`, with `STATUS` being `trusted` or
    `MISSING (vrg-package trust add <org> …, on the host)`.
  - The CLI catches `(packages.TrustError, VersionError)` and exits 1.

- [ ] **Step 8: Write `ansible/group_vars/all/packages.yml`.**

  ```yaml
  ---
  # Package catalog (epic .github#319 §5.3): repos + exact pins, read from the ONE
  # authoritative lab/versions.yaml on the controller (as mq_version reads lab/mq-version).
  # Per-user trust anchors are NOT here: mqlab passes them as package_trust (-e @trust-vars).
  package_catalog: "{{ (lookup('file', playbook_dir + '/../lab/versions.yaml') | from_yaml).packages }}"
  package_runtime_name: "{{ package_catalog.pins.keys() | select('match', '^vergil-python[0-9]') | first }}"
  ```

- [ ] **Step 9: Write `package-repos/tasks/main.yml`.** Tasks in this order, with these
  exact names, so the structural tests can find them:

  ```yaml
  ---
  # Trust each org's package repository with the standard third-party recipe (spec §5.3):
  # fetch the published key -> verify its PRIMARY fingerprint against the per-user anchor ->
  # only then install the key and write the source. No keyring package (#324 owns rotation).
  - name: every catalog org has a trust anchor
    ansible.builtin.assert:
      that: package_trust is defined and (package_catalog.repos.keys() | difference(package_trust.keys()) | length == 0)
      fail_msg: >-
        no package trust anchor for {{ package_catalog.repos.keys() | difference((package_trust | default({})).keys()) | join(', ') }}
        — on the HOST run `vrg-package trust add <org> <fingerprint>`, re-enter the VM
        (vrg-vm session), and run this through mqlab (it passes the anchors).

  - name: per-org trust
    ansible.builtin.include_tasks: org.yml
    loop: "{{ package_catalog.repos | dict2items }}"
    loop_control: { loop_var: _org, label: "{{ _org.key }}" }
  ```

  `package-repos/tasks/org.yml`:

  ```yaml
  ---
  - name: fetch the published key
    ansible.builtin.get_url:
      url: "{{ _org.value.url }}/keys/{{ _org.key }}.asc"
      dest: "/var/tmp/{{ _org.key }}.asc"
      mode: "0644"
      force: true
    become: true

  - name: read the key's fingerprints
    ansible.builtin.command: gpg --batch --show-keys --with-colons /var/tmp/{{ _org.key }}.asc
    register: _keyinfo
    changed_when: false

  - name: assert the primary fingerprint equals the anchor
    vars:
      _lines: "{{ _keyinfo.stdout_lines }}"
      _pubs: "{{ _lines | select('match', '^pub:') | list }}"
      # the first fpr record follows the (single) pub record: it is the PRIMARY key's
      _primary: "{{ (_lines | select('match', '^fpr:') | first).split(':')[9] }}"
    ansible.builtin.assert:
      that:
        - _pubs | length == 1
        - _primary == package_trust[_org.key]
      fail_msg: >-
        {{ _org.key }} key at {{ _org.value.url }}/keys/{{ _org.key }}.asc has primary
        fingerprint {{ _primary }}, anchored {{ package_trust[_org.key] }} — refusing to trust it

  - name: install the verified apt key
    when: ansible_os_family == "Debian"
    ansible.builtin.copy:
      src: "/var/tmp/{{ _org.key }}.asc"
      remote_src: true
      dest: "/etc/apt/keyrings/{{ _org.key }}.asc"
      mode: "0644"
    become: true

  - name: write the apt source
    when: ansible_os_family == "Debian"
    ansible.builtin.copy:
      dest: "/etc/apt/sources.list.d/{{ _org.key }}.sources"
      mode: "0644"
      content: |
        Types: deb
        URIs: {{ _org.value.url }}/deb
        Suites: {{ ansible_distribution_release }}
        Components: main
        Signed-By: /etc/apt/keyrings/{{ _org.key }}.asc
    become: true
    register: _apt_source

  - name: refresh apt indexes when a source changed
    when: ansible_os_family == "Debian" and _apt_source is changed
    ansible.builtin.apt: { update_cache: true }
    become: true

  - name: install the verified rpm key
    when: ansible_os_family == "RedHat"
    ansible.builtin.copy:
      src: "/var/tmp/{{ _org.key }}.asc"
      remote_src: true
      dest: "/etc/pki/rpm-gpg/RPM-GPG-KEY-{{ _org.key }}"
      mode: "0644"
    become: true

  - name: import the verified rpm key into the rpm database
    when: ansible_os_family == "RedHat"
    ansible.builtin.rpm_key:
      key: "/etc/pki/rpm-gpg/RPM-GPG-KEY-{{ _org.key }}"
      fingerprint: "{{ package_trust[_org.key] }}"
    become: true

  - name: write the dnf repository
    when: ansible_os_family == "RedHat"
    ansible.builtin.yum_repository:
      name: "{{ _org.key }}"
      description: "{{ _org.key }} packages"
      baseurl: "{{ _org.value.url }}/rpm/el{{ ansible_distribution_major_version }}/$basearch"
      gpgcheck: true
      repo_gpgcheck: true
      gpgkey: "file:///etc/pki/rpm-gpg/RPM-GPG-KEY-{{ _org.key }}"
      enabled: true
    become: true
  ```

  The structural test flattens `include_tasks` targets: `_tasks("package-repos")` loads
  `main.yml` and inlines `org.yml` at the include. If the helper can't, extend it.

- [ ] **Step 10: Write `package-install/tasks/main.yml`.** Set
  `defaults/main.yml: package_names: []`.

  ```yaml
  ---
  # Install exact pins from the trusted repositories (spec §5.4). The runtime pin is ALWAYS
  # installed explicitly, so it never floats as a dependency. M1 -> M3 is a cold rebuild:
  # never install over an in-lab layout (dpkg/rpm would follow the venv symlink).
  - name: the names to install
    ansible.builtin.set_fact:
      _pkg_names: "{{ (package_names + [package_runtime_name]) | unique }}"

  - name: the exact specs (apt name=pin, dnf name-pin)
    ansible.builtin.set_fact:
      _apt_specs: "{{ _pkg_names | zip(_pkg_names | map('extract', package_catalog.pins)) | map('join', '=') | list }}"
      _dnf_specs: "{{ _pkg_names | zip(_pkg_names | map('extract', package_catalog.pins)) | map('join', '-') | list }}"
    when: _pkg_names | difference(package_catalog.pins.keys()) | length == 0

  - name: every name is pinned
    ansible.builtin.assert:
      that: _pkg_names | difference(package_catalog.pins.keys()) | length == 0
      fail_msg: "unpinned package(s) {{ _pkg_names | difference(package_catalog.pins.keys()) }} — pin them in lab/versions.yaml packages.pins"

  - name: look for an in-lab layout
    ansible.builtin.stat: { path: "/opt/logical-minds-foundry/{{ item }}/releases" }
    loop: "{{ _pkg_names }}"
    register: _inlab

  - name: look for a symlinked venv
    ansible.builtin.stat: { path: "/opt/logical-minds-foundry/{{ item }}/venv", follow: false }
    loop: "{{ _pkg_names }}"
    register: _venvlink

  - name: refuse to install over an in-lab layout
    ansible.builtin.assert:
      that:
        - _inlab.results | selectattr('stat.exists') | list | length == 0
        - _venvlink.results | selectattr('stat.islnk', 'defined') | selectattr('stat.islnk') | list | length == 0
      fail_msg: >-
        {{ inventory_hostname }} carries an in-lab install where a package would go —
        M1 -> M3 is a cold rebuild: mqlab box build <box>, then bring the stack up again

  - name: trust the repositories
    ansible.builtin.include_role: { name: package-repos }

  - name: install the pinned packages
    block:
      - name: install the pinned packages (apt)
        when: ansible_os_family == "Debian"
        ansible.builtin.apt:
          name: "{{ _apt_specs }}"
          allow_downgrade: true
          state: present
        become: true
      - name: install the pinned packages (dnf)
        when: ansible_os_family == "RedHat"
        ansible.builtin.dnf:
          name: "{{ _dnf_specs }}"
          allow_downgrade: true
          state: present
        become: true
    rescue:
      - name: name the pins
        ansible.builtin.fail:
          msg: >-
            installing {{ _pkg_names }} at their pins failed (see the package tool's error
            above). Check lab/versions.yaml packages.pins: the app pins and the runtime pin
            ({{ package_runtime_name }} {{ package_catalog.pins[package_runtime_name] }}) must
            be published and mutually installable.

  - name: read the installed versions
    ansible.builtin.package_facts: {}

  - name: assert each installed version equals its pin
    vars:
      _got: >-
        {{ ansible_facts.packages[item][0].version if ansible_os_family == 'Debian'
           else ansible_facts.packages[item][0].version ~ '-' ~ ansible_facts.packages[item][0].release }}
    ansible.builtin.assert:
      that: _got == package_catalog.pins[item]
      fail_msg: "{{ item }} is {{ _got }} on {{ inventory_hostname }}, pinned {{ package_catalog.pins[item] }} (lab/versions.yaml)"
    loop: "{{ _pkg_names }}"
  ```

- [ ] **Step 11: Run the tests and see them pass. Then validate** with
  `vrg-container-run -- vrg-validate`. This includes `ansible-lint`/`yamllint`, if
  vrg-validate runs them.
- [ ] **Step 12: Live smoke (if a lab is running; otherwise say so in the PR).** This step
  needs TRUST-SYNC or a hand-written VM anchor file. Run
  `mqlab package trust list`, then `cd ansible && ansible-playbook package-install.yml --limit <one ubuntu guest> -e @$(mqlab build path work)/packages/trust-vars.json`.
  This trusts both orgs and installs only the runtime pin. Expected: the
  `assert each installed version equals its pin` task passes for `vergil-python3.14.8`.
  Run it twice; the second run reports `changed=0` for the source and key tasks. Record the
  output in the PR body.
- [ ] **Step 13: Commit and report ready.** Commit with
  `vrg-commit --type feat --scope packages --message "add the package catalog, trust anchors and package roles"`,
  then run `report-ready`.

---

## Task L2: Lab — in-lab tier on `vergil-python3.14.8`; retire `runtime-install`

**Files:**

- Modify: `lab/versions.yaml`. Delete `runtime:`, lines 66-78 including its comment block.
- Modify: `src/mqlab/versions.py`. Delete `RuntimePin`, `_RUNTIME_KEYS`, `_PYTHON_KEYS`,
  `_PBS_DOWNLOADS`, `_runtime()` and `Catalog.runtime`; `_TOP_KEYS` drops `"runtime"`. Add
  `BoxEntry.packages: tuple[str, ...]`, which is the runtime name when `components` is
  non-empty, else `()` (L3 extends it).
- Modify: `src/mqlab/runtime.py`. Replace the tarball code (`ensure_tarball`,
  `ensure_interpreter`, `unpack_root`, `interpreter_path`) with
  `dev_interpreter(pkg: RuntimePackage, runner) -> Path`. Keep `host_arch()` and
  `RuntimePinError`.
- Modify: `src/mqlab/component.py`:
  - `build()` and `ensure_built()` take `runtime: RuntimePackage` instead of
    `pin`/`fetch`, and call `dev_interpreter`.
  - `install_vars(artifacts)` writes
    `{"package_runtime": {"name","pin","interpreter"}, "component_artifacts": {…}}`.
  - `install_command` adds `*packages.trust_args(catalog.packages)`.
  - `render_status` reads `runtime_installed` as the package version.
- Modify: `ansible/component-install.yml`. Roles become `[package-install, component-install]`.
  `package-install` gets the default `package_names: []`, so only the runtime is installed.
- Modify: `ansible/roles/component-install/{defaults,tasks}/main.yml`:
  - the precondition asserts `package_facts` show `package_runtime.name` at
    `package_runtime.pin`, instead of the `PIN` file;
  - `_c_runtime: "{{ package_runtime.interpreter }}"`;
  - `component_config_prefix: /etc/logical-minds-foundry`;
  - `INSTALLED.json` `runtime_installed` is the package pin.
- Delete: `ansible/roles/runtime-install/`.
- Modify: the six bake playbooks (`bake-san.yml:80`, `bake-nativeha-rhel.yml:73`,
  `bake-nativeha-ubuntu.yml:89`, `bake-pcmk-ubuntu.yml:93`, `bake-mq-rdqm.yml:84`,
  `bake-mq-ubuntu.yml:71`). Replace the `runtime-install` include with:

  ```yaml
      - name: pinned packages for this box (the runtime + any packaged products; epic .github#319)
        ansible.builtin.include_role:
          name: package-install
        vars:
          package_names: "{{ baked_packages }}"
  ```

  Update each playbook's comment to name `package_runtime` and the trust vars.
- Modify: `ansible/roles/mq-client/tasks/main.yml:7` (its `runtime-install` reference).
- Modify: `ansible/roles/app-requester/tasks/main.yml:81` and
  `ansible/roles/mq-inter-qm/tasks/main.yml:110`, changing `/etc/opt/logical-minds-foundry`
  to `/etc/logical-minds-foundry`.
- Modify: `components/mq-resiliency-clients/systemd/mq-app-requester.service:18` and
  `mq-svc-responder@.service:19` (`EnvironmentFile`), plus `components/mq-resiliency-clients/README.md:51`
  and `components/README.md:53,92`.
- Modify: `src/mqlab/box.py`:
  - `BoxSpec.runtime_pin` becomes `packages: tuple[tuple[str, str], ...]` (name and pin,
    sorted);
  - `_component_args` becomes `_package_args`, which emits
    `--packages "<n>=<pin>,…"` (an empty string when there are none), `--components …`, and,
    when packages or components are non-empty,
    `--install-vars <path> --package-trust <trust-vars path>`;
  - `_build_plan` calls `packages.write_trust_vars(catalog.packages)` **before** any step
    when any box in the plan will bake with packages.
- Modify: `lab/boxes/_manifest-hash.sh`. `--runtime-pin` becomes `--packages`: entries match
  `^[a-z0-9][a-z0-9.+-]*=[0-9][0-9A-Za-z.+~]*-[0-9]+$`, and the digest line
  `runtime_pin=` becomes sorted `package=<name>=<pin>` lines, emitted when non-empty.
- Modify: `lab/boxes/build-fatbox.sh`:
  - accept `--packages` and `--package-trust`; `--package-trust` is required when
    `--packages` is non-empty, failing in the existing style;
  - pass `--packages` to `_manifest-hash.sh`;
  - `BAKE_EXTRA_VARS += (-e "@$PACKAGE_TRUST" -e "{\"baked_packages\": <json list of names>}")`;
  - drop the unused `baked_components` extra-var.
- Modify: `lab/boxes/_components.sh`. `components_record <packages> <components>` writes
  `{"packages": {…}, "components": {…}}`.
- Modify tests:
  - `tests/test_versions.py`: drop the `runtime` tests and add
    `test_box_packages_include_runtime_iff_components`;
  - `tests/test_runtime.py`: rewrite it for `dev_interpreter`;
  - `tests/test_component.py`: the `PIN` fixture becomes a `RuntimePackage`;
  - `tests/test_install_roles.py`: delete the `runtime-install` tests, and rewrite
    `test_component_install_asserts_runtime_pin_first` as
    `test_component_install_asserts_runtime_package_first`;
  - `tests/test_box_components.py`, `tests/boxfleet.py`, `tests/test_box_build.py`: replace
    the flag name `--runtime-pin` with `--packages`;
  - `tests/test_client_roles_rewired.py:31`, `tests/test_collector_roles_rewired.py:31`
    (`ENV_DIR` for clients only; observability keeps `/etc/opt` until L3),
    `tests/test_clients_component_baked.py`, `tests/test_stage_off_tmpfs.py:304,322`.
- Modify docs: `docs/site/docs/operate/guest-components.md` (runtime section, lines 56,
  61, 71 and 75) and `docs/development/box-bake-manifest.md` (the `runtime_pin` /
  `runtime-install` mentions become `--packages` / `package-install`).

**Interfaces:**

- Consumes (L1): `PackagesCatalog`, `RuntimePackage`, `packages.trust_args`,
  `packages.write_trust_vars`, `packages.trust_vars_path`, the `package-install` role,
  `package_catalog`, `package_runtime_name`.
- Produces (used by L3):
  - `BoxEntry.packages`, `BoxSpec.packages`;
  - the builder flags `--packages` and `--package-trust`;
  - the bake var `baked_packages`;
  - `install_vars()` output key `package_runtime`.

```python
def dev_interpreter(pkg: RuntimePackage, runner: Runner) -> Path:
    """The dev VM's interpreter for `mqlab component build`: the SAME package the guests get.

    Asserts `dpkg-query -W -f='${Version}' <pkg.name>` == pkg.pin and that the interpreter
    exists. The dev VM gets vergil-python only as vergil-tooling's dependency (vrg-vm's
    profile packages are installed before the vergil repo is configured), so a mismatch means
    the lab pin and vergil-tooling's runtime have diverged.
    """
```

Its error, when the package is absent or the version differs:

```text
mqlab: the dev VM has <name> <got|not installed>; lab/versions.yaml pins <pin>.
  The dev VM's runtime comes from vergil-tooling (installed vergil-python packages: <list>).
  Move the lab's runtime pin to a runtime the dev VM has (lab/versions.yaml packages.pins),
  so the components are tested on exactly what the guests run.
```

- [ ] **Step 1: Write the failing tests:**

  ```python
  # tests/test_runtime.py
  def test_dev_interpreter_asserts_package_pin(tools) -> None:
      tools.on("dpkg-query", stdout="3.14.8+20261003-1")
      assert runtime.dev_interpreter(RT, tools.runner) == Path("/opt/vergil/python/3.14.8/bin/python3.14")

  def test_dev_interpreter_refuses_mismatch_naming_pin(tools) -> None:
      tools.on("dpkg-query", stdout="3.14.8+20261101-1")
      with pytest.raises(RuntimePinError, match="lab/versions.yaml pins 3.14.8\\+20261003-1"):
          runtime.dev_interpreter(RT, tools.runner)

  # tests/test_versions.py
  def test_box_packages_include_runtime_iff_components() -> None:
      cat = load_catalog()
      assert cat.box("mq-client", OsRef("ubuntu", 24)).packages == ("vergil-python3.14.8",)
      assert cat.box("infra", OsRef("ubuntu", 26)).packages == ()

  # tests/test_box_build.py
  def test_build_plan_requires_anchors_before_any_bake(no_anchors, baking) -> None:
      with pytest.raises(packages.TrustError):
          box.build_boxes(["mq-ubuntu24"], force=True)
      assert baking.commands == []          # nothing ran: no builder, no ansible

  def test_manifest_hash_flips_on_package_pin_change() -> None:
      a = _hash(manifest_hash_args("mq-ubuntu24"))
      b = _hash(_with(manifest_hash_args("mq-ubuntu24"), "--packages", "vergil-python3.14.8=3.14.8+20261101-1"))
      assert a != b

  # tests/test_install_roles.py
  def test_component_install_asserts_runtime_package_first() -> None:
      t = _flat(_tasks("component-install"))
      assert "package_runtime" in str(t[0]) or "package_facts" in str(t[0])
      assert not (REPO / "ansible/roles/runtime-install").exists()

  def test_bake_playbooks_install_packages_not_tarball() -> None:
      for pb in BAKES:  # the six bake playbooks
          text = (REPO / "ansible" / pb).read_text()
          assert "runtime-install" not in text and "package-install" in text

  def test_no_retired_runtime_paths_anywhere() -> None:
      for p in _tracked(["ansible", "components", "src", "lab"]):
          assert "cpython-3.14" not in p.read_text(errors="ignore")
  ```

  Also: `test_client_env_files_under_etc_logical_minds_foundry` (`test_client_roles_rewired.py`).

- [ ] **Step 2: Run the tests and see them fail** (`uv run pytest -x`).
- [ ] **Step 3: Make the source edits listed above, in this order:**
  1. `versions.py`;
  2. `runtime.py`;
  3. `component.py`;
  4. `box.py`;
  5. the shell builders;
  6. the roles and playbooks;
  7. the clients config path;
  8. the docs.

  Run `grep -rn "runtime_python\|runtime-install\|cpython-3.14\|RuntimePin\b\|runtime_pin" src ansible lab components tests docs/site docs/development`.
  The only remaining hits should be in historical specs and plans under `docs/` (leave
  those).
- [ ] **Step 4: Run the tests and see them pass. Then validate** with
  `vrg-container-run -- vrg-validate`.
- [ ] **Step 5: Live check (this requires anchors in the VM).**
  - `mqlab component build mq-resiliency-clients` passes on
    `/opt/vergil/python/3.14.8/bin/python3.14`. Its log names the interpreter.
  - `mqlab box rebuild mq-ubuntu24` bakes it. The bake log shows
    `--packages vergil-python3.14.8=3.14.8+20261003-1`, and `package-install` asserts the
    pin. Then `mqlab component install mq-resiliency-clients --host <a client node>` on a
    running stack is green.

  If no lab is available, say so in the PR; V1 does the cold rebuild. Record the output in
  the PR.
- [ ] **Step 6: Commit and report ready.** Commit with
  `vrg-commit --type feat --scope components --message "run in-lab components on the vergil-python package; retire runtime-install"`,
  then run `report-ready`.

---

## Task L3: Lab — observability from the package; remove the in-lab copy

Everything below lands in **one PR** (spec §7), so develop never lacks a working
observability install.

**Files:**

- Modify: `lab/versions.yaml` roles `san`, `mq-nativeha`, `pcmk`, `mq-rdqm` (lines 43, 45-48,
  50, 51). Replace `components: [mq-resiliency-observability]` with
  `packages: [mq-resiliency-observability]`.
- Modify: `src/mqlab/versions.py`:
  - `_ROLE_KEYS` += `"packages"`;
  - `_role_packages(raw, where, pins)`: a list of strings with no duplicates; each must be a
    pins key and must **not** be the runtime (the runtime is implicit);
  - `BoxEntry.packages` = role packages ∪ {runtime when packages **or** components}, sorted;
  - a new `Catalog.packaged_names() -> set[str]`, the union of every role's packages.
- Delete: `components/mq-resiliency-observability/`.
- Modify: `components/README.md`. Add an "Extracted components" section:
  `mq-resiliency-observability` now lives in
  `logical-minds-foundry/mq-resiliency-observability`, is installed from the
  logical-minds-foundry package repository at the `lab/versions.yaml` pin, and is changed by
  release-to-test (`vergil-project/.github#370` is the future pre-release channel).
- Modify: the bake playbooks `bake-san.yml`, `bake-nativeha-rhel.yml`,
  `bake-nativeha-ubuntu.yml`, `bake-pcmk-ubuntu.yml` and `bake-mq-rdqm.yml`. Delete the
  `component-install` include for observability. The `package-install` include from L2 now
  installs it through `baked_packages`; units arrive inert from the package.
- Modify: `ansible/roles/{cluster-state,nativeha-state,rdqm-state}/tasks/main.yml`:
  - Replace the "baked?" stat and assert (cluster-state 8-21, nativeha-state 10-23, rdqm-state
    10-23) with:

    ```yaml
    - name: the pinned observability package (re-installed at provision; spec §5.4)
      ansible.builtin.include_role:
        name: package-install
      vars:
        package_names: [mq-resiliency-observability]
    ```

    `package-install` refuses an in-lab layout, installs or downgrades to the pin, and
    asserts the installed version.
  - Create the config directory `/etc/logical-minds-foundry/mq-resiliency-observability`
    (`0755`) before rendering.
  - The env-file `dest` becomes `/etc/logical-minds-foundry/mq-resiliency-observability/<unit>.env`
    (cluster-state:70, nativeha-state:90, rdqm-state:72).
  - Update the header comments: the collector is the packaged product, and the units come
    from the package.
  - Update each env template's line 1.
- Modify: `src/mqlab/phases.py`. Append `*packages.trust_args(load_catalog().packages)` to the
  provision command (`:543`) and the `observability.yml` command (`:632`). Then check every
  other `ansible-playbook` call that reaches a state role:
  `grep -rn "observability.yml\|validate-nativeha-log-lifecycle.yml" src/mqlab` together with
  the `include_role` lines in `ansible/validate-nativeha-log-lifecycle.yml:255,284`. Any call
  that includes `cluster-state`, `nativeha-state` or `rdqm-state` gets the same args.
  `package-repos`' first assert makes a missed call fail loudly rather than silently.
- Modify: `src/mqlab/component.py`:
  - `render_status` gains a `TIER` column (`in-lab` / `packaged`);
  - packaged rows come from `read_packages(names, hosts, runner)`, which runs ad-hoc
    `ansible <hosts> -m ansible.builtin.package_facts` with the JSON callback (the same
    pattern as `read_installed`, `:542`);
  - a host row reads `installed <ver> — matches pin` or `installed <ver> — DIFFERS from pin <pin>`
    or `not installed`.
- Modify: `src/mqlab/cli.py` `_select_components` (`:1964`). A name in
  `catalog.packaged_names()` exits 1 with:

  ```text
  mqlab: mq-resiliency-observability is an extracted, packaged component (lab/versions.yaml
  packages.pins = 0.2.0-1). Change it by releasing a new version, then bump the pin and
  reprovision (mqlab bootstrap <stack> --from provision) or rebake its box
  (mqlab box rebuild <box>).
  ```

  `component status` includes packaged names by default, and `--all` lists both tiers.
  `component build` and `component install` refuse them.
- Modify tests:
  - `tests/test_collector_roles_rewired.py`: `ENV_DIR` becomes `/etc/logical-minds-foundry/…`,
    `test_collector_roles_assert_baked_component` is replaced, and `OBSERVABILITY_BAKES` no
    longer expects `component-install`;
  - `tests/test_san_baked.py:70`, `tests/test_box_components.py:35`,
    `tests/test_versions.py:209,212`;
  - new tests below.
- Modify docs:
  - `docs/site/docs/operate/guest-components.md`: the two tiers, `component status` tiers,
    and how to upgrade the packaged product;
  - `docs/development/box-bake-manifest.md` and `docs/development/box-model.md`: the
    observability lines.

  The doc review (#1444) sweeps the rest.

**Interfaces:**

- Consumes:
  - L1: `package-install`, `packages.trust_args`.
  - L2: `BoxEntry.packages`, `--packages`, `baked_packages`.
  - X2: `mq-resiliency-observability 0.2.0-1` is on the LMF index.
- Produces:
  - `Catalog.packaged_names() -> set[str]`;
  - `component.read_packages(names: list[str], hosts: list[str], runner) -> dict[str, dict[str, str | None]]`
    (host → name → version, or None).

- [ ] **Step 1: Write the failing tests:**

  ```python
  # tests/test_versions.py
  def test_committed_roles_install_observability_as_a_package() -> None:
      cat = load_catalog()
      for role in ("san", "mq-nativeha", "pcmk", "mq-rdqm"):
          assert cat.roles[role]["packages"] == ["mq-resiliency-observability"]
          assert cat.roles[role]["components"] == []
      assert cat.packaged_names() == {"mq-resiliency-observability"}

  def test_role_packages_refuse_unpinned_and_runtime(tmp_path) -> None:
      with pytest.raises(VersionError, match="not pinned"):
          load_catalog(_write(tmp_path, _mutated(("roles", "pcmk", "packages"), ["nope"])))
      with pytest.raises(VersionError, match="implicit"):
          load_catalog(_write(tmp_path, _mutated(("roles", "pcmk", "packages"), ["vergil-python3.14.8"])))

  def test_box_packages_for_observability_boxes() -> None:
      assert load_catalog().box("pcmk", OsRef("ubuntu", 24)).packages == (
          "mq-resiliency-observability", "vergil-python3.14.8")

  # tests/test_collector_roles_rewired.py
  @pytest.mark.parametrize("role", ["cluster-state", "nativeha-state", "rdqm-state"])
  def test_collector_roles_install_the_pinned_package_first(role) -> None:
      first = _tasks(role)[0]
      assert first["ansible.builtin.include_role"]["name"] == "package-install"
      assert first["vars"]["package_names"] == ["mq-resiliency-observability"]

  @pytest.mark.parametrize("role", ["cluster-state", "nativeha-state", "rdqm-state"])
  def test_collector_env_under_etc_logical_minds_foundry(role) -> None:
      assert "/etc/opt/" not in yaml.safe_dump(_tasks(role))

  def test_observability_bakes_have_no_component_install() -> None:
      for pb in OBSERVABILITY_BAKES:
          assert "component-install" not in (REPO / "ansible" / pb).read_text()

  # tests/test_cli_component.py
  @pytest.mark.parametrize("verb", ["build", "install"])
  def test_component_refuses_extracted(verb, runner) -> None:
      args = [verb, "mq-resiliency-observability"] + (["--host", "pcmk-a1"] if verb == "install" else [])
      r = runner.invoke(app, ["component", *args])
      assert r.exit_code == 1 and "extracted, packaged component" in r.output and "mqlab box build" in r.output

  # tests/test_component.py
  def test_render_status_shows_packaged_tier(lab, tools) -> None:
      tools.on_package_facts({"pcmk-a1": {"mq-resiliency-observability": "0.2.0-1"}})
      out = component.render_status(["mq-resiliency-observability"], tools.runner, hosts=["pcmk-a1"])
      assert "packaged" in out and "matches pin" in out

  def test_render_status_flags_drift(lab, tools) -> None:
      tools.on_package_facts({"pcmk-a1": {"mq-resiliency-observability": "0.1.9-1"}})
      out = component.render_status(["mq-resiliency-observability"], tools.runner, hosts=["pcmk-a1"])
      assert "DIFFERS from pin 0.2.0-1" in out

  # tests/test_phases.py (or where provision commands are asserted today)
  def test_provision_and_observability_plays_carry_trust_vars(anchors, stack) -> None:
      for cmd in (_provision_cmd(stack), _observability_cmd(stack)):
          assert "-e" in cmd.argv and any(a.endswith("trust-vars.json") for a in cmd.argv)
  ```

  `on_package_facts` is a new `FakeTools` helper. It returns the JSON-callback payload for
  `ansible … -m ansible.builtin.package_facts`, mirroring the existing slurp fake.

- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Make the edits listed above.** Delete the component directory last. Then run
  `grep -rn "mq-resiliency-observability" src ansible lab components tests`. The only
  remaining hits should be the catalog, the roles' `package_names`, tests and the README note.
- [ ] **Step 4: Run the tests and see them pass. Then validate** with
  `vrg-container-run -- vrg-validate`.
- [ ] **Step 5: Live check (this requires X2, and anchors in the VM).**
  1. `mqlab box rebuild pcmk-ubuntu24` bakes it with `--packages mq-resiliency-observability=0.2.0-1,vergil-python3.14.8=…`.
  2. Bring up one stack and run `mqlab component status --host <a pcmk node>`; it shows
     `packaged … matches pin`.
  3. `systemctl cat lab-cluster-state.service` on a node shows the unit coming from
     `/usr/lib/systemd/system` with the new `EnvironmentFile`.

  Record the output in the PR. If no lab is available, say so in the PR; V1 covers it.
- [ ] **Step 6: Commit and report ready.** Commit with
  `vrg-commit --type feat --scope components --message "install mq-resiliency-observability from its published package"`.
  The body notes the in-lab copy removal and `0.2.0-1`. Then run `report-ready`.

---

## Operational V1 (validation): Cold rebuild, macOS arm64

Created with `--kind validation` in the lab repo, `--blocked-by` L3, with a dependency note
on TRUST-SYNC.

- **Precondition:**
  - develop contains L1-L3;
  - TRUST-SYNC is released, and the host has both anchors;
  - `mqlab package trust list` in the VM shows both orgs as `trusted`.
- **Rows** (each: `mqlab box rebuild --all`, then `mqlab bootstrap <stack>` from scratch):
  - `nativeha-ubuntu` @ `ubuntu:24` (**must pass**);
  - `pcmk-ubuntu` @ `ubuntu:24`, whose SAN nodes run `ubuntu:26` (**best effort**: record the
    outcome; failures here don't block acceptance).
- **Checks per row:**
  1. On every observability host, `apt-cache policy mq-resiliency-observability` shows the
     installed `0.2.0-1` from `https://logical-minds-foundry.github.io/packages/deb`, and
     `vergil-python3.14.8` from vergil's repo.
  2. `systemctl cat lab-*-state.service` shows the unit loaded from `/usr/lib/systemd/system`
     with `EnvironmentFile=/etc/logical-minds-foundry/…`, and the timers are active.
  3. Prometheus has `cluster_*` series (pcmk/san), or `cluster_*` plus
     `cluster_nha_*`/`lab_loglifecycle_*` (nativeha). The cockpits are populated (human
     dashboard review).
  4. `mqlab component status --host <all>` shows observability as `packaged … matches pin`
     and clients as `in-lab … matches CURRENT`, with runtime `3.14.8+20261003-1`.
  5. The clients run on `/opt/vergil/python/3.14.8/bin/python3.14`
     (`readlink -f /opt/logical-minds-foundry/mq-resiliency-clients/venv/bin/python`).
     `mq-bench` runs.
  6. `/opt/vergil/cpython-3.14` and `/etc/opt/logical-minds-foundry` are absent on every
     guest.
  7. **Trust failure paths:**
     - Remove the `logical-minds-foundry` anchor on the host, then `vrg-vm session`, then
       `mqlab box rebuild pcmk-ubuntu24`. It fails before any bake, naming
       `vrg-package trust add`.
     - Write a wrong fingerprint for that org on the host (using `--replace`), sync, then
       run `ansible-playbook package-install.yml --limit <one node> -e @…trust-vars.json`.
       It fails at `assert the primary fingerprint equals the anchor`, and no
       `/etc/apt/sources.list.d/logical-minds-foundry.sources` change happens.
     - Restore the anchor afterwards.
  8. **Pin downgrade/upgrade path:** set the observability pin to a version that doesn't
     exist (`0.2.0-9`). Provision fails, naming `lab/versions.yaml packages.pins`. Restore
     the pin; provision is green with `changed=0` on the package tasks.
- **Acceptance:** the must-pass row is green. Record `Outcome: SUCCESS` with the best-effort
  row's outcome noted.

## Operational V2 (validation): Cold rebuild, x86 cloud

Created with `--kind validation`, `--blocked-by` L3, with a dependency note on TRUST-SYNC.
It's run by the cloud agent via `issue-validate`, as #1358 was.

- **Precondition:** develop contains L1-L3; the cloud VM was created or started after
  TRUST-SYNC's release, so its anchors are synced; the host is x86-64-v3 (required for
  `rhel:10`); the RHEL DVDs are staged.
- **Rows:**
  - `nativeha-rhel-crr` @ `rhel:10` (**must pass**);
  - `rdqm-rhel` @ `rhel:9` (best effort);
  - the shared `mq-client` box at x86_64.
- **Checks:** V1 checks 1-6, using `dnf info --installed` / `dnf repoquery` for the origin.
  Plus: the generated `/etc/yum.repos.d/logical-minds-foundry.repo` uses `el10`
  (respectively `el9`) literally, and `dnf makecache` succeeds against the RHEL minor pin.
- **Acceptance:** the must-pass row is green. Record `Outcome: SUCCESS`.

---

## Operational X3 (deployment + code): `1.0.0`, pin bump, reprovision check

Created with `--kind deployment` in the product repo, `--blocked-by` V1 and V2.

- **Product (human-gated release):**
  - First, a product PR (agent) bumps `VERSION`/`pyproject` to `1.0.0` and adds
    `releases/v1.0.0.md` and a CHANGELOG entry ("1.0 after the lab dogfoods it", D9).
  - After merge, the human releases `v1.0.0`. The agent confirms `1.0.0-1` on the LMF index,
    as in X2.
- **Lab (agent; a code PR in the lab repo, filed as its own lab task under #319 when X3
  starts, per the placement law):**
  - set `lab/versions.yaml` `mq-resiliency-observability: "1.0.0-1"`;
  - update `test_committed_*` pin assertions;
  - validate and report ready.
- **Reprovision check (after the lab PR merges):** on a running `nativeha-ubuntu` stack,
  provision **without a rebake**.
  - `mqlab component status` shows `1.0.0-1 matches pin` on every observability host, and
    the collectors keep emitting.
  - `mqlab box status` reports the observability boxes as STALE (the pin is in the
    manifest hash).
- **Acceptance:** record `Outcome: SUCCESS` with the status output.

---

## Closing bookends (already filed)

- **Documentation review:** `mq-resiliency-lab-for-linux#1444`, blocked by X3. It's a
  multi-repo sweep:
  - the lab's `docs/site` and `docs/development`;
  - the product's `docs/site`;
  - `.github` spec errata, if any.
- **Follow-on brainstorm, config-driven org registry:** `.github#321` (it removes REG's
  interim entry, and absorbs `vrg-package trust` into the registry design).
- **Follow-on brainstorm, holistic key and credential rotation:** `.github#324`.
- **Retrospective:** `.github#322` (terminal).
