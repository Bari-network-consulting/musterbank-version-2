# musterbank-version-2 — Roadmap

## Version 3 (planned)

### Inter-tenant traffic control (Fortinet FortiGate NGFW)
- Service-leaf-attached firewall in one-arm mode
- Explicit allow-list of PAYMENT ↔ TRADING flows (stateful policy)
- Default-deny between tenants preserved outside the allow-list
- HA active/passive FortiGate pair (target design; single unit acceptable as
  a starting point)

### DCI path redundancy
- Add a simulated MPLS L3VPN provider path as a secondary DCI, terminated on
  the existing BGW pair (F-LEAF-05 / N-LEAF-05)
- Existing direct dual-/31 kept as "dark fiber primary"
- BGP path selection via local-preference / MED to drive primary/backup
  behaviour and clean failover

### Design constraints preserved from Version 2
- All DCI paths terminate at the BGW pair (no leaf-to-leaf or spine-to-spine
  shortcuts across the site boundary)
- EVPN Multi-Site remains the site-boundary demarcation
- Tenant isolation remains default-deny; the NGFW creates named exceptions
  only, never blanket leaks

## Version 2 (delivered)
- Dual-DC VXLAN EVPN fabric (DC1 Frankfurt / DC2 Nürnberg)
- IS-IS L2 unnumbered underlay with ECMP
- iBGP EVPN with spine RRs, symmetric IRB, anycast gateway
- EVPN Multi-Site DCI with rewrite-evpn-rt-asn
- Two tenants (PAYMENT / TRADING) with hard VRF isolation
- Ansible / AWX automation across the full lifecycle
- Three-image NX-OS test matrix (10.2.3, 10.4.2, 10.6.1)
- W-series platform findings log (Nexus 9300v virtual platform)
