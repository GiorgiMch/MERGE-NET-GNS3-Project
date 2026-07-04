# MERGE-NET — Post-Merger Network Integration Lab

**Scenario:** Two companies merge. Both sites run `192.168.10.0/24`. Neither can renumber — production DHCP scopes, static servers, printers, monitoring all depend on existing addressing. The sites need secure site-to-site connectivity anyway.

This lab solves the overlapping-subnet problem the way it's solved in production: **NAT-before-crypto over an IPsec tunnel**, with dual-ISP redundancy, gateway failover, and path-aware failure detection.

Built entirely in **GNS3 with Dynamips** (c7200 / c3745 images) — no VM, no hypervisor.

---

## Topology

<img width="1296" height="707" alt="GNS3 VPN Project Topology" src="https://github.com/user-attachments/assets/0b51bd85-3f3d-47fc-8f2c-cb391118a101" />


```
Site A (HQ)                      "Internet"                Site B (Branch)
PC1 / PC3                                                  PC2
  |                              ISP1 (primary A)            |
SW-A (EtherSwitch, VLAN 10/20)     |                        R5
  |         |                      |                         |
RDNR1 --- RDNR2  (HSRP pair)       |                        R4 (edge)
     \     /                     ISP2 (primary B)          /
    EDGE-ABR  ---- dual-homed ----+------- dual-homed ----+
```

| Device | Role |
|---|---|
| SW-A | EtherSwitch (NM-16ESW), VLAN 10 (Users) / VLAN 20 (MGMT), dot1q trunks |
| RDNR1 / RDNR2 | HSRP gateway pair, per-VLAN active/standby split, OSPF area 0 |
| EDGE-ABR | Site A edge: NAT, IPsec, dual-ISP, IP SLA tracked failover |
| ISP1 / ISP2 | Transit-only routers. No IGP, no route to private space — deliberately ignorant, like real providers |
| R4 | Site B edge: mirrored NAT/IPsec/dual-ISP design, primary flipped to ISP2 |
| R5 | Site B core, OSPF, LAN gateway |

---

## The Core Problem: Overlapping Subnets

Both LANs use `192.168.10.0/24`. Plain routing between them is impossible:

- A routing table cannot hold two different destinations for the same prefix
- Worse — hosts never even send the traffic to their gateway. A host pinging `192.168.10.x` believes the destination is on its own LAN and ARPs locally. **No edge configuration can fix a decision the host makes before the packet leaves.**

### The Fix: Static Network NAT Before Encryption

Each site presents a translated identity to the other:

```
Site A real 192.168.10.0/24  →  172.16.10.0/24  (what Site B sees)
Site B real 192.168.10.0/24  →  172.16.20.0/24  (what Site A sees)

ip nat inside source static network 192.168.10.0 172.16.10.0 /24
```

Host bits are preserved (`.10` stays `.10`), making translation predictable and troubleshootable. Hosts reach the remote site via its **translated** range — the collision never enters the routed path.

**Order of operations (the detail that makes or breaks this design):**

```
Outbound:  NAT → crypto ACL match → encrypt → ISP
Inbound:   decrypt → reverse NAT → deliver to LAN
```

The crypto ACL references **post-NAT addresses only**. `192.168.10.0/24` appears nowhere in VPN policy.

**Why the translated range is private (172.16.x.x), not public:** this traffic is encapsulated in ESP before ever touching the ISP. Only the tunnel's outer endpoints need ISP-routable addressing — the translated space is never routed natively across the transport network.

---

## IPsec Site-to-Site VPN

| Phase | Parameters |
|---|---|
| ISAKMP (Phase 1) | AES-256, SHA-256, pre-shared key, DH group 14 |
| IPsec (Phase 2) | ESP-AES-256, ESP-SHA256-HMAC, tunnel mode |
| Interesting traffic | `172.16.10.0/24 ↔ 172.16.20.0/24` (translated only) |

Tunnel endpoints ride **one transit path end-to-end** (ISP1), with peer host-routes pinned so a default-route failover can never silently reroute tunnel packets into the wrong provider.

---

## Dual-ISP Failover — IP SLA + Tracked Floating Statics

