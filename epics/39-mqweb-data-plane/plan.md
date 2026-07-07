# mqweb REST as data-plane infrastructure — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make mqweb faithfully a per-QM data-plane infrastructure surface — classify it, give it a topology-derived canonical published REST address (both sites), close the Native HA coverage gap, and fix the `rdqm-rhel` site-B VIP the DR invariant surfaces — ratifying design §8.3 without redesigning it.

**Architecture:** Five independent tasks landing in the **`mq-resiliency-lab-for-linux`** repo (this plan document lives in `.github`). Task 2 adds a pure topology→endpoint renderer in `mqlab` (mirroring `dns.py`/`scrape.py`) that emits site-A + site-B endpoints and fails loud on a DR-VIP omission; Task 3 adds four bare `roles: [mqweb]` plays to the Native HA building blocks; Task 5 declares the `rdqm-rhel` site-B VIP in topology and de-hardcodes the DR cutover script; Tasks 1 and 4 are docs and live-lab verification.

**Tech Stack:** Python 3 + Typer (`mqlab` CLI), pytest w/ 100% branch coverage, Ansible (roles/plays), IBM MQ 9.4 mqweb (Liberty, 9443/HTTPS), RDQM DR (`rdqmint`/`rdqmdr`), libvirt/Vagrant lab.

## Global Constraints

- **Ratify §8.3; audit-and-align only.** Do not redesign mqweb's placement. `httpHost=*` **stays** — no socket/firewall restriction in this epic.
- **Derive, never hardcode.** Endpoints and VIPs come from `lab/topology.yaml` (QM-as-a-variable). No QM name or IP literal in shipped code or scripts.
- **svc-sim is the counterparty special case** — reached over `net-ext`, explicitly outside the "our data plane" framing.
- **DR invariant:** a non-Native-HA (VIP) stack with a site-A VIP must declare a site-B VIP; the renderer fails loud otherwise.
- **Validation gate is exactly one command:** `vrg-container-run -- vrg-validate`. The dev-loop test command is `uv run pytest ...` (build-tool use only; never embed `uv run` in shipped code/scripts).
- **Coverage floor: 100% branch** — `uv run pytest --cov=src --cov-branch --cov-fail-under=100`; `# pragma: no cover` only for genuinely unreachable guards.
- **Cold-rebuild acceptance gate** applies to Tasks 3 and 5 (provisioning/DR changes): a full VM cold rebuild must prove them one-pass; lint-green ≠ done. The human operates the lab.
- **Each task = one GitHub issue** in the lab repo, on `feature/<issue>-<slug>` off `develop`; commit with `vrg-commit`; PR into `develop`.

---

### Task 1: Classify mqweb as data-plane infrastructure (docs)

**Files:**
- Modify: `docs/reference/dns-fqdn-inventory.md`
- Modify: `docs/specs/2026-06-03-mq-cluster-lab-design.md` (§8.3 ~line 1145; §1 ~line 192)

**Interfaces:**
- Consumes: the canonical-address scheme (conceptual). Parallel-safe with Task 2.
- Produces: the authoritative prose classification other tasks cite.

- [ ] **Step 1: Add mqweb to the FQDN inventory taxonomy.** In `docs/reference/dns-fqdn-inventory.md`, add a **Migrate → FQDN** row for the mqweb REST endpoint on pcmk/RDQM (both sites: `pcmk-vip-a.client.com:9443` / `pcmk-vip-b.client.com:9443`, `rdqm-vip-a.client.com:9443` / `rdqm-vip-b.client.com:9443`), a **Stay literal IP** row for the Native HA endpoint (active-instance data-plane node IP; reason: no VIP, runtime-resolved), and a one-line note that svc-sim's mqweb is the counterparty surface on `net-ext`, administered lab-only. Example:

```markdown
| mqweb admin REST/Console endpoint (pcmk/RDQM) | `mqweb` on each QM node; published to clients | site A/B VIP `:9443` | `pcmk-vip-a/-b.client.com:9443` / `rdqm-vip-a/-b.client.com:9443` |
```

