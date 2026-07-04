# MERGE-NET — Design Notes

Why each major decision was made, and what the alternative would have cost.

---

## 1. NAT instead of renumbering

**Constraint:** both sites run `192.168.10.0/24` and cannot renumber. Renumbering a live LAN touches DHCP scopes, static servers, printers, firewall rules and monitoring — expensive and risky, so real companies avoid it until a planned migration window.

**Decision:** static **network** NAT at each edge, translating the whole /24 in one statement:

```
Site A real 192.168.10.0/24 -> 172.16.10.0/24
Site B real 192.168.10.0/24 -> 172.16.20.0/24
```

**Why static network NAT and not PAT or dynamic pools:** site-to-site needs a predictable, bidirectional, one-to-one mapping so return traffic routes deterministically. Host bits are preserved (`.10` stays `.10`), which makes translations self-evident in `show ip nat translations` and keeps troubleshooting sane. PAT (overload) is a many-to-one tool for general internet access — the wrong shape for this problem.

**Why hosts must target the translated range:** a host pinging an address inside its own subnet mask ARPs locally and never consults its gateway. No edge configuration can override a decision the host makes before the packet leaves. The translated range is therefore not cosmetic — it is the only address space through which cross-site communication is even possible.

---

## 2. Why the translated range is private (172.16.x.x), not public

The translated traffic is ESP-encapsulated before it ever touches a provider. The ISPs natively forward only the tunnel's outer header — the `100.64.x.x` WAN endpoints. The translated space never appears in any provider routing table, so public routability is not required. Real-world overlap deployments use secondary RFC1918 ranges for exactly this reason.

The `100.64.0.0/10` WAN addressing is RFC 6598 shared address space — public-flavored, reserved for carrier-side use, and guaranteed not to squat on real internet blocks.

---

## 3. NAT-before-crypto ordering

```
Outbound:  LAN -> NAT (source translated) -> crypto ACL match -> encrypt -> ISP
Inbound:   ISP -> decrypt -> reverse NAT (destination) -> LAN
```

The crypto ACL therefore references **post-NAT addresses only**. The real overlapping subnet appears nowhere in VPN policy. Writing the real subnets into the crypto ACL is the classic failure on this scenario — the ACL would describe traffic that can never exist at the point where it is evaluated.

---

## 4. Deterministic failover, not load balancing

Two identical default routes would ECMP-load-balance across both ISPs. That is wrong here: the tunnel requires deterministic pathing (one tunnel rides one transit end-to-end). Instead:

- **Tracked primary default** (AD 1, tied to `track 1`)
- **Floating backup default** (AD 10 — invisible while the primary exists)
- **IP SLA probe beyond the first hop** (the provider loopback), because real failures happen deeper in the path while the local interface stays green. Interface-down detection alone cannot see them.
- **Untracked host route to the probe target** — without it, the probe depends on the very default route it controls: track down -> route withdrawn -> probe has no path -> probe can never succeed again. The monitoring path must exist independently of anything the monitor controls.

AD defines *which* route is preferred; SLA + track define *when* the preferred route stops being a candidate. Neither replaces the other.

---

## 5. Split-primary dual-ISP design

Site A's primary exit is ISP1; Site B's is ISP2. Both providers carry production traffic daily instead of one idling as a paid cold spare; each site's backup path activates only during genuine failure. This is standard multi-homed branch design — and it is also what forced the tunnel-transit discipline documented in the failure analysis.

---

## 6. Tunnel transit pinning

The tunnel rides ISP1 end-to-end (`100.64.1.2 <-> 100.64.2.2`). Host routes pin the peer address and the remote translated prefix to the ISP1 next-hop on both sides. Without pinning, a default-route failover (R4's default already points at ISP2 in steady state) sends tunnel or return traffic out an interface with no crypto map — packets leave unencrypted into a provider that has no route to the translated space, and the tunnel silently dies. Crypto maps only act on traffic egressing the interface they are applied to; routing must deliver traffic to that interface deliberately.

---

## 7. Deliberately ignorant ISPs

The ISP routers run no IGP and hold no route to any private or translated prefix — exactly like real providers filtering RFC1918. This is the lab's proof-of-work mechanism: if the sites can communicate, only the tunnel and NAT can be carrying it. Any temptation to "fix" a problem by adding an inside route on an ISP is the signal that the edge design is broken. The real internet hides that class of mistake; an honest lab exposes it immediately.

---

## 8. HSRP per-VLAN split

RDNR1 is active for VLAN 10, RDNR2 for VLAN 20 (priority 110 vs default 100, `preempt` on both for automatic failback). Both gateways forward in steady state — load distribution, not just redundancy theater. Hosts point only at the virtual IP; a physical gateway failure is invisible to them.

---

## 9. Interior/edge separation

`default-information originate` on each edge injects reachability into OSPF. Interior routers never learn which ISP is active, that an SLA probe exists, or that a failover occurred — they know only "unknown traffic -> edge." All path-selection complexity is contained at the edge, where it belongs. If both ISPs die, the advertisement stops and the interior correctly learns the internet is gone rather than blackholing at the edge.

---

## 10. Management plane

- SSH v2, local auth, VTY `access-class` restricted to the MGMT VLAN (`192.168.20.0/24`) on all managed devices.
- ISPs intentionally unhardened — outside the administrative boundary; applying internal policy to them would misrepresent the scenario.
- SW-A's IOS image lacks crypto entirely -> telnet-only, documented rather than hidden. Legacy gear without crypto images is a common real audit finding; documenting the limitation honestly is the professional response.

## Platform honesty (EtherSwitch)

The NM-16ESW module is a router-hosted switching module, not a Catalyst: VLANs live in `vlan database` EXEC mode (absent from running-config), verification is `show vlan-switch`, and port-security / modern STP tooling are unsupported. Features it cannot do were dropped from scope and practiced on an exam-accurate simulator instead — knowing a platform's limits is part of the design.
