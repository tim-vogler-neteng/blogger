---
title: IP Tunnels for JNCIS-SP
date: 2026-04-15
tags: [juniper, tunnels, gre, networking]
---

## IP Tunnels

Tunnels encapsulate one protocol inside another, creating a virtual point-to-point link across a network that wouldn't otherwise carry that traffic. The encapsulating network is called the **underlay**; the encapsulated traffic and the logical topology it creates is called the **overlay**. Both GRE and IP-IP are **stateless** — they hold no session state and provide no encryption or reliability guarantees.

**Common use cases:**
- Carry IPv6 traffic across an IPv4-only core (6in4)
- Carry IPv4 traffic across an IPv6-only core (4in6)
- Extend IGP adjacencies across a WAN that doesn't support multicast
- Tunnel MPLS across a non-MPLS network
- Bridge Layer 2 domains across a routed network

---

### Tunnel Concepts

**Underlay vs Overlay:**
- The **underlay** is the physical network that actually carries the encapsulated packets. The tunnel endpoints must be reachable via the underlay.
- The **overlay** is the logical network seen by the encapsulated traffic — it has no visibility into the underlay topology.

**Tunnel endpoints:**
- Every tunnel has a **source** IP and a **destination** IP that identify the two endpoints in the outer header
- The source IP should be reachable from the remote end — using loopback addresses is best practice (stable, unaffected by individual link failures)
- The underlay route to the tunnel destination must exist before the tunnel can come up

**Recursive routing (important gotcha):**
If the tunnel itself is used to carry the route to the tunnel endpoint, the router enters an infinite lookup loop and the tunnel never comes up. The route to the tunnel destination must be resolved via a path that does **not** go through the tunnel — typically a directly connected route, static route, or a different routing table.

---

### IP-IP Tunnels

IP-IP encapsulation prepends a new 20-byte IP header to the original packet. It is the simplest and lowest-overhead tunnel type.

**Packet structure:**

```
[ Outer IP Header (20B) | Original IP Header | Payload ]
```

| Field | Value |
|-------|-------|
| Overhead | **20 bytes** |
| IP Protocol number | **4** (IP-in-IP) |
| Can encapsulate | IP unicast only |
| Multicast | No |
| L2 frames | No |
| Junos interface | `ip-0/0/0` |

**TTL behavior:** The outer IP header carries its own TTL (default 64). The inner IP header TTL is decremented normally at each logical hop — from the tunnel source's perspective, the tunnel is one hop.

---

### GRE Tunnels

GRE (Generic Routing Encapsulation) adds a 4-byte shim header between the outer IP header and the encapsulated payload. The shim's **Protocol Type** field (2 bytes) identifies what is inside, allowing GRE to carry almost any network-layer protocol.

**Packet structure:**

```
[ Outer IP Header (20B) | GRE Header (4B+) | Inner Packet ]
```

**GRE header fields:**

| Field | Bits | Description |
|-------|------|-------------|
| C (Checksum present) | 1 | If set, a 2-byte checksum and 2-byte reserved field follow |
| Reserved | 12 | Must be zero |
| Version | 3 | Always 0 for standard GRE |
| Protocol Type | 16 | Identifies the encapsulated protocol (e.g., `0x0800` = IPv4, `0x86DD` = IPv6, `0x8847` = MPLS) |

Minimum GRE overhead = **24 bytes** (20B outer IP + 4B GRE header). Optional fields (checksum, key, sequence number) can add more.

| Field | Value |
|-------|-------|
| Minimum overhead | **24 bytes** |
| IP Protocol number | **47** |
| Can encapsulate | IPv4, IPv6, MPLS, L2 frames, multicast |
| Multicast | Yes |
| L2 frames | Yes (with virtual-switch instance) |
| Junos interface | `gr-0/0/0` |

**GRE advantages over IP-IP:**
- Carries multicast → enables IGP neighbor discovery across WAN links that don't support multicast
- Carries MPLS → extends MPLS networks across non-MPLS infrastructure
- Carries L2 frames → bridges VLANs across routed networks
- Carries IPv6 over IPv4 and IPv4 over IPv6

---

### 6in4 — IPv6 over IPv4

6in4 tunnels carry IPv6 packets inside IPv4. The outer header uses **IP protocol 41**. This is used to connect IPv6 islands across an IPv4-only underlay.

- Configured identically to IP-IP but with `family inet6` on the tunnel interface unit
- The tunnel source and destination are IPv4 addresses
- Junos uses the `ip-0/0/0` interface for 6in4 as well

```
# VMX 1
set interfaces ip-0/0/0 unit 0 tunnel source 100.1.1.1
set interfaces ip-0/0/0 unit 0 tunnel destination 200.2.2.2
set interfaces ip-0/0/0 unit 0 family inet6 address 2001:db8:1::1/64

# IPv6 route pointing into the tunnel
set routing-options rib inet6.0 static route 2001:db8::/32 next-hop ip-0/0/0.0
```

---

### MTU Considerations

Tunnel headers add overhead, which reduces the effective MTU available to the inner payload. This is one of the most operationally important aspects of tunnel deployment.

| Tunnel Type | Header Overhead | Effective MTU (on 1500B link) |
|-------------|----------------|-------------------------------|
| IP-IP | 20 bytes | **1480 bytes** |
| GRE (no options) | 24 bytes | **1476 bytes** |
| GRE + MPLS label | 28 bytes | **1472 bytes** |

**Problems caused by MTU mismatch:**
- Large packets are dropped silently if they exceed the tunnel MTU
- ICMP "Fragmentation Needed" / "Packet Too Big" messages may be filtered by firewalls, preventing Path MTU Discovery from working
- TCP sessions may hang while UDP and ICMP work fine (since TCP uses MSS negotiation)

