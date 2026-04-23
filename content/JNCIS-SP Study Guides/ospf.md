---
title: OSPF Concepts for JNCIS-SP
date: 2026-04-15
tags: [juniper, ospf, networking]
---

## OSPF (Open Shortest Path First)

OSPF is a link-state interior gateway protocol (IGP). Each router floods Link-State Advertisements (LSAs) describing its interfaces and neighbors. Every router builds an identical Link-State Database (LSDB) and runs the Dijkstra SPF algorithm to compute the shortest path tree. OSPF runs directly over IP (protocol 89) and uses multicast for efficiency.

**Default route preferences in Junos:**
- OSPF Internal routes: **10**
- OSPF AS External routes: **150**

---

### Terms

- **LSDB** (Link-State Database) - The topological database. Within a single area, all routers must have an identical LSDB.
- **SPF** (Shortest Path First) - The Dijkstra algorithm each router runs against the LSDB to compute best paths.
- **Router ID (RID)** - A 32-bit identifier unique to each OSPF router. Junos selects the RID in this order: explicitly configured → highest active loopback IP → highest physical interface IP. Best practice is to configure it explicitly.
- **ABR** (Area Border Router) - A router with interfaces in multiple OSPF areas. Generates Type 3 (Summary) LSAs between areas.
- **ASBR** (AS Boundary Router) - A router that redistributes routes from outside OSPF into the OSPF domain. Generates Type 5 LSAs.
- **Backbone Router** - Any router with at least one interface in Area 0.
- **Internal Router** - All interfaces are in the same single area.

Remember, best practice to explicitly configure the router-id. See below: 
```
set routing-options router-id 4.4.4.4
```

---

### OSPF Packet Types

OSPF has five packet types. All run directly over IP (protocol 89) — no TCP/UDP — so reliability is handled by LSAcks.

| Type | Name | Purpose | Key Detail |
|------|------|---------|------------|
| 1 | **Hello** | Discovers neighbors, elects DR/BDR, maintains adjacencies. | Hello: 10s (broadcast/P2P), 30s (NBMA). Dead: 4× Hello. |
| 2 | **Database Description (DBD)** | Exchanges a summary (table of contents) of each router's LSDB. | Uses interface MTU. MTU mismatch causes ExStart stall. |
| 3 | **Link-State Request (LSR)** | Requests specific LSAs missing from the local LSDB. | Sent during the Loading state after comparing DBDs. |
| 4 | **Link-State Update (LSU)** | The workhorse — carries the actual LSAs. | Sent in response to an LSR or proactively on topology changes. |
| 5 | **Link-State Ack (LSAck)** | Confirms receipt of an LSU. | Required because OSPF runs over IP, not TCP. |

**Multicast addresses:**
- `224.0.0.5` — AllSPFRouters. Every OSPF router listens here.
- `224.0.0.6` — AllDRouters. Used by DROther routers to send LSUs/LSAcks to the DR and BDR only.
- OSPFv3 equivalents: `FF02::5` and `FF02::6`

---

### Adjacency States

OSPF adjacencies progress through seven states. Understanding where a session stalls tells you what's broken.

| State | What's Happening | Troubleshooting Note |
|-------|-----------------|---------------------|
| **Down** | No Hellos received from this neighbor. | Check physical layer, interface status, firewall filters. |
| **Init** | A Hello was received but this router's RID is not yet in the neighbor's Hello. | One-way communication — likely a filter blocking return Hellos. |
| **2-Way** | Bidirectional Hellos confirmed. Both routers see each other. | DR/BDR election occurs here. DROther↔DROther relationships stop here permanently. |
| **ExStart** | Master/slave negotiation before database exchange. | **MTU mismatch** is the most common cause of stalling here. |
| **Exchange** | DBD packets (LSDB table of contents) are exchanged. | MTU or corrupt packets can also appear here. |
| **Loading** | Missing LSAs identified from DBD comparison; LSRs sent, LSUs received. | Hangs here indicate memory/CPU exhaustion or packet loss. |
| **Full** | LSDBs are fully synchronized. SPF can now run. | This is the healthy end state. |

