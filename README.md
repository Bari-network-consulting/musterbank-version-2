<!--
  musterbank-version-2
  Copyright (c) 2026 Abdelkrim Bari - CISCATEL / Bari Network Consulting
  https://www.ciscatel.com  ·  Licensed under the MIT License (see LICENSE)
-->

# musterbank-version-2 - Ansible fabric automation (Lab v2 / NXOS-IMAGE-TESTING)

Dual-DC VXLAN EVPN fabric for Muster Bank AG, lab **version 2**. Same design
pillars as version 1 (IS-IS L2 unnumbered underlay, iBGP EVPN with spine RRs,
symmetric IRB, anycast gateway 0000.2222.3333, EVPN Multi-Site DCI with
rewrite-evpn-rt-asn) and the **same two tenants**:

| Tenant | L3VNI | L3-VLAN | L2 VLANs (VNI) | Subnets (GW .254) |
|---|---|---|---|---|
| TENANT-PAYMENT | 50001 | 3901 | 10 (30010), 20 (30020) | 172.16.10.0/24, 172.16.20.0/24 |
| TENANT-TRADING | 50002 | 3902 | 30 (30030), 40 (30040) | 172.16.30.0/24, 172.16.40.0/24 |

**Server IP convention: `172.16.<VLAN>.<SRV-number>/24`** - the address itself
identifies the VLAN (e.g. SRV22 = 172.16.10.22 -> VLAN 10 / PAYMENT).

## Lab v2 image test matrix (why this lab exists)
| Pair | Image | vPC domain |
|---|---|---|
| F-LEAF-01/02 | 10.2(3) - control | 12 |
| F-LEAF-03/04 | 10.4(2) | 34 |
| N-LEAF-01/02 | 10.6(1) | 12 |
| N-LEAF-03/04 | 9.3(1) | 34 |
| F/N-LEAF-05 (BGW), all spines | 10.2(3) | - (W5: no vpc on BGW) |

## Cabling (per server leaf)
E1/1-2 uplinks to spines (unnumbered IS-IS) | E1/3 vPC Po1 dual-homed IOL |
E1/4 vPC Po2 dual-homed WinServer (manual OS config) | E1/5-6 Po500 peer-link |
E1/7 orphan single-homed IOL (VLAN 20 on odd leafs, 40 on even) |
E1/8 subifs: .99 keepalive (VRF VPC-KEEPALIVE), .98 IS-IS backup.
BGW: E1/1-2 fabric, E1/5-6 dual /31 DCI (eBGP, timers 3/9, no BFD - v1 finding).

## Usage
```
ansible-playbook playbooks/deploy-dc1.yml -k       # full DC1 build (phases 0-5)
ansible-playbook playbooks/deploy-dc2.yml -k
ansible-playbook playbooks/deploy-all.yml -k       # one-command dual-DC build
ansible-playbook playbooks/servers-dc1.yml -k      # IOL server farm (cisco.ios)
ansible-playbook playbooks/validate-fabric.yml -k  # NX-OS only, servers excluded
ansible-playbook playbooks/reset-dc1-fabric.yml -k # typed gate: RESET-DC1
ansible-playbook playbooks/backup-configs.yml -k
```
Both labs share the same management addressing - **run only one lab at a time.**

## v1 lessons baked in
1. **Host scoping fix**: fabric phases target `dc1_fabric`/`dc2_fabric` only;
   IOS servers can no longer be hit by NX-OS tasks ('output value text...').
2. **W8/W10 NVE pre-flight gate**: overlay/multisite phases attempt an isolated
   `interface nve1` FIRST and fail fast with the documented W-series remedy,
   instead of aborting a large config push with a raw parser error.
3. **Split-task reset**: `no interface nve1` is gated on actual presence in the
   running config, and each removal domain is a separate task - one rejected
   line can no longer leave a host fully configured.
4. **Parent-interface admin state**: vPC phase explicitly enables E1/5, E1/6
   and the E1/8 *parent* (today's F-LEAF-01 lesson: subif up, parent shut).
5. Po500 gets **no explicit MTU** (members carry 9216); no
   `no anycast-gateway-mac` anywhere in reset; state-aware `no switchport`;
   jumbo-MTU check-before-push; `arp_tcam_carved` gate for suppress-arp;
   `save_when: changed` everywhere (copy run start discipline).

## VERIFY BEFORE FIRST RUN (assumptions to align with v1)
- Management IPs in `inventory/hosts.yml` follow the v1 scheme anchored on the
  known F-LEAF-01 = 192.168.1.21. Spines .11/.12 (DC1) and .31/.32 (DC2),
  DC2 leafs .41-.45, servers .10x/.12x are **best-guess** - correct any that
  differ from your v1 inventory.
- Underlay/VTEP loopbacks, keepalive/backup /31s, VLAN<->VNI numbering and
  vPC domain 34 for the second pairs: adjust in `inventory/` if v1 used
  different values (F-LEAF-01/02 values match the v1 evidence: lo0 10.1.0.11/.12,
  domain 12, E1/8.98 backup 10.1.98.x/31, BGP 65001).
- IOL server mgmt: e0/3 is assumed `no switchport` + IP on the mgmt segment.

## Restore from backups
```
ansible-playbook playbooks/backup-configs.yml -k                       # snapshot all devices to backups/
ansible-playbook playbooks/restore-configs.yml -k                      # restore ALL devices (typed gate: RESTORE)
ansible-playbook playbooks/restore-configs.yml -k -e target=F-LEAF-01  # restore ONE host
ansible-playbook playbooks/restore-configs.yml -k -e target=dc1_fabric # restore ONE DC's fabric
```
Restore MERGES the backup content back onto each device. Effective for undoing template-owned drift; use reset+deploy to remove objects added manually outside the templates. Hosts without a backup file are skipped cleanly.

## Roadmap — Version 3 (planned)

Version 2 delivers a fully functional dual-DC VXLAN EVPN fabric with EVPN Multi-Site
DCI, hard tenant isolation, and Ansible/AWX automation across a three-image NX-OS
test matrix. Version 3 is planned to add:

- **Controlled inter-tenant traffic via a Fortinet FortiGate NGFW** — service-leaf
  attached, one-arm mode, with an explicit allow-list of PAYMENT ↔ TRADING flows
  (default-deny between tenants otherwise preserved). Reflects how banks implement
  zone-boundary controls: strict isolation with named, inspected exceptions.
- **DCI path redundancy at the BGW layer** — a simulated MPLS L3VPN provider path
  as a secondary DCI alongside the existing direct dual-/31 (modelled as the
  "dark fiber primary"), with BGP path selection driving active/backup behaviour.
  Reflects how banks achieve carrier-diverse DR connectivity between paired sites.

Both changes remain terminated at the BGW pair (F-LEAF-05 / N-LEAF-05); the site
boundary and Multi-Site design stay intact.
