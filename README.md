## 🐛 Challenges & Troubleshooting

Real-world networking is rarely "configure once, works immediately." Below are three issues encountered during this lab and how they were diagnosed and resolved — using a systematic bottom-up approach (Layer 1 → Layer 3).

### Issue 1: EIGRP Adjacency Failed to Form Between WAN Routers

**Symptom:** After configuring EIGRP 1000 on R6 and R7 with matching network statements, no neighbor relationship appeared in `show ip eigrp neighbors`, while other links converged immediately.

**Diagnosis:**
```
R6# show ip int brief
GigabitEthernet0/0   176.16.67.1   YES manual up   up
```

**Root cause:** A typo — `176.16.67.1` instead of `172.16.67.1`. The interface was technically "up/up," masking the misconfiguration, since both ends still had link connectivity but were on completely different subnets, preventing EIGRP hello packets from forming an adjacency.

**Fix:** Corrected the IP address to `172.16.67.1`, matching the subnet on R7. Adjacency formed within seconds (`%DUAL-5-NBRCHANGE: ... is up: new adjacency`).

**Lesson:** Always verify `show ip int brief` before troubleshooting routing protocol behavior — an "up/up" interface does not guarantee correct addressing.

---

### Issue 2: End-to-End Connectivity Broken After Re-IPing a Point-to-Point Link

**Symptom:** After correcting an IP addressing mistake on the R3–R4 link, ping and traceroute from internal VLANs to the internet (`8.8.8.1`) stopped working entirely — traffic died exactly at the WAN edge.

**Diagnosis:** Compared interface IPs on both ends of the point-to-point link:
```
R3# show ip int brief → GigabitEthernet0/1   123.123.123.1
R4# show ip int brief → GigabitEthernet0/0   123.123.123.2
```

The addressing itself was valid (no IP conflict), but the **static default route on R3 still pointed to the old next-hop**:
```
S* 0.0.0.0/0 [1/0] via 123.123.123.1
```

**Root cause:** After changing R3's own interface IP to `.1`, the existing default route — which had been written for the *old* topology — was now pointing R3 to itself, creating a routing black hole.

**Fix:**
```
R3(config)# no ip route 0.0.0.0 0.0.0.0 123.123.123.1
R3(config)# ip route 0.0.0.0 0.0.0.0 123.123.123.2
```

**Lesson:** Changing one side of a point-to-point link has a ripple effect — any static routes, NAT statements, or ACLs referencing the old IP must be updated in the same change window. This is a common real-world cause of outages during IP renumbering.

---

### Issue 3: NAT Configured Correctly but ACL Logic Silently Failed

**Symptom:** A Telnet-restriction ACL (denying a specific host from reaching the ISP router) showed zero `deny` matches and all traffic was being permitted, despite the `deny` statement clearly being present in the ACL.

**Diagnosis:**
```
R2# show access-lists BLOCK_TELNET_VLAN20
Extended IP access list BLOCK_TELNET_VLAN20
    permit ip any any (11 match(es))
    deny tcp host 10.2.20.100 host 123.123.123.2 eq telnet
```

**Root cause:** Editing a named ACL with `no <line>` followed by re-adding a line does not preserve original ordering — Cisco IOS appends the new entry with a sequence number *higher* than existing lines. The `deny` statement ended up positioned *after* `permit ip any any`, meaning it could never be evaluated (ACLs process top-down, first match wins).

**Fix:** Removed both entries and re-added them with explicit sequence numbers to enforce correct order:
```
R2(config)# ip access-list extended BLOCK_TELNET_VLAN20
R2(config-ext-nacl)# 10 deny tcp host 10.2.20.100 host 123.123.123.2 eq 23
R2(config-ext-nacl)# 20 permit ip any any
```

**Lesson:** Named ACLs in IOS auto-assign sequence numbers on insertion — always verify final order with `show access-lists` after any edit, and use explicit sequence numbers when precision matters.

---

### Key Takeaway

Each issue above was resolved using the same systematic method: **isolate the failure domain first** (Is it L1? L2? L3? NAT? ACL?) before changing configuration — rather than guessing. This bottom-up troubleshooting approach (verify physical/IP → verify routing → verify NAT → verify ACL) is the same methodology used in production network operations.
