---
title: IPv6 Concepts for JNCIS-SP
date: 2026-04-15
tags:
  - juniper
  - ipv6
  - networking
  - commands_verified
---

## IPv6

IPv6 was designed to solve IPv4 address exhaustion while simplifying the protocol. The header is fixed-length and streamlined, broadcast is eliminated in favor of multicast, and address configuration can be fully automatic. For service providers, the most exam-relevant areas are address types, NDP, autoconfiguration, and how routing protocols (OSPF, IS-IS) extend to support IPv6.

---

### IPv4 vs IPv6 Key Differences

| Feature | IPv4 | IPv6 |
|---------|------|------|
| Address size | 32 bits | 128 bits |
| Header size | Variable (20–60 bytes) | Fixed 40 bytes |
| Header checksum | Yes | No (relies on L4) |
| Fragmentation | Routers and source | Source only |
| Broadcast | Yes | No — replaced by multicast |
| Address resolution | ARP | NDP (ICMPv6) |
| Autoconfiguration | DHCP only | SLAAC + DHCPv6 |
| IPsec | Optional | Built into extension header framework |

---

### IPv6 Header

The base IPv6 header is always exactly **40 bytes**. It is simpler than IPv4 — no checksum, no options field, and no fragmentation fields (those are handled by extension headers when needed).

| Field | Size | Description |
|-------|------|-------------|
| Version | 4 bits | Always `6` |
| Traffic Class | 8 bits | QoS / DSCP marking (equivalent to IPv4 ToS) |
| Flow Label | 20 bits | Identifies a specific flow for QoS treatment |
| Payload Length | 16 bits | Length of the payload (excluding the 40-byte header) |
| Next Header | 8 bits | Identifies what follows: an extension header or an upper-layer protocol (TCP=6, UDP=17, ICMPv6=58) |
| Hop Limit | 8 bits | Equivalent to IPv4 TTL — decremented at each hop |
| Source Address | 128 bits | |
| Destination Address | 128 bits | |

**Extension Headers** are chained after the base header using the Next Header field. Each extension header identifies the next via its own Next Header field. Common extension headers:

| Type | Next Header Value | Purpose |
|------|-------------------|---------|
| Hop-by-Hop Options | 0 | Must be processed by every router along the path |
| Routing Header | 43 | Source routing (used by SRv6) |
| Fragment Header | 44 | Fragmentation — used only by the source |
| ESP | 50 | IPsec encryption |
| AH | 51 | IPsec authentication |

> **No router fragmentation:** If an IPv6 packet is too large, the router drops it and sends an ICMPv6 "Packet Too Big" back to the source. The source is responsible for fragmenting. This is why Path MTU Discovery is critical in IPv6 networks.

---

### Address Format

- **128 bits** written as 8 groups of 4 hex digits (hextets) separated by colons
- Leading zeros within a hextet can be omitted: `0001` → `1`
- One contiguous run of all-zero hextets can be replaced with `::` — only once per address

```
Full:        2001:0db8:0000:0000:0000:0000:0000:0001
Compressed:  2001:db8::1
```

---

### Address Types and Ranges

| Prefix | Type | Notes |
|--------|------|-------|
| `2000::/3` | **Global Unicast** | Publicly routable. Equivalent to public IPv4. Current allocations are under `2000::/4` and `3000::/4`. |
| `FC00::/7` | **Unique Local (ULA)** | Private addresses. Range FCxx–FDxx. Equivalent to RFC 1918. Not routed on the public internet. |
| `FE80::/10` | **Link-Local** | Automatically assigned to every IPv6 interface. Never routed beyond the local link. Used by NDP and routing protocols. |
| `FF00::/8` | **Multicast** | Replaces broadcast. Scope is encoded in the address. |
| `::1/128` | **Loopback** | Equivalent to 127.0.0.1. |
| `2002::/16` | **6to4** | Transition mechanism embedding an IPv4 address in IPv6. |

> **`FEC0::/10` (site-local) is deprecated.** Do not use. Replaced by Unique Local (`FC00::/7`).

**Anycast:** An anycast address is assigned to multiple interfaces (typically on different routers). Traffic is routed to the topologically nearest one. Indistinguishable from a unicast address in format — the anycast behavior is configured, not derived from the prefix.

---

### Multicast Scopes and Key Addresses

IPv6 multicast addresses encode a scope in bits 12–15 of the address.

| Scope Value | Scope Name | Reach |
|-------------|-----------|-------|
| `1` | Node-local | Same device only |
| `2` | Link-local | Same link (`FF02::`) |
| `5` | Site-local | Within a site |
| `E` | Global | Internet-wide |

**Well-known link-local multicast addresses (`FF02::`):**