- [ ] **Step 2: State the plane in the design doc.** In `docs/specs/2026-06-03-mq-cluster-lab-design.md` §8.3 (~line 1145), append: mqweb is a **data-plane infrastructure** surface reached at the QM's data-plane VIP (pcmk/RDQM, each site) or active-instance node IP (Native HA); it is **not** a management/observability-plane service. In §1 (~line 192), add that "REST on every QM" means REST **available and addressable on the data plane**.

- [ ] **Step 3: Validate.** Run: `vrg-container-run -- vrg-validate` — Expected: PASS.

- [ ] **Step 4: Commit.**

```bash
vrg-commit --type docs --scope obs --message "classify mqweb REST as data-plane infrastructure (#<issue>)" \
  --body "Add the mqweb 9443 endpoint to the FQDN inventory (both-site VIP for pcmk/RDQM; active-instance node IP for Native HA; svc-sim net-ext counterparty) and state the plane in design §8.3/§1. Epic logical-minds-foundry/.github#39."
```

---

### Task 2: Topology-derived canonical published REST address (both sites + DR invariant)

**Files:**
- Create: `src/mqlab/rest.py`
- Create: `tests/test_rest.py`
- Modify: `src/mqlab/cli.py` (register a `rest` sub-Typer with `render` — mirror `dns render` at `cli.py:529`)
- Create: `tests/test_cli_rest.py`
- Modify: `src/mqlab/apply.py` (retire the stale `10.30.0.10` docstring example; mark legacy)
- Modify: `docs/development/lab-bringup-capture.md` (flag the step-5 `apply.py` path as legacy Phase-B)

**Interfaces:**
- Produces:
  - `rest.rest_endpoints(topo: dict) -> list[dict]` — each record `{"stack": str, "kind": "vip"|"active-instance"|"counterparty", "endpoints": dict}` where `endpoints` maps a site key (`"site-a"`, optionally `"site-b"`) to a `list[str]` of full `https://<ip>:9443` URLs.
  - `rest.lab_rest_endpoints() -> list[dict]` — same, over the real `lab/topology.yaml`.
  - `rest.RestError` — raised (fail-loud) when a VIP stack lacks `vip_b`, a stack has neither VIP nor `cluster_group`, or a node lacks its data-plane NIC.
  - CLI `mqlab rest render` — writes `build/work/rest/endpoints.json`, echoes each site's endpoints.
- Consumes: `mqlab.paths.repo_root`, `mqlab.paths.work`.

- [ ] **Step 1: Write the failing renderer test** — `tests/test_rest.py`:

```python
"""Canonical mqweb REST endpoint derivation, both sites + DR invariant (epic #39)."""
import pytest

from mqlab import rest


def _topo():
    return {
        "stacks": {
            "pcmk-ubuntu": {
                "cluster_group": "pcmk_a",
                "groups": ["pcmk_a", "pcmk_b"],
                "qm": {"vip": "10.10.1.200", "vip_b": "10.10.2.200"},
            },
            "nativeha-ubuntu": {
                "cluster_group": "nha_ubuntu_a",
                "groups": ["nha_ubuntu_a", "nha_ubuntu_b"],
                "qm": {},
            },
        },
        "groups": {
            "pcmk_a": ["pcmk-a1"],
            "nha_ubuntu_a": ["nha-ubuntu-a1", "nha-ubuntu-a2"],
            "nha_ubuntu_b": ["nha-ubuntu-b1"],
        },
        "nodes": {
            "nha-ubuntu-a1": {"nics": {"net-data-a": "10.10.1.11"}},
            "nha-ubuntu-a2": {"nics": {"net-data-a": "10.10.1.12"}},
            "nha-ubuntu-b1": {"nics": {"net-data-b": "10.10.2.11"}},
            "svc-sim": {"nics": {"net-ext": "10.60.0.50"}},
        },
    }


def test_vip_stack_yields_both_site_endpoints():
    recs = {r["stack"]: r for r in rest.rest_endpoints(_topo())}
    assert recs["pcmk-ubuntu"]["kind"] == "vip"
    assert recs["pcmk-ubuntu"]["endpoints"] == {
        "site-a": ["https://10.10.1.200:9443"],
        "site-b": ["https://10.10.2.200:9443"],
    }


def test_native_ha_yields_per_node_endpoints_per_site():
    recs = {r["stack"]: r for r in rest.rest_endpoints(_topo())}
    assert recs["nativeha-ubuntu"]["kind"] == "active-instance"
    assert recs["nativeha-ubuntu"]["endpoints"] == {
        "site-a": ["https://10.10.1.11:9443", "https://10.10.1.12:9443"],
        "site-b": ["https://10.10.2.11:9443"],
    }


def test_svc_sim_is_counterparty_on_net_ext():
    recs = {r["stack"]: r for r in rest.rest_endpoints(_topo())}
    assert recs["svc-sim"]["kind"] == "counterparty"
    assert recs["svc-sim"]["endpoints"] == {"site-a": ["https://10.60.0.50:9443"]}
```