Interface-down detection isn't enough: real failures happen deeper in the path while the local interface stays green. Each edge router probes *through* its primary ISP to a loopback beyond the first hop:

```
ip sla 1
 icmp-echo <ISP loopback> source-interface <primary WAN>
 frequency 5
track 1 ip sla 1 reachability
ip route 0.0.0.0 0.0.0.0 <primary ISP> track 1
ip route 0.0.0.0 0.0.0.0 <backup ISP> 10
```

- **AD** makes two candidate routes coexist and swap cleanly
- **SLA + track** decides *when* the primary stops being a candidate — path health, not just link state
- A permanent host-route to the probe target prevents the classic **circular dependency** (probe needs the route the probe controls — see Lessons below)

Site A's primary is ISP1; Site B's primary is ISP2 — both providers carry real traffic day-to-day instead of one idling as a cold spare.

Interior routers learn reachability via OSPF `default-information originate` — the LAN side never knows which ISP is active or that a failover happened. Complexity stays contained at the edge.

---

## Redundancy at the Gateway — HSRP

- Per-VLAN active/standby split: RDNR1 active for VLAN 10, RDNR2 active for VLAN 20 → both routers forward in steady state
- `preempt` on both for automatic failback
- Hosts point at the virtual IP only; physical router failure is invisible to them

---

## Security Baseline

- SSH v2 only, local auth, `service password-encryption` on all managed devices
- VTY `access-class` restricting management to the MGMT VLAN (`192.168.20.0/24`)
- ISP routers intentionally unhardened — they model infrastructure outside the administrative boundary
- SW-A limitation documented honestly: IOS image lacks crypto → telnet only (a realistic legacy-gear audit finding)

---

## Bugs Hit and What They Taught

The most valuable part of the lab. Every one of these is a realistic production failure mode:

1. **Tunnel endpoints split across isolated ISPs.** Peer was set to the remote router's ISP2 address while sourcing from ISP1 — no common transit, IKE stuck in `MM_NO_STATE`. The real internet hides this mistake; isolated lab ISPs expose it instantly. *Rule: both tunnel endpoints must share one transit path. Redundancy = one tunnel per path, not mixed addresses.*

2. **IP SLA circular dependency.** The probe target was only reachable via the default route the probe itself controlled → track down → route withdrawn → probe can never succeed → permanent failure. *Fix: an untracked host-route to the probe target, independent of anything the SLA controls.*

3. **Crypto ACLs are not filters — they must be exact mirrors.** Putting both directions in one ACL made each router claim the remote site's identity as local traffic, corrupting Phase 2 proxy identities. *One direction per side; the SA handles the return.*

4. **Crypto maps only grab traffic egressing their interface.** Return traffic followed the default route out the ISP2 interface while the crypto map sat on the ISP1 interface — replies left unencrypted and died. *Fix: route the remote translated prefix toward the tunnel's transit ISP explicitly.*

5. **Router-originated pings don't test NAT/VPN.** Traffic generated by the edge router itself bypasses inside→outside NAT and never matches the crypto ACL. *Always test from a LAN host.*

---

## Verification

```
show standby brief                 ! HSRP active/standby split
show ip ospf neighbor              ! FULL adjacencies
show track 1 / show ip sla stat    ! path health
show crypto isakmp sa              ! QM_IDLE
show crypto ipsec sa               ! encaps/decaps incrementing
show ip nat translations           ! live 192.168.10.x → 172.16.x.x mappings
```

Failover tested live: shutting ISP1's far-side interface (a failure that never touches the edge router's own link) flips the track object and swaps the default route within seconds.

---

## Roadmap

- [ ] Backup IPsec tunnel over ISP2 (tunnel-level survivability, tied to existing SLA tracking)
- [ ] QoS on tunnel traffic (LLQ for priority classes)
- [ ] Syslog + SNMP to the MGMT-VLAN monitoring host
- [ ] Inbound edge ACLs (permit IKE/ESP/SLA probes, deny the rest)
- [ ] PAT for direct internet access, with VPN-traffic NAT bypass

---

## Repo Structure

```
configs/     final running-configs per device
docs/        design notes and failure analysis
```

*Built as part of my CCNA → CCNP study path. Feedback welcome.*
