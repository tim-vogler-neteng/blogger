---
title: High Availability for JNCIS-SP
date: 2026-04-15
tags: [juniper, high-availability, networking]
---

## High Availability

Junos provides a layered HA architecture. Link aggregation handles physical link redundancy. Graceful Restart, GRES, and NSR handle control plane failures at increasing levels of sophistication. BFD accelerates failure detection for all routing protocols. Understanding which technology does what — and what each one requires — is the core exam objective for this topic.

---

### Link Aggregation Groups (LAG / LACP)

LAG bundles multiple physical interfaces into a single logical `ae` (aggregated Ethernet) interface, providing both redundancy and increased bandwidth. IEEE standard **802.3ad** — not to be confused with 802.1ad (Q-in-Q).

**LACP** (Link Aggregation Control Protocol) negotiates and maintains the bundle. It exchanges LACPDUs between peers to verify link health and membership.

**LACP modes:**

| Mode | Behavior |
|------|---------|
| **Active** | Initiates LACP negotiation. Sends LACPDUs unconditionally. |
| **Passive** | Only responds to LACPDUs — does not initiate. |

> At least one side of a LAG must be in **active** mode. Passive/passive means neither side initiates and the bundle never forms.

**LACP timers:**

| Rate | Interval | Detection Time (3× missed) |
|------|----------|--------------------------|
| **Fast** (default) | 1 second | 3 seconds |
| **Slow** | 30 seconds | 90 seconds |

**LACP port states:**
- **Collecting** — the port is receiving valid LACPDUs from the peer
- **Distributing** — the port is sending traffic into the bundle
- Both must be active on a port before it passes traffic

**Hashing:** flows are assigned to member links using a hash of the 5-tuple (src/dst IP, protocol, src/dst port) at L3, or src/dst MAC at L2. This keeps individual flows on the same link to prevent reordering.

**Configuration:**

```
# Step 1: reserve AE interface slots in the chassis
set chassis aggregated-devices ethernet device-count 2

# Step 2: assign member interfaces to the bundle
set interfaces ge-0/0/0 gigether-options 802.3ad ae0
set interfaces ge-0/0/1 gigether-options 802.3ad ae0

# Step 3: configure the AE interface
set interfaces ae0 aggregated-ether-options lacp active
set interfaces ae0 aggregated-ether-options minimum-links 1
set interfaces ae0 unit 0 family inet address 10.0.0.1/30
```

`minimum-links` sets the number of active member links required before the AE interface stays up. If active links drop below this threshold, the whole AE goes down — useful for forcing a failover rather than forwarding on a degraded bundle.

**Monitoring:**

```
show lacp status
show lacp interfaces ae0
show lacp statistics interfaces ae0
show interfaces ae0 detail
```

---

### Multichassis LAG (MC-LAG)

MC-LAG extends a LAG across two separate physical chassis, allowing a downstream device to dual-home to two upstream switches/routers while appearing as a single logical bundle.

- An **ICL** (Inter-Chassis Link) connects the two MC-LAG peers and carries control traffic and flooded frames
- Both chassis present the **same LACP system ID** to the downstream device so it sees one logical peer
- Provides chassis-level redundancy — the downstream device stays connected if one MC-LAG peer fails
- Commonly used in access/aggregation designs where a server or CE needs redundant uplinks to two PE/aggregation switches

**Key protocols and concepts:**
- **ICCP** (Inter-Chassis Control Protocol) — the control plane between MC-LAG peers. Runs over TCP, synchronizes LACP state, link state, and MAC/ARP tables between chassis. Must be reachable between peers (typically via the ICL or a dedicated management path).
- **ICL** — the physical or LAG link between the two MC-LAG peers. Carries ICCP control traffic and, in active-active mode, BUM (broadcast/unknown-unicast/multicast) frames that must reach both peers.
- **mc-ae-id** — a numeric ID that ties the MC-LAG bundle together across the two peers. Must match on both.
- **chassis-id** — differentiates the two peers: `0` on one, `1` on the other.
- **status-control** — one peer is designated `active` for LACP control purposes; the other is `standby`.

**MC-LAG modes:**

| Mode | Behavior |
|------|---------|
| **active-active** | Both peers forward traffic simultaneously. Load-balanced across both uplinks from the downstream device. |
| **active-standby** | Only the active peer forwards. Standby takes over on failure. |

**Configuration (both peers — differences noted inline):**

