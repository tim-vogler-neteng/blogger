---
title: MPLS Concepts for JNCIS-SP
date: 2026-04-15
tags:
  - juniper
  - mpls
  - networking
  - commands_verified
---

## MPLS (Multiprotocol Label Switching)

MPLS is a forwarding mechanism that uses short, fixed-length labels to make packet-forwarding decisions instead of performing a full IP lookup at every hop. Labels are applied at the ingress of an MPLS domain and stripped at the egress, with each transit router performing only a label swap — making forwarding fast and enabling traffic engineering, VPNs, and QoS capabilities.

---

### Terms

- **LSR** (Label Switching Router) - Any router participating in MPLS forwarding. Performs label actions push, swap, or pop.
- **LSP** (Label Switched Path) - The unidirectional path a labeled packet takes from ingress to egress LSR.
- **FEC** (Forwarding Equivalence Class) - A group of packets that receive identical forwarding treatment and are assigned the same label at ingress. The ingress router decides the FEC assignment; downstream routers just label-switch.
- **Ingress LSR** - The first router in an LSP. Classifies traffic into FECs and pushes labels.
- **Egress LSR** - The last router in an LSP. Removes the label and forwards the original packet.
- **Transit LSR** (P router) - An interior provider router. Swaps labels and forwards without examining the inner IP header.
- **PE** (Provider Edge) - ISP router at the edge of the MPLS domain that interfaces with customer equipment. Performs label push/pop for customer traffic.
- **CE** (Customer Edge) - Customer device that connects to the PE. Not aware of MPLS.
- **LIB** (Label Information Base) - The full table of all label bindings a router has received. Not all entries are actively used for forwarding.
- **LFIB** (Label Forwarding Information Base) - The active subset of the LIB used for actual forwarding decisions. This is what the data plane uses.
- **TED** (Traffic Engineering Database) - Populated by IGP TE extensions; stores link-state info (bandwidth, admin groups) used by CSPF to calculate constrained paths.
- **MBB** (Make-before-Break) - Default Junos behavior where the new LSP is fully signaled and verified before traffic is switched over from the old path.

---

### Label Operations

| Operation | Description |
|-----------|-------------|
| **Push**  | Add a new label to the top of the label stack. Done by the ingress LSR. |
| **Swap**  | Replace the top label with a new one. Done by transit LSRs. |
| **Pop**   | Remove the top label from the stack. Done by the egress LSR or the penultimate hop. |

---

### MPLS Label Structure

Each MPLS label is a 32-bit field inserted between the Layer 2 and Layer 3 headers (sometimes called a "shim header"). Multiple labels can be stacked.

| Field | Bits | Description |
|-------|------|-------------|
| **Label** | 20 | The forwarding label value (0–1,048,575). |
| **TC** (Traffic Class) | 3 | Formerly called EXP. Used for QoS/DiffServ marking. |
| **S** (Bottom of Stack) | 1 | `1` = this is the last label in the stack; `0` = more labels below. |
| **TTL** | 8 | Time to Live. Decremented at each hop, similar to IP TTL. |

---

### Reserved Labels

Labels 0–15 are reserved and have fixed meanings:

| Label | Name | Purpose |
|-------|------|---------|
| 0 | IPv4 Explicit Null | Signals the egress to apply IPv4 QoS processing before popping. |
| 1 | Router Alert | Causes the receiving router to process the packet locally. |
| 2 | IPv6 Explicit Null | Same as label 0 but for IPv6. |
| 3 | Implicit Null | Used by the egress to signal PHP to the penultimate hop. Never actually appears in a packet. |

---

### Penultimate Hop Popping (PHP)

PHP is the default behavior in MPLS. The egress LSR advertises label **3** (Implicit Null) to its upstream neighbor, signaling that neighbor (the penultimate hop router) to pop the label before forwarding. This way the egress LSR receives a plain IP packet and performs only one lookup (IP) instead of two (label + IP).