> **Exam tip — the 2-Way ceiling:** On a broadcast segment (e.g., Ethernet), DROther routers only reach **Full** with the DR and BDR. DROther-to-DROther relationships stay at **2-Way** permanently — this is normal and expected. Don't mistake it for a problem.

---

### Designated Router (DR) and Backup Designated Router (BDR)

On multi-access (broadcast) segments, every router forms a full adjacency only with the DR and BDR — not with every other router. This reduces the O(n²) adjacency and flooding problem.

**Election:**
- Based on **interface priority** (0–255, default 128). Higher wins.
- Ties broken by highest **Router ID**.
- Election is **non-preemptive** — the DR/BDR holds their role even when a higher-priority router joins. Election re-runs only when the OSPF process restarts.
- Priority 0 = ineligible for DR/BDR.
```
set protocols ospf area 0 interface ge-0/0/0 priority 0
```

**BDR role:** Takes over as DR immediately when the DR fails, without re-running a full election.

**Avoiding DR election (P2P):** Configuring a link as point-to-point skips the DR/BDR election and the 2-Way state entirely, taking adjacency to Full faster (saves up to 40 seconds). It also suppresses Type 2 LSA generation. The interface type needs to match on both sides of the link otherwise the adjacency won't come up.

```
set protocols ospf area 0 interface ge-0/0/0 interface-type p2p
```

---

### Metrics and Cost

OSPF uses **cost** as its metric. Lower cost is preferred.

- `cost = reference-bandwidth / interface-bandwidth`
- Default reference bandwidth: **100 Mbps**
- On a 1 Gbps interface with the default reference, cost = 100/1000 = **0.1**, which rounds to **1** — same as a 100 Mbps link. This is a well-known scaling problem.

**Best practice:** Raise the reference bandwidth to match your fastest links.

```
set protocols ospf reference-bandwidth 100g
```

You can also set cost manually per interface:

```
set protocols ospf area 0 interface ge-0/0/0.0 metric 10
```

---

### Adjacency Requirements

The following Hello packet fields must match between two routers to form an adjacency:

- Subnet mask of the link
- Hello interval
- Dead interval
- Options field (includes area type bits: E-bit and N-bit)
- Authentication

> **E-bit and N-bit:** These are carried in Hello packets to signal area type. **E-bit = 1** for standard/backbone areas (external routes allowed), **E-bit = 0** for stub areas. **N-bit = 1** for NSSA routers. Routers with mismatched bits will not form an adjacency — this is how all routers in a stub/NSSA area are forced to agree on the area type.

---

### OSPF Areas

Areas limit the scope of LSA flooding and allow route summarization at area borders. All non-backbone areas must connect to Area 0 either physically or via a virtual link — this prevents routing loops.

> **Split Horizon Rule:** An ABR only accepts and re-floods a Type 3 LSA if it received it via a backbone (Area 0) interface. This is why discontiguous areas cause routing problems and why all areas must connect to Area 0.

**Area types:**

| Area Type | Type 3 (Inter-area) | Type 4 | Type 5 (External) | Default Route From ABR | Notes |
|-----------|--------------------|---------|--------------------|------------------------|-------|
| Standard | Yes | Yes | Yes | No | Full LSA support. |
| Backbone (0) | Yes | Yes | Yes | No | All areas connect here. |
| Stub | Yes | No | No | Yes (metric 1) | ABR injects a default. No ASBRs allowed. |
| Totally Stubby | No | No | No | Yes | ABR blocks Type 3/4/5. Only default route enters. |
| NSSA | Yes | No | No (Type 7 allowed) | No (configurable) | Local ASBR can inject Type 7. No Type 5 from outside. |
| Totally NSSA | No | No | No (Type 7 allowed) | Yes | Type 3/4/5 blocked. Local ASBR Type 7 still allowed. |

> **"Totally" areas are not separate area types.** Totally Stubby and Totally NSSA are just Stub/NSSA areas where `no-summaries` is added to the ABR config only. All other routers in the area still think they are in a plain Stub or NSSA. The ABR is the only one doing the filtering.

