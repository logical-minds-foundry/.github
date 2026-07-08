# Restore RDQM build functionality — strategy & findings

- **Epic:** `logical-minds-foundry/.github#45`
- **Documentation task:** `logical-minds-foundry/.github#46`
- **Origin:** live-lab validation under epic `logical-minds-foundry/.github#39` (mqweb data-plane), while bringing up `rdqm-rhel` for the RDQMAPP mqweb check.
- **Status:** strategy (repair epic — see note below)
- **Date:** 2026-07-08

## 0. What kind of epic this is

This is a **repair / triage epic, not a feature epic.** The usual `epic-create`
flow brainstorms and designs a thing to build, then specs and plans that design.
Here we already know something is broken and that fixing it is **more than one
quick PR** — but we do **not** yet have a full solution to spec, because part of
the work *is the investigation*. So this document is a **strategy + findings**
record: what we have proven is wrong, how we intend to run it to ground, and the
task shape. The detailed fix designs emerge as the investigation tasks (#562
re-diagnose, #564 audit, #565 validate) complete, and are captured there.

## 1. Problem

The RDQM (`rdqm-rhel`) DR/HA cold build **has never reliably worked**. It was
never validated from a genuine cold rebuild — we have primarily exercised the
Native HA arm — and when we finally did, every attempt to create the queue
manager failed. The failures presented differently depending on TLS, which sent
early diagnosis down the wrong path.

## 2. What we have proven (findings)

All grounded in IBM MQ 9.4 docs (cached under `build/refs/ibm-docs/ibm-mq/9.4.x/`):

1. **Missing prerequisite — `mqm` passwordless SSH + sudo.** IBM's DR/HA create is
   **one `crtmqm -sx` per site**, which auto-creates the HA secondaries by SSH-ing
   to the peer nodes as `mqm` (with sudo). The lab never provisioned this, so
   `crtmqm -sx` errors *"the secondary queue manager must first be created."*
2. **A broken workaround.** The lab hand-rolls a manual **secondaries-first**
   order (`lab/scripts/rdqm-qm-create.sh`). That is IBM's documented no-SSH
   fallback, but it **does not work for DR/HA** — creating the secondaries
   `Inconsistent` first forces a full initial DRBD resync that never reaches
   `UpToDate`, so the create fails: `AMQ3879E` "not connected after 10 seconds"
   (with `-re` TLS, whose slower connect trips the fixed 10 s wait first) or
   `AMQ3817E` / `rdqmapp.dr: (-2) Need access to UpToDate data` (plaintext, at the
   DR-promote step). The plaintext bisect proved TLS is **not** the cause.
3. **The coordinated path works.** With `mqm` SSH+sudo provisioned live, a single
   `crtmqm -sx` **auto-created both secondaries** (*"Secondary queue manager
   created on rdqm-a2/a3"*).
4. **`mqm` home must move to `/home/mqm`.** `/var/mqm` is group-writable *and*
   SELinux `var_t`; `sshd` `StrictModes` + `authorized_keys` reads fail there
   (confirmed via AVC denial). This matches IBM's documented step.
5. **`crtmqm` SSHes over the `172.16.x` HA_Replication subnet**, not the
   node-name/primary address — a missing host-key acceptance there failed
   silently as `Connection closed [preauth]`.
6. **#556 (ClusterSyslog) is incomplete.** Once the working coordinated create
   *reaches* diagnostic-template processing (the manual path died at `AMQ3879E`
   first, masking it), `crtmqm` still fails `AMQ7059E` "details for service
   'ClusterSyslog' conflict". The merged distinct-name fix does not resolve it.
7. **#558 (silent `tlshd`) hid the truth.** `tlshd.conf` shipped `loglevel=0`,
   silencing the handshake logs; raising it showed the TLS handshakes **succeed**,
   flipping the diagnosis away from TLS. (Fixed, PR in flight.)

## 3. Strategy

Align the RDQM build with IBM's documented coordinated procedure, prove it on a
clean cold rebuild, and document the IBM-side fallback defect. Concretely:

1. **Provision the prerequisite** (`#560`): an `rdqm-ssh-access` role that enables
   `mqm` SSH+sudo the IBM way (home → `/home/mqm`, controller-managed key,
   sudoers, replication-subnet host-key acceptance), runs **early** (before
   mqweb/cluster), with an **enable → create → remove** lifecycle so the access is
   not standing infrastructure.
2. **Adopt the documented create** (`#561`): rewrite `rdqm-qm-create.sh` to the
   one-`crtmqm`-per-site coordinated flow; delete the secondaries-first workaround.
3. **Re-diagnose + fix the diagnostic conflict** (`#562`): find the true source of
   the `ClusterSyslog` collision (now reproducible through the working create) and
   fix `mq-diag-logging`; supersede #556.
4. **Prove it green** (`#565`, validation): a clean `rdqm-rhel` cold rebuild —
   coordinated create, HA+DR healthy, diagnostics intact, mqweb answers — gates
   the epic.
5. **Report the IBM defect** (`#563`): a `docs/reports/` bug report of the
   manual-fallback failure for a human to file with IBM (potential APAR); open
   question — does it reproduce on v10 (future).
6. **Probe pcmk** (`#564`): audit whether the Pacemaker stacks share this class of
   prerequisite / create-flow gap. If so, the closing follow-on-brainstorm (`#47`)
   mints a separate "Restore pcmk build" epic.

## 4. Open questions (resolved by the tasks, not here)

- The exact `ClusterSyslog` fix (#562 investigation).
- Whether pcmk is similarly affected (#564 audit).
- v10 behavior of the fallback (out of scope; upgrade/downgrade mechanism not yet
  built).

## 5. Acceptance

Epic done when: `#565` validation PASSes on a cold rebuild, `#563` IBM report
exists, `#564` pcmk audit is reported (feeding `#47`), and the closing doc-review
(`#48`) confirms the site docs reflect the changes.
