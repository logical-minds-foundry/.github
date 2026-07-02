# DNS hostname resolution — implementation plan

> **For agentic workers:** this is an **epic-level plan**. Each work item
> below is filed as a member-repo issue (`mq-resiliency-lab-for-linux`),
> linked to epic #21 with `vrg-epic-link`, and built with
> `/vergil:issue-implement <N>` — which runs its own TDD cycle. Steps use
> checkbox (`- [ ]`) syntax for tracking. Sub-items are issue-sized, not
> per-function; the detailed test-first steps live in each issue's own build.

**Goal:** Give the lab production-shaped DNS — authoritative BIND9 forward +
reverse zones, generated from `lab/topology.yaml`, served from new per-side
infrastructure nodes — then point the lab's configuration at names instead of
raw IPs.

**Architecture:** The network is already segregated (sites A/B = HA+DR;
`net-ext` = the org boundary), so nothing is re-addressed. The only new
component is a per-side **infra node**: our-side authoritative for `client.com`
(our whole estate), a mock service-side authoritative for `service.com`
(`svc-sim`). Zones are generated from `topology.yaml` — whose `nics:` map
already carries the plane names the generator needs — never hand-edited.
Guests resolve via BIND by IP; `/etc/hosts` shrinks to self-identity + the
HA/replication fabric.

