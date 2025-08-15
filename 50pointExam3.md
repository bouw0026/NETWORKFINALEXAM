# Exam Review — 50 Points (OSPF • NAT • ACL)

A compact, GitHub-ready practice set. Each item includes **realistic show output**, **targeted questions**, **answers with reasoning**, and **Cisco/Putty CLI** to fix/verify.

**Points:** Q1: 8 • Q2: 8 • Q3: 7 • Q4: 9 • Q5: 9 • Q6: 9 = **50**

---

## Q1 — OSPF Hello/Dead Timer Mismatch on `10.10.245.0/28` (8 pts)

**Given** (CORE & RB share the broadcast segment `10.10.245.0/28` via an L2 switch):

```text
CORE# show ip ospf interface g0/2
GigabitEthernet0/2 is up, line protocol is up
  Internet Address 10.10.245.5/28, Area 0, Network Type BROADCAST, Cost: 1
  State DROTHER, Priority 1
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
  Designated Router (ID) 10.255.0.5, Interface address 10.10.245.5
  Backup Designated router (ID) 10.255.0.4, Interface address 10.10.245.4

RB# show ip ospf interface g0/2
GigabitEthernet0/2 is up, line protocol is up
  Internet Address 10.10.245.4/28, Area 0, Network Type BROADCAST, Cost: 1
  State DROTHER, Priority 1
  Timer intervals configured, Hello 1, Dead 4, Wait 4, Retransmit 5

RB# show ip ospf neighbor
<no output>  ! (no neighbors seen)
```

**Questions**

- Will CORE and RB form a FULL adjacency on this segment? Why/why not?
- What exact fix aligns the two devices?
- After the fix, what should you verify?

**Answer (scored items)**

- The adjacency **will not form** because **Hello/Dead timers differ** (CORE **10/40** vs RB **1/4**). OSPF neighbors must match **area, network type, authentication, MTU, and timers**; timer mismatch causes Hellos to be ignored.
- **Fix** RB’s interface timers to **Hello 10 / Dead 40** to match CORE.
- After fixing, **verify** with `show ip ospf neighbor` that **FULL** adjacency appears and **Dead** timer counts down from ~**40s**.
- On a **broadcast** segment, **L2 switch presence does not add OSPF cost**; cost remains the interface cost (**1** here).
- Confirm **DR/BDR** are stable on the segment **after adjacency forms**.

**Fix / Verify (Cisco CLI)**

```console
RB# conf t
RB(config)# interface g0/2
RB(config-if)# ip ospf hello-interval 10
RB(config-if)# ip ospf dead-interval 40
RB(config-if)# end
RB# show ip ospf neighbor
RB# show ip ospf interface g0/2
```

---

## Q2 — DR/BDR & “Does L2 Add Cost?” Reinforcement (8 pts)

**Given** (RA, RB, CORE on `10.10.245.0/28` via L2-SW):

```text
RA# show ip ospf neighbor
Neighbor ID     Pri   State     Dead Time   Address        Interface
10.255.0.5        1   FULL/DR   00:00:33    10.10.245.5    GigabitEthernet0/2
10.255.0.4        1   FULL/BDR  00:00:37    10.10.245.4    GigabitEthernet0/2

RA# show ip ospf interface g0/2
GigabitEthernet0/2 ...
  Network Type BROADCAST, Cost: 1
  State DROTHER, Priority 1
  Designated Router (ID) 10.255.0.5, Interface address 10.10.245.5
  Backup Designated router (ID) 10.255.0.4, Interface address 10.10.245.4
```

**Questions**

- Who is DR and BDR on `10.10.245.0/28`?
- Does the presence of the Layer-2 switch increase OSPF cost by **+1 hop** on this segment?
- How would you force **RA** to become DR in the future?

**Answer (scored items)**

- **DR = 10.255.0.5 (CORE)**, **BDR = 10.255.0.4 (RB)** as shown by states **FULL/DR** and **FULL/BDR**.
- An **L2 switch does not add OSPF cost or hops**; OSPF sums **router interface costs only**.
- To prefer **RA** as DR at next election, **raise OSPF interface priority** on RA above others (e.g., **200**). DR/BDR elections are **non-preemptive**; roles change only after **adjacency reset/election**.