- [ ] **Step 2: Run it — expect failure.** Run: `uv run pytest tests/test_rest.py -q` — Expected: FAIL (`ModuleNotFoundError: mqlab.rest`).

- [ ] **Step 3: Implement `src/mqlab/rest.py`:**

```python
"""Canonical published mqweb REST endpoints, derived from lab/topology.yaml (epic #39).

Each QM's admin REST/Console (Liberty, 9443/HTTPS) is a data-plane infrastructure
surface reached at the QM's service address, published for BOTH sites (the endpoint
must be known on whichever site is live after a DR cutover):
- pcmk / RDQM -> the data-plane VIP per site (vip / vip_b).
- Native HA   -> the active instance's data-plane node IP, resolved at runtime among
                 each site's candidate nodes (no VIP exists).
- svc-sim     -> the counterparty, over net-ext (lab-only; outside "our data plane").

Fail-loud DR invariant: a VIP stack with a site-A VIP MUST declare vip_b.

Renders the *published* endpoint(s); does not connect. Current provisioning applies
QM content via Ansible runmqsc, not REST (epic #39 spec §3).
"""

from typing import Any, cast

import yaml

from mqlab.paths import repo_root

REST_PORT = 9443
DATA_NET_A = "net-data-a"
DATA_NET_B = "net-data-b"
EXT_NET = "net-ext"


class RestError(Exception):
    """A stack's REST endpoint cannot be derived from topology (fail loud)."""


def _lab_topo() -> dict[str, Any]:
    text = (repo_root() / "lab" / "topology.yaml").read_text()
    return cast(dict[str, Any], yaml.safe_load(text))


def _url(ip: str) -> str:
    return f"https://{ip}:{REST_PORT}"


def _node_urls(topo: dict[str, Any], group: str, net: str) -> list[str]:
    nodes = topo.get("nodes") or {}
    hosts = (topo.get("groups") or {}).get(group) or []
    urls: list[str] = []
    for host in hosts:
        ip = ((nodes.get(host) or {}).get("nics") or {}).get(net)
        if not ip:
            raise RestError(f"node {host!r} has no {net} address")
        urls.append(_url(ip))
    return urls


def rest_endpoints(topo: dict[str, Any]) -> list[dict[str, Any]]:
    """One record per QM REST surface, with both-site endpoints, derived from `topo`."""
    out: list[dict[str, Any]] = []
    nodes = topo.get("nodes") or {}
    for name, cfg in (topo.get("stacks") or {}).items():
        cfg = cfg or {}
        qm = cfg.get("qm") or {}
        vip, vip_b = qm.get("vip"), qm.get("vip_b")
        if vip:
            if not vip_b:
                raise RestError(
                    f"stack {name!r}: site-A VIP but no vip_b — a DR/HA stack must "
                    f"publish a VIP on both sites"
                )
            out.append(
                {
                    "stack": name,
                    "kind": "vip",
                    "endpoints": {"site-a": [_url(vip)], "site-b": [_url(vip_b)]},
                }
            )
            continue
        group_a = cfg.get("cluster_group")
        if not group_a:
            raise RestError(f"stack {name!r}: no vip and no cluster_group to derive a REST endpoint")
        endpoints: dict[str, list[str]] = {"site-a": _node_urls(topo, group_a, DATA_NET_A)}
        groups_b = [g for g in (cfg.get("groups") or []) if g != group_a]
        if groups_b:
            endpoints["site-b"] = _node_urls(topo, groups_b[0], DATA_NET_B)
        out.append({"stack": name, "kind": "active-instance", "endpoints": endpoints})
    svc_ext = ((nodes.get("svc-sim") or {}).get("nics") or {}).get(EXT_NET)
    if svc_ext:
        out.append(
            {"stack": "svc-sim", "kind": "counterparty", "endpoints": {"site-a": [_url(svc_ext)]}}
        )
    return out


def lab_rest_endpoints() -> list[dict[str, Any]]:
    """`rest_endpoints` over the real lab/topology.yaml."""
    return rest_endpoints(_lab_topo())
```