| Address | Group |
|---------|-------|
| `FF02::1` | All nodes |
| `FF02::2` | All routers |
| `FF02::5` | OSPFv3 all routers |
| `FF02::6` | OSPFv3 all DRs |
| `FF02::9` | RIPng routers |
| `FF02::1:2` | All DHCPv6 servers/relays |

**Solicited-Node Multicast:**
Every unicast/anycast address has a corresponding solicited-node multicast address used by NDP for neighbor resolution (replacing ARP). Format:

```
FF02::1:FF00:0/104 + last 24 bits of the interface address
```

Example: interface address `2001:db8::1` → solicited-node group `FF02::1:FF00:0001`

---

### Neighbor Discovery Protocol (NDP)

NDP replaces ARP and ICMP Router Discovery. All NDP messages are ICMPv6.

| Message | ICMPv6 Type | Purpose |
|---------|------------|---------|
| **Router Solicitation (RS)** | 133 | Host asks "is there a router on this link?" |
| **Router Advertisement (RA)** | 134 | Router announces its presence, prefix info, and flags for autoconfiguration |
| **Neighbor Solicitation (NS)** | 135 | Resolves a neighbor's MAC address (like ARP Request) or performs DAD |
| **Neighbor Advertisement (NA)** | 136 | Response to NS — provides MAC address (like ARP Reply) |
| **Redirect** | 137 | Router tells a host about a better first-hop for a destination |

**Boot sequence:**

1. Interface auto-assigns a link-local address: `FE80::/10` + EUI-64 derived from MAC
2. **DAD** (Duplicate Address Detection) — sends an NS for the tentative address to verify no one else is using it. If no NA is received, the address is confirmed unique.
3. Host sends **RS** to `FF02::2` (all routers)
4. Router responds with **RA** containing prefix, default gateway, and M/O flags
5. Host uses prefix to generate a global unicast address (SLAAC) and performs DAD again

**RA flags:**
- **M flag** (Managed) — tells hosts to use DHCPv6 for address assignment
- **O flag** (Other) — tells hosts to use DHCPv6 for other configuration (DNS) but not the address itself

---

### EUI-64 Interface ID Generation

EUI-64 derives a 64-bit interface ID from a 48-bit MAC address:

1. Split the MAC in half: `XX:XX:XX` | `YY:YY:YY`
2. Insert `FF:FE` in the middle: `XX:XX:XX:FF:FE:YY:YY:YY`
3. Flip the **7th bit** (U/L bit) of the first byte: if it was 0, set it to 1 (and vice versa)

**Example:**

```
MAC:    00:50:56:AB:CD:EF
Step 1: 00:50:56 | AB:CD:EF
Step 2: 00:50:56:FF:FE:AB:CD:EF
Step 3: Flip bit 7 of 0x00 → 0x02
EUI-64: 0250:56FF:FEAB:CDEF

Full link-local: FE80::250:56FF:FEAB:CDEF
```

---

### Address Assignment Methods

| Method | Address From | Options (DNS etc.) | Notes |
|--------|-------------|-------------------|-------|
| **SLAAC** | Self-generated (EUI-64 or random) | No | Stateless — no server needed. RA M=0, O=0. |
| **SLAAC + DHCPv6** (hybrid) | Self-generated | DHCPv6 | RA M=0, O=1. Address from SLAAC, DNS from DHCPv6. |
| **DHCPv6 stateful** | DHCPv6 server | DHCPv6 | RA M=1. Full address and options from server. |
| **Static** | Manual config | Manual | Used on infrastructure (routers, switches). |

---

### Junos IPv6 Configuration

**Static address:**

```
set interfaces ge-0/0/0 unit 0 family inet6 address 2001:db8:1::1/64
```

**EUI-64 (auto-generated host bits):**

```
set interfaces ge-0/0/0 unit 0 family inet6 address 2001:db8:1::/64 eui-64
```

**DHCPv6 client:**

```
set interfaces ge-0/0/0 unit 0 family inet6 dhcpv6-client client-identifier duid-type duid-ll
set interfaces ge-0/0/0 unit 0 family inet6 dhcpv6-client client-ia-type ia-na
set interfaces ge-0/0/0 unit 0 family inet6 dhcpv6-client req-option dns-server
```

- `ia-na` — Identity Association for Non-temporary Address (permanent address)
- `duid-ll` — use the interface's link-local address as the DHCP client identifier

**IPv6 static routes (go in `inet6.0`):**

```
set routing-options rib inet6.0 static route ::/0 next-hop 2001:db8::1
```

**Junos routing tables:**

| Table | Contents |
|-------|---------|
| `inet6.0` | IPv6 unicast routes |
| `inet6.2` | IPv6 multicast routes (used for RPF checks) |

---

### OSPFv3

OSPFv3 is a redesigned version of OSPF for IPv6. It is a **separate protocol instance** from OSPFv2 (`protocols ospf3` in Junos).

