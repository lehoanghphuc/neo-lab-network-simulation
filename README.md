# NEO Lab — Enterprise Network Infrastructure Simulation

> A Cisco Packet Tracer lab simulating a realistic 3-tier enterprise network: internal LAN switching and routing, an ISP-style WAN backbone, and internet-facing DMZ services — built to practice the kind of end-to-end design, configuration, and troubleshooting skills expected of a Network/Systems Engineer.

---

## 📌 Overview

This lab models three interconnected environments:

1. **LAN NEO NETWORK** — A redundant enterprise LAN with multi-VLAN switching, dual-area OSPF, and internet-facing security controls.
2. **WAN NEO ISP** — A 4-router EIGRP backbone simulating an ISP transit network, with engineered path preference and failover.
3. **LAN GOOGLE** — A DMZ environment hosting a public-facing web server reachable from the internet via NAT.

The goal was to go beyond a basic CCNA-style topology and build something closer to how these technologies are actually combined in production: multi-area routing, redistribution between routing domains, asymmetric NAT policies, and deliberate path engineering — then debug every issue that came up along the way using a structured, layer-by-layer methodology.

---

## 🗺️ Topology

**LAN NEO NETWORK** — Redundant Layer 2/3 design across OSPF Area 0 and Area 1.

![LAN NEO NETWORK topology](screenshots/web-server-verification.png)

**WAN NEO ISP** — 4-router EIGRP backbone connecting NEO NETWORK to the Google DMZ.

![WAN NEO ISP topology](PASTE_IMAGE_HERE_wan_neo_isp.png)

**LAN GOOGLE** — DMZ with a public web server behind static NAT.

![LAN GOOGLE topology](PASTE_IMAGE_HERE_lan_google.png)

---

## 🎯 Objectives

- Design a redundant, multi-VLAN enterprise LAN with deterministic Layer 2 and Layer 3 failover roles
- Implement multi-area OSPF with controlled DR/BDR election
- Build and tune an EIGRP-based WAN backbone, including manual path preference via metric manipulation
- Apply selective NAT/PAT policies and ACL-based access restrictions
- Expose an internal web server to the internet through static NAT (port forwarding)
- Practice systematic, layer-by-layer network troubleshooting

---

## ⚙️ Technologies Implemented

**Layer 2**
- 802.1Q VLAN trunking with synchronized VLANs across switches
- LACP EtherChannel (port-channel bundling between switch links)
- Rapid-PVST root bridge control (primary/secondary root per VLAN)
- PortFast on access ports connecting end devices
- Port Security (single static MAC per access port, violation mode: restrict/log)

**Layer 3**
- OSPF multi-area design (Area 0 / Area 1) with ABR redundancy (R1, R2)
- Controlled DR/BDR election via interface priority
- Inter-VLAN routing (router-on-a-stick) on dual routers
- EIGRP (AS 1000) for WAN backbone routing
- EIGRP path preference engineering via `delay` metric tuning, with verified feasible-successor failover

**IP Services**
- PAT (NAT overload) — restricted to specific internal subnets only
- Static NAT (port forwarding) — TCP/80 and TCP/443 to an internal DMZ server
- Extended ACLs — selective ICMP and Telnet restrictions by host/range

---

## 🏗️ Architecture Summary

```
LAN NEO NETWORK (OSPF Area 0 / Area 1)
 ├─ VLAN 10 / VLAN 20 — dual-homed via R1 & R2 (inter-VLAN routing + ACLs)
 ├─ R1 / R2 — ABRs, DR/BDR controlled, NAT-restricted PAT egress
 └─ R3 — Area 0 router, DR/BDR excluded, PAT (NAT overload) edge

        │  123.123.123.0/30
        ▼

WAN NEO ISP (EIGRP 1000)
 ├─ R4 — ISP edge toward NEO NETWORK, engineered path preference (delay)
 ├─ R6 — preferred transit path
 ├─ R5 — backup transit path (feasible successor)
 └─ R7 — ISP edge toward Google DMZ

        │  8.8.8.0/28
        ▼

LAN GOOGLE (DMZ)
 └─ R8 — Static NAT (8.8.8.8:80/443 → Web Server), PAT for outbound web server traffic
```

---

## 🔑 Key Configurations

- **OSPF DR/BDR control:** R1 = DR (`priority 255`), R2 = BDR (`priority 100`), R3 excluded (`priority 0`)
- **EIGRP path engineering:** `delay 5000` applied on R4's link toward R5, forcing EIGRP to select the R6 path as the sole successor while keeping the R5 path as a feasible successor for instant failover
- **Selective PAT:** Only VLAN 10 and VLAN 20 subnets (on both R1 and R2) are permitted in the NAT access-list — all other traffic is excluded from internet access by design
- **Host/range-based ACLs:** ICMP echo denied for a specific 11-host range in VLAN 10 toward the WAN edge subnet; Telnet denied for a single host in VLAN 20 toward the ISP router — both using precise wildcard masks rather than blanket subnet denies
- **Static NAT with PAT coexistence:** The DMZ router (R8) runs both static port-forwarding rules (inbound) and a PAT overload rule (outbound) simultaneously without conflict