- [ ] **Step 4: Run — expect pass.** Run: `uv run pytest tests/test_rest.py -q` — Expected: PASS (3 tests).

- [ ] **Step 5: Add the fail-loud + edge-branch tests** (100% branch coverage) in `tests/test_rest.py`:

```python
def test_vip_stack_without_vip_b_raises_dr_invariant():
    topo = {"stacks": {"rdqm-rhel": {"qm": {"vip": "10.10.1.100"}}}, "groups": {}, "nodes": {}}
    with pytest.raises(rest.RestError, match="must .*publish a VIP on both sites"):
        rest.rest_endpoints(topo)


def test_native_ha_site_a_only_when_no_peer_group():
    topo = {
        "stacks": {"nh": {"cluster_group": "g_a", "groups": ["g_a"], "qm": {}}},
        "groups": {"g_a": ["n1"]},
        "nodes": {"n1": {"nics": {"net-data-a": "10.10.1.11"}}},
    }
    recs = rest.rest_endpoints(topo)
    assert recs[0]["endpoints"] == {"site-a": ["https://10.10.1.11:9443"]}


def test_stack_without_vip_or_group_raises():
    topo = {"stacks": {"broken": {"qm": {}}}, "groups": {}, "nodes": {}}
    with pytest.raises(rest.RestError, match="no vip and no cluster_group"):
        rest.rest_endpoints(topo)


def test_node_missing_data_nic_raises():
    topo = {
        "stacks": {"nh": {"cluster_group": "g", "groups": ["g"], "qm": {}}},
        "groups": {"g": ["n1"]},
        "nodes": {"n1": {"nics": {}}},
    }
    with pytest.raises(rest.RestError, match="no net-data-a address"):
        rest.rest_endpoints(topo)


def test_absent_svc_sim_yields_no_counterparty_record():
    assert rest.rest_endpoints({"stacks": {}, "groups": {}, "nodes": {}}) == []


def test_lab_rest_endpoints_reads_real_topology():
    recs = {r["stack"]: r for r in rest.lab_rest_endpoints()}
    # Real topology (after Task 5 declares rdqm vip_b): both VIP stacks have site-a+site-b.
    assert recs["pcmk-ubuntu"]["endpoints"]["site-b"] == ["https://10.10.2.200:9443"]
    assert recs["svc-sim"]["kind"] == "counterparty"
```

> **Ordering note:** `test_lab_rest_endpoints_reads_real_topology` asserts `rdqm-rhel` resolves cleanly, which requires Task 5's `vip_b` declaration. If Task 2 lands first, the real-topology renderer will (correctly) raise the DR invariant on `rdqm-rhel` — that is the intended fail-loud signal. Sequence Task 5 before this assertion is expected to pass green in CI, or land Task 2 and Task 5 in the same PR. Note this on the issue.

- [ ] **Step 6: Run coverage on the module.** Run: `uv run pytest tests/test_rest.py --cov=src/mqlab/rest --cov-branch --cov-report=term-missing -q` — Expected: PASS, `rest.py` 100% (no missing branches).

- [ ] **Step 7: Register the CLI command.** In `src/mqlab/cli.py`, near `dns_app` (~line 525), add:

```python
rest_app = typer.Typer(help="Canonical published mqweb REST endpoints (#39)", no_args_is_help=True)
app.add_typer(rest_app, name="rest")


@rest_app.command("render")
def rest_render() -> None:
    """Render per-QM canonical mqweb REST endpoints (both sites) to build/work/rest/endpoints.json (#39)."""
    from mqlab.rest import lab_rest_endpoints

    out_dir = work("rest")
    out_dir.mkdir(parents=True, exist_ok=True)
    recs = lab_rest_endpoints()
    (out_dir / "endpoints.json").write_text(json.dumps(recs, indent=2, sort_keys=True) + "\n")
    for rec in recs:
        for site, urls in rec["endpoints"].items():
            typer.echo(f"{rec['stack']:<16} {rec['kind']:<16} {site:<7} {' '.join(urls)}")
    typer.echo(f"rendered {len(recs)} REST surfaces -> {out_dir}")
```