```
# Step 1: ICCP peering — configure on both peers with the other peer's address
# Peer 1
set protocols iccp local-ip-addr 10.0.0.1
set protocols iccp peer 10.0.0.2 session-establishment-hold-time 50
set protocols iccp peer 10.0.0.2 redundancy-group-id-list 1
set protocols iccp peer 10.0.0.2 liveness-detection minimum-interval 500 multiplier 3

# Peer 2 (mirror with addresses swapped)
set protocols iccp local-ip-addr 10.0.0.2
set protocols iccp peer 10.0.0.1 session-establishment-hold-time 50
set protocols iccp peer 10.0.0.1 redundancy-group-id-list 1
set protocols iccp peer 10.0.0.1 liveness-detection minimum-interval 500 multiplier 3

# Step 2: ICL between the two MC-LAG peers (shown as ae1)
set interfaces ge-0/0/2 gigether-options 802.3ad ae1
set interfaces ae1 aggregated-ether-options minimum-links 1
set interfaces ae1 unit 0 family inet address 10.0.0.1/30   # peer 2 uses 10.0.0.2/30

# Step 3: MC-LAG bundle toward the downstream device (ae0)
# Both peers — same system-id so downstream sees one logical peer
set interfaces ge-0/0/0 gigether-options 802.3ad ae0
set interfaces ae0 aggregated-ether-options lacp active
set interfaces ae0 aggregated-ether-options lacp system-id 00:01:02:03:04:05   # identical on both

# mc-ae options — mc-ae-id and redundancy-group must match; chassis-id differs
set interfaces ae0 aggregated-ether-options mc-ae mc-ae-id 1
set interfaces ae0 aggregated-ether-options mc-ae redundancy-group 1
set interfaces ae0 aggregated-ether-options mc-ae mode active-active
set interfaces ae0 aggregated-ether-options mc-ae chassis-id 0   # peer 2 uses chassis-id 1
set interfaces ae0 aggregated-ether-options mc-ae status-control active   # peer 2 uses standby

set interfaces ae0 unit 0 family inet address 192.168.1.1/24
```

> `status-control active` on one peer means it originates LACP PDUs. Only one peer should be `active`; the other must be `standby`.

**Monitoring:**

```
show iccp
show iccp peer detail
show multi-chassis mc-lag interfaces
show interfaces ae0 detail
show lacp interfaces ae0
```

---

### Graceful Restart (GR)

Graceful Restart allows a router's routing protocols to restart without withdrawing routes or dropping adjacencies. Neighboring routers (helpers) are notified and told to maintain their forwarding state and hold the restarting router's routes for a grace period.

**Roles:**
- **Restarting router** — the router whose control plane is restarting. Sends a GR notification before restarting.
- **Helper router** — a neighbor that receives the GR notification and agrees to maintain routes and adjacencies during the grace period.

**Key behaviors:**
- Forwarding continues on the restarting router (PFE is unaffected)
- Helpers hold stale routes and do not reconverge during the grace period
- After restart, the protocols re-sync and stale routes are cleaned up
- Requires both sides to support GR — if a neighbor doesn't support GR, it will reconverge normally

```
set routing-options graceful-restart
```

```
show ospf overview          # shows GR status
show bgp neighbor           # shows GR capability negotiated
```

---

### Graceful Routing Engine Switchover (GRES)

GRES provides hardware redundancy for platforms with two Routing Engines (re0 and re1). If the primary RE fails, the backup takes over the control plane.

**What GRES does:**
- Continuously syncs **interface state** and **kernel/forwarding state** between master and backup RE
- On failover, the backup RE takes over and the **Packet Forwarding Engine (PFE) continues forwarding without interruption** — no traffic drop during the switchover
- The backup RE becomes the new master in ~2 seconds

**What GRES does NOT do (without NSR):**
- Routing protocol adjacencies and tables are **not** synced
- Routing protocols restart on the new master and must reconverge from scratch

```
set chassis redundancy graceful-switchover
```

```
show system switchover
show chassis routing-engine
```

---

### Nonstop Active Routing (NSR)

NSR extends GRES by also replicating **routing protocol state** to the backup RE. When the backup takes over, routing protocols are already in sync — no reconvergence occurs and neighbors never know a switchover happened.

**NSR vs Graceful Restart:**
- GR requires neighbor cooperation (neighbors must support and honor the GR capability)
- NSR is **fully transparent to neighbors** — they see no event at all