- **Default behavior:** PHP is enabled by default in Junos.
- **Ultimate Hop Popping:** Disables PHP, forcing the egress router to do both the label pop and the IP lookup. Required when the egress needs the label for VPN or QoS decisions.

```
set protocols mpls label-switched-path ToR4 ultimate-hop-popping
```

---

### MPLS Routing Tables in Junos

| Table | Purpose |
|-------|---------|
| `inet.0` | Standard IPv4 routing table. Does not contain MPLS paths. |
| `inet.3` | Populated by MPLS signaling protocols (LDP/RSVP). BGP uses this table to resolve next-hops for VPNs and labeled unicast routes. |
| `mpls.0` | The MPLS forwarding table. Contains label-to-label swap/pop entries used by the LFIB. |

---

### LDP — Label Distribution Protocol

LDP is a standards-based signaling protocol that automatically distributes labels for IGP-learned prefixes. It does not support traffic engineering — it simply mirrors the IGP topology and creates one LSP per prefix.

**Key characteristics:**
- Builds a full mesh of label bindings between all LDP-speaking routers.
- Every potential LSP is created — one for each reachable loopback prefix by default.
- Relies entirely on the IGP for loop prevention and route failover.
- Primarily used to enable iBGP next-hop resolution in service provider cores.

**Discovery and Session Establishment**

1. Hello packets are sent to multicast `224.0.0.2` on UDP port 646 — this discovers neighbors on directly connected links.
2. Once a neighbor is discovered, a TCP session is established to port 646 using the router's loopback (transport address) as the source.
3. After the TCP session is up, label bindings are exchanged.

**Label advertisement behavior:**
- **Downstream Unsolicited (DU):** Default. Labels are advertised to all neighbors without being requested.
- **Liberal Label Retention:** Default in Junos. The LIB stores all received label bindings, even from non-best-path neighbors. This speeds up convergence if the IGP path changes.
- By default, only `/32` loopback addresses generate LDP LSPs.

**Configuration**

```
set protocols ldp interface ge-0/0/0.0
set protocols ldp interface ge-0/0/1.0
set protocols ldp interface lo0.0
```

**Monitoring and Troubleshooting**

```
show ldp neighbor
show ldp interface
show ldp session
show ldp database
show ldp database session 192.168.101.1
traceroute mpls ldp 192.168.101.3 no-resolve
```

---

### RSVP — Resource Reservation Protocol

RSVP is a signaling protocol used to establish explicitly routed, traffic-engineered LSPs with bandwidth reservations. Unlike LDP, RSVP LSPs are manually or CSPF-calculated rather than IGP-driven.

**Key characteristics:**
- Relies on an IGP to be running (OSPF or IS-IS).
- Supports **explicit routing** via EROs (Explicit Route Objects).
- Supports **bandwidth reservation** per LSP.
- Uses **soft-state** — RSVP state must be refreshed periodically or it times out.
- Default action is penultimate hop popping.

**RSVP path configuration**

```
set protocols mpls label-switched-path ToR4 from 1.1.1.1
set protocols mpls label-switched-path ToR4 to 4.4.4.4
```

RSVP requires traffic engineering enabled and you need to ensure the IGP is configured for it
- For OSPF
```
set protocols ospf traffic-engineering
```

- For ISIS
```
set protocols isis level 2 wide-metrics-only 
set protocols isis traffic-engineering
```

**RSVP Message Flow**

1. **PATH message** — Sent ingress → egress. Carries the ERO (the desired path), requested bandwidth, and PHOP (previous hop). Installs path state on each router along the way.
2. **RESV message** — Sent egress → ingress. Allocates resources and distributes labels hop-by-hop back to the ingress.
3. **RRO** (Record Route Object) — Populated in the RESV message as it travels back. Records each hop that was traversed, useful for verifying the active path.

**LSP Priority**

RSVP uses setup and hold priorities (0 = highest, 7 = lowest) to control which LSPs can preempt others when bandwidth is constrained.

