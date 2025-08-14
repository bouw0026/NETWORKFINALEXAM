# FINAL EXAM (Practice) — Routing, OSPF, NAT, and ACLs

---

## PART A — OSPF Neighbours, Costs, and Path Interpretation (18 points)

### 1) OSPF DR/BDR on the shared L2 segment (3 pts)

**Prompt (student):**  
On the `10.10.245.0/28` multi-access segment (`RA g0/2`, `RB g0/2`, `CORE g0/2` to L2-SW), all three routers run OSPF Area 0 with default interface priorities. RIDs are loopbacks: `RA=10.255.0.2`, `RB=10.255.0.4`, `CORE=10.255.0.5`.  
**1a)** Who is DR? **1b)** Who is BDR? **1c)** Will roles change if CORE later raises its priority after election without resetting anything?

**Answer:**  
- **1a)** DR = CORE (highest RID .5)  
- **1b)** BDR = RB (next highest RID .4)  
- **1c)** No. OSPF DR elections are non-preemptive; roles persist until adjacency reset.

> *Why this is likely:* Your notes emphasise neighbour relationships and multi-access behaviour; the L2 segment with RA/RB/CORE is explicit in the topology and the loopback-RID convention is called out. Students must recall DR/BDR rules and non-preemption.

---

### 2) Reference bandwidth mismatch (3 pts)

**Prompt (student):**  
RA and CORE use auto-cost reference-bandwidth 10000, but RB is left at default (100). All inter-router links are 1 Gbps. Explain how this can change RA’s chosen next-hop to reach `10.10.4.0/24` (PC-B LAN) versus CORE’s chosen next-hop. Be specific about interface cost calculation.

**Answer:**  
With 10 Gb reference, a 1 Gb interface cost ≈ 10 (RA, CORE). On RB (100 Mb ref), 1 Gb rounds to 1. Different totals can make RA/CORE prefer different exits, while RB’s LSAs advertise cost=1 links — leading to inconsistent SPF views and possible asymmetric paths. Fix by aligning reference bandwidth on all routers.

> *Why this is likely:* The “High-Priority Topics” explicitly mention cost calc and ref-bandwidth mismatches as a trap that changes actual paths.

---

### 3) Traceroute vs neighbours (2 pts)

**Prompt (student):**  
True or False: A traceroute proves which OSPF neighbours a router has on each hop. Explain.

**Answer:**  
False. Traceroute shows forwarding path decisions (routing table), not OSPF adjacencies. You can have an adjacency that isn’t used for forwarding (and vice-versa).

> *Why this is likely:* Your sheet explicitly warns: “Traceroute shows forwarding paths, not OSPF neighbour relationships.”

---

### 4) Per-hop path thinking (3 pts)

**Prompt (student):**  
A ping from PC-C (`10.10.6.0/24` via RC) to PC-A (`10.10.2.0/24` via RA) appears to take RC→CORE→RA. Explain why “per-hop routing-table decisions” can produce this outcome even if the physical diagram looks like a shorter diagonal path.

**Answer:**  
Each router chooses the lowest-cost next-hop from its own table; equal costs or ref-bandwidth differences can steer RC to CORE first, then CORE to RA. Visual proximity is irrelevant; SPF sums interface costs, and ties can ECMP.

> *Why this is likely:* The note tells students to think per hop and warns that ECMP/costs may contradict a visual guess.

---

### 5) OSPF defaults and E2 routes (4 pts)

**Prompt (student):**  
EDGE-A has a static default to REMOTE and runs default-information originate. Internal routers are Area 0 only.  
**5a)** What route type do internal routers see for `0.0.0.0/0`?  
**5b)** Why inject the default from a single edge in this topology?

**Answer:**  
- **5a)** O*E2 `0.0.0.0/0` (external type-5, metric type E2).  
- **5b)** A single ASBR keeps exit deterministic and avoids asymmetric return paths, which would break NAT state when both edges translate.

> *Why this is likely:* Your files keep Area 0 only and highlight default origination and asymmetry/NAT pitfalls for the exam.

---

## PART B — NAT Behaviour and Troubleshooting (16 points)

### 6) Static vs PAT design (4 pts)

**Prompt (student):**  
You must publish an internal host `10.10.2.100` (PC-A LAN) to be reachable from REMOTE using `203.0.113.100`.  
**6a)** Write the two commands needed for the static one-to-one mapping (assume EDGE-A g0/0 is outside, g0/1 inside).  
**6b)** Contrast this with a PAT design for inside-to-outside browsing using the EDGE-A outside interface.

