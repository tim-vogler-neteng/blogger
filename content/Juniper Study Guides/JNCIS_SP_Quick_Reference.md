---
title: JNCIS-SP Quick Reference
date: 2026-06-30
tags:
  - juniper
  - networking
  - quick-reference
---

## OSPF


### OSPF Packet Types

| Type | Name | Purpose |
|------|------|---------|
| 1 | Hello | Neighbor discovery, DR/BDR election, keepalive |
| 2 | DBD | LSDB table of contents exchange |
| 3 | LSR | Request specific missing LSAs |
| 4 | LSU | Delivers actual LSAs |
| 5 | LSAck | Confirms LSU receipt |

### LSA Types

| Type | Name | Originated By | Scope | Blocked By |
|------|------|--------------|-------|-----------|
| 1 | Router | Every router | Area | — |
| 2 | Network | DR | Area | — |
| 3 | Summary | ABR | Domain | Totally Stubby, Totally NSSA |
| 4 | ASBR Summary | ABR | Domain | Totally Stubby, Totally NSSA, NSSA, Stub |
| 5 | AS-External | ASBR | Domain | All stub and NSSA areas |
| 7 | NSSA External | ASBR (NSSA) | NSSA only | — (translated to Type 5 at ABR) |

### Area Types

| Area | Type 3 | Type 4 | Type 5 | Type 7 | Default Route |
|------|--------|--------|--------|--------|--------------|
| Standard | Yes | Yes | Yes | No | No |
| Backbone | Yes | Yes | Yes | No | No |
| Stub | Yes | No | No | No | Yes (auto) |
| Totally Stubby | No | No | No | No | Yes (auto) |
| NSSA | Yes | No | No | Yes | No (configurable) |
| Totally NSSA | No | No | No | Yes | Yes (auto) |

### Adjacency States

Down → Init → 2-Way → ExStart → Exchange → Loading → **Full**


## IS-IS


### PDU Types

| PDU Type | Name                         | Sent By       | Network Type | Purpose                                                                                                            |
|----------|------------------------------|---------------|--------------|---------------------------------------------------------------------------------------------------------------------|
| IIH      | IS-IS Hello                  | All Routers   | LAN or P2P   | Establishes and maintains adjacencies. P2P hellos cover both levels; LAN hellos are level-specific.                 |
| LSP      | Link State PDU               | All Routers   | All          | The IS-IS equivalent of an LSA. Carries routing info (prefixes, neighbors, metrics) in TLVs. Flooded per level.    |
| CSNP     | Complete Sequence Number PDU | DIS (on LAN)  | LAN or P2P   | Acts as a table of contents for the LSDB. Sent periodically on LANs (default 10s) to keep routers in sync.        |
| PSNP     | Partial Sequence Number PDU  | Any Router    | LAN or P2P   | Requests a missing LSP or acknowledges receipt of an LSP (on P2P links only).                                      |

### TLV Types

| TLV Type | Name                       | Primary Usage                  |
|----------|----------------------------|--------------------------------|
| 1        | Area Addresses             | Adjacency formation            |
| 22       | Extended IS Reachability   | Wide metrics for neighbors     |
| 129      | Protocols Supported        | IPv4 / IPv6 capability         |
| 135      | Extended IP Reachability   | Wide metrics for IPv4 prefixes |
| 236      | IPv6 Reachability          | IPv6 prefix advertisements     |
| 242      | Router CAPABILITY          | Advertising SR/TE features     |


## MPLS


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


## Tunnels


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

| Command                                 | Purpose                  |
| --------------------------------------- | ------------------------ |
| `show interfaces gr-0/0/0 detail`       | Tunnel state, counters   |
| `show interfaces gr-0/0/0 extensive`    | Includes keepalive stats |
| `ping <dest> size 1472 do-not-fragment` | MTU path test            |


## High Availability


### HA Technology Comparison