> **NSSA does not automatically inject a default route.** Unlike a regular stub area, the NSSA ABR does not inject a default by default. You must configure it explicitly.

**Configuration:**

```
# Stub
set protocols ospf area 0.0.0.1 stub

# Totally Stubby (no-summaries on the ABR only)
set protocols ospf area 0.0.0.1 stub no-summaries

# NSSA
set protocols ospf area 0.0.0.1 nssa

# NSSA with default route
set protocols ospf area 0.0.0.1 nssa default-lsa type-7

# Totally NSSA (no-summaries on ABR only)
set protocols ospf area 0.0.0.1 nssa no-summaries
```

---

### LSA Types

LSAs are the data structures flooded inside LSUs. Each type has a defined originator and flooding scope.

| Type | Name | Originated By | Flooding Scope | Purpose |
|------|------|--------------|---------------|---------|
| **1** | Router | Every router | Local area only | Describes the router's interfaces and connected neighbors. B-bit = ABR, E-bit = ASBR. |
| **2** | Network | DR | Local area only | Represents a multi-access segment and lists all attached routers. Not generated on P2P links. |
| **3** | Summary | ABR | Inter-area (domain) | Carries prefix info from one area to another. Re-generated by every ABR it passes through. |
| **4** | ASBR Summary | ABR | Inter-area (domain) | Tells routers in other areas how to reach an ASBR. |
| **5** | AS-External | ASBR | Domain-wide | Carries externally redistributed prefixes. Blocked by all stub and NSSA areas. |
| **7** | NSSA External | ASBR (in NSSA) | NSSA area only | External routes within an NSSA. Translated to Type 5 by the ABR. |

> **Type 4 only needed across areas:** If you are in the same area as the ASBR, you already have its Type 1 LSA and can reach it directly. Type 4 only exists so routers in *other* areas can find the ASBR. The ABR is essentially saying: "I know you can't see the ASBR — let me advertise a route to it for you."

> **Type 3 re-generation:** Unlike a Type 5 (which is flooded as-is across the entire domain), a Type 3 LSA is **re-originated** by every ABR it crosses. Each ABR rewrites the LSA with its own Router ID as the advertising router.

> **NSSA P-bit:** When an ASBR in an NSSA originates a Type 7, it sets a **Propagate (P) bit**. When the NSSA ABR sees P-bit = 1, it translates the Type 7 to a Type 5 and floods it into the rest of the domain. P-bit = 0 means the Type 7 stays local.

> **Forwarding Address (FA):** Type 5 and Type 7 LSAs carry a Forwarding Address. If FA = `0.0.0.0`, route traffic toward the ASBR's Router ID (using a Type 1 or Type 4 LSA). If FA is a specific IP, traffic is forwarded to that address instead — useful when the ASBR is reachable through a different next-hop.

---

### External Routes — E1 vs E2

When an ASBR redistributes external routes into OSPF, each route is tagged with a metric type:

- **E2 (default in Junos):** The external metric stays constant across the entire domain. Every router sees the same cost that the ASBR set. Simple but ignores internal topology.
- **E1:** The external metric is cumulative. Each router adds its own internal cost to reach the ASBR on top of the original metric. More accurate but more work for the ASBR.

> **E1 always beats E2 for the same prefix.** OSPF prefers E1 over E2 regardless of the numeric metric values. E1 is considered more precise because it accounts for internal topology cost.

---

### Opaque LSAs (Types 9, 10, 11)

Opaque LSAs are extension containers — OSPF carries them without caring about the payload. They enable new features without changing the core protocol.

| Type | Scope | Common Use | SP Relevance |
|------|-------|-----------|-------------|
| **9** | Link-local | Graceful Restart signaling | Used to maintain adjacency state during hitless restarts. |
| **10** | Area-local | Traffic Engineering (TE) | Carries link bandwidth, delay, and admin group info for RSVP-TE CSPF. Enabled with `set protocols ospf traffic-engineering`. |
| **11** | AS-wide | AS-wide data | Rarely used compared to Type 10. |