**Config to influence DR/BDR (next election)**

```console
RA# conf t
RA(config)# interface g0/2
RA(config-if)# ip ospf priority 200
RA(config-if)# end
! Reset adjacency or wait for a new election to see effect.
```

---

## Q3 — Reference-Bandwidth Mismatch & RC FastEthernet (7 pts)

**Given** (RA & CORE ref-bw raised; RB left default; RC uses 100Mb e-links):

```text
RA# show ip ospf
Routing Process "ospf 10"
  Reference bandwidth unit is 100000 Mbps   ! 100 Gb

RB# show ip ospf
Routing Process "ospf 10"
  Reference bandwidth unit is 100 Mbps      ! default

RC# show ip ospf interface e0/1
Ethernet0/1 ...
  Cost: 100
```

**Symptom**

```text
CORE# show ip route 192.0.2.80
O*E2 0.0.0.0/0 [110/1] via 10.10.25.1, GigabitEthernet0/3   (CORE prefers via RA)

RB# show ip route 192.0.2.80
O*E2 0.0.0.0/0 [110/1] via 10.10.34.1, GigabitEthernet0/1   (RB prefers via EDGE-B; ECMP/choices differ from expected)
```

**Questions**

- Why might routers pick different “best” exits even with identical topology?
- What uniform change restores consistent cost calculations?
- Why is **RC**’s **100 Mb** interface cost shown as **100** here?

**Answer (scored items)**

- **Reference-bandwidth mismatch** causes per-router cost calculations to differ, **skewing SPF** results.
- **Set the same** `auto-cost reference-bandwidth` on **every OSPF router** (e.g., **100000** for 100 Gb scale).
- With higher ref-bw, **1 Gb ≠ 100 Mb**; **100 Mb becomes cost ≈ 100**, **1 Gb ≈ 10 (or 1)**. RC’s e-link (100 Mb) shows **cost 100** under high ref-bw, correctly making it undesirable transit.

**Remediation (uniform ref-bw OR manual cost pinning)**

```console
! On EVERY router:
router ospf 10
 auto-cost reference-bandwidth 100000

! (Optionally pin cost on a specific interface)
interface e0/1
 ip ospf cost 100
```

---

## Q4 — NAT PAT Fails Due to Narrow Source ACL (9 pts)

**Given** (EDGE-A PAT config):

```text
EDGE-A# show run | sec ip nat
ip access-list standard NAT_INSIDE
 permit 10.10.2.0 0.0.0.255
!
ip nat inside source list NAT_INSIDE interface g0/0 overload
!
interface g0/1
 ip address 10.10.25.1 255.255.255.240
 ip nat inside
!
interface g0/0
 ip address 203.0.113.1 255.255.255.0
 ip nat outside

EDGE-A# show ip nat statistics
Total active translations: 0 (0 static, 0 dynamic; 0 extended)
Hits: 0  Misses: 0
```

**Symptom:** Only `10.10.2.0/24` users browse; **VLAN10/20/30** cannot.

**Questions**

- Why aren’t other subnets getting PAT?
- What’s the minimal safe change to match all inside `10.10.x.x` subnets?
- How do you immediately verify success?

**Answer (scored items)**

- The `NAT_INSIDE` ACL only matches **10.10.2.0/24**, so other inside sources **never hit NAT**.
- **Widen** the match to the enterprise range (e.g., **10.10.0.0/16**) or **add explicit lines** per subnet.
- Verify with `show ip nat statistics` (**hits rising**) and `show ip nat translations` (**active PAT entries**). Ensure **inside/outside** roles are correct (already correct above).

**Fix / Verify (Cisco CLI)**

```console
EDGE-A# conf t
EDGE-A(config)# ip access-list standard NAT_INSIDE
EDGE-A(config-std-nacl)# no 10
EDGE-A(config-std-nacl)# permit 10.10.0.0 0.0.255.255
EDGE-A(config)# end
EDGE-A# clear ip nat translation *   ! optional to reset
EDGE-A# show ip nat statistics
EDGE-A# show ip nat translations
```