- [ ] **Step 8: Write the CLI test** — `tests/test_cli_rest.py` (mirror `tests/test_cli_dns.py`, seed via `MQLAB_REPO_ROOT`):

```python
import json

from typer.testing import CliRunner

from mqlab import cli

_TOPO = """
stacks:
  pcmk-ubuntu:
    cluster_group: pcmk_a
    groups: [pcmk_a, pcmk_b]
    qm: {vip: 10.10.1.200, vip_b: 10.10.2.200}
groups:
  pcmk_a: [pcmk-a1]
nodes:
  pcmk-a1: {nics: {net-data-a: 10.10.1.51}}
  svc-sim: {nics: {net-ext: 10.60.0.50}}
"""


def test_rest_render_writes_endpoints_json(tmp_path, monkeypatch):
    (tmp_path / "lab").mkdir()
    (tmp_path / "lab" / "topology.yaml").write_text(_TOPO)
    monkeypatch.setenv("MQLAB_REPO_ROOT", str(tmp_path))
    result = CliRunner().invoke(cli.app, ["rest", "render"])
    assert result.exit_code == 0, result.output
    data = json.loads((tmp_path / "build" / "work" / "rest" / "endpoints.json").read_text())
    by_stack = {r["stack"]: r for r in data}
    assert by_stack["pcmk-ubuntu"]["endpoints"]["site-b"] == ["https://10.10.2.200:9443"]
    assert by_stack["svc-sim"]["kind"] == "counterparty"
```

- [ ] **Step 9: Run the CLI test.** Run: `uv run pytest tests/test_cli_rest.py -q` — Expected: PASS.

- [ ] **Step 10: Retire the legacy `apply.py` example.** In `src/mqlab/apply.py`, replace the docstring usage line:

```python
Usage: python -m mqlab.apply content/qm-main.yaml https://10.30.0.10:9443
```

with:

```python
LEGACY (Phase-B). Current stacks apply QM content via Ansible runmqsc, not REST.
Kept as a pymqrest usage example. For a QM's real REST address, use `mqlab rest render`
(never a hardcoded IP). Usage: python -m mqlab.apply content/<qm>.yaml <base_url>
```

Do not change `apply_spec`/`VERIFY_TLS` behaviour — `tests/test_apply.py` (incl. the `importlib.reload` VERIFY_TLS test at `test_apply.py:59`) must still pass unchanged.

- [ ] **Step 11: Flag the runbook step.** In `docs/development/lab-bringup-capture.md` (step 5, the `python -m mqlab.apply ... https://10.30.0.10:9443` row), add: **"(legacy Phase-B single-QM path; superseded by Ansible `runmqsc`. Canonical REST address: `mqlab rest render`.)"**

- [ ] **Step 12: Full validation gate.** Run: `vrg-container-run -- vrg-validate` — Expected: PASS (ruff, `pytest --cov=src --cov-branch --cov-fail-under=100` green). Fix any suite-wide coverage gap.

- [ ] **Step 13: Commit.**

```bash
vrg-commit --type feat --scope obs --message "derive canonical mqweb REST endpoints (both sites) from topology (#<issue>)" \
  --body "Add mqlab.rest: per-site VIP endpoints for pcmk/RDQM, per-node data-plane IPs per site for Native HA, svc-sim net-ext counterparty, and a fail-loud DR invariant (VIP stack must declare vip_b). Add 'mqlab rest render'. Retire the stale 10.30.0.10 apply.py/runbook example. Epic logical-minds-foundry/.github#39."
```

---

### Task 3: Native HA mqweb coverage

**Files:**
- Modify: `ansible/_nativeha-cluster-ha.yml` (add `- hosts: nha_rhel_a` / `roles: [mqweb]` after the `mq-nativeha` play ~line 24)
- Modify: `ansible/_nativeha-ubuntu-cluster-ha.yml` (add `- hosts: nha_ubuntu_a` / `roles: [mqweb]` after ~line 24)
- Modify: `ansible/_nativeha-dr-replication.yml` (add `- hosts: nha_rhel_b` / `roles: [mqweb]`)
- Modify: `ansible/_nativeha-ubuntu-dr-replication.yml` (add `- hosts: nha_ubuntu_b` / `roles: [mqweb]`)
- Verify (read-only): the fixed `mqweb` PKI entity distributes to Native HA hosts.