**Solutions:**

1. **Set tunnel interface MTU explicitly** to prevent oversized packets from entering the tunnel:

```
set interfaces gr-0/0/0 unit 0 family inet mtu 1476
```

2. **TCP MSS clamping** — rewrite the TCP MSS option in SYN packets to a safe value, so TCP sessions never try to send packets larger than the tunnel can handle:

```
set interfaces gr-0/0/0 unit 0 family inet tcp-mss 1436
```

3. **Increase physical MTU** (jumbo frames) on the underlay so the tunnel overhead fits within the physical MTU without reducing the effective payload size.

> **Exam tip:** If a tunnel is up but large transfers fail while pings work, MTU is almost always the cause. Pings use small packets by default; TCP bulk transfers hit the MTU ceiling.

---

### GRE Keepalives

GRE tunnels are stateless by nature — neither endpoint knows if the other end is still alive. Keepalives add a lightweight liveness check.

- Junos sends GRE keepalive packets through the tunnel; the remote endpoint loops them back
- If the configured number of keepalives go unacknowledged, the tunnel interface goes down and the routing protocol removes the route

```
set protocols oam gre-tunnel interface gr-0/0/0.0 keepalive-time 10 set protocols oam gre-tunnel interface gr-0/0/0.0 hold-time 30
```

- `interval` — how often keepalives are sent (seconds)
- `holdtime` — how long to wait before declaring the tunnel down (seconds)

---

### Configuration

**Enable tunnel services on the PIC** (required on some platforms/line cards):

```
set chassis fpc 0 pic 0 tunnel-services bandwidth 1g
```

**Complete GRE tunnel (IPv4 over IPv4):**

```
# Router A
set interfaces gr-0/0/0 unit 0 tunnel source 10.0.0.1          # use loopback
set interfaces gr-0/0/0 unit 0 tunnel destination 10.0.0.2
set interfaces gr-0/0/0 unit 0 family inet address 172.16.0.1/30
set interfaces gr-0/0/0 unit 0 family inet mtu 1476

# Router B
set interfaces gr-0/0/0 unit 0 tunnel source 10.0.0.2
set interfaces gr-0/0/0 unit 0 tunnel destination 10.0.0.1
set interfaces gr-0/0/0 unit 0 family inet address 172.16.0.2/30
set interfaces gr-0/0/0 unit 0 family inet mtu 1476
```

**Complete IP-IP tunnel (IPv4 over IPv4):**

```
# Router A
set interfaces ip-0/0/0 unit 0 tunnel source 10.0.0.1
set interfaces ip-0/0/0 unit 0 tunnel destination 10.0.0.2
set interfaces ip-0/0/0 unit 0 family inet address 172.16.1.1/30
set interfaces ip-0/0/0 unit 0 family inet mtu 1480

# Router B
set interfaces ip-0/0/0 unit 0 tunnel source 10.0.0.2
set interfaces ip-0/0/0 unit 0 tunnel destination 10.0.0.1
set interfaces ip-0/0/0 unit 0 family inet address 172.16.1.2/30
set interfaces ip-0/0/0 unit 0 family inet mtu 1480
```

**GRE tunnel bridging L2 over L3** (requires virtual-switch routing instance):

```
set interfaces gr-0/0/0 unit 0 tunnel source 10.0.0.1
set interfaces gr-0/0/0 unit 0 tunnel destination 10.0.0.2
set interfaces gr-0/0/0 unit 0 family bridge interface-mode trunk
set interfaces gr-0/0/0 unit 0 family bridge vlan-id-list 100
set interfaces gr-0/0/0 unit 0 family bridge core-facing
```

---

### Monitoring and Troubleshooting

```
show interfaces gr-0/0/0 detail
show interfaces ip-0/0/0 detail
show interfaces gr-0/0/0 extensive      # includes keepalive stats
show route table inet.0
ping 172.16.0.2 source 172.16.0.1       # ping across the tunnel
ping 172.16.0.2 source 172.16.0.1 size 1472 do-not-fragment   # MTU test
traceroute 172.16.0.2 source 172.16.0.1
```

**Troubleshooting checklist:**

| Symptom | Likely Cause |
|---------|-------------|
| Tunnel interface stays Down | Underlay route to tunnel destination missing; recursive routing loop; tunnel services PIC not configured |
| Tunnel up but no traffic flows | MTU mismatch; firewall filter blocking encapsulated traffic; routing issue in overlay |
| Pings work, large transfers fail | MTU — set tunnel MTU or enable TCP MSS clamping |
| Tunnel flaps | Keepalive failures; underlay instability |

---

## Quick Reference

### IP-IP vs GRE

| Feature | IP-IP | GRE |
|---------|-------|-----|
| Overhead | 20 bytes | 24 bytes (minimum) |
| IP protocol number | 4 | 47 |
| IPv4 unicast | Yes | Yes |
| IPv6 | No (use 6in4 `proto 41`) | Yes |
| Multicast | No | Yes |
| MPLS | No | Yes |
| L2 frames | No | Yes |
| Keepalives | No | Yes |
| Junos interface | `ip-0/0/0` | `gr-0/0/0` |

### MTU Impact

| Tunnel | Overhead | Effective MTU (1500B link) |
|--------|----------|--------------------------|
| IP-IP | 20B | 1480 |
| GRE | 24B | 1476 |
| GRE + 1 MPLS label | 28B | 1472 |

### Key Commands

| Command | Purpose |
|---------|---------|
| `show interfaces gr-0/0/0 detail` | Tunnel state, counters |
| `show interfaces gr-0/0/0 extensive` | Includes keepalive stats |
| `ping <dest> size 1472 do-not-fragment` | MTU path test |
