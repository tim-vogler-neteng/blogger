---
title: IS-IS Concepts for JNCIS-SP
date: 2026-04-15
tags:
  - juniper
  - is-is
  - networking
  - commands_verified
---

## IS-IS

IS-IS (Intermediate System to Intermediate System) is a link-state routing protocol used primarily in service provider networks. Like OSPF, it uses the Dijkstra SPF algorithm to compute shortest paths, but runs natively over CLNS rather than IP, making it protocol-agnostic and well-suited for multi-protocol environments.

---

### Terms

- **ES** (End System) - A host that originates and receives packets. ES-to-ES communication is host-to-host.
- **IS** (Intermediate System) - A router that forwards packets. IS-IS describes routing between intermediate systems.
- **CLNS/CLNP** - IS-IS runs natively over the Connectionless Network Service (CLNS) using CLNP, not IP. This is a key distinction from OSPF.
- **NSAP** (Network Service Access Point) - The addressing scheme IS-IS uses instead of IP addresses.
- **NET** (Network Entity Title) - The IS-IS address configured on a router. Format: `Area ID . System ID . NSEL`
  - Example: `49.0001.1921.6800.1001.00`
    - `49.0001` — Area ID
    - `1921.6800.1001` — System ID (6 bytes, often derived from an IP like 192.168.1.1)
    - `00` — NSEL (always 00 for a router)
- **System ID** - 6-byte unique identifier for a router within an area (similar to OSPF Router ID).
- **NSEL** (N-Selector) - The last byte of a NET, always `00` for routers.
- **L1 router** - Routes only within its area; sends traffic to unknown destinations toward the nearest L1/L2 router.
- **L2 router** - Routes between areas and toward other ASes.
- **L1/L2 router** - Does both; this is the Junos default.

---

### Link-State Database

- Runs the Dijkstra SPF algorithm.
- L1 and L2 maintain **separate LSDBs** — SPF is run independently for each level.
- Each router originates its own LSP and floods it throughout its level.
- LSDB synchronization is handled by CSNPs (full sync) and PSNPs (fill gaps).

---

### IS-IS Protocol Data Units (PDUs)

- **IIH** (IS-IS Hello) - Used to discover neighbors and maintain adjacencies. Contains the router's identity, capabilities, and configured area.
  - **L1 LAN IIH**: Sent by Level 1 routers on multi-access networks (like Ethernet).
  - **L2 LAN IIH**: Sent by Level 2 routers on multi-access networks.
  - **P2P IIH**: A single format used for point-to-point links, regardless of level.
- **LSP** (Link State PDU) - Carries the actual routing information, including connected neighbors, configured prefixes, and metric costs. Each LSP has a sequence number, checksum, and remaining lifetime.
  - **L1 LSP**: Contains routing information for the local area.
  - **L2 LSP**: Contains backbone routing information.
- **CSNP** (Complete Sequence Number PDU) - Contains a complete list of all LSPs in a router's LSDB. Used to ensure every router in the area has a consistent view of the network.
  - **L1 CSNP**: Summarizes the Level 1 LSDB.
  - **L2 CSNP**: Summarizes the Level 2 LSDB. On LAN segments, the DIS sends these periodically. On point-to-point links, they are typically sent only when the link first comes up.
- **PSNP** (Partial Sequence Number PDU) - Used to request missing LSPs or acknowledge receipt of specific LSPs. Unlike CSNPs, they only reference a subset of LSPs.
  - **L1 PSNP / L2 PSNP**: Used to fill gaps after a CSNP reveals a missing LSP, or as an explicit ACK on point-to-point links.

---

### Type, Length, Value (TLVs)

TLVs are the data structures embedded inside LSPs that carry routing information. Key TLVs to know for JNCIS-SP:

