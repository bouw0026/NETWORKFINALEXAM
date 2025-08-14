# Predictive FINAL PRACTICE — OSPF, NAT, ACLs (RC Ethernet Focus)

**Total:** 50 pts · 13 questions  
*(OSPF/Routing ↑, NAT ↑, IPv6 = 0)*

---

## A) OSPF & Per-Hop Routing (26 pts)

### 1. Default ref-bw pitfall with RC’s Ethernet (3 pts)

**Prompt:**  
With default OSPF reference bandwidth (100 Mbps), what cost will IOS compute for a 100 Mbps Ethernet link and a 1 Gbps link? Why is that a problem in this topology where RC uses Ethernet e0/x while others use Gigabit?

**Answer:**  
Both compute to cost 1, so OSPF can’t tell 100 Mb from 1 Gb; RC’s slower e-links can be treated as “equal” to gig links, skewing path choices.

**Why likely:**  
Your notes emphasize ref-bw mismatches and cost calculation as a high-priority exam theme.

---

### 2. After tuning ref-bw to 10,000 (4 pts)

**Prompt:**  
You set `auto-cost reference-bandwidth 10000` on all routers. What are the OSPF costs now for 1 Gb vs 100 Mb links, and how does this change the network’s preferred transit through RC?

**Answer:**  
1 Gb → 10; 100 Mb → 100. RC’s 100 Mb links become expensive, so routers avoid transiting RC unless the destination is RC’s LAN.

**Why likely:**  
The lab’s OSPF-Tuning explicitly pushes raising ref-bw to reveal real path differences.

---

### 3. “bandwidth” vs ip ospf cost (4 pts)

**Prompt:**  
RC’s e0/1 physically negotiates 100 Mb. Should you change the interface bandwidth or directly set `ip ospf cost` to influence OSPF? Explain the side-effects.

**Answer:**  
Prefer `ip ospf cost` for OSPF-only tuning; changing bandwidth also affects QoS, EIGRP metrics, etc., so it’s best avoided for cost tuning.

**Why likely:**  
The text calls out three cost methods and warns against misusing bandwidth.

---

### 4. OSPF network type on RC’s Ethernet P2P (3 pts)

**Prompt:**  
By default, what OSPF network type runs on Ethernet links like CORE g0/1 ↔ RC e0/1? What would switching to point-to-point change?

**Answer:**  
Default is broadcast (with DR/BDR). Switching to point-to-point removes DR/BDR on that two-router segment and eliminates Type-2 network LSAs there.

**Why likely:**  
Your materials highlight network types and DR/BDR behavior on Ethernet.

---

### 5. Per-hop path from PC-B → 192.0.2.80 with tuned costs (5 pts)

**Prompt:**  
With ref-bw=10000 on all routers, predict whether RB will prefer a path through RC or direct to the edges for traffic bound to the Remote HTTP server (192.0.2.80). Give the reason tied to link costs.

**Answer:**  
RB will prefer direct edge paths (RB↔EDGE-B or via the L2-SW/RA), not through RC, because RC’s e-links (100 Mb, cost≈100) are worse than gig uplinks (cost≈10).

**Why likely:**  
The topology explicitly shows RC on e-interfaces and Remote servers on 192.0.2.0/24, inviting cost-driven path questions.

---

### 6. What cost would you expect on RC e0/1? (3 pts)

**Prompt:**  
After ref-bw=10000, what cost do you expect for RC e0/1 (100 Mb) in `show ip ospf interface e0/1`?

**Answer:**  
Approximately 100. (Reference/Interface → 10000/100).

**Why likely:**  
Straight application of the formula tested in prior labs.

---

### 7. Multi-access DR/BDR and who’s on it (4 pts)

**Prompt:**  
On the 10.10.245.0/28 L2 segment (RA, RB, CORE), which router is most likely DR at defaults and why doesn’t RC factor here?

**Answer:**  
CORE (highest RID among those three at default priority). RC isn’t cabled to that segment; its links are e0/1 to CORE and e0/3 to RB only. DR elections are non-preemptive.