**Interfaces:**
- Consumes: the existing `mqweb` role (QM-agnostic; **no vars**), `mqweb_admin_password` from `group_vars/all/admin.yml`, the `mqweb` PKI entity via `pki-distribute`.
- Produces: a running `mqweb.service` on every Native HA instance; admin served by the active instance via the existing "find the active instance" step.

- [ ] **Step 1: Add the site-A RHEL mqweb play.** In `ansible/_nativeha-cluster-ha.yml`, after the `mq-nativeha` play (~line 24), append:

```yaml
- hosts: nha_rhel_a
  roles: [mqweb]  # REST on every QM (design §1), per-node/stateless (spec §8.3)
```

- [ ] **Step 2: Add the site-A Ubuntu mqweb play.** Same in `ansible/_nativeha-ubuntu-cluster-ha.yml` with `hosts: nha_ubuntu_a`.

- [ ] **Step 3: Add the site-B (Recovery) mqweb plays.** `- hosts: nha_rhel_b` / `roles: [mqweb]` in `ansible/_nativeha-dr-replication.yml`, and `- hosts: nha_ubuntu_b` / `roles: [mqweb]` in `ansible/_nativeha-ubuntu-dr-replication.yml`. (mqweb is per-node/stateless; fine to install while `nha_start: false`.)

- [ ] **Step 4: Confirm mqweb PKI enrollment (read-only).** Run: `grep -n "mqweb" src/mqlab/cli.py ansible/roles/mqweb/tasks/main.yml` — confirm the `mqweb` entity is fixed/global (`_FIXED_PKI_ENTITIES`, `cli.py:573`) and `pki-distribute` runs inside the role (`roles/mqweb/tasks/main.yml:17`, gated `mqweb_tls`). No edit expected — the role self-distributes its cert on whatever host it runs.

- [ ] **Step 5: Lint gate.** Run: `vrg-container-run -- vrg-validate` — Expected: PASS (ansible-lint clean; bare-role plays need no task names).

- [ ] **Step 6: Commit.**

```bash
vrg-commit --type feat --scope obs --message "deploy mqweb on every Native HA instance (#<issue>)" \
  --body "Add bare 'roles: [mqweb]' plays to the four _nativeha* building blocks (rhel+ubuntu, site-A HA + site-B CRR), mirroring pcmk/RDQM. Closes the 'REST on every QM' gap for Native HA. Epic logical-minds-foundry/.github#39."
```

- [ ] **Step 7: Cold-rebuild live proof (human-operated — acceptance gate).**

```bash
mqlab vm provision --stack nativeha-ubuntu     # or the documented provisioning entry point
for h in nha-ubuntu-a1 nha-ubuntu-a2 nha-ubuntu-a3; do
  ssh $h 'systemctl is-active mqweb.service'   # expect: active (x3)
done
mqlab rest render                              # shows nativeha-ubuntu site-a candidate URLs
curl -sk -u "$MQWEB_ADMIN_USER:$MQWEB_ADMIN_PASSWORD" \
  https://<active-node-data-a-ip>:9443/ibmmq/rest/v2/admin/qmgr | head   # returns NHAUAPP
```

Expected: `mqweb.service` active on all three; the active instance's REST returns `NHAUAPP`. Record on the PR (cold-rebuild acceptance).

---

### Task 4: Cross-stack verification + failover proof

**Files:**
- Modify/Create: a verification note under `docs/reference/` recording the acceptance procedure + results; reference epic #38 as the automation home.

**Interfaces:**
- Consumes: `mqlab rest render` (Task 2), deployed mqweb (Task 3), `rdqm-rhel` vip_b (Task 5), the running lab.
- Produces: recorded evidence that each published endpoint is reachable/authenticated and survives failover on both sites.

- [ ] **Step 1: Probe every stack's published endpoints (human-operated).**