**Requirements:**
- GRES must be configured and working before enabling NSR
- Both commands are required:

```
set routing-options nonstop-routing
set system commit synchronize
```

```
show task replication         # verify protocol state is syncing to backup RE
show system switchover        # verify GRES is healthy
```

---

### Nonstop Bridging (NSB)

NSB is the Layer 2 equivalent of NSR. It replicates Layer 2 protocol state (STP, LACP, LLDP, IGMP snooping) to the backup RE so that bridging protocols survive an RE switchover without reconvergence.

- Required for ISSU on platforms running Layer 2 protocols
- Enabled alongside GRES

```
set protocols layer2-control nonstop-bridging
```

---

### Unified In-Service Software Upgrade (ISSU)

ISSU performs a software upgrade with minimal traffic disruption by upgrading one RE at a time while the other continues forwarding.

**Prerequisites (all must be in place before starting):**
1. Dual REs installed and healthy
2. GRES configured and verified
3. NSR configured and verified (`show task replication` shows in-sync)
4. NSB configured if running L2 protocols
5. Software image staged on the router

**High-level process:**
1. Backup RE upgrades to the new software and reboots
2. Mastership switches to the newly upgraded RE (GRES handles this hitlessly)
3. Old master RE (now backup) upgrades and reboots
4. Normal dual-RE operation resumes

```
request system software in-service-upgrade /var/tmp/junos-image.tgz reboot
```

> **ISSU is not universally supported.** Check platform and release notes — some Junos versions and hardware combinations do not support ISSU.

---

### Bidirectional Forwarding Detection (BFD)

BFD is a lightweight hello protocol designed for one purpose: **fast failure detection**. It runs independently of any routing protocol and notifies the protocol immediately when a path goes down.

**Why it's needed:** OSPF Dead timers default to 40 seconds. BGP hold timers default to 90 seconds. BFD can detect failures in under a second.

**How it works:**
- Two BFD peers exchange control packets at the negotiated interval
- If `multiplier` consecutive packets are missed, the session is declared **Down**
- The routing protocol (OSPF, IS-IS, BGP, etc.) is notified immediately and reconverges

**Detection time formula:**

```
Detection time = negotiated-interval × multiplier
```

The negotiated interval is the **maximum** of the local `minimum-interval` and the remote's `minimum-interval` (both sides agree on the slower rate). Example: local = 300ms, remote = 500ms → negotiated = 500ms. With multiplier 3 → detection time = **1500ms**.

**BFD modes:**
- **Asynchronous** (default) — both peers send BFD packets at the negotiated interval. Standard mode.
- **Echo mode** — the local router sends echo packets that loop back through the remote's forwarding plane. Tests the forwarding path rather than the control plane. Can achieve faster detection because it uses the PFE, not the RE.

**Configuration:**

```
# OSPF
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 \
    bfd-liveness-detection minimum-interval 300 multiplier 3

# IS-IS
set protocols isis interface ge-0/0/0.0 \
    bfd-liveness-detection minimum-interval 300 multiplier 3

# BGP (multi-hop — for non-directly connected peers)
set protocols bgp group IBGP neighbor 10.0.0.2 \
    bfd-liveness-detection minimum-interval 300 multiplier 3

# Enable echo mode
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 \
    bfd-liveness-detection transmit-interval minimum-interval 300 \
    bfd-liveness-detection detection-time threshold 1000
```

**BFD parameters:**
- `minimum-interval` — minimum time between BFD packets (ms). Actual rate is negotiated with peer.
- `minimum-receive-interval` — minimum rate at which this router is willing to receive packets (if different from send rate)
- `multiplier` — number of missed packets before declaring the session down

**Monitoring:**

```
show bfd session
show bfd session detail
show bfd session extensive
```

**Sample `show bfd session` output:**

```
                                                  Detect   Transmit
Address          State     Interface              Time     Interval  Multiplier
10.0.0.2         Up        ge-0/0/0.0             0.900    0.300        3
```

---

## Quick Reference

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

| Command | Purpose |
|---------|---------|
| `show lacp status` | LAG member link states |
| `show interfaces ae0 detail` | AE interface stats and member links |
| `show system switchover` | GRES status |
| `show task replication` | NSR sync status |
| `show chassis routing-engine` | RE status and mastership |
| `show bfd session` | BFD session states and timers |