---

## 🐛 Challenges & Troubleshooting

Real-world networking is rarely "configure once, works immediately." Below are three issues encountered during this lab and how they were diagnosed and resolved — using a systematic bottom-up approach (Layer 1 → Layer 3).

### Issue 1: EIGRP Adjacency Failed to Form Between WAN Routers

**Symptom:** After configuring EIGRP 1000 on two routers with matching network statements, no neighbor relationship appeared in `show ip eigrp neighbors`, while other links converged immediately.

**Diagnosis:**
```
R6# show ip int brief
GigabitEthernet0/0   176.16.67.1   YES manual up   up
```

**Root cause:** A typo — `176.16.67.1` instead of `172.16.67.1`. The interface was technically "up/up," masking the misconfiguration, since both ends still had link connectivity but were on completely different subnets, preventing EIGRP hello packets from forming an adjacency.

**Fix:** Corrected the IP address to `172.16.67.1`, matching the subnet on the neighboring router. Adjacency formed within seconds (`%DUAL-5-NBRCHANGE: ... is up: new adjacency`).

**Lesson:** Always verify `show ip int brief` before troubleshooting routing protocol behavior — an "up/up" interface does not guarantee correct addressing.

---

### Issue 2: End-to-End Connectivity Broken After Re-IPing a Point-to-Point Link

**Symptom:** After correcting an IP addressing mistake on a WAN edge link, ping and traceroute from internal VLANs to the internet stopped working entirely — traffic died exactly at the WAN edge.

**Diagnosis:** Compared interface IPs on both ends of the point-to-point link — addressing was valid (no conflict), but the **static default route on the internal edge router still pointed to the old next-hop**:
```
S* 0.0.0.0/0 [1/0] via 123.123.123.1
```

**Root cause:** After changing the router's own interface IP, the existing default route — written for the *previous* addressing scheme — now pointed the router to itself, creating a routing black hole.

**Fix:**
```
no ip route 0.0.0.0 0.0.0.0 123.123.123.1
ip route 0.0.0.0 0.0.0.0 123.123.123.2
```

**Lesson:** Changing one side of a point-to-point link has a ripple effect — any static routes, NAT statements, or ACLs referencing the old IP must be updated in the same change window. This is a common real-world cause of outages during IP renumbering.

---

### Issue 3: NAT Configured Correctly but ACL Logic Silently Failed

**Symptom:** A Telnet-restriction ACL (denying a specific host from reaching the ISP router) showed zero `deny` matches and all traffic was being permitted, despite the `deny` statement clearly being present in the ACL.

**Diagnosis:**
```
show access-lists BLOCK_TELNET_VLAN20
Extended IP access list BLOCK_TELNET_VLAN20
    permit ip any any (11 match(es))
    deny tcp host 10.2.20.100 host 123.123.123.2 eq telnet
```

**Root cause:** Editing a named ACL with `no <line>` followed by re-adding the line does not preserve original ordering — Cisco IOS appends the new entry with a sequence number *higher* than existing lines. The `deny` statement ended up positioned *after* `permit ip any any`, meaning it could never be evaluated (ACLs process top-down, first match wins).

**Fix:** Removed both entries and re-added them with explicit sequence numbers to enforce correct order:
```
ip access-list extended BLOCK_TELNET_VLAN20
 10 deny tcp host 10.2.20.100 host 123.123.123.2 eq 23
 20 permit ip any any
```

**Lesson:** Named ACLs in IOS auto-assign sequence numbers on insertion — always verify final order with `show access-lists` after any edit, and use explicit sequence numbers when precision matters.

---

### Key Takeaway

Each issue above was resolved using the same systematic method: **isolate the failure domain first** (Is it L1? L2? L3? NAT? ACL?) before changing configuration — rather than guessing. This bottom-up troubleshooting approach is the same methodology used in production network operations.

---

## ✅ Verification

End-to-end functionality was validated at each layer of the design — from internal routing convergence to full internet-edge connectivity through NAT and port forwarding.

### 1. OSPF Neighbor Adjacency & DR/BDR Election (Area 0)

Confirms R1 is elected DR, R2 is BDR, and R3 is excluded from the election (`priority 0`) as required by design:

```
R1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address      Interface
2.2.2.2         100   FULL/BDR        00:00:30    10.0.0.2     GigabitEthernet0/0
3.3.3.3         0     FULL/DROTHER    00:00:30    10.0.0.3     GigabitEthernet0/0
```

### 2. OSPF Multi-Area Route Propagation

Inter-area summary routes (Area 1 → Area 0) are correctly advertised by both ABRs:

```
R2# show ip ospf database
                Summary Net Link States (Area 0)
Link ID         ADV Router      Age    Seq#       Checksum
10.1.10.0       1.1.1.1         34     0x80000006 0x006ed1
10.1.20.0       1.1.1.1         24     0x80000007 0x00fd37
10.2.10.0       2.2.2.2         11     0x80000006 0x0044f6
10.2.20.0       2.2.2.2         1      0x80000007 0x00d35c
```

