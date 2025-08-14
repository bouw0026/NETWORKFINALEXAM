# Final Exam Prep Playbook — OSPF, NAT, ACLs (GitHub Edition)

> Focused on practical skills, pattern recognition, and the fastest troubleshooting paths for your final topology. IPv6 removed; emphasis on OSPF & NAT.

---

## Table of Contents
- [What You Should Know Cold](#what-you-should-know-cold-practical-not-theory)
- [Pattern Recognition (“Cheat Codes”)](#pattern-recognition-cheat-codes)
- [Command Playbooks (60-second triage)](#command-playbooks-60-second-triage)
  - [A) OSPF Triage](#a-ospf-triage-any-router)
  - [B) Path Sanity (Per-hop Thinking)](#b-path-sanity-per-hop-thinking)
  - [C) NAT Triage (Edge)](#c-nat-triage-edge)
  - [D) ACL Triage](#d-acl-triage)
- [“If X then Y” Troubleshooting Trees](#if-x-then-y-troubleshooting-trees)
- [Tiny Mental Cheat Cards](#tiny-mental-cheat-cards)
- [RC-Specific Tells](#rc-specific-tells)
- [30-Second Pre-Exam Drill](#30-second-pre-exam-drill)

---

## What You Should Know Cold (practical, not theory)

### OSPF (Area 0 only)
- **Router-ID**: Set via loopback for stability (e.g., `10.255.0.<ID>/32`).
- **Cost math**: `cost = reference_bw / interface_bw` (Mb). With `ref=10000` (10 Gb):  
  - 10 Gb → **1**, 1 Gb → **10**, 100 Mb → **100**.  
  - Align the **reference on all routers** or paths diverge.
- **DR/BDR on broadcast**: **Priority wins**, then **RID**; **non-preemptive** once elected.
- **Passive-interface**: Still **advertises** network but sends no Hellos. Use it on user LANs/SVIs.
- **Traceroute ≠ neighbors**: Traceroute shows **forwarding**, not adjacencies.

### NAT (edges)
- **Inside/Outside roles** must be correct (top cause of “NAT not working”).  
- **PAT (overload)**: Many-to-one using the **outside interface IP**; widen source ACL to match **all inside** (e.g., `10.10.0.0/16`).  
- **Asymmetry breaks NAT**: Same edge must see **outbound and return**. One default + NAT is the safe baseline.

### ACLs
- **Standard** = source-only → place **near destination**.
- **Extended** = src/dst/port → place **near source**.
- There is always an **implicit `deny any`** at the end; add explicit permits.

### Topology “tells”
- Link subnets encode router IDs (e.g., `10.10.13.0/29` is IDs 1↔3).
- **RC uses Ethernet (100 Mb)** on `e0/x`. If reference-bw is default (100), RC may appear “equal” to 1 Gb links — misleading. Fix with `auto-cost reference-bandwidth` or `ip ospf cost`.

---

## Pattern Recognition (“Cheat Codes”)

**OSPF path oddities**
- “Shorter-looking” path not chosen → suspect **ref-bw mismatch**, **RC’s 100 Mb costs**, or **ECMP** hidden by a non-equal cost.

**Neighbor up but traffic fails**
- If adjacencies are **FULL** but pings fail → usually **ACL** or **NAT**, not OSPF.

**Default route logic**
- `O*E2 0.0.0.0/0` everywhere → a single edge is ASBR (typical).  
- Both edges inject defaults + PAT → intermittent timeouts = **asymmetric NAT**.

**NAT telltales**
- Some subnets browse, others don’t → **NAT ACL** too narrow.  
- No translations → wrong inside/outside or ACL mismatch.  
- Outside sees RFC1918 → NAT not hit or bypassed.

**ACL gotchas**
- Only a **deny** line = you blocked **everything** (implicit deny).  
- Standard ACL near source = **collateral damage** → use extended ACL near source for surgical control.

---

## Command Playbooks (60-second triage)

### A) OSPF Triage (any router)
```console
show ip ospf | include Router ID|Reference
show ip ospf neighbor
show ip ospf interface brief
show ip ospf interface <IF>     ! check Hello/Dead, passive, cost
show ip route ospf
```
**Look for**
- Correct **RID** (loopback value).  
- Same **Reference bandwidth** everywhere.  
- Expected **FULL** neighbors; DR/BDR on multi-access.  
- **Passive** on user SVIs (Hello 0).  
- Reasonable `O` routes (and defaults if used).

### B) Path Sanity (Per-hop Thinking)
```console
show ip route <DEST>
traceroute <DEST>
show ip cef <DEST> detail       ! if available
```
- Each hop makes its **own** decision. If forward/back paths differ, check **ECMP**, **ref-bw mismatch**, or **default source**.

### C) NAT Triage (Edge)
```console
show run | section ip nat
show access-lists <NAT_ACL>
show ip nat statistics
show ip nat translations
```
- **Roles**: `ip nat inside` on internal links; `ip nat outside` on edge uplink.  
- **ACL hits** increment during tests; translations appear with **outside IP:port** for PAT.

### D) ACL Triage
```console
show access-lists <N>
show run interface <IF> | include access-group
```
- Check **rule order** (specific denies before broad permits).  
- **Placement**: extended near source; standard near destination.  
- During testing, consider a temporary `deny ip any any log` at the end (lab only).

---

## “If X then Y” Troubleshooting Trees

**OSPF path not what you expected**
1. `show ip ospf | include Reference` → mismatch? **Align ref-bw**.
2. `show ip ospf interface <IF>` → unexpected **cost**? Set `ip ospf cost N`.
3. `show ip route <DEST>` at **every hop** → find where next-hop choice diverges.

**Neighbors won’t go FULL on Ethernet**
- Verify **network type** (broadcast vs p2p) and **mask** match.  
- For a 2-node Ethernet link, consider `ip ospf network point-to-point` on both ends.

**Inside hosts can’t reach 192.0.2.x**
1. On edge: `show ip nat translations` → empty?  
   - Fix **inside/outside roles**.  
   - Fix **NAT ACL** scope (include that subnet).  
2. If translations exist but no reply → check for **asymmetric NAT**; enforce a single exit or use FHRP/PBR.

**Only Guest (VLAN30) is broken**
- `show access-lists` → lone **deny** without trailing permit?  
- Ensure **extended ACL** near source when being surgical.

---

## Tiny Mental Cheat Cards

**OSPF cost (ref=10000 Mb)**  
- 10 Gb = **1** · 1 Gb = **10** · 100 Mb = **100**  
*(If ref=100, 1 Gb rounds ≈ 1 — misleading!)*

**DR/BDR tiebreak**  
- **Priority** (0 = never DR) → **RID** → **Non-preemptive**

**NAT sanity**  
- Roles correct? **Inside/Outside**  
- Is source matched? **ACL hits**  
- Is return **symmetric**? (One default + PAT is simplest.)

**ACL placement**  
- **Standard → destination**  
- **Extended → source**  
- Plan your **permits**; implicit deny lurks at end

**Traceroute vs OSPF**  
- Traceroute = **forwarding path**  
- Neighbors/LSDB = **control plane**  
- Use both together

---

## RC-Specific Tells

- With default ref-bw (100), **RC’s 100 Mb** looks equal to 1 Gb links → wrong paths.  
- With aligned ref-bw (10000), **RC becomes costly transit** (≈100) and is used mainly to reach **RC’s own LAN**.  
- If you must keep RC in-path, you can lower cost via **`ip ospf cost`** (accept the throughput trade).

---

## 30-Second Pre-Exam Drill

1. On every router:  
   ```console
   show ip ospf | include Router ID|Reference
   ```
2. On CORE:  
   ```console
   show ip ospf neighbor         # confirm DR/BDR on shared L2 segment
   ```
3. On EDGE-A:  
   ```console
   show run | section ip nat
   show ip nat statistics
   ```
4. On CORE/RA/RB:  
   ```console
   show access-lists
   ```
5. End-to-end quick test:  
   ```console
   traceroute 192.0.2.80
   ```
   Expect path toward **EDGE-A** (or your chosen single egress). If not, investigate **costs, defaults, or asymmetry**.

---

### License
This study guide is provided for personal educational use.