**Tech Stack:** Ansible (roles + site playbooks), BIND9 (`named`), libvirt /
Vagrant via `lab/topology.yaml`, Python (`src/mqlab` — zone generation reuses
`inventory.py`'s topology reader), pytest, `vrg-validate`.

## Global Constraints

- **Zones are generated from `lab/topology.yaml`, never hand-edited.** The
  BIND zone file is a generated artifact; author only YAML.
- **`REVDNS` stays `ENABLED`** (the QM default) — it is required for the
  hostname CHLAUTH rules the security workstream depends on. Do **not**
  disable it as a shortcut.
- **Minimal-dependency dividing line:** `/etc/hosts` = self-identity +
  HA/replication fabric only; everything on the app↔service axis is DNS-only.
- **Management (`net-mgmt`) is non-transit — The Watcher.** Nothing routes
  through it.
- **`nsswitch` order stays `files dns`.**
- **Cold-rebuild acceptance gate:** each task is accepted only after a full VM
  cold rebuild proves it one-pass. Lint-green ≠ done.
- **Validation is `vrg-container-run -- vrg-validate` only.** Never suppress a
  gate. Secrets never enter git.

---

## Task B — BIND9 + infra nodes

Delivers working DNS. Sequence B1→B5; the cold-reboot gate is at B5.

### B1: Add the infra nodes to the topology

**Files:** `lab/topology.yaml` (new nodes); `lab/networks/*.xml` if a new
attachment is needed; `src/mqlab/inventory.py` + `tests/test_inventory.py`
(new `infra` group).

- [ ] Add `infra-client` (our side) and `infra-svc` (mock service side) nodes
  with NICs on the planes they must serve — at minimum `net-mgmt`, `net-data-a`,
  `net-data-b` (client), and `net-ext` (both, to span the org boundary).
- [ ] Add an `infra` group (and `org` tag groundwork) so playbooks can target
  them; assert the inventory renders them (extend `test_inventory.py`).
- [ ] **Deliverable:** both nodes cold-boot, are SSH-ready, and appear in the
  rendered inventory.
- [ ] **Validation:** `vrg-validate` green; `mqlab vm status` shows the nodes;
  they answer on `net-mgmt`.
- **Open decision (resolve in this issue's build):** one infra node hosting
  both zones vs. two separate nodes — the spec's failure-domain-independence
  argues two; confirm against VM budget.

### B2: Zone generator (topology.yaml → zone records)

**Files:** `src/mqlab/dns.py` (new — pure functions); `tests/test_dns.py`
(new); `lab/topology.yaml` (add per-node `org: client|service`).

- [ ] Add the `org` tag to every node in `topology.yaml`.
- [ ] Write `dns.py` to read the topology (reuse `inventory.py`'s loader) and
  emit, as plain data: forward records (`<host>-<plane>` A per NIC; base-name
  CNAME → primary interface) per `org` zone, and reverse (PTR) records per
  subnet. This is a **pure, unit-testable function** — the primary TDD target.
- [ ] **Deliverable:** `dns.py` produces correct forward + reverse record sets
  from a fixture topology, including the Native-HA 3-instance names and the
  pcmk/rdqm VIP service names.
- [ ] **Validation:** `uv run pytest tests/test_dns.py`; 100% branch coverage
  per the repo's coverage gate.
- **Open decision:** which interface is "primary" (base-name CNAME target) per
  node role; `net-ext` reverse-zone authority (single-owner default — our
  infra owns `0.60.10.in-addr.arpa`).

### B3: BIND role — serve the zones

**Files:** `ansible/roles/bind-dns/{tasks,defaults,templates,handlers}/`
(new); templates `named.conf.j2`, `zone.forward.j2`, `zone.reverse.j2`.

- [ ] Install `bind9`; template `named.conf` (authoritative for this side's
  zone + reverse zones; forward cross-org to the peer infra) and the generated
  zone files (fed by B2's records); `notify` a `named` reload handler.
- [ ] **Deliverable:** each infra node answers authoritative forward + reverse
  queries for its zone and forwards the other org's names.
- [ ] **Validation:** `dig @<infra> <host>-data-a.client.com` and
  `dig -x <ip>` return the expected records; cross-org forward resolves.

### B4: Resolver + minimal `/etc/hosts`

**Files:** `ansible/roles/host-resolver/` (new, or extend `host-net-state`);
resolver template (netplan/`systemd-resolved`); `/etc/hosts` template.

- [ ] Point each guest's resolver at its side's infra node **by IP**; keep
  `nsswitch` `files dns`.
- [ ] Seed the minimal `/etc/hosts`: `localhost`, self FQDN+short → primary
  address, and the HA/replication fabric peers (heartbeat/replication names) —
  reusing the data `pcmk-cluster`/`rdqm-ha` already assemble.
- [ ] **Deliverable:** guests resolve lab names via BIND; `/etc/hosts` carries
  only the survive-DNS-loss set.
- [ ] **Validation:** on a guest, `getent hosts <host>-data-a.client.com`
  resolves via DNS; `getnameinfo` on a QM data-plane IP returns **fast** (the
  #454 ~10 s tax is gone); cluster/replication names still resolve with DNS
  stopped.

### B5: Wire into site playbooks, remove the stopgap, cold-reboot gate

**Files:** `ansible/site-distributed-shared.yml` and each arm's site playbook
(apply `bind-dns` to `infra`, `host-resolver` to all); remove the tactical
`/etc/hosts` stopgap task.

- [ ] Apply the roles in the provisioning flow; **remove the stopgap only
  after** B1–B4 prove DNS resolves (no coverage gap).
- [ ] **Deliverable:** a full stack provisions with DNS as the resolution path.
- [ ] **Cold-reboot gate:** full VM cold rebuild; the message flow runs and a
  DR cutover succeeds resolving via DNS; the #454 reverse tax stays gone.

---

## Task C — FQDN migration sweep

Gated on Task B being cold-reboot-validated (addresses known-correct). Default
to **minimal dependency**.

### C1: Inventory the raw-IP reference sites

**Files:** none (analysis); output a checklist in the Task C issue.

- [ ] Grep the config surface for raw IPs: channel `CONNAME`/`LOCLADDR`,
  listener `IPADDR`, cluster CONNAMEs, `mqweb` `httpHost`, LDAP `AUTHINFO
  CONNAME`, exporter/scrape targets, VIP references.
- [ ] **Deliverable:** a per-surface table — raw IP → proposed FQDN (or "stays
  IP, with reason").

### C2: Apply FQDNs where they make sense

**Files:** the roles/playbooks/templates each surface lives in.

- [ ] Replace IPs with FQDNs per the C1 table. **Heartbeat/replication stays
  literal IP** (fabric must not depend on DNS); name-in-hosts for admin
  readability only. Decide genuine 50/50s case-by-case, defaulting to
  least-dependency.
- [ ] **Deliverable:** the app↔service axis references names; the fabric stays
  IP.
- [ ] **Validation:** `vrg-validate` green.

### C3: Cold-reboot validation

- [ ] **Cold-reboot gate:** full rebuild; message flow + DR cutover succeed
  with names in the config; no raw IPs remain on the app↔service axis except
  those C1 justified.

---

## Epic done-criteria

Tasks B and C both cold-reboot-validated. Out of scope (separate epics):
DNS fault-injection; detailed client-connectivity design (#472 is the
precursor client fix).
