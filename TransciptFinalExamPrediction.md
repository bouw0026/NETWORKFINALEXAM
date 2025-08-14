# Final Exam Prediction — 50 Points (GitHub Edition)

> IPv4-only topology. Heavier weight on **OSPF & Routing** and **NAT**, ACLs still present.  
> Each question includes a collapsible **Answer** with a short “why this is likely” note tied to your course docs (e.g., *05-OSPF-Tuning.md*, *04-OSPF-Basic.md*, *11-NAT.md*, *NAT_Static_Example.md*, *09-Standard-ACLs.md*, *10-Extended-ACLs.md*, *networktopology.md*).

---

## Table of Contents
- [Part A — OSPF & Routing (28 pts)](#part-a--ospf--routing-28-pts)
  - [Q1. OSPF hello/dead mismatch (4 pts)](#q1-ospf-hellodead-mismatch-4-pts)
  - [Q2. DR/BDR election on the shared VLAN (3 pts)](#q2-drbdr-election-on-the-shared-vlan-3-pts)
  - [Q3. RC is FastEthernet — cost and path (5 pts)](#q3-rc-is-fastethernet--cost-and-path-5-pts)
  - [Q4. Per-hop reasoning vs. the picture (4 pts)](#q4-per-hop-reasoning-vs-the-picture-4-pts)
  - [Q5. OSPF neighbor and interface verification (3 pts)](#q5-ospf-neighbor-and-interface-verification-3-pts)
  - [Q6. Default route origination at the edge (4 pts)](#q6-default-route-origination-at-the-edge-4-pts)
  - [Q7. Floating static for WAN failover (5 pts)](#q7-floating-static-for-wan-failover-5-pts)
- [Part B — NAT (14 pts)](#part-b--nat-14-pts)
  - [Q8. PAT on EDGE-A using outside interface (5 pts)](#q8-pat-on-edge-a-using-outside-interface-5-pts)
  - [Q9. Static NAT / Port-forward for HTTP (4 pts)](#q9-static-nat--port-forward-for-http-4-pts)
  - [Q10. “It pings but TFTP fails” after PAT config (3 pts)](#q10-it-pings-but-tftp-fails-after-pat-config-3-pts)
  - [Q11. Order of operations with ACLs and NAT (2 pts)](#q11-order-of-operations-with-acls-and-nat-2-pts)
- [Part C — ACLs (8 pts)](#part-c--acls-8-pts)
  - [Q12. Permit web from Faculty only (5 pts)](#q12-permit-web-from-faculty-only-5-pts)
  - [Q13. ISP-edge anti-RFC1918 filter (3 pts)](#q13-isp-edge-anti-rfc1918-filter-3-pts)
- [Quick Command Checklist](#quick-command-checklist)

---

## Part A — OSPF & Routing (28 pts)

### Q1. OSPF hello/dead mismatch (4 pts)
You see CORE–RB neighbors stuck in `EXSTART/EXCHANGE` on the 10.10.245.0/28 segment. `show ip ospf interface` on CORE shows **Hello 10 / Dead 40**. On RB, **Hello 1 / Dead 4**.  
**a)** Name the problem. **b)** Fix with exact config on RB’s interface.

<details><summary><strong>Answer</strong></summary>

**a)** Hello/Dead timer mismatch prevents adjacency.  
**b)**
```console
interface GigabitEthernet0/2
 ip ospf hello-interval 10
 ip ospf dead-interval 40
```
Optionally reset adjacency: `clear ip ospf process` (during a window).

**Why likely:** Carolina emphasized timer pitfalls; Lab *05-OSPF-Tuning.md* shows timer alignment and when to reset OSPF.  
**Refs:** 05-OSPF-Tuning.md, 04-OSPF-Basic.md
</details>

---

### Q2. DR/BDR election on the shared VLAN (3 pts)
On 10.10.245.0/28 (CORE, RA, RB on a broadcast segment), you want CORE to be DR. What single per-interface knob would you set on CORE and RB to influence the election?

<details><summary><strong>Answer</strong></summary>

Set OSPF interface **priority** (higher wins). Example:  
```console
interface GigabitEthernet0/2   ! CORE's interface on the shared segment
 ip ospf priority 200
!
interface GigabitEthernet0/2   ! RB and RA on that segment
 ip ospf priority 1
```
**Why likely:** DR/BDR control via priority is a staple in *05-OSPF-Tuning.md*; non-preemptive elections are often tested.  
**Refs:** 05-OSPF-Tuning.md, 08-Cloud-OSPF-Routing.md
</details>

---

### Q3. RC is FastEthernet — cost and path (5 pts)
RC’s uplinks are **Ethernet (100 Mb/s)**. Other backbone links are **Gigabit**. With default OSPF reference-bandwidth (100 Mb/s), what cost distortion happens and what global fix should you apply? Give the exact command.

<details><summary><strong>Answer</strong></summary>

With **ref-bw=100**, both **100 Mb** and **1 Gb** round to **cost 1**, hiding the faster path and skewing SPF.  
Fix: raise ref-bw on **all** OSPF routers so costs reflect speed:
```console
router ospf 10
 auto-cost reference-bandwidth 100000   ! 100 Gb scale
```
**Why likely:** Reference bandwidth mismatches were a highlighted trap; your topology has RC on 100 Mb e-interfaces.  
**Refs:** 05-OSPF-Tuning.md, Implementing optional OSPF features.docx, networktopology.md
</details>

---

### Q4. Per-hop reasoning vs. the picture (4 pts)
From PC-VLAN20 (10.10.20.0/24) to the HTTP server **192.0.2.80**, explain **why traceroute might not match** your “visual” shortest line across the diagram. Give two technical reasons tied to OSPF behavior.

<details><summary><strong>Answer</strong></summary>

- **ECMP**: equal-cost multipath can split flows across parallel links.  
- **Per-hop cost differences**: each router computes its own shortest path; ref-bw and interface costs may differ; traceroute shows **forwarding paths**, not neighbor adjacencies.

**Why likely:** Your notes stress per-hop thinking and “traceroute ≠ neighbors.”  
**Refs:** 07-Applied-Learning.md, 08-Cloud-OSPF-Routing.md, networktopology.md
</details>

---

### Q5. OSPF neighbor and interface verification (3 pts)
Name **two** show commands you would capture to verify (i) neighbors and (ii) per-interface OSPF role/cost on CORE–EDGE-A.

<details><summary><strong>Answer</strong></summary>

```console
show ip ospf neighbor
show ip ospf interface GigabitEthernet0/x
```
**Why likely:** Standard verification flow drilled in OSPF labs.  
**Refs:** Basic OSPF theory part 2.docx, 04-OSPF-Basic.md
</details>

---

### Q6. Default route origination at the edge (4 pts)
EDGE-A connects toward REMOTE/Internet. You want everyone to learn a default. Which configuration style do you expect: **redistribute static** vs. **default-information originate**? State one and why.

<details><summary><strong>Answer</strong></summary>

Use **`default-information originate`** after adding a static default on EDGE-A:
```console
ip route 0.0.0.0 0.0.0.0 203.0.113.254
router ospf 10
 default-information originate
```
**Why likely:** Clean ASBR default injection is the pattern in your OSPF labs.  
**Refs:** 04-OSPF-Basic.md, 05-OSPF-Tuning.md
</details>

---

### Q7. Floating static for WAN failover (5 pts)
A backup internet link exists on EDGE-B. Write a **floating static default** on CORE that prefers OSPF’s learned default but kicks in if OSPF is lost. State the **AD** you’d pick.

<details><summary><strong>Answer</strong></summary>

```console
ip route 0.0.0.0 0.0.0.0 10.10.34.1 120
```
Administrative Distance **120** (> OSPF 110) keeps OSPF preferred; static installs only if OSPF goes away.

**Why likely:** Carolina discussed AD ordering and floating statics in review; your topology supports CORE↔RB/EDGE-B paths.  
**Refs:** Midterm Exam example.txt (style), 05-OSPF-Tuning.md
</details>

---

## Part B — NAT (14 pts)

### Q8. PAT on EDGE-A using outside interface (5 pts)
Configure dynamic **PAT** for **10.10.10.0/24** out EDGE-A’s outside interface. Include the ACL and **verification** commands.

<details><summary><strong>Answer</strong></summary>

```console
interface GigabitEthernet0/1   ! inside
 ip nat inside
interface GigabitEthernet0/0   ! outside (to REMOTE/ISP)
 ip nat outside
access-list 10 permit 10.10.10.0 0.0.0.255
ip nat inside source list 10 interface GigabitEthernet0/0 overload

! Verify:
show ip nat translations
show ip nat statistics
```
**Why likely:** PAT via outside interface with overload + verification are core tasks.  
**Refs:** 11-NAT.md, NAT_Dynamic_Example.md, NAT_Theory.md
</details>

---

### Q9. Static NAT / Port-forward for HTTP (4 pts)
Publish **192.0.2.80:80** (REMOTE-SERVER HTTP) via EDGE-B at outside global **203.0.113.3:80**. Show the single-line mapping and one verification.

<details><summary><strong>Answer</strong></summary>

```console
ip nat inside source static tcp 192.0.2.80 80 203.0.113.3 80
show ip nat translations | include 203.0.113.3:80
```
**Why likely:** Static one-to-one/one-to-port mapping is a frequent item.  
**Refs:** NAT_Static_Example.md, 11-NAT.md
</details>

---

### Q10. “It pings but TFTP fails” after PAT config (3 pts)
PAT seems configured, pings to **192.0.2.69** work, but **TFTP uploads fail**. Name **two** likely misconfigs you’d check first.

<details><summary><strong>Answer</strong></summary>

1) Missing `overload` (using dynamic pool, hitting pool exhaustion).  
2) ACL matches the wrong **source** (must match **inside locals** pre-NAT).

**Why likely:** These are the two most common PAT mistakes from the NAT labs.  
**Refs:** NAT_Tables_Examples.md, NAT_Dynamic_Example.md
</details>

---

### Q11. Order of operations with ACLs and NAT (2 pts)
For **inbound** traffic on an inside interface, what happens first—ACL or NAT—and why does it matter for matching?

<details><summary><strong>Answer</strong></summary>

Inbound **ACL** is evaluated **before** inside source NAT. Your ACL must match the **pre-NAT** (inside local) addresses.

**Why likely:** Matching order is essential to troubleshoot “ACL appears right but no hits.”  
**Refs:** 11-NAT.md (troubleshooting section)
</details>

---

## Part C — ACLs (8 pts)

### Q12. Permit web from Faculty only (5 pts)
Only **PC-VLAN10** hosts (10.10.10.0/24) may reach **HTTP/HTTPS** on **192.0.2.80**. Everyone else denied; **ICMP still allowed from any**. Write an **extended ACL** (name it `WEB-GATE`) and say **where** you’d apply it.

<details><summary><strong>Answer</strong></summary>

```console
ip access-list extended WEB-GATE
 permit icmp any any echo
 permit icmp any any echo-reply
 permit tcp 10.10.10.0 0.0.0.255 host 192.0.2.80 eq 80
 permit tcp 10.10.10.0 0.0.0.255 host 192.0.2.80 eq 443
 deny   ip any host 192.0.2.80
 permit ip any any
!
interface GigabitEthernet0/0   ! EDGE toward REMOTE
 ip access-group WEB-GATE out
```
**Placement:** Extended ACLs near **source** are best; edge-outbound is also acceptable for central enforcement.

**Why likely:** Tests order, specificity, and correct placement.  
**Refs:** 09-Standard-ACLs.md, 10-Extended-ACLs.md
</details>

---

### Q13. ISP-edge anti-RFC1918 filter (3 pts)
Write a single summarized statement set for numbered **ACL 101** to **block private sources** entering from the ISP side while permitting the rest.

<details><summary><strong>Answer</strong></summary>

```console
access-list 101 deny   ip 10.0.0.0 0.255.255.255 any
access-list 101 deny   ip 172.16.0.0 0.15.255.255 any
access-list 101 deny   ip 192.168.0.0 0.0.255.255 any
access-list 101 permit ip any any
interface GigabitEthernet0/0   ! outside
 ip access-group 101 in
```
**Why likely:** Classic ISP-edge hygiene (stop RFC1918 leakage); Carolina mentioned provider-side filtering.

**Refs:** 10-Extended-ACLs.md, networktopology.md
</details>

---

## Quick Command Checklist

**OSPF neighbors & interfaces**
```console
show ip ospf neighbor
show ip ospf interface brief
show ip ospf interface <IF>
show ip route ospf
```
**OSPF tuning**
```console
interface <IF>
 ip ospf priority <n>
 ip ospf hello-interval <n>
 ip ospf dead-interval <n>
router ospf 10
 auto-cost reference-bandwidth 100000
 ! maintenance: clear ip ospf process
```
**NAT build & verify**
```console
show run | section ip nat
show access-lists <NAT_ACL>
show ip nat statistics
show ip nat translations
```
**ACL build & verify**
```console
show access-lists
show run interface <IF> | include access-group
```

---

### License
This study resource is provided for personal educational use.
