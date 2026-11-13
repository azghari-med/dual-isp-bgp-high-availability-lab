# LAB CHALLENGE: "One Cable Away From Offline" — Dual-ISP Edge with BGP

**Difficulty:** Intermediate to Advanced · **Est. time:** 4-6 hours · **Platform:** PNetLab / EVE-NG (IOL or CSR routers, no heavy VMs)
**Technologies:** eBGP · iBGP · local-preference · AS-path prepending · route filtering · HSRP/VRRP · BFD (optional)

> This file contains the scenario and tasks only. There are no configurations in it.
> Try each phase before you look anything up.

---

## The Scenario

**Brückner Handel GmbH** runs an online order system out of its Düsseldorf office. It has **one ISP, one router, one cable.**

Last month a contractor digging in the street cut that cable. The company was offline for **six hours**. No orders, no email, no phone (VoIP). The loss was around €40,000.

The post-incident review found:

| # | Finding | Consequence |
|---|---|---|
| 1 | Single ISP, single circuit | One cable cut took the whole company offline |
| 2 | Single edge router | Even with a second ISP, the router would still be a single point of failure |
| 3 | Static default route only | Nothing detected the failure, so there was no automatic recovery |
| 4 | Provider-assigned IP addresses | Moving to another ISP would mean renumbering every public service |

**Management brief:**

> "I want a second ISP. If one fails, nobody should notice. And I want to decide which ISP carries our traffic normally, not have it left to chance."

**Your job:** build the edge so the company is multihomed to two ISPs with BGP, with a clear primary and backup, automatic failover, and no way for the company to accidentally become a transit network for the internet.

---

## The Environment You Must Build

```
                       [ INTERNET ]  AS 65000
                       loopback 192.0.2.1/24 (simulated "internet")
                        /                  \
                  ISP-A  AS 65001        ISP-B  AS 65002
                       |                      |
                  EDGE-R1  ====  iBGP  ====  EDGE-R2        AS 65100 (the company)
                       \                      /
                        ---- HSRP / VRRP ----
                                  |
                          Edge LAN 203.0.113.0/24
                                  |
                          HOST-1 (a PC or VPC)
```

### Addressing

| Link / Network | Subnet | Notes |
|---|---|---|
| Company public block (PI space) | 203.0.113.0/24 | The prefix you advertise to both ISPs |
| EDGE-R1 to ISP-A | 198.51.100.0/30 | R1 `.2`, ISP-A `.1` |
| EDGE-R2 to ISP-B | 198.51.100.4/30 | R2 `.6`, ISP-B `.5` |
| ISP-A to INTERNET | 198.51.100.8/30 | |
| ISP-B to INTERNET | 198.51.100.12/30 | |
| EDGE-R1 to EDGE-R2 (iBGP link) | 10.0.12.0/30 | Plus loopbacks for iBGP peering |
| Edge LAN | 203.0.113.0/24 | R1 `.2`, R2 `.3`, HSRP VIP `.1`, HOST-1 `.10` |

| Router | ASN |
|---|---|
| INTERNET | 65000 |
| ISP-A | 65001 |
| ISP-B | 65002 |
| EDGE-R1 / EDGE-R2 (company) | 65100 |

> **Lab note:** you are using private ASNs and a documentation prefix. In real life you would need a public ASN and your own provider-independent (PI) block from your regional registry (RIPE in Germany). The routing technique is identical.

---

## The Requirements

### R1: Multihoming
The company must be connected to **both ISPs at the same time** over eBGP, so it has a live path to the internet through each one.

### R2: Provider-independent addressing
The company's public block must be **advertised to both ISPs**, so the internet can reach it through either one and no renumbering is needed if an ISP is dropped.

### R3: Primary and backup (outbound)
Outbound traffic from the company must **normally leave through ISP-A**. ISP-B carries it only if ISP-A fails.

### R4: Steer inbound traffic
Traffic coming **into** the company from the internet should also prefer ISP-A. You must influence this from your side without controlling the ISPs.

### R5: No transit
The company's AS must **never** advertise routes learned from one ISP to the other. It may only advertise **its own prefix**.

### R6: Automatic failover
If ISP-A fails (link down or session dead), traffic must shift to ISP-B **without manual intervention**, and shift back when ISP-A returns. You must measure how long failover takes.

### R7: Router redundancy
The loss of **one edge router** must not take the company offline. The inside hosts must keep a working gateway.

---

## Your Tasks

### Phase 0: Build and baseline
- [ ] Build the topology and address every link
- [ ] Confirm **every directly connected neighbor pings** before you touch BGP
- [ ] Build the INTERNET, ISP-A and ISP-B routers (they only need basic eBGP and a simulated internet prefix)

### Phase 1: eBGP to both ISPs
- [ ] Bring up eBGP from EDGE-R1 to ISP-A and from EDGE-R2 to ISP-B
- [ ] Confirm both sessions reach **Established**
- [ ] Verify you receive the simulated internet prefix from each ISP

### Phase 2: iBGP between the edge routers
- [ ] Bring up iBGP between EDGE-R1 and EDGE-R2 (use loopbacks and an IGP or static routes so they can reach each other)
- [ ] Make sure each router can use the routes learned by the other
- [ ] Confirm both routers see **both** paths to the internet prefix

### Phase 3: Advertise your prefix
- [ ] Originate the company's /24 into BGP on both edge routers
- [ ] Confirm the INTERNET router sees your prefix **via both ISPs**