```
set protocols mpls label-switched-path ToR4 priority 4 4
```

**Fast Reroute (FRR)**

FRR pre-computes and pre-signals backup paths so that traffic can be rerouted in under 50ms when a link or node failure is detected.

- **Link protection:** Protects against failure of the next-hop link.
- **Node protection:** Protects against failure of the next-hop router.

```
set protocols mpls label-switched-path ToR4 fast-reroute
```

**Primary and Secondary Paths**

LSPs can have a primary path (preferred) and one or more secondary paths (standby). If the primary fails, traffic switches to the secondary.

```
set protocols mpls label-switched-path ToCust1nyc to 172.16.0.5
set protocols mpls label-switched-path ToCust1nyc primary via-dallas
set protocols mpls label-switched-path ToCust1nyc secondary via-chicago

set protocols mpls path via-dallas 10.1.2.2 strict
set protocols mpls path via-dallas 10.2.3.3 strict
set protocols rsvp interface ge-0/0/0.0
```

**Monitoring and Troubleshooting**

```
show mpls lsp
show mpls lsp ingress extensive
show rsvp session
show rsvp interface
show route table inet.3
show route table mpls.0
```

Sample `show mpls lsp` output:

```
root@vRouter1> show mpls lsp
Ingress LSP: 1 sessions
To              From            State Rt P     ActivePath       LSPname
4.4.4.4         1.1.1.1         Up     0 *                      ToR4
Total 1 displayed, Up 1, Down 0

Egress LSP: 1 sessions
To              From            State   Rt Style Labelin Labelout LSPname
1.1.1.1         4.4.4.4         Up       0  1 FF       3        - ToR1
Total 1 displayed, Up 1, Down 0

Transit LSP: 0 sessions
Total 0 displayed, Up 0, Down 0
```

Sample `show rsvp interface` output:

```
root@vRouter1> show rsvp interface
RSVP interface: 5 active
                          Active  Subscr- Static      Available   Reserved    Highwater
Interface          State  resv    iption  BW          BW          BW          mark
ge-0/0/0.0             Up       1   100%  1000Mbps    1000Mbps    0bps        0bps
ge-0/0/1.0             Up       0   100%  1000Mbps    1000Mbps    0bps        0bps
ge-0/0/5.0             Up       0   100%  1000Mbps    1000Mbps    0bps        0bps
ge-0/0/6.0             Up       1   100%  1000Mbps    1000Mbps    0bps        0bps
lo0.0                  Up       0   100%  0bps        0bps        0bps        0bps
```

Sample `show mpls lsp ingress extensive` output:

```
root@lumen-sf-pe1> show mpls lsp ingress name ToCust1nyc extensive
Ingress LSP: 1 sessions
172.16.0.5
  From: 172.16.0.1, State: Up, ActiveRoute: 0, LSPname: ToCust1nyc, LSPid: 2
  ActivePath: via-dallas (primary)
  LSPtype: Static Configured, Penultimate hop popping
  LoadBalance: Random
  Encoding type: Packet, Switching type: Packet, GPID: IPv4
 *Primary   via-dallas       State: Up
    Priorities: 7 0
    Bandwidth: 100Mbps
    Flap Count: 2
    Received RRO (ProtectionFlag 1=Available 2=InUse 4=B/W 8=Node 10=SoftPreempt 20=Node-ID):
          10.1.3.3(Label=299872) 10.3.5.5(Label=3)
```

---

### CSPF — Constrained Shortest Path First

CSPF is the path calculation algorithm used by RSVP to find a path that satisfies configured constraints. It runs SPF on the TED rather than on the standard IGP topology.

**How it works:**
1. The TED is populated by IGP TE extensions — links that do not meet constraints are pruned first.
2. Dijkstra SPF is run on the pruned topology to find the shortest path.
3. The result is encoded as an ERO and placed in the RSVP PATH message.
4. CSPF runs locally on the ingress LSR only.