---

## Q5 — Inbound Static NAT Blocked by Outside ACL (9 pts)

**Given** (server `10.10.10.5` statically mapped to `203.0.113.5`):

```text
EDGE-A# show run | sec ip nat
ip nat inside source static 10.10.10.5 203.0.113.5
!
interface g0/1
 ip nat inside
interface g0/0
 ip nat outside

EDGE-A# show run | sec access-list
101 access-list 101 permit tcp any host 10.10.10.5 eq 80
101 access-list 101 permit tcp any host 10.10.10.5 eq 443
101 access-list 101 deny ip any any

EDGE-A# show run | sec g0/0
interface g0/0
 ip access-group 101 in
```

**Symptom:** External users **cannot** reach `http(s)://203.0.113.5`.

**Questions**

- Why do the ACL permits not work?
- What exact change fixes it?
- Where should you apply it?

**Answer (scored items)**

- On the **outside interface inbound**, ACL sees packets **before NAT**; destination is **203.0.113.5**, **not 10.10.10.5**.
- The ACL should **match the public (global) IP** of the server. Replace the ACL to permit `tcp any host 203.0.113.5 eq 80/443`; apply **inbound on the outside interface**.
- After ACL allows, NAT translates `203.0.113.5 → 10.10.10.5` and the session **succeeds**.

**Corrected ACL (Cisco CLI)**

```console
EDGE-A# conf t
EDGE-A(config)# no access-list 101
EDGE-A(config)# access-list 101 permit tcp any host 203.0.113.5 eq 80
EDGE-A(config)# access-list 101 permit tcp any host 203.0.113.5 eq 443
EDGE-A(config)# access-list 101 deny ip any any
EDGE-A(config)# end
! Applied already inbound on g0/0
```

---

## Q6 — Enterprise ACL: Block Guest→Faculty, Allow Internet (9 pts)

**Given** (initial, wrong standard ACL near source):

```text
CORE# show run | sec access-list 50
access-list 50 deny 10.10.30.0 0.0.0.255
!
interface Vlan30
 ip access-group 50 in

! Symptom: Guest (10.10.30.0/24) is blocked from everything, including internet.
```

**Questions**

- Why is this the wrong ACL type/placement?
- Provide an extended ACL that blocks **Guest→Faculty only** (`10.10.30.0/24 → 10.10.10.0/24`) while allowing Guest to the internet.
- Where should it be applied?

**Answer (scored items)**

- **Standard ACL** matches only **source**; placed near source, it **blocks all Guest traffic**.
- Use an **extended ACL** to match **src & dst**, **blocking only** Guest→Faculty.
- **Best practice:** place extended ACL **near the source** (**Vlan30 inbound**) to stop unwanted traffic early while allowing other destinations (internet).

**Corrected ACL (Cisco CLI)**

```console
CORE# conf t
CORE(config)# no access-list 50
CORE(config)# ip access-list extended GUEST-FILTER
CORE(config-ext-nacl)# deny ip 10.10.30.0 0.0.0.255 10.10.10.0 0.0.0.255
CORE(config-ext-nacl)# permit ip 10.10.30.0 0.0.0.255 any
CORE(config-ext-nacl)# exit
CORE(config)# interface Vlan30
CORE(config-if)# ip access-group GUEST-FILTER in
CORE(config-if)# end
```

---

## One-Minute Verification Cheat Sheet (optional practice)

```console
! OSPF health
show ip ospf neighbor
show ip ospf interface <if>
show ip ospf | i Reference

! Path & cost
show ip route <DEST>
traceroute <DEST>

! NAT/PAT
show ip nat statistics
show ip nat translations
show run | sec ip nat

! ACLs
show access-lists
show run interface <if> | i access-group
```

> **Remember for `10.10.245.0/28`:** the **Layer-2 switch adds no OSPF cost**; only **router interfaces** contribute to the metric.