### 3. PAT — Internal VLANs Reaching the Internet

NAT overload translation confirmed for a host reaching the Google web server, with a full round-trip (inside local ↔ outside global match):

```
R3# show ip nat translations
Pro  Inside global        Inside local         Outside local   Outside global
icmp 123.123.123.2:8      10.1.20.100:8        8.8.8.1:8       8.8.8.1:8
icmp 123.123.123.2:9      10.1.20.100:9        8.8.8.1:9       8.8.8.1:9
...
```

### 4. EIGRP WAN Convergence & Path Engineering

EIGRP 1000 successfully formed adjacencies across all four ISP backbone routers, with the engineered path preference confirmed in the topology table — a single active successor via the preferred path, and a feasible successor on the backup path ready for instant failover:

```
R4# show ip eigrp topology
P 8.8.8.0/28, 1 successors, FD is 3328
  via 172.16.46.2 (3328/3072), GigabitEthernet0/2     ← active path
  via 172.16.45.2 (1283072/3072), GigabitEthernet0/0  ← backup (feasible successor)
```

### 5. ACL Enforcement — Access Restriction

Named ACLs confirmed enforcing both the ICMP range restriction and the Telnet restriction, with correct deny-before-permit ordering and live match counters:

```
R1# show access-lists
Extended IP access list BLOCK_PING_VLAN10
 10 deny icmp host 10.1.10.13 123.123.123.0 0.0.0.3 echo
 20 deny icmp 10.1.10.14 0.0.0.1 123.123.123.0 0.0.0.3 echo
 30 deny icmp 10.1.10.16 0.0.0.7 123.123.123.0 0.0.0.3 echo
 40 permit ip any any (67 match(es))
Extended IP access list BLOCK_TELNET_VLAN20
 10 deny tcp host 10.1.20.100 host 123.123.123.2 eq telnet
 20 permit ip any any (1 match(es))
```

### 6. Static NAT — Port Forwarding to DMZ Web Server

TCP port 80 successfully forwarded from the public-facing IP (`8.8.8.8`) to the internal web server, verified via direct TCP probe from the NEO Network edge router:

```
R3# telnet 8.8.8.8 80
Trying 8.8.8.8 ...Open
```

Confirmed end-to-end via browser from an internal client (Laptop0, VLAN 10):

> **URL:** `http://8.8.8.8` → Successfully renders the web page, confirming the static NAT + PAT + multi-hop WAN path (LAN NEO → R3 → EIGRP backbone → R7/R8 → Web Server) functions correctly end-to-end.

![Web browser successfully loading http://8.8.8.8 through static NAT port forwarding](PASTE_IMAGE_HERE_web_server_verification.png)

*Laptop0 (VLAN 10, internal NEO Network) successfully reaches the Google DMZ web server through the full path: LAN → OSPF → PAT → EIGRP WAN backbone → Static NAT → Web Server.*

---

### Summary

| Layer | Feature Tested | Result |
|---|---|---|
| L2 | VLAN trunking, LACP EtherChannel, Port Security | ✅ Pass |
| L3 (LAN) | OSPF multi-area, DR/BDR election control | ✅ Pass |
| L3 (WAN) | EIGRP 1000, engineered path preference & failover | ✅ Pass |
| IP Services | PAT, Static NAT (port forwarding), ACL restrictions | ✅ Pass |
| End-to-End | Internal client → Internet → DMZ web server | ✅ Pass |

---

## 📁 Repository Structure

```
neo-lab-network-simulation/
├── configs/
│   ├── R1.txt   — LAN NEO: ABR, DR (Area 0), inter-VLAN routing, ACLs
│   ├── R2.txt   — LAN NEO: ABR, BDR (Area 0), inter-VLAN routing, ACLs
│   ├── R3.txt   — LAN NEO: Area 0 edge, PAT, default route origination
│   ├── R4.txt   — WAN ISP: edge to NEO NETWORK, EIGRP path engineering
│   ├── R5.txt   — WAN ISP: backup transit path
│   ├── R6.txt   — WAN ISP: preferred transit path
│   ├── R7.txt   — WAN ISP: edge to Google DMZ
│   └── R8.txt   — LAN GOOGLE: Static NAT + PAT, DMZ edge
├── Hphvc_Prefinal.pkt
└── README.md
```

Full running-configs for every router are available in [`/configs`](./configs) for direct review without needing to open Cisco Packet Tracer.

---

## 💡 Skills Demonstrated

- Multi-area OSPF design with deterministic DR/BDR election control
- EIGRP path manipulation via delay-based metric tuning, with verified feasible-successor failover
- Selective NAT/PAT policy design (subnet-restricted internet access)
- Static NAT (port forwarding) for exposing internal services to the internet
- Precise ACL design using host and range-based wildcard masks
- Systematic, layer-by-layer network troubleshooting (L1 → L2 → L3 → NAT → ACL)
- End-to-end verification methodology across a multi-domain topology (LAN → WAN → DMZ)

---

## 🛠️ Tools Used

Cisco Packet Tracer (IOS 15.1)