---

### Security

**Authentication:**
OSPF supports MD5 authentication on a per-interface basis. Disabled by default in Junos.

```
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 authentication md5 1 key "s3cr3t"
```

**Passive interfaces:**
A passive interface is advertised into OSPF (so the prefix appears in the LSDB) but does not send Hellos and does not form adjacencies. Best practice is to make all interfaces passive by default and explicitly activate only the interfaces that should peer.

```
set protocols ospf area 0.0.0.0 interface lo0.0 passive
```

---

### Miscellaneous Options

- **BFD** — Enables sub-second failure detection independent of OSPF Hello/Dead timers.

```
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 bfd-liveness-detection minimum-interval 300
```

- **Graceful Restart** — Informs neighbors before the OSPF process restarts so they continue forwarding as if the router is still up. Not enabled by default.

```
set protocols ospf graceful-restart
```

- **prefix-export-limit** — Caps the number of external routes accepted into the OSPF domain to protect against route table explosion.

```
set protocols ospf prefix-export-limit 1000
```

- **Virtual Links** — Allows a non-backbone area to connect to Area 0 through another area when a direct physical connection is not possible. Configured between two ABRs.

```
set protocols ospf area 0.0.0.2 virtual-link neighbor-id 3.3.3.3 transit-area 0.0.0.2
```

---

### Monitoring and Troubleshooting

**Common adjacency problems:**

| Problem | What to Check |
|---------|--------------|
| No neighbor detected | Physical/datalink connectivity, IP subnet/mask match, area ID match, area type match, authentication, Hello/Dead timers, network type. |
| Stuck in ExStart | MTU mismatch between neighbors. |
| Stuck in 2-Way | Normal for DROther↔DROther on a broadcast segment. If unexpected, check DR/BDR election. |
| Stuck in Loading | Memory/CPU exhaustion, packet loss dropping LSUs. |

**Useful show commands:**

```
show ospf neighbor
show ospf neighbor detail
show ospf interface
show ospf interface lo0.0 extensive
show ospf database
show ospf database detail
show ospf database external
show ospf database nssa
show ospf route
show ospf statistics
show ospf overview
```

**`show ospf overview` — sample output:**

```
root@ABR> show ospf overview
Instance: master
  Router ID: 2.2.2.2
  Route table index: 0
  Area border router
  LSA refresh time: 50 minutes
  Area: 0.0.0.0
    Stub type: Not Stub
    Authentication Type: None
    Area border routers: 0, AS boundary routers: 1
    Neighbors
      Up (in full state): 1
  Area: 0.0.0.1
    Stub type: Stub, Stub cost: 10
    Authentication Type: None
    Area border routers: 0, AS boundary routers: 0
    Neighbors
      Up (in full state): 1
  Topology: default (ID 0)
    Full SPF runs: 6
    SPF delay: 0.200000 sec, SPF holddown: 5 sec, SPF rapid runs: 3
```

**`show ospf interface lo0.0 extensive` — passive interface:**

```
root@ABR> show ospf interface lo0.0 extensive
Interface           State   Area            DR ID           BDR ID          Nbrs
lo0.0               DRother 0.0.0.0         0.0.0.0         0.0.0.0            0
  Type: LAN, Address: 2.2.2.2, Mask: 255.255.255.255, MTU: 65535, Cost: 0
  Adj count: 0, Passive
  Hello: 10, Dead: 40, ReXmit: 5, Not Stub
  Auth type: None
  Topology default (ID 0) -> Passive, Cost: 0
```

**`show ospf statistics` — packet counters:**

```
root@ABR> show ospf statistics
Packet type             Total                  Last 5 seconds
                   Sent      Received        Sent      Received
   Hello            450          1853           0             2
     DbD              5             5           0             0
   LSReq              2             2           0             0
LSUpdate             18            10           0             0
   LSAck              9            18           0             0
LSAs flooded           :                   16, last 5 seconds :            0
LSAs retransmitted     :                    0, last 5 seconds :            0
```

---

## Quick Reference

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