**Answer:**  
**6a)**
```shell
ip nat inside source static 10.10.2.100 203.0.113.100
(on interfaces) ip nat inside (g0/1) and ip nat outside (g0/0)
```
**6b)** PAT: ACL for `10.10.0.0/16`, then  
`ip nat inside source list <ACL> interface g0/0 overload` — many hosts share the single outside IP by ports.

> *Why this is likely:* NAT types (Static/Dynamic/PAT) and behaviour are listed as high-priority; the REMOTE /203.0.113.0/24 links are explicit in the topology.

---

### 7) Asymmetric routing breaks NAT (4 pts)

**Prompt (student):**  
Both EDGE-A and EDGE-B are configured with PAT and both inject OSPF defaults. Inside traffic to `192.0.2.80` (web) sometimes times out. Provide the root cause and one clean design fix.

**Answer:**  
- **Cause:** Asymmetric return. Flows exiting via EDGE-A may return via EDGE-B; the return hits a router with no matching NAT state and is dropped.
- **Fix:** Pick one edge as the only default+NAT (others standby), or use first-hop redundancy (HSRP/VRRP) to keep flows symmetric through a single active egress.

> *Why this is likely:* The sheet defines asymmetric routing and links it to NAT breakage — a classic exam scenario.

---

### 8) NAT pool boundary mistake (4 pts)

**Prompt (student):**  
On EDGE-B a student configures:
```shell
ip nat pool PUB 205.0.113.60 205.0.113.50 netmask 255.255.255.0
ip nat inside source list 10 pool PUB
```
Yet inside users can’t browse. Identify two errors and the fixes.

**Answer:**  
- Pool range reversed (start must be ≤ end) → fix order `205.0.113.50 205.0.113.60`.
- Missing/incorrect inside/outside roles or ACL; ensure `ip nat inside` on internal interfaces, `ip nat outside` on `205.0.113.x` uplink, and ACL 10 matches all inside subnets (`10.10.0.0/16`).

> *Why this is likely:* “NAT pools, subnet boundaries, and misconfigs” are in your exam cues; `205.0.113.0/24` is in your remote-edge link list.

---

### 9) Verifying translations (4 pts)

**Prompt (student):**  
You ping from PC-VLAN20 (`10.10.20.0/24`) to `192.0.2.53` (DNS). On EDGE-A, what two commands confirm NAT operation, and what do you expect to see?

**Answer:**  
- **Commands:** `show ip nat translations`, `show ip nat statistics`.
- **Expect:** dynamic entries mapping inside local `10.10.20.x` to inside global `203.0.113.1` with port numbers; statistics showing active translations increasing.

> *Why this is likely:* NAT behaviour/verification is listed; `192.0.2.53` is named as DNS in your server block.

---

## PART C — ACL Design, Placement, and Interpretation (16 points)

### 10) Standard vs Extended placement (3 pts)

**Prompt (student):**  
You must block Guest VLAN (`10.10.30.0/24`) from reaching the Server LAN `192.0.2.0/24` but allow Guest to go anywhere else.  
**10a)** Which ACL type fits best, and where would you apply it (in/out, which device)?  
**10b)** Why not use a standard ACL on CORE inbound from VLAN30?

**Answer:**  
- **10a)** Extended ACL near the source (CORE or first hop toward edge), applied outbound toward the edges or inbound on VLAN30 SVI, matching src `10.10.30.0/24` → dst `192.0.2.0/24`.
- **10b)** A standard ACL can only match source, so placed near source it would block Guest to everywhere, not just the server LAN.

> *Why this is likely:* The notes emphasize Standard vs Extended rules and placement decisions.

---

### 11) Crafting the rule set (5 pts)

**Prompt (student):**  
Write an extended ACL that:
- permits Faculty (`10.10.10.0/24`) to `192.0.2.80` tcp/80,
- denies Student (`10.10.20.0/24`) to `192.0.2.80` tcp/80,
- permits all other IP traffic from both subnets.  
Apply it on the edge toward REMOTE in the correct direction.

**Answer:**
```shell
access-list 110 permit tcp 10.10.10.0 0.0.0.255 host 192.0.2.80 eq 80
access-list 110 deny   tcp 10.10.20.0 0.0.0.255 host 192.0.2.80 eq 80
access-list 110 permit ip 10.10.10.0 0.0.0.255 any
access-list 110 permit ip 10.10.20.0 0.0.0.255 any
interface g0/0          ! (EDGE facing REMOTE)
 ip access-group 110 out
```