**TED population by IGP:**

| IGP | TE Extension | Configuration |
|-----|--------------|---------------|
| IS-IS | Extra TLVs in LSPs (enabled with wide-metrics) | `set protocols isis level 2 wide-metrics-only` |
| OSPF | Type 10 (Opaque) LSAs | `set protocols ospf traffic-engineering` |

**TED scope limitations:**
- IS-IS: TED is only populated from routers at the **same level**. Level 1 TE info does not cross into Level 2.
- OSPF: Type 10 LSAs do not cross area boundaries by default. TED is limited to the local area.

**CSPF constraints:**
- **Bandwidth:** Path must have sufficient available bandwidth on every link.
- **Admin Groups (Colors):** 32-bit bitmask allowing interfaces to be grouped. LSPs can include or exclude groups.
- **Hop limit:** Maximum number of hops allowed in the path.

**Bandwidth tie-breakers (when multiple equal-cost paths exist):**
- `least-fill` — Prefer the path with the most available bandwidth (spread load).
- `most-fill` — Prefer the path with the least available bandwidth (pack links before using new ones).
- `random` — Choose randomly among equal-cost paths.

**Admin Groups configuration:**

```
set protocols mpls admin-groups red 0
set protocols mpls admin-groups blue 1
set protocols mpls interface ge-0/0/0.0 admin-group red
set protocols mpls label-switched-path ToR4 admin-group include-any red
```

---

### MPLS Fragmentation

By default, MPLS does not allow fragmentation. If a labeled packet exceeds the MTU, it is dropped. There are two options to address this:

```
set protocols mpls path-mtu allow-fragmentation
set protocols mpls path-mtu rsvp mtu-signaling
```

---

### Segment Routing (SR)

Segment Routing is a source-routing paradigm where the ingress node encodes the entire forwarding path as an ordered list of instructions — called **segments** — directly in the packet header. Transit nodes execute each instruction in sequence. There is no per-flow state required in the network core.

**Why it matters:** SR replaces the need for LDP (and in many cases RSVP-TE) by using only the IGP to signal forwarding paths. Simpler control plane, same capabilities.

---

#### SR-MPLS

In SR-MPLS, each Segment ID (SID) is represented as an **MPLS label**. The segment list becomes a label stack. Transit routers perform standard MPLS label operations — they don't need to understand SR at all, only the ingress node needs to build the stack.

**Advantages over LDP:**
- No separate signaling protocol — just the IGP
- Explicit paths without per-hop RSVP state
- Inherently ECMP-aware
- Simpler control plane

---

#### SID Types

| SID Type | Significance | Description |
|----------|-------------|-------------|
| **Prefix SID** | Global | Advertised by the IGP. Tied to a prefix (usually a loopback). All routers in the domain allocate the same label for this SID. |
| **Node SID** | Global | A special prefix SID representing the router itself (its loopback). The most common SID type in practice. |
| **Adjacency SID** | Local | Represents a specific link between two nodes. Locally assigned (not globally unique). Used to steer traffic over a particular interface, bypassing normal SPF. |

---

#### Label Ranges — SRGB and SRLB

- **SRGB** (Segment Routing Global Block) — the label range reserved for globally significant SIDs. Every router allocates labels from its SRGB for prefix/node SIDs. The actual label value = `SRGB base + SID index`.
  - Junos default SRGB: **800000–900000**
  - Example: SID index 100 → label **800100**
- **SRLB** (Segment Routing Local Block) — the label range reserved for locally significant SIDs such as adjacency SIDs. Locally assigned by each router.

> **Exam tip:** Junos SRGB starts at 800,000. If the exam gives you a SID index, add it to 800,000 to get the label value.

---

#### IGP Extensions for SR

SR requires the IGP to advertise SID information alongside normal topology data. No additional signaling protocol is needed.

