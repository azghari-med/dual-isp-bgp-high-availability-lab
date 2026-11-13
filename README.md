# One Cable Away From Offline — Dual-ISP BGP Edge (Complete Solution)

> \*\*Cisco IOS lab walkthrough | PNetLab / EVE-NG | eBGP, iBGP, local preference, AS-path prepending, export filtering, HSRP, failure testing\*\*

This repository answers the supplied *LAB-CHALLENGE-bgp-dual-isp.md*: implementation phases 0–8, all six challenge injections, all nine design questions, and the acceptance/evidence checklist. It is an **implementation reference**, not a claim that an actual lab was executed. Replace the evidence placeholders with authentic captures from your virtual environment.

## Contents

1. [Design and caveats](#1-design-and-caveats)
2. [Topology and IP plan](#2-topology-and-ip-plan)
3. [Requirements mapping](#3-requirements-mapping)
4. [Build: phases 0–3](#4-build-phases-03)
5. [Traffic engineering: phases 4–6](#5-traffic-engineering-phases-46)
6. [Gateway redundancy: phase 7](#6-gateway-redundancy-phase-7)
7. [Failover and recovery: phase 8](#7-failover-and-recovery-phase-8)
8. [Challenge injections](#8-challenge-injections)
9. [Design-question answers](#9-design-question-answers)
10. [Acceptance criteria and evidence](#10-acceptance-criteria-and-evidence)
11. [References](#11-references)

## 1\. Design and caveats

**Business goal:** Company AS **65100** is multihomed to ISP-A AS **65001** and ISP-B AS **65002**. ISP-A is the normal outbound and inbound path; ISP-B is backup. Both edge routers attach to the same inside LAN with one HSRP gateway. Only `203.0.113.0/24` may leave AS 65100 as an eBGP announcement.

**Scope:** Cisco IOS-style IPv4 unicast syntax. Tested-in-lab status: **NOT VERIFIED** — router image versions, interface names, and feature support vary between IOL and CSR1000v images. Each `.cfg` uses `GigabitEthernet0/0` through `0/2`; if your device uses `Ethernet0/x` or `GigabitEthernet1`, substitute actual names. Some older IOL images do not support `standby version 2`, object tracking, or some show command variants. Check `?` and `show version`.

### Critical networking subtleties

* **BGP only advertises its chosen best path by default.** With LOCAL\_PREF 200 on ISP-A and 100 on ISP-B, EDGE-R2 receives A via iBGP and prefers it over its own B route. It therefore does *not* normally advertise its inferior B route back to EDGE-R1. Thus EDGE-R2 can see both paths while EDGE-R1 may see **only its A path**. The lab's request that *both* routers display *both* paths at the same time cannot be guaranteed by this basic design. BGP **additional-paths** (with compatible implementations and carefully chosen send/receive selection) or extra routing-policy architecture is required to meet that literal condition. **Do not falsify screenshots to satisfy it.** The core primary/backup forwarding and failover goals still work.
* **HSRP only protects the first-hop gateway.** It does not synchronize firewall session state, detect every BGP-only failure, or make all disruptions invisible. This reference tracks EDGE-R1's ISP interface *line protocol* to change HSRP active router on a physical failure. For a BGP session failure with link still up, traffic may still get to ISP-B through iBGP via R2, but the HSRP active device remains R1 until additional BGP-aware tracking is implemented.
* **Prepending is a hint, not a guarantee.** The internet-side AS chooses its own best route. ISP/customer LOCAL\_PREF, commercial policies, and other attributes may override AS-path length. The simple INTERNET simulator should prefer A when all other earlier criteria tie.
* **This is a simulated public internet.** `192.0.2.0/24`, `198.51.100.0/24`, and `203.0.113.0/24` are RFC 5737 documentation blocks and **are not publicly routable**, and all ASNs are private-use ranges. There is no actual external internet access.
* **No actual failover duration is asserted.** You must collect timing from your emulator and image. A physical link-down event is usually detected faster than a silent BGP transport failure.

## 2\. Topology and IP plan

```text
                   INTERNET (AS 65000)
                   Lo0 192.0.2.1/24
                   Gi0/0        Gi0/1
                   .10             .14
                 /                    \\
      198.51.100.8/30        198.51.100.12/30
               /                        \\
          .9 ISP-A                    ISP-B .13
          AS 65001                    AS 65002
          Gi0/0 .1                    Gi0/0 .5
               |                        |
       198.51.100.0/30        198.51.100.4/30
               |                        |
          Gi0/0 .2                    Gi0/0 .6
          EDGE-R1 ================== EDGE-R2
         AS 65100       iBGP        AS 65100
         Lo0 10.255.0.1/32       Lo0 10.255.0.2/32
          Gi0/1 .1 --- 10.0.12.0/30 --- .2 Gi0/1
          Gi0/2 .2                    Gi0/2 .3
                 \\                   /
                  \[ L2 Ethernet switch ]
                        |
                  203.0.113.0/24
                  HSRP VIP .1
                  HOST-1 .10/24
                  gateway .1
```

**Wiring note:** The `EDGE-R1 === EDGE-R2` label denotes an iBGP session carried over the **single physical 10.0.12.0/30** inter-router connection shown immediately below it; do not add an unnecessary second link. The edge LAN connections share one virtual Ethernet switch (no router or VLAN separation between them).

|Node|Interface|IPv4 / mask|Connection|
|-|-|-|-|
|INTERNET|Gi0/0|198.51.100.10/30|ISP-A Gi0/1|
|INTERNET|Gi0/1|198.51.100.14/30|ISP-B Gi0/1|
|INTERNET|Lo0|192.0.2.1/24|Simulated internet service|
|ISP-A|Gi0/0|198.51.100.1/30|EDGE-R1 Gi0/0|
|ISP-A|Gi0/1|198.51.100.9/30|INTERNET Gi0/0|
|ISP-B|Gi0/0|198.51.100.5/30|EDGE-R2 Gi0/0|
|ISP-B|Gi0/1|198.51.100.13/30|INTERNET Gi0/1|
|EDGE-R1|Gi0/0|198.51.100.2/30|ISP-A Gi0/0|
|EDGE-R1|Gi0/1|10.0.12.1/30|EDGE-R2 Gi0/1|
|EDGE-R1|Lo0|10.255.0.1/32|iBGP update-source|
|EDGE-R1|Gi0/2|203.0.113.2/24|LAN switch|
|EDGE-R2|Gi0/0|198.51.100.6/30|ISP-B Gi0/0|
|EDGE-R2|Gi0/1|10.0.12.2/30|EDGE-R1 Gi0/1|
|EDGE-R2|Lo0|10.255.0.2/32|iBGP update-source|
|EDGE-R2|Gi0/2|203.0.113.3/24|LAN switch|
|HSRP group 1|virtual gateway|203.0.113.1/24|Hosts' default gateway|
|HOST-1|Ethernet|203.0.113.10/24|LAN switch|

## 3\. Requirements mapping

|Requirement|Solution|Verification|
|-|-|-|
|R1: simultaneous ISPs|R1–ISP-A eBGP; R2–ISP-B eBGP|`show ip bgp summary`|
|R2: independent prefix|`network 203.0.113.0 mask 255.255.255.0` on both edge routers|INTERNET's BGP table|
|R3: outbound ISP-A|ISP-A import LOCAL\_PREF 200; ISP-B 100|R2 BGP best-path + HOST traceroute|
|R4: inbound ISP-A|AS 65100 prepended three times toward ISP-B|INTERNET path/AS\_PATH|
|R5: no transit|exact `/24` prefix-list enforced outbound on both edges|`advertised-routes`|
|R6: failover|iBGP alternative + BGP withdrawals|shut/no shut + repeated tests|
|R7: router redundancy|HSRP VIP .1, R1 priority 120, R2 priority 100; track R1 outside line|`show standby brief`, router-shutdown test|

## 4\. Build: phases 0–3

### Phase 0 — Preflight and baseline

Create **five Cisco routers**, one Ethernet switch, one VPCS (the topology contains INTERNET, two providers and two edge routers). Wire interfaces exactly as in the IP plan. On Cisco images, paste the contents of these files in global configuration mode:

```text
configs/INTERNET.cfg
configs/ISP-A.cfg
configs/ISP-B.cfg
configs/EDGE-R1.cfg
configs/EDGE-R2.cfg
configs/HOST-1.txt
```

For a strict learning sequence, initially configure **interfaces and static loopback reachability only**, and add BGP, policy and HSRP phase by phase from the same files. Configuration order does not affect the finished solution. Cisco router files contain `end` and may be pasted from privileged EXEC after `configure terminal`. Do not paste a router file onto the wrong node.

Before enabling BGP, verify directly connected IPv4 reachability:

```cisco
! EDGE-R1
show ip interface brief
ping 198.51.100.1
ping 10.0.12.2
ping 203.0.113.3
ping 10.255.0.2

! EDGE-R2
ping 198.51.100.5
ping 10.0.12.1
ping 203.0.113.2
ping 10.255.0.1

! ISP-A
ping 198.51.100.2
ping 198.51.100.10

! ISP-B
ping 198.51.100.6
ping 198.51.100.14

! INTERNET
ping 198.51.100.9
ping 198.51.100.13
```

**Expected:** all directly adjacent probes succeed; the edge loopbacks are reachable via `/32` static routes over 10.0.12.0/30. If a directly connected ping fails, check cabling, address/mask, interface state and ARP *before debugging BGP*.

### Phase 1 — eBGP

The provider ASN is the neighbor's `remote-as`:

```cisco
! EDGE-R1
router bgp 65100
 neighbor 198.51.100.1 remote-as 65001

! EDGE-R2
router bgp 65100
 neighbor 198.51.100.5 remote-as 65002
```

The complete configurations also establish ISP-A–INTERNET and ISP-B–INTERNET eBGP sessions. INTERNET originates `192.0.2.0/24` with `network` because that exact prefix exists as a connected Loopback0 route. Providers pass their selected BGP paths to the company without special route-maps in this educational simulator.

```cisco
show ip bgp summary
show ip bgp 192.0.2.0
show ip route 192.0.2.0
show ip bgp neighbors 198.51.100.1
```

Run the corresponding ISP-B commands on EDGE-R2 with neighbor `198.51.100.5`. **Expected:** the eBGP neighbor state is Established (on classic IOS, an established neighbor is represented by a number in the `State/PfxRcd` column), and the simulated internet prefix appears.

### Phase 2 — Loopback-sourced iBGP

```cisco
! EDGE-R1
ip route 10.255.0.2 255.255.255.255 10.0.12.2
router bgp 65100
 neighbor 10.255.0.2 remote-as 65100
 neighbor 10.255.0.2 update-source Loopback0
 neighbor 10.255.0.2 next-hop-self

! EDGE-R2
ip route 10.255.0.1 255.255.255.255 10.0.12.1
router bgp 65100
 neighbor 10.255.0.1 remote-as 65100
 neighbor 10.255.0.1 update-source Loopback0
 neighbor 10.255.0.1 next-hop-self
```

**Why `next-hop-self` is required:** The ISP-A eBGP route arrives on R1 with NEXT\_HOP `198.51.100.1`. When R1 passes that path over iBGP, the ordinary BGP behavior can leave this next hop unchanged. R2 has no connected route to `198.51.100.1`, so the iBGP path may be unusable. With `next-hop-self`, R1 advertises NEXT\_HOP `10.255.0.1`, which R2 reaches through its explicit `/32` static route. The reverse holds for R2 and ISP-B.

```cisco
show ip bgp summary
show ip bgp neighbors 10.255.0.1
show ip bgp neighbors 10.255.0.2
show ip route 10.255.0.1
show ip route 10.255.0.2
show ip bgp 192.0.2.0
```

**Important:** At this intermediate phase, before LOCAL\_PREF is changed, eBGP routes generally win the tie over iBGP routes at each edge. After Phase 4, R2 selects the A path. With ordinary iBGP R1 should not be expected to retain an independently advertised B path while R2 is selecting A. `show ip bgp` is not proof that all physically available paths are being distributed across iBGP.

### Phase 3 — Originate the company's prefix twice

```cisco
! Both EDGE-R1 and EDGE-R2
router bgp 65100
 network 203.0.113.0 mask 255.255.255.0
```

The `network` statement only originates an exact-prefix BGP route if `203.0.113.0/24` is present in the local RIB. Here the LAN interface provides that connected route. No fake null route or extra redistribution is necessary.

```cisco
! On each edge router
show ip route 203.0.113.0
show ip bgp 203.0.113.0

! On INTERNET
show ip bgp 203.0.113.0
```

**Expected at INTERNET:** two AS paths for the company's `/24`, one via AS 65001 and one via AS 65002. Only one is normally selected for the global routing table; seeing two **BGP paths** is different from having two installed FIB routes. Both will remain available only while both providers can advertise their customer-derived path; read their next hops and AS\_PATH values, not just a summary count.

## 5\. Traffic engineering: phases 4–6

### Phase 4 — Prefer ISP-A for outgoing traffic

The inbound route-map on R1 sets LOCAL\_PREF **200** on the simulated internet prefix; the equivalent on R2 sets LOCAL\_PREF **100**. Local preference is propagated **inside** AS 65100 through iBGP, but is not sent to external ASes.

```cisco
! EDGE-R1
ip prefix-list SIMULATED-INTERNET seq 5 permit 192.0.2.0/24
route-map ISP-A-IN permit 10
 match ip address prefix-list SIMULATED-INTERNET
 set local-preference 200
router bgp 65100
 neighbor 198.51.100.1 route-map ISP-A-IN in

! EDGE-R2
ip prefix-list SIMULATED-INTERNET seq 5 permit 192.0.2.0/24
route-map ISP-B-IN permit 10
 match ip address prefix-list SIMULATED-INTERNET
 set local-preference 100
router bgp 65100
 neighbor 198.51.100.5 route-map ISP-B-IN in
```

The route-maps have **implicit deny**: neither edge accepts unrelated announcements from its ISP. In a production design, a different acceptance policy may be appropriate (especially if receiving default or full tables); here `192.0.2.0/24` is the only intended service prefix.

**Expected forwarding:** R1 sends `192.0.2.0/24` to ISP-A. R2's best BGP path becomes the route through R1 (LOCAL\_PREF 200), *despite* R2's directly learned eBGP route through ISP-B (LOCAL\_PREF 100). LOCAL\_PREF is checked before the eBGP-vs-iBGP preference in common Cisco best-path selection.

```cisco
! EDGE-R2
show ip bgp 192.0.2.0
show ip route 192.0.2.0
traceroute 192.0.2.1 source 203.0.113.3

! HOST-1 (VPCS)
trace 192.0.2.1
```

From HOST-1, the normal traceroute is expected to follow **HSRP VIP -> EDGE-R1 -> ISP-A -> INTERNET**. Depending on the emulator, routers may show their physical interface address rather than the HSRP virtual address at hop 1, and some hops may not answer TTL-expired probes. Check routing tables to corroborate the path.

### Phase 5 — Make inbound traffic prefer ISP-A

Apply AS-path prepending on **R2's outbound eBGP announcements only**:

```cisco
! EDGE-R2
ip prefix-list CUSTOMER-ONLY seq 5 permit 203.0.113.0/24
route-map ISP-B-OUT permit 10
 match ip address prefix-list CUSTOMER-ONLY
 set as-path prepend 65100 65100 65100
router bgp 65100
 neighbor 198.51.100.5 route-map ISP-B-OUT out
```

R1 does **not** prepend toward ISP-A. On the internet simulator the path via A should look like `65001 65100` and via B should look like `65002 65100 65100 65100 65100` (three manual prepends plus the ordinary customer AS added during eBGP export). The precise displayed AS\_PATH formatting depends on IOS version; compare lengths, not whitespace.

```cisco
! INTERNET
show ip bgp 203.0.113.0
show ip route 203.0.113.0
traceroute 203.0.113.10 source 192.0.2.1
```

**Expected:** INTERNET prefers ISP-A *if earlier best-path decision attributes are equal*. AS-path prepending does not override a remote AS's local preference or its commercial traffic policy.

### Phase 6 — Strict non-transit filtering

The only externally advertised prefix is `203.0.113.0/24`:

```cisco
! Both edge routers
ip prefix-list CUSTOMER-ONLY seq 5 permit 203.0.113.0/24

! EDGE-R1
route-map ISP-A-OUT permit 10
 match ip address prefix-list CUSTOMER-ONLY
router bgp 65100
 neighbor 198.51.100.1 route-map ISP-A-OUT out

! EDGE-R2
route-map ISP-B-OUT permit 10
 match ip address prefix-list CUSTOMER-ONLY
 set as-path prepend 65100 65100 65100
router bgp 65100
 neighbor 198.51.100.5 route-map ISP-B-OUT out
```

The implicit deny on each route-map rejects any other eBGP announcement, including the INTERNET service and any learned provider routes. The `network` statements do not themselves bypass these export policies.

```cisco
! EDGE-R1
show ip bgp neighbors 198.51.100.1 advertised-routes
show ip prefix-list CUSTOMER-ONLY
show route-map ISP-A-OUT

! EDGE-R2
show ip bgp neighbors 198.51.100.5 advertised-routes
show ip prefix-list CUSTOMER-ONLY
show route-map ISP-B-OUT

! ISP-A / ISP-B: view incoming path, if any, for simulated service
show ip bgp 192.0.2.0
show ip bgp 203.0.113.0
```

**Expected export proof:** the only advertised IPv4 unicast NLRI toward either upstream is `203.0.113.0/24`. The providers may legitimately learn `192.0.2.0/24` **directly from INTERNET**; the key is proving they do *not* learn an alternate ISP path with customer AS **65100** as transit. For a strong injection test, temporarily (only in the simulator) remove both customer export route-maps, clear outbound policies, examine what happens, then restore filters; never leave the edge unfiltered. Even without the filters, ordinary iBGP best-path behavior can affect exactly which accidental routes are forwarded, so an absence of observed transit in one run is **not** evidence that the filters are unnecessary.

> \*\*BGP security note:\*\* Protect both eBGP edges even if the current simple topology seems to prevent some routes from propagating. For production also implement maximum-prefix limits, monitoring and upstream prefix/AS-path filters; consider RPKI origin validation and registries. The company prefix-list is the main defense required for this lab.

## 6\. Gateway redundancy: phase 7

Both edges share the LAN `203.0.113.0/24`, and HOST-1 uses `.1` as gateway. HSRP group 1 on R1 has **priority 120**, and R2 has **priority 100**. HSRP `preempt` lets the preferred router retake ownership. Track R1's ISP-A *physical* interface with object 10 and decrement 40 on failure, reducing its effective priority to **80** so R2 becomes Active.

```cisco
! EDGE-R1
track 10 interface GigabitEthernet0/0 line-protocol
interface GigabitEthernet0/2
 standby version 2
 standby 1 ip 203.0.113.1
 standby 1 priority 120
 standby 1 preempt delay minimum 30
 standby 1 track 10 decrement 40

! EDGE-R2
interface GigabitEthernet0/2
 standby version 2
 standby 1 ip 203.0.113.1
 standby 1 priority 100
 standby 1 preempt
```

```cisco
show standby brief
show standby
show track 10
show ip interface brief
```

|Condition|R1 effective priority|R2 priority|Expected Active|
|-|-:|-:|-|
|Normal, ISP-A uplink up|120|100|R1|
|ISP-A interface down, both routers alive|80|100|R2|
|EDGE-R1 router powered off|unavailable|100|R2|
|ISP-A restored|120 after tracking recovery|100|R1 after preempt delay|

**Limit:** Interface tracking is not true BGP-neighbor tracking. If the outside Ethernet stays up but the BGP session to ISP-A dies, HSRP stays active on R1 even though BGP can reroute R1's traffic through iBGP to R2 (hairpin). For a real deployment, couple HSRP to a **BGP-session-aware track object** or upstream health check, and test that signal cannot accidentally succeed via the *other* provider. One approach on a capable image is an IP SLA probe pinned to the ISP-A link coupled to `track` + HSRP; another is automation that adjusts a track object based on the actual BGP neighbor state. An ICMP check to the provider-facing IP proves L3 reachability, **not** that its BGP process/session is healthy.

**Symmetry limit:** HSRP plus inbound AS-path prepending makes primary path alignment likely, but cannot guarantee bidirectional symmetry. ISP/customer policy can produce a different return path; stateful firewalls need a proper HA/session-sync design or a deliberately symmetric forwarding architecture.

## 7\. Failover and recovery: phase 8

### Test A — ISP-A physical uplink failure

1. Establish the normal baseline from HOST-1: `ping 192.0.2.1` and `trace 192.0.2.1`. Verify HSRP Active R1 and INTERNET's best company-prefix path via ISP-A.
2. Start continuous pings with timestamps from a host capable of logging them (or repeat pings and capture the console with a clock).
3. On **EDGE-R1**, execute:

```cisco
configure terminal
interface GigabitEthernet0/0
 shutdown
end
```

4. Confirm `show standby brief` now identifies **EDGE-R2 as Active**, and on R2 `show ip bgp 192.0.2.0` / `show ip route 192.0.2.0` select ISP-B. On INTERNET, `show ip bgp 203.0.113.0` should select the via-ISP-B path. Retest HOST-1 reachability and trace.
5. Recover the link:

```cisco
configure terminal
interface GigabitEthernet0/0
 no shutdown
end
```

6. Wait for the BGP relationship to re-establish and path selection to change. Verify return to the ISP-A best route on R2 and INTERNET, and R1's HSRP Active status (configured preempt delay can intentionally postpone failback).

**Important direction distinction:** Outbound convergence and *inbound* route convergence are separate events. The traceroute from HOST-1 tests the outbound route, while INTERNET's route table and reverse traceroute test inbound routing. HSRP failover timing can differ from BGP reconvergence.

### Test B — EDGE-R1 complete failure

Power off or stop EDGE-R1 in PNetLab/EVE-NG (do not merely shut its ISP-A interface). R2 must become HSRP Active after timers expire, keep the service reachable using ISP-B, and continue advertising `203.0.113.0/24`. Recover R1 and confirm the primary configuration returns. In real life a routing convergence interval is expected; "no outage" as an availability objective is not literally zero lost packets.

### Test C — Silent BGP failure (critical extra credit)

Simulate loss of the R1–ISP-A BGP session while the interface stays `up/up` (e.g., misconfigure the neighbor in an isolated test). Demonstrate that interface-only HSRP tracking **does not** take R1 out of Active. Check whether R1's route via iBGP/R2 works. This identifies the gap between link redundancy and routing-session redundancy. Remedy with a supported, independently verified BGP-health tracking mechanism before presenting the design as production-ready.

### Timing methodology

Record these fields using timestamps in a terminal/logging tool (host time, router syslog with timestamps, or network capture). A continuous ping interval determines the precision of the packet-loss estimate.

|Test|Failure trigger time|First lost reply|First recovered reply|Recovery duration|Lost probes|
|-|-|-|-|-|-|
|ISP-A down|**Measure**|**Measure**|**Measure**|**Measure**|**Measure**|
|ISP-A up (failback)|**Measure**|**Measure**|**Measure**|**Measure**|**Measure**|
|EDGE-R1 off|**Measure**|**Measure**|**Measure**|**Measure**|**Measure**|
|BGP-only failure|**Measure**|**Measure**|**Measure**|**Measure**|**Measure**|

Calculate **recovery duration = timestamp of first recovered reply − failure trigger timestamp**. Also state RTT/timeout settings and whether the first lost packet was captured, because single-second ping intervals can make a sub-second outage difficult to measure. Save time-series captures in `evidence/13-failover-timings.csv`.

### Optional: faster failure detection

* **Direct interface shutdown:** local interface loss is generally detected immediately by the router; remaining delay includes BGP export/withdrawal, propagation, HSRP and ARP/MAC movement.
* **BGP hold timer:** when a link stays up but packets disappear, default-like IOS BGP keepalive/hold values commonly are 60/180 seconds (negotiated by peers), so detection can be slow. `neighbor ... timers <keepalive> <holdtime>` may shorten silent-peer detection, but needs support/testing and can cause flaps under transient delay.
* **BFD:** fast bidirectional forwarding detection with BGP integration can bring detection down to low hundreds of milliseconds or faster where supported. First verify your IOL/CSR image and ISP simulator support BFD before configuring it; avoid unsupported copied commands.
* **HSRP timers:** smaller hello/hold or use supported millisecond HSRP settings if available, but aggressive timings can increase false handovers. Coordinate with route availability to avoid moving the gateway before the backup path is usable.

## 8\. Challenge injections

### Injection 1 — BGP stuck Active/Idle: causes and confirmation

|Cause|Test|Fix|
|-|-|-|
|Wrong peer IP address|`show run|section router bgp`, `show ip interface brief`|
|Incorrect `remote-as`|`show ip bgp summary`, compare both running configs|Set each peer to its *actual* ASN|
|Link down / miswired / disabled|`show interfaces`, `show ip interface brief`|Repair link or `no shutdown`|
|Bad subnet mask / no IP reachability|`ping <neighbor>`, `show ip route <neighbor>`|Correct interface IP/mask or route|
|TCP/179 blocked by ACL/firewall|`show access-lists`, `show ip bgp neighbors`, `telnet <peer> 179` where supported|Permit bidirectional BGP TCP session establishment|
|iBGP loopback transport lacks route|`ping <peer-loopback> source Loopback0`; `show ip route <peer-loopback>`|Add the `/32` loopback route or IGP|
|`update-source` missing / wrong|`show run|section router bgp` and neighbor details|
|Wrong TTL for non-direct eBGP peers|`show ip bgp neighbors`; examine hops/TTL|Use appropriate `ebgp-multihop` only when topology calls for it|
|BGP MD5/password mismatch|Syslog and neighbor debug/logs|Make password match|
|Peer not activated under address-family|`show run|section router bgp`|

Debug commands (`debug ip bgp` or platform equivalent) can be noisy; use `terminal monitor`, syslog, and `undebug all` after a controlled observation.

### Injection 2 — Route exists in BGP but not installed

**Most likely:** unresolvable iBGP NEXT\_HOP. R1 learns a route over eBGP with next hop `198.51.100.1`, then forwards it to R2 over iBGP while retaining that original next hop. R2 has no route toward this provider link, so the path cannot enter the RIB and cannot be used. Verify with `show ip bgp 192.0.2.0`, `show ip route 198.51.100.1` and `ping 10.255.0.1 source Loopback0`. Resolve with `neighbor 10.255.0.2 next-hop-self` on R1 (and reverse on R2), plus functional loopback transport routes. **Other possibilities:** another path has better BGP attributes, an already-installed route with lower administrative distance wins RIB installation, or a policy denies/invalidates it. The absence of `>` alone is not proof of next-hop failure.

### Injection 3 — Company becomes unintended transit

**Root mistake:** missing or overpermissive outbound BGP export policy. If AS 65100 propagates routes learned from ISP-A toward ISP-B, traffic between networks can begin traversing corporate circuits. Risks include bandwidth exhaustion, increased latency, billing/overage, service outages, provider contract violations, and increased DDoS exposure. **Fix:** the exact-match `CUSTOMER-ONLY` prefix-list enforced by both `ISP-A-OUT` and `ISP-B-OUT`; verify `advertised-routes` on both neighbor sessions and AS\_PATH at both ISPs. Remember that implicit route-map denies prevent route leakage even if more routes later appear in the corporate BGP table.

### Injection 4 — AS prepend did not change inbound path

Possible reasons:

1. The remote AS sets a higher **LOCAL\_PREF** for ISP-B, evaluated before AS path length.
2. The route-map was applied in the wrong direction/neighbor or did not match the prefix; inspect `advertised-routes` and remote `show ip bgp` AS\_PATH.
3. A more-specific prefix through ISP-B wins **longest-prefix match** regardless of AS\_PATH on the aggregate; check the FIB and all advertisements.
4. A provider strips/rewrites AS\_PATH or other attributes, applies internal policies, or INTERNET's competing route is filtered/unavailable.
5. The alternative path's other decision criteria differ or changes were not propagated yet.

**Alternatives:** provider-supported BGP communities to change inbound LOCAL\_PREF, ISP portal traffic-engineering controls, MED where providers agree to honor it (mainly when comparing routes from the same neighboring AS), or better prefix-specific steering if provider policies permit and operational cost is justified. You cannot mandate an external AS's choice from the customer side.

### Injection 5 — Failover took three minutes

A **\~180-second** delay strongly suggests hold-time expiration for a session that stopped receiving BGP messages while its interface remained up. Differentiate this from an administrative shutdown, which usually signals link-down locally. Other contributors include keepalive interval, negotiated hold time, TCP behavior, BGP best-path processing, MRAI/update propagation, route flap dampening if configured, HSRP hello/hold/preemption delay and ARP/MAC relearning.

Fix options and risks: lower BGP hold timers (false neighbor drops), enable BFD (extra operational load/compatibility, sensitivity to packet loss), use BGP neighbor fall-over or connected-interface detection where applicable (only detects failures its mechanism can observe), tune HSRP timers (false gateway elections), or use reliable IP SLA tracking (a flawed probe can falsely switch to backup). Measure rather than assume faster is always better.

### Injection 6 — Asymmetric routing and firewalls

Outbound selection is governed by the company's LOCAL\_PREF; inbound selection is made independently by remote networks. Even though you prepend toward ISP-B, upstream policy may still prefer ISP-B. A stateful firewall seeing outbound on one appliance/interface and inbound on a different stateful path may drop return packets because the flow is absent from its session table or reverse-path checks fail. Use provider-supported policy/communities, carefully designed NAT and edge firewall placement, synchronized HA firewalls, or active/standby internet exit policy. Check actual flows in *both* directions with traceroute and firewall sessions; prepend alone never guarantees symmetry.

## 9\. Acceptance criteria and evidence

Mark a box only after reproducing and capturing the check on your chosen router image:

* \[ ] R1–ISP-A, R2–ISP-B and R1–R2 BGP sessions are Established (`show ip bgp summary` on both edges)
* \[ ] INTERNET receives two BGP paths to `203.0.113.0/24` (`show ip bgp 203.0.113.0`)
* \[ ] R2's `192.0.2.0/24` best path goes via R1 / ISP-A (`show ip bgp 192.0.2.0`)
* \[ ] HOST-1 reaches `192.0.2.1` over ISP-A under normal conditions (`trace 192.0.2.1`)
* \[ ] INTERNET prefers ISP-A to reach company `/24` (`show ip bgp 203.0.113.0`)
* \[ ] Both edges advertise **only** `203.0.113.0/24` to upstreams (`advertised-routes`)
* \[ ] HSRP Active on R1 when healthy; Active on R2 on R1 shutdown/outside link-down (`show standby brief`)
* \[ ] Traffic uses ISP-B after primary link or router failure (host ping/trace, R2 route, INTERNET BGP)
* \[ ] Traffic returns to ISP-A when recovered (host trace + INTERNET BGP)
* \[ ] Actual failure/recovery times and probe loss counted and documented
* \[ ] Note the standard iBGP best-path visibility caveat rather than claim R1 also displays both paths

### Evidence capture commands

```cisco
show version
show ip interface brief
show ip bgp summary
show ip bgp 192.0.2.0
show ip bgp 203.0.113.0
show ip route 192.0.2.0
show ip route 203.0.113.0
show ip bgp neighbors 198.51.100.1 advertised-routes
show ip bgp neighbors 198.51.100.5 advertised-routes
show standby brief
show track 10
show logging
```

Run commands on the **relevant** router; commands referencing a specific BGP neighbor or track object apply only to the edge configured for it. For screenshots and CSV naming see [`evidence/README.md`](evidence/README.md).

\---

**Repository integrity note:** All configuration files are authored examples derived from the challenge; no terminal verification or failover timings have been fabricated. Validate every command against your router image, then include authentic logs and screenshots when publishing final lab results.