**Key differences from OSPFv2:**

| Feature | OSPFv2 | OSPFv3 |
|---------|--------|--------|
| Addressing info in LSAs | Yes (in Router and Network LSAs) | No — separated into Type 8/9 LSAs |
| Runs over | IPv4 | IPv6 link-local addresses |
| Router ID | 32-bit (can auto-select from IPv4) | 32-bit — **must be configured manually** if no IPv4 addresses exist |
| Authentication | In OSPF header | Uses IPv6 IPsec extension headers |
| Multiple instances per link | No | Yes (instance ID in header) |

**OSPFv3 LSA Types unique to IPv6:**

| Type | Name | Purpose |
|------|------|---------|
| **8** | Link LSA | Each router originates one per interface. Contains the link-local address and all IPv6 prefixes on that link. |
| **9** | Intra-Area Prefix LSA | Originated per router or per transit network. Carries IPv6 prefix reachability within the area. Replaces the addressing role of Type 1 and 2 LSAs. |

> OSPFv3 separates **topology** (who's connected to whom) from **addressing** (what prefixes exist). Types 1 and 2 LSAs still describe the topology, but they carry no IP addressing info — that's handled entirely by Types 8 and 9.

**Configuration:**

```
set protocols ospf3 area 0.0.0.0 interface ge-0/0/0.0
set protocols ospf3 area 0.0.0.0 interface lo0.0
set routing-options router-id 1.1.1.1       # required if no IPv4 addresses
```

```
show ospf3 neighbor
show ospf3 database
show ospf3 interface
show ospf3 route
```

---

### IS-IS with IPv6

IS-IS supports IPv6 through TLV extensions without requiring a separate protocol version. The same IS-IS process advertises both IPv4 and IPv6 prefixes simultaneously — a significant operational advantage over OSPFv2/v3.

**IPv6-specific TLVs:**

| TLV | Name | IPv4 Equivalent |
|-----|------|----------------|
| 232 | IPv6 Interface Address | TLV 132 (IPv4 Interface Address) |
| 236 | IPv6 Reachability | TLV 135 (Extended IP Reachability) |

**Multi-Topology IS-IS (MT-ISIS):**
By default IS-IS runs a single topology for both IPv4 and IPv6. If the IPv4 and IPv6 topologies differ (e.g., some links only run one protocol), multi-topology mode can be enabled to run separate SPF calculations.

```
set protocols isis interface ge-0/0/0.0
set protocols isis interface lo0.0
# TLV 236 is advertised automatically when inet6 is configured on interfaces
```

```
show isis adjacency
show isis database detail        # shows TLV 236 entries for IPv6 prefixes
show route table inet6.0 protocol isis
```

---

### Monitoring and Troubleshooting

```
show interfaces ge-0/0/0 detail          # IPv6 addresses and NDP state
show ipv6 neighbors                       # NDP neighbor cache (like ARP table)
show route table inet6.0                  # IPv6 routing table
show route table inet6.0 protocol static
show ospf3 neighbor
show ospf3 database
show ospf3 interface detail
show isis database detail                 # look for TLV 236 entries
```

---

## Quick Reference

### Address Types

| Prefix | Type | Routable |
|--------|------|---------|
| `2000::/3` | Global Unicast | Yes |
| `FC00::/7` | Unique Local | No (private) |
| `FE80::/10` | Link-Local | No (link only) |
| `FF00::/8` | Multicast | Scope-dependent |
| `::1/128` | Loopback | No |

### NDP Message Types

| Message | ICMPv6 | Purpose |
|---------|--------|---------|
| RS | 133 | Host solicits router |
| RA | 134 | Router announces prefix/config |
| NS | 135 | MAC resolution / DAD |
| NA | 136 | Response to NS |
| Redirect | 137 | Better next-hop notification |

### Address Assignment Methods

| Method | RA M flag | RA O flag | Address Source |
|--------|-----------|-----------|---------------|
| SLAAC | 0 | 0 | Self-generated |
| SLAAC + DHCPv6 | 0 | 1 | Self-generated + DHCPv6 options |
| DHCPv6 stateful | 1 | — | DHCPv6 server |

### OSPFv2 vs OSPFv3

| Feature | OSPFv2 | OSPFv3 |
|---------|--------|--------|
| Junos hierarchy | `protocols ospf` | `protocols ospf3` |
| Addressing in Router LSA | Yes | No (Type 8/9) |
| Router ID required | Auto or manual | Manual if no IPv4 |
| Runs over | IPv4 | IPv6 link-local |

### OSPFv3 LSA Types for IPv6

| Type | Name | Purpose |
|------|------|---------|
| 8 | Link LSA | Link-local address + prefixes on the link |
| 9 | Intra-Area Prefix LSA | IPv6 prefix reachability within an area |