| Feature | GR | GRES | NSR | ISSU |
|---------|-----|------|-----|------|
| Protects against | Control plane restart | RE hardware failure | RE hardware failure | Software upgrade |
| Requires dual RE | No | Yes | Yes | Yes |
| Requires GRES | No | — | Yes | Yes |
| Requires NSR | No | No | — | Yes |
| Neighbor cooperation needed | Yes | No | No | No |
| Forwarding interrupted | No | No | No | No |
| Protocol reconvergence | No (helpers hold routes) | Yes | No | No |
| Transparent to neighbors | No | No | Yes | Yes |

### LACP Modes

| Mode | Behavior |
|------|---------|
| Active | Initiates LACP — sends PDUs unconditionally |
| Passive | Responds only — at least one side must be active |

### BFD Detection Time

```
Detection time = negotiated-interval × multiplier
Negotiated interval = max(local min-interval, remote min-interval)
```

### Key Commands

| Command                       | Purpose                             |
| ----------------------------- | ----------------------------------- |
| `show lacp status`            | LAG member link states              |
| `show interfaces ae0 detail`  | AE interface stats and member links |
| `show system switchover`      | GRES status                         |
| `show task replication`       | NSR sync status                     |
| `show chassis routing-engine` | RE status and mastership            |
| `show bfd session`            | BFD session states and timers       |


## Protocol Independent Routing


### Default Route Preferences

| Protocol | Preference |
|----------|-----------|
| Direct / Local | 0 |
| Static | 5 |
| OSPF Internal | 10 |
| IS-IS L1 Internal | 15 |
| IS-IS L2 Internal | 18 |
| RIP | 100 |
| Aggregate / Generated | 130 |
| OSPF External | 150 |
| IS-IS L1 External | 160 |
| IS-IS L2 External | 165 |
| BGP | 170 |

### Static Route Next-Hop Options

| Next-hop | Behavior |
|----------|---------|
| IP address | Forward to next-hop |
| `reject` | Drop + ICMP unreachable |
| `discard` | Silent drop |
| `next-table` | Redirect to another routing table |

### Routing Instance Types

| Type | Purpose |
|------|---------|
| `forwarding` | FBF — separate forwarding table, no protocols |
| `virtual-router` | Full isolated routing domain with protocols |
| `vrf` | MPLS L3VPN |
| `vpls` | MPLS L2VPN (multipoint) |
| `l2vpn` | MPLS L2VPN (point-to-point) |

### Key Commands

| Command | Purpose |
|---------|---------|
| `show route` | Show active routing table |
| `show route hidden` | Show inactive / suppressed routes |
| `show route martians table inet.0` | Show martian prefix list |
| `show route instance` | List all routing instances |
| `show route <prefix> exact detail` | Show a specific route with contributing details |
| `show route forwarding-table` | Show the forwarding table (LFIB) |


## Layer 2 / VLANs


### 802.1ad Tag Operations

| Operation | Where | Description |
|-----------|-------|-------------|
| Push | Ingress PEB | Add S-Tag to customer frame |
| Pop | Egress PEB | Remove S-Tag |
| Swap | Inter-provider handoff | Replace S-Tag value |

### STP Port States

| State | BPDUs | Learn MACs | Forward Data |
|-------|-------|-----------|-------------|
| Blocking | Rx only | No | No |
| Listening | Yes | No | No |
| Learning | Yes | Yes | No |
| Forwarding | Yes | Yes | Yes |

### RSTP Port Roles

| Role | Description |
|------|-------------|
| Root | Best path to root |
| Designated | Best port on segment toward root |
| Alternate | Backup root port — instant failover |
| Backup | Backup designated port (same bridge, same segment) |

### STP Timers

| Timer | Default | Impact |
|-------|---------|--------|
| Hello | 2s | BPDU send interval |
| Forward Delay | 15s | Time in Listening + Learning states |
| Max Age | 20s | BPDU expiry |

### Key Commands

| Command | Purpose |
|---------|---------|
| `show bridge mac-table` | Layer 2 forwarding table |
| `show bridge domain` | Bridge domain summary |
| `show spanning-tree bridge detail` | STP topology and root info |
| `show spanning-tree interface` | Per-port STP state and role |


## IPv6


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