| IGP | Extension | Notes |
|-----|-----------|-------|
| IS-IS | New TLVs in existing LSPs | Wide metrics required (`wide-metrics-only`) |
| OSPF | Type 10 (Opaque) LSAs | Enable with `set protocols ospf traffic-engineering` |

---

#### Basic Configuration

**IS-IS SR:**

```
set chassis network-services enhanced-ip
set protocols isis level 2 wide-metrics-only
set protocols isis source-packet-routing srgb start-label 800000 index-range 1000
set protocols isis source-packet-routing node-segment ipv4-index 1
```

**OSPF SR:**

```
set chassis network-services enhanced-ip
set protocols ospf traffic-engineering
set protocols ospf source-packet-routing node-segment ipv4-index 11
set protocols ospf source-packet-routing srgb start-label 800000 index-range 1000
```

---

#### Segment Lists and Explicit Paths

A **segment list** is an ordered stack of SIDs that defines an explicit path. The ingress node pushes the full stack; each node pops the top label and forwards.

- **Node SIDs only:** traffic follows IGP shortest path between each node in the list
- **Adjacency SID included:** forces traffic over a specific link, regardless of IGP metric

This provides RSVP-TE-like traffic engineering without the signaling overhead.

---

#### SRv6 (Brief Overview)

SRv6 uses IPv6 as the data plane instead of MPLS. SIDs are encoded as **IPv6 addresses** in a **Segment Routing Header (SRH)** — an IPv6 extension header (type 4).

- Transit nodes that don't understand SRv6 simply forward the packet based on the IPv6 destination address
- The active SR node pops the current SID from the SRH and updates the IPv6 destination with the next SID
- Allows seamless tunneling through non-SRv6 infrastructure

---

#### Monitoring

```
show ospf spring sid-database

show route protocol spring-te
show route table inet.3
```

---

## Quick Reference

### Label Operations

| Operation | Who Does It | Description |
|-----------|-------------|-------------|
| Push | Ingress LSR | Adds a label to the packet |
| Swap | Transit LSR | Replaces the top label |
| Pop | Egress or penultimate hop | Removes the top label |

### Reserved Labels

| Label | Name | Meaning |
|-------|------|---------|
| 0 | IPv4 Explicit Null | Signal QoS intent to egress |
| 1 | Router Alert | Trap to local CPU |
| 2 | IPv6 Explicit Null | Signal QoS intent for IPv6 |
| 3 | Implicit Null | Trigger PHP at penultimate hop |

### LDP vs RSVP

| Feature | LDP | RSVP |
|---------|-----|------|
| Traffic Engineering | No | Yes |
| Explicit paths | No | Yes (ERO) |
| Bandwidth reservation | No | Yes |
| Path calculation | IGP topology | CSPF |
| LSPs created | All prefixes (loopbacks by default) | Manually configured |
| Soft-state | No | Yes (requires refresh) |
| Fast Reroute | No | Yes |
| Use case | iBGP next-hop resolution | TE, bandwidth guarantees |

### Key Junos Routing Tables

| Table | Contents |
|-------|----------|
| `inet.0` | Standard IP routes |
| `inet.3` | MPLS-signaled routes (BGP next-hop resolution) |
| `mpls.0` | Label forwarding entries (LFIB) |

### Monitoring Commands

| Command | Purpose |
|---------|---------|
| `show mpls lsp` | Summary of all LSPs |
| `show mpls lsp ingress extensive` | Detailed ingress LSP info including RRO |
| `show rsvp session` | Active RSVP sessions |
| `show rsvp interface` | RSVP interface state and bandwidth |
| `show ldp neighbor` | LDP neighbor adjacencies |
| `show ldp session` | LDP TCP sessions |
| `show ldp database` | LDP label bindings |
| `show ldp interface` | LDP-enabled interfaces |
| `show route table inet.3` | MPLS routes used for BGP resolution |
| `show route table mpls.0` | MPLS forwarding table |
| `traceroute mpls ldp <prefix>` | Trace an LDP LSP path |