> *Why this is likely:* It tests rule ordering, specific service control, and device/dir placement — all called out under “Writing ACLs” and “Interpreting behaviour.”

---

### 12) Implicit deny & logging (4 pts)

**Prompt (student):**  
An engineer creates only:  
`access-list 15 deny 10.10.30.0 0.0.0.255` and applies it inbound on CORE’s Vlan30. Guest loses all connectivity. Explain why, and provide an improved end of ACL to help troubleshooting without breaking traffic.

**Answer:**  
- **Reason:** implicit deny any at end of every ACL. With only a deny line, everything else is dropped.
- **Fix:** Add `access-list 15 permit any` or, during testing, `deny ip any any log` after explicit permits to observe unexpected matches.

> *Why this is likely:* The notes explicitly ask you to interpret ACL behaviour from config; implicit-deny is a classic grading point.

---

## PART D — Routing Table & Per-Hop Decisions (20 points)

### 13) Fill-in next-hop / exit interface (8 pts)

**Prompt (student):**  
Given the topology (Area 0 only) and that EDGE-A originates the default, complete the table for CORE’s forwarding decisions.

| Destination   | Next-Hop IP (from CORE) | Exit Interface | Via (route type) |
|---------------|------------------------|---------------|------------------|
| 10.10.2.25    | next-hop RA (10.10.25.x) | g0/3         | O (intra-area)   |
| 10.10.6.50    | next-hop RC (10.10.56.x) | g0/1         | O (intra-area)   |
| 192.0.2.80    | next-hop toward EDGE-A (via 10.10.25.x or 10.10.245.x path per lowest cost), exit corresponding CORE link | O*E2 (default) |

> *Why this is likely:* Your sheet asks for routing-table interpretation and per-hop tracing; the default-via-EDGE-A design is baked into your materials.

---

### 14) ECMP awareness (4 pts)

**Prompt (student):**  
If CORE has two equal-cost paths to EDGE-A (via RA over `10.10.25.0/28` and via the shared `10.10.245.0/28`), what will OSPF do, and how could a ref-bandwidth mismatch on one router break that symmetry?

**Answer:**  
OSPF installs ECMP; IOS will keep multiple equal-cost routes (platform default often 4). If one router advertises different interface costs (mismatched reference), totals won’t tie, ECMP disappears, and path selection changes.

> *Why this is likely:* Your OSPF doc excerpt explains ECMP and maximum-paths; the exam brief flags reference-bandwidth pitfalls.

---

### 15) Flapping link impact (4 pts)

**Prompt (student):**  
The RA↔EDGE-A link intermittently drops. Describe two observable effects in OSPF output and one user-visible symptom.

**Answer:**  
- **OSPF:** adjacency change logs (`%OSPF-5-ADJCHG … to DOWN/FULL`), LSDB reflood/age changes; neighbours cycling states.
- **User:** transient loss/latency while SPF reconverges; possible path change via alternative links.

> *Why this is likely:* “Flapping network” is specifically defined in your exam notes; OSPF neighbour-state logs are shown in your OSPF theory docs.

---

## PART E — Short Practicals (Hands-on) (10 points)

### 16) Make LANs “quiet” but routable (5 pts)

**Prompt (student):**  
On RA, ensure the PC-A LAN is advertised in OSPF but does not send/receive Hellos. Provide the exact line(s).

**Answer:**  
Under OSPF:  
`passive-interface GigabitEthernet0/0` (RA’s LAN). The network remains advertised; no Hellos on that LAN.

> *Why this is likely:* Passive-interface is a frequent check; your materials focus on clean OSPF hygiene on user LANs.

---

### 17) Confirm the default on RB (5 pts)

**Prompt (student):**  
Which single command on RB proves it learned the default via OSPF from EDGE-A, and what code would you see?

**Answer:**  
`show ip route` → look for `O*E2 0.0.0.0/0` and the next-hop toward EDGE-A.

> *Why this is likely:* Your tasks repeatedly test default-injection visibility and route code literacy.

---

## Notes on style alignment

- I used checkboxes/tables and “interpret these outputs” format exactly like the Midterm Exam example (tick-boxes, per-hop tables, and short explanations).