| TLV Type | Name                       | Primary Usage                  |
|----------|----------------------------|--------------------------------|
| 1        | Area Addresses             | Adjacency formation            |
| 22       | Extended IS Reachability   | Wide metrics for neighbors     |
| 129      | Protocols Supported        | IPv4 / IPv6 capability         |
| 135      | Extended IP Reachability   | Wide metrics for IPv4 prefixes |
| 236      | IPv6 Reachability          | IPv6 prefix advertisements     |
| 242      | Router CAPABILITY          | Advertising SR/TE features     |

---

### Adjacencies and Neighbors

An IS-IS adjacency is formed after a successful IIH exchange. Routers must agree on the following to form an adjacency:

- Area ID (for Level 1)
- Authentication (if configured)
- MTU

**Adjacency states (point-to-point):** Down → Init → Up

On LAN segments, all routers form a full adjacency with each other. This differs from OSPF, where only the DR and BDR form full adjacencies with other routers.

---

### Designated Intermediate System (DIS)

The DIS is elected on multi-access (LAN) segments to reduce flooding overhead. It is similar to OSPF's DR but with some important differences.

**Election**
- The router with the highest interface priority wins (default: 64, range: 0–127).
- Ties are broken by the highest SNPA (MAC address).
- Election is **preemptive** — a higher-priority router that joins later immediately takes over (unlike OSPF's DR).
- There is **no backup DIS** (no BDR equivalent).

**Role**
- Periodically floods CSNPs every 10 seconds to keep the LSDB synchronized on the LAN.
- Creates and maintains a **pseudonode LSP** that represents the LAN segment itself.
- All routers on the segment form an adjacency with the pseudonode rather than directly with each other — this simplifies the topology and reduces LSP count.

**Key differences from OSPF DR**

| Feature       | IS-IS DIS                          | OSPF DR                              |
|---------------|------------------------------------|--------------------------------------|
| Backup        | None                               | BDR exists                           |
| Preemptive    | Yes                                | No                                   |
| Hello rate    | 3x more frequently than normal     | Same as other routers                |
| Adjacencies   | All routers form full adjacencies  | Only DR/BDR form full adjacencies    |

**Identifying the DIS**

The `!` in `show isis adjacency` output marks the neighbor that is the DIS on that segment.

```
set protocols isis interface ge-0/0/1.0 level 2 priority 100
```

---

### Metrics

- In IS-IS, metrics are the "cost" assigned to an interface to influence path selection.
- Unlike OSPF, which can automatically calculate cost based on bandwidth, IS-IS metrics are traditionally static and require manual configuration.
- **Narrow Metrics** - The default. Supports values 0–63 (field is only 6 bits).
- **Wide Metrics** - Expands the metrics field to 24 bits. Required for Traffic Engineering (TE) and Segment Routing.

```
set protocols isis level 1 wide-metrics-only
set protocols isis level 2 wide-metrics-only
```

---

### Configuration, Monitoring, and Troubleshooting

**show isis adjacency**

```
root@R2-P> show isis adjacency
Interface             System         L State         Hold (secs) SNPA
ge-0/0/1.0            R4-P          ! 2 Up                     8  50:0:0:4:0:3
ge-0/0/2.0            R1-PE         ! 1 Up                    23  50:0:0:1:0:2
```

**Header breakdown:**

- **Interface**: The local interface where the neighbor was discovered.
- **System**: The System ID or hostname of the neighbor. If resolvable, you see a name (like `R1-PE`); otherwise the raw hex ID.
- **L**: Level. IS-IS uses Level 1 (intra-area) and Level 2 (inter-area).
- **State**: Adjacency status. `Up` is expected; `Down`, `Initialize`, or `Reject` indicate a problem.
- **Hold**: The dead timer. Counts down until the next Hello is expected. If it hits 0, the adjacency drops.
- **SNPA**: Subnetwork Point of Attachment. On Ethernet, this is the neighbor's MAC address.
- **!**: Indicates the neighbor is the DIS on that segment.

---

## Quick Reference

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
