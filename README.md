```mermaid
flowchart LR
    %% =========================
    %% CORE / OSPF AREA (PID 10)
    %% IDs use loopbacks: 10.255.0.ID/32
    %% =========================

    subgraph LEFT_LANS[Left LANs]
      direction TB
      PCA[PC-A\n(PC-VLAN10)\n10.10.2.0/24]
      V10[Faculty VLAN 10\n10.10.10.0/24]
      V20[Student VLAN 20\n10.10.20.0/24]
      V30[Guest VLAN 30\n10.10.30.0/24]
      V00[10.10.0.0/24]
    end

    subgraph CORE[Core / Distribution]
      direction TB
      R2[R2 (ID:2)]
      R5[R5 (ID:5)]
      SW[L3-SWITCH]
      R6[R6 (ID:6)]
      R4[R4 (ID:4)]
      R1[R1 (EDGE-A • ID:1)]
      R3[R3 (EDGE-B • ID:3)]
    end

    subgraph RIGHT_EDGE[Right Edge & Remote]
      direction TB
      NETA[203.0.113.0/24]
      NETB[205.0.113.0/24]
      REMRTR[REMOTE\n192.0.2.2/24]
      REMSRV[REMOTE-SERVER\n192.0.2.0/24]
    end

    %% =========================
    %% L3 SWITCH SVI to LANs
    %% =========================
    V10 ---|SVI Vlan10| SW
    V20 ---|SVI Vlan20| SW
    V30 ---|SVI Vlan30| SW
    V00 ---|SVI VlanXX| SW
    PCA ---|Access port in VLAN10| SW

    %% =========================
    %% CORE LINKS (WITH SUBNETS & IF NAMES)
    %% =========================
    R2 ---|G0/0 ↔ G0/0 • 10.10.12.0/29| R1
    R2 ---|G0/2 ↔ G0/1 • 10.10.25.0/28| R5
    R5 ---|G0/1 ↔ G1/0 • 10.10.245.0/28| SW
    SW ---|G0/1 ↔ G0/1 • 10.10.56.0/28| R6
    R6 ---|G0/2 ↔ G0/1 • 10.10.46.0/27| R4
    R4 ---|G0/2 ↔ G0/2 • 10.10.34.0/29| R3
    R3 ---|G0/0 ↔ G0/2 • 10.10.13.0/29| R1

    %% =========================
    %% RIGHT-SIDE / EDGE NETWORKS
    %% =========================
    R1 ---|G0/1 • 203.0.113.0/24| NETA
    R3 ---|G0/1 • 205.0.113.0/24| NETB

    %% Remote router & server off EDGE-A side (per diagram)
    R1 ---|G0/3 • 192.0.2.0/24| REMRTR
    REMRTR ---|LAN 192.0.2.0/24| REMSRV
```