```bash
mqlab rest render
# VIP stacks — probe BOTH sites' VIPs:
for ip in 10.10.1.200 10.10.2.200 10.10.1.100 10.10.2.100; do
  openssl s_client -connect $ip:9443 </dev/null 2>/dev/null | openssl x509 -noout -subject
done
curl -sk -u "$MQWEB_ADMIN_USER:$MQWEB_ADMIN_PASSWORD" https://10.10.1.200:9443/ibmmq/rest/v2/admin/qmgr | head
# native-ha: active instance (Task 3 Step 7). svc-sim (counterparty):
curl -sk -u "$MQWEB_ADMIN_USER:$MQWEB_ADMIN_PASSWORD" https://10.60.0.50:9443/ibmmq/rest/v2/admin/qmgr | head
```

Expected: each live endpoint returns the QM over TLS with the `mqweb` cert.

- [ ] **Step 2: pcmk/RDQM HA failover proof.** Move the QM within site A (Pacemaker `pcs resource move` / RDQM HA failover); re-probe the site-A VIP. Expected: VIP follows, REST answers on the new owner.

- [ ] **Step 3: Cross-site DR cutover proof.** Run `lab/scripts/rdqm-dr-cutover.sh a2b`; probe the **site-B** VIP (`10.10.2.100:9443`). Expected: `mqlab rest render`'s published site-B endpoint matches the address the cutover bound, and REST answers there. Fail back `b2a` and re-probe site A.

- [ ] **Step 4: Native HA switchover proof.** Trigger a switchover (`site-nativeha-ubuntu-switchover.yml`), re-resolve the active instance, re-probe. Expected: the resolution re-points to the new active member's mqweb.

- [ ] **Step 5: Record + reference the framework.** Write results into a short `docs/reference/` note; add a line pointing to epic #38 (live-lab validation framework) as where this becomes an automated `validate` playbook. Run: `vrg-container-run -- vrg-validate` — Expected: PASS.

- [ ] **Step 6: Commit.**

```bash
vrg-commit --type docs --scope obs --message "record mqweb endpoint verification + both-site failover proof (#<issue>)" \
  --body "Cross-stack acceptance: each published mqweb endpoint reachable/authenticated on its data-plane address; survives HA failover, cross-site DR cutover (site-B VIP matches the renderer), and Native HA switchover. Note epic #38 as the automation home. Epic logical-minds-foundry/.github#39."
```

---

### Task 5: Fix the `rdqm-rhel` site-B VIP (declare in topology, de-hardcode the DR script)

