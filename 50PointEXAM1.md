# Revised FINAL EXAM (Practice) — Routing, OSPF, NAT, ACLs

**Total:** 50 pts · 13 questions · Difficulty: ~3.5/5  
*(IPv6 removed; extra weight on OSPF & NAT.)*

---

## A) OSPF Neighbours, Costs, Path Logic (24 pts total)

### 1) DR/BDR with explicit priorities (4 pts)

**Prompt:**  
On the shared `10.10.245.0/28` segment, priorities are set: CORE=100, RA=50, RB=1. RIDs remain `10.255.0.5` / `.2` / `.4`. After all three reload at the same time, who becomes DR and BDR? If you later bump RA’s priority to 110 without touching links, do roles change immediately?

**Answer:**  
DR=CORE (prio 100), BDR=RA (prio 50). Raising RA to 110 doesn’t preempt; roles change only if the adjacency resets.

**Why likely:**  
Multi-access DR/BDR with priority + non-preemption is a classic.

---

### 2) Reference-bandwidth alignment and path choice (4 pts)

**Prompt:**  
All inter-router links are 1 Gb. RA/CORE use auto-cost reference-bandwidth 100000 (100 Gb), RB remains default (100 Mb). How could this produce different chosen next-hops on RA vs CORE to reach `10.10.4.0/24`?

**Answer:**  
With ref=100 Gb, 1 Gb has cost ≈ 100 (RA/CORE), but RB advertises 1 Gb links as 1 (default ref=100 Mb). Mismatched costs skew SPF totals; RA and CORE may compute different best paths than expected. Fix: same reference on every router.

**Why likely:**  
Your notes call out ref-bw mismatches as a trap.

---

### 3) Passive-interface side-effects (4 pts)

**Prompt:**  
`show ip ospf interface Vlan20` on CORE shows Hello 10s / Dead 40s and neighbors present. You intended VLAN20 to be advertised but not form adjacencies. What setting is missing and what changes in output confirm the fix?

**Answer:**  
Missing `passive-interface Vlan20`. After enabling it, OSPF still advertises the subnet but Hellos stop (Hello=0), and no neighbors appear on Vlan20.

**Why likely:**  
Passive on user LANs is stressed for hygiene.

---

### 4) LSA types on a multi-access net (4 pts)

**Prompt:**  
On `10.10.245.0/28`, which LSA type represents the multi-access network, and who originates it? What human-readable clue in `show ip ospf database` ties it to the elected role?

**Answer:**  
Type-2 (Network LSA), originated by the DR for that segment. DB output shows the DR’s interface IP/RID as the LSA’s advertising router.

**Why likely:**  
Connects DR election to LSDB interpretation.

---

### 5) Traceroute vs neighbour tables (4 pts)

**Prompt:**  
A traceroute from PC-VLAN20 to `192.0.2.80` shows CORE→RA→EDGE-A→REMOTE. Name two things traceroute confirms and one thing it does not confirm.

**Answer:**  
Confirms the forwarding path and each hop’s reachable next-hop. Does not confirm OSPF neighborships (adjacencies).

**Why likely:**  
Your sheet warns “traceroute ≠ neighbors.”

---

### 6) Competing defaults: E1 vs E2 (4 pts)

**Prompt:**  
EDGE-A originates an E2 default. EDGE-B originates an E1 default. All else equal, which default do internal routers prefer and why?

**Answer:**  
Prefer the E1 default (from EDGE-B) because E1 adds internal path cost to the ASBR, allowing a lower-total-cost route to win over a flat E2 metric.

**Why likely:**  
Tests understanding of external type behavior and route selection.

---

## B) NAT Behaviour & Troubleshooting (16 pts total)

### 7) PAT ACL scope mistake (5 pts)

**Prompt:**  
On EDGE-A:
```shell
ip access-list standard NAT_INSIDE
 permit 10.10.2.0 0.0.0.255
ip nat inside source list NAT_INSIDE interface g0/0 overload
```
Users on `10.10.20.0/24` can’t browse; `10.10.2.0/24` can. Inside/outside roles are correct. Fix only the matching so all inside subnets work, and say how you’d verify.

**Answer:**  
Widen the match (e.g., `permit 10.10.0.0 0.0.255.255`) or add lines per subnet. Verify with `show ip nat translations` / `show ip nat statistics` and rising hit-counts on the ACL.

**Why likely:**  
Common real error—ACL too narrow for NAT.

---

### 8) Keep dual-edge, keep symmetry (4 pts)

**Prompt:**  
You must keep both EDGE-A and EDGE-B advertising a default and doing PAT, but avoid asymmetric return drops. Give two viable designs.

**Answer:**  
1. HSRP/VRRP on inside + outside, active-only NAT at the active edge; failover preserves symmetry.  
2. PBR on internal routers to steer all outbound to a single chosen edge, with tracked failover; replies follow the same path.

**Why likely:**  
Your brief emphasizes asymmetric routing and NAT state.

---

### 9) Read a NAT table (3 pts)

**Prompt:**  
Given a translation:  
`tcp 10.10.20.25:49512 203.0.113.1:49512 192.0.2.80:80 192.0.2.80:80`  
Label inside local, inside global, outside local, outside global.

**Answer:**  
- inside local = `10.10.20.25:49512`
- inside global = `203.0.113.1:49512`
- outside local = `192.0.2.80:80`
- outside global = `192.0.2.80:80` (often same)

**Why likely:**  
Interpretation of NAT table fields is a staple.

---

### 10) Static NAT for a published host (4 pts)

**Prompt:**  
Make `10.10.2.100` reachable from REMOTE as `203.0.113.100`. Write the mapping and the minimal interface role commands.

**Answer:**
```shell
ip nat inside source static 10.10.2.100 203.0.113.100
```
`ip nat inside` on the internal interface, `ip nat outside` on the `203.0.113.0/24` link.

**Why likely:**  
Simple, high-yield static NAT task.

---

## C) ACL Design, Placement, Interpretation (10 pts total)

### 11) Extended ACL with least collateral (4 pts)

**Prompt:**  
Block Student (`10.10.20.0/24`) from `192.0.2.53` (DNS) while allowing Student to reach everything else, and don’t affect Faculty. Provide two lines minimum and choose a placement that fits best-practice.

**Answer:**
```shell
access-list 110 deny   ip 10.10.20.0 0.0.0.255 host 192.0.2.53
access-list 110 permit ip 10.10.20.0 0.0.0.255 any
```
Apply near the source (e.g., CORE outbound toward the edges or Vlan20 inbound).

**Why likely:**  
Tests precise match + placement logic.

---

### 12) Lock down device management with a standard ACL (3 pts)

**Prompt:**  
Allow only Faculty (`10.10.10.0/24`) to access CORE’s vty. Provide the minimal commands.

**Answer:**
```shell
access-list 50 permit 10.10.10.0 0.0.0.255
line vty 0 4
 access-class 50 in
```

**Why likely:**  
Standard ACLs are perfect for mgmt plane.

---

### 13) Stop advertising a LAN in OSPF (3 pts)

**Prompt:**  
You decide VLAN30 should be local-only on CORE (routed internally but not learned by others). With interface-mode OSPF, what exact change prevents its advertisement?

**Answer:**  
Remove OSPF from that SVI:  
`interface Vlan30 → no ip ospf 10 area 0.`  
(Note: passive-interface still advertises; you must not enable OSPF on that SVI.)

**Why likely:**  
Subtle distinction—passive vs not-advertised.