### Phase 4: Outbound policy (primary / backup)
- [ ] Make ISP-A the preferred exit for the whole company
- [ ] Confirm EDGE-R2 sends its outbound traffic via R1 and ISP-A, not directly to ISP-B
- [ ] Prove it with a traceroute from HOST-1

### Phase 5: Inbound policy
- [ ] Make the internet prefer ISP-A to reach your /24
- [ ] From the INTERNET router, confirm which path it chooses and why

### Phase 6: Stop being a transit
- [ ] Restrict what each edge router advertises so only your own /24 goes out
- [ ] Prove that ISP-A does **not** learn ISP-B's routes through you (and the reverse)

### Phase 7: Gateway redundancy
- [ ] Give the edge LAN a redundant gateway using HSRP or VRRP
- [ ] Make the gateway's active router match your BGP primary, so the traffic path is not asymmetric

### Phase 8: Failover test
- [ ] Shut the ISP-A link and watch the routes move to ISP-B
- [ ] Time how long it takes
- [ ] Restore ISP-A and confirm traffic returns to it
- [ ] Optional: improve failover speed and compare

---

## Challenge Injections

**Injection 1: "The neighbor won't come up."**
One of your BGP sessions is stuck in **Active** (or bouncing between Active and Idle).
> *List at least five distinct causes, and the command or check you would use to confirm each.*

**Injection 2: "It's in the BGP table, but it's not used."**
EDGE-R2 shows a route from EDGE-R1 in `show ip bgp`, but there is no `>` (best) marker and nothing in the routing table.
> *What is the most likely reason? Why does it only happen to routes learned from an eBGP peer and passed over iBGP?*

**Injection 3: "We're a transit network."**
An engineer at ISP-B calls to say your AS is appearing in the path between ISP-A and ISP-B, and your 100 Mbps link is carrying their customers' traffic.
> *What did you forget to configure, and what could this cost you? Fix it and prove it.*

**Injection 4: "I prepended, but nothing changed."**
You added AS-path prepending toward ISP-B, yet the INTERNET router still prefers the path via ISP-B.
> *Give at least three reasons prepending might not work, and what you would try instead.*

**Injection 5: "Failover took three minutes."**
You shut the ISP-A link and it took far longer than you expected for traffic to move.
> *What timers are involved? What are the options for making it faster, and what is the risk of each?*

**Injection 6: "Asymmetric routing."**
Outbound traffic leaves via ISP-A, but the replies come back via ISP-B, and a stateful device in the path drops them.
> *Why does this happen? How do you keep the traffic path symmetric?*

---

## Design Questions

**Q1.** Why use **BGP** to connect to two ISPs instead of two static default routes with IP SLA tracking (like your bank lab)? When is each the right answer?

**Q2.** **Local-preference, AS-path prepending, MED.** Which of these influences **outbound** traffic and which influences **inbound**? Why can you directly control one but only suggest the other?

**Q3.** Why does iBGP need `next-hop-self` in this design? What does the next hop look like when a route crosses from eBGP into iBGP?

**Q4.** What is a **transit AS**, and why would a company never want to be one? What is the actual risk if you accidentally advertise one ISP's routes to the other?

**Q5.** Your lab uses private ASNs and a documentation prefix. What would a real company need to obtain to do this for real, and from whom? What happens to a private ASN if it reaches the public internet?

**Q6.** Do you want the **full internet routing table** from your ISPs, or just a **default route**? What are the trade-offs for a company this size, in memory, control and complexity?

**Q7.** BGP is slow to converge by default. Why? What is **BFD**, and what does it add that BGP timers alone cannot?

**Q8.** You run HSRP on the inside of the edge routers and BGP on the outside. **What links the two?** If EDGE-R1 loses its ISP-A session but is still the HSRP active router, what happens to the traffic, and how would you fix that?

**Q9.** Provider-assigned addresses versus provider-independent addresses: which one made this company's original problem worse, and why?

---

## Acceptance Criteria

- [ ] Both eBGP sessions and the iBGP session are Established
- [ ] The INTERNET router sees the company /24 through both ISPs
- [ ] Outbound traffic from HOST-1 normally leaves through ISP-A (proven by traceroute)
- [ ] The INTERNET router prefers ISP-A to reach the company /24
- [ ] The company advertises **only its own prefix**, and the transit test proves it
- [ ] Shutting ISP-A moves traffic to ISP-B automatically, and restoring it moves it back
- [ ] Losing one edge router does not take the company offline
- [ ] You measured and documented the failover time

---

## Evidence to Capture

1. `show ip bgp summary` on both edge routers (all sessions Established)
2. `show ip bgp` showing both paths and the best-path marker
3. A traceroute from HOST-1 going out through ISP-A
4. The INTERNET router's view of your prefix and its chosen path
5. The outbound filter proving only the /24 is advertised
6. The failover test: before, during and after shutting ISP-A, with the time measured

---

## Why This Lab Matters

Every campus, VPN and security design you have built ends at the same place: the edge, where your network meets the internet. If that one cable fails, nothing inside matters. Multihoming with BGP is how real organizations keep that edge alive, and it is the standard answer to "what happens when the ISP goes down?"

It also covers a large part of ENARSI: BGP neighbor states, path attributes, route filtering and route maps, and failover behavior. The troubleshooting injections are the kind of scenario the exam tests, and each one is a ready-made interview story.

---

*Try every phase before you look anything up. When you are ready, ask for the solution file with full configs and answers.*