**Files:**
- Modify: `lab/topology.yaml` (add `vip_b: 10.10.2.100` to the `rdqm-rhel` stack's `qm:` block ~line 291)
- Modify: `lab/scripts/rdqm-dr-cutover.sh` (`:26`/`:28` — the hardcoded `TO_VIP` literals become topology-sourced or guarded)
- Create/Modify: `tests/test_rest.py` already asserts real-topology `rdqm-rhel` resolves (Task 2 Step 5) — this task makes it green.

**Interfaces:**
- Consumes: Task 2's renderer + DR invariant.
- Produces: `rdqm-rhel` publishing both-site endpoints; a single source of truth for its VIPs.

- [ ] **Step 1: Declare the site-B VIP.** In `lab/topology.yaml`, in the `rdqm-rhel` stack `qm:` block (currently `qm: {vip: 10.10.1.100, ...}`), add `vip_b: 10.10.2.100` (the value already bound as a literal in `rdqm-dr-cutover.sh:26`). Keep it consistent with the RDQM one-FIP model — `vip_b` is the *site-B incarnation* of the single floating IP, live only after cutover.

- [ ] **Step 2: Prove the renderer now resolves `rdqm-rhel`.** Run: `uv run pytest tests/test_rest.py::test_lab_rest_endpoints_reads_real_topology -q` — Expected: PASS (previously the DR invariant would raise on `rdqm-rhel`). Also run `mqlab rest render` and confirm `rdqm-rhel` shows `site-a https://10.10.1.100:9443` and `site-b https://10.10.2.100:9443`.

- [ ] **Step 3: De-hardcode the DR cutover script.** In `lab/scripts/rdqm-dr-cutover.sh`, replace the literal `TO_VIP` assignments (`:26` `10.10.2.100`, `:28` `10.10.1.100`) with values sourced from topology so the script and the renderer cannot drift. Minimal approach — source the rendered endpoints:

```bash
# after `mqlab rest render` has written build/work/rest/endpoints.json
VIP_A=$(python3 -c "import json,sys; d=json.load(open('build/work/rest/endpoints.json')); print([r for r in d if r['stack']=='rdqm-rhel'][0]['endpoints']['site-a'][0].split('//')[1].split(':')[0])")
VIP_B=$(python3 -c "import json,sys; d=json.load(open('build/work/rest/endpoints.json')); print([r for r in d if r['stack']=='rdqm-rhel'][0]['endpoints']['site-b'][0].split('//')[1].split(':')[0])")
if [ "$DIR" = a2b ]; then FROM_PRIMARY=rdqm-a1; TO_PRIMARY=rdqm-b1; TO_VIP=$VIP_B
elif [ "$DIR" = b2a ]; then FROM_PRIMARY=rdqm-b1; TO_PRIMARY=rdqm-a1; TO_VIP=$VIP_A
fi
```

If sourcing JSON in bash is deemed too heavy for the script, the acceptable fallback is a **guard**: keep the literals but add a check that they equal the topology-declared `vip`/`vip_b`, failing loud on drift. Either way, topology is the source of truth. (Do **not** embed `uv run` in the script — call `python3`/`mqlab` by bare name.)

- [ ] **Step 4: Validate.** Run: `vrg-container-run -- vrg-validate` — Expected: PASS (shellcheck/ansible-lint/pytest green; the real-topology renderer test passes).

- [ ] **Step 5: DR cutover cold-rebuild proof (human-operated — acceptance gate).** On the box: provision `rdqm-rhel`, run `lab/scripts/rdqm-dr-cutover.sh a2b`, confirm the QM goes live at site B and REST answers on `10.10.2.100:9443`; fail back `b2a`. Expected: cutover works with the topology-sourced VIP (parity with the Phase-C drill), and `mqlab rest render`'s site-B endpoint matches.

- [ ] **Step 6: Commit.**

```bash
vrg-commit --type fix --scope obs --message "declare rdqm-rhel site-B VIP in topology; de-hardcode the DR cutover (#<issue>)" \
  --body "rdqm-rhel declared vip: 10.10.1.100 but no vip_b, though 10.10.2.100 was a hardcoded literal in rdqm-dr-cutover.sh. Declare vip_b in topology (single source of truth) and source the cutover VIPs from it. Renderer now publishes both sites; the DR invariant passes. Epic logical-minds-foundry/.github#39."
```

---

## Self-Review

**Spec coverage:**
- §4.1 classify → Task 1. §4.2 canonical address (both sites) + DR invariant + legacy cleanup → Task 2. §4.3 Native HA coverage → Task 3. §4.4 rdqm-rhel site-B VIP fix → Task 5. §5 binding (`httpHost=*` stays) → Global Constraints. §7 verification (both-site probes, HA failover, DR cutover, Native HA switchover, cold-rebuild) → Task 4 (+ Task 3 Step 7, Task 5 Step 5). §6 non-goals → not implemented, correct. §8 relationships → epic-level. **No gaps.**
- svc-sim special case → Task 2 (`counterparty`) + Task 1. DR invariant → Task 2 (unit-tested) + surfaced-and-fixed in Task 5. Consistent.

**Placeholder scan:** No TBD/TODO; every code/edit step shows exact content or an exact command with expected output. Live-lab steps (Task 3 Step 7, Task 4, Task 5 Step 5) are human-operated by design and shown concretely.

**Type consistency:** `rest_endpoints`/`lab_rest_endpoints`/`RestError`/`_node_urls` and the record shape (`stack`/`kind`/`endpoints` where `endpoints` is a `dict[str, list[str]]` keyed `site-a`/`site-b`) are used identically across `rest.py`, both test files, and the CLI `rest render` (which iterates `rec["endpoints"].items()`). `kind` values (`vip`/`active-instance`/`counterparty`) match module, tests, and Task 1 prose. Constants `9443`/`net-data-a`/`net-data-b`/`net-ext` match the topology facts. Task 2 Step 5's real-topology test depends on Task 5's `vip_b` declaration — flagged inline (land together or sequence Task 5 first).