**Why likely:**  
DR/BDR with who’s present on the broadcast net is a frequent check.

---

## B) NAT Behaviour & Throughput Reality (16 pts)

### 8. “Slow only for PC-C” complaint (5 pts)

**Prompt:**  
Users on PC-C (10.10.6.0/24) report slow web to 192.0.2.80 while others are fine. NAT at EDGE-A is correct. Name two OSPF checks and one reason tied to RC’s Ethernet links.

**Answer:**  
Check (1) ref-bw alignment on all routers, (2) per-hop route from RC with `show ip route`/`traceroute`. Reason: RC’s 100 Mb e-links (cost≈100) create a bottleneck vs the 1 Gb paths others take.

**Why likely:**  
The “High-Priority Topics” link cost warnings plus Remote server context.

---

### 9. PAT scope must include PC-C subnet (4 pts)

**Prompt:**  
EDGE-A PAT uses a standard ACL that only matches 10.10.2.0/24. PC-C cannot browse. Fix the ACL so all inside (10.10.x.x) can PAT, and state the two show commands you’ll use to confirm.

**Answer:**  
Widen to `permit 10.10.0.0 0.0.255.255` (or add per-subnet lines); verify with `show ip nat translations` and `show ip nat statistics`.

**Why likely:**  
NAT-ACL scope errors are explicitly listed in the brief.

---

### 10. Dual-edge defaults but symmetric NAT (4 pts)

**Prompt:**  
Both edges must advertise a default and do PAT, but you must avoid asymmetric return. Give two viable designs.

**Answer:**  
(i) Use HSRP/VRRP with active-only NAT at the active edge;  
(ii) PBR on CORE/RB/RA to steer all egress to one edge, with tracked failover. (Either keeps forward/return symmetric through the same NAT box.)

**Why likely:**  
Asymmetric routing + NAT are called out as exam terms.

---

### 11. Reading a PAT entry (3 pts)

**Prompt:**  
You see:  
`tcp 10.10.6.25:50832 203.0.113.1:50832 192.0.2.80:80 192.0.2.80:80`  
Label inside local, inside global, outside local, outside global.

**Answer:**  
- inside local = 10.10.6.25:50832  
- inside global = 203.0.113.1:50832  
- outside local = 192.0.2.80:80  
- outside global = 192.0.2.80:80

**Why likely:**  
NAT table interpretation is a perennial quick-hit.

---

## C) ACLs to Protect the Bottleneck (8 pts)

### 12. Surgical block to protect RC’s link (4 pts)

**Prompt:**  
Temporarily block Student (10.10.20.0/24) from 192.0.2.80 TCP/80 only (to ease RC congestion) while allowing everything else. Provide the two key lines and a best-practice placement.

**Answer:**
```shell
access-list 110 deny   tcp 10.10.20.0 0.0.0.255 host 192.0.2.80 eq 80
access-list 110 permit ip  10.10.20.0 0.0.0.255 any
```
Apply near the source (CORE toward edges or Vlan20 inbound).

**Why likely:**  
Extended ACL placement vs standard—explicit in your prep sheet.

---

### 13. Don’t advertise VLAN30 (4 pts)

**Prompt:**  
You want VLAN30 local only on CORE (no OSPF advertisement). With interface-mode OSPF, what exact change prevents its LSA?

**Answer:**  
Remove OSPF from the SVI:  
`interface Vlan30 → no ip ospf 10 area 0`  
(not just passive-interface, which still advertises).

**Why likely:**  
Subtle, high-value distinction your prof likes to probe.

---

## Why this exam shape fits the prof’s M.O.

- Heavy OSPF emphasis (RC’s Ethernet links) → exact match to “OSPF cost & ref-bw mismatches” and “per-hop tracing” priorities.
- NAT items lean on ACL scope and symmetry (common, grading-friendly errors).
- Concise ACL tasks test placement and minimal rules (the prof’s usual quick-check format).
