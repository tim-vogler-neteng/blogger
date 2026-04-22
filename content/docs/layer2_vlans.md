---
title: Layer 2 Bridging, VLANs, and STP for JNCIS-SP
date: 2026-04-15
tags: [juniper, layer2, vlan, stp, networking]
---

## Layer 2 Bridging and VLANs

Service provider networks often need to deliver Layer 2 connectivity between geographically separated customer sites. Junos implements this using bridge domains, which define the L2 forwarding boundaries, and 802.1ad (Q-in-Q) to tunnel customer VLAN spaces across the provider network without overlap.

---

### Terms

- **Bridge Domain** — a Layer 2 forwarding domain. Like a VLAN. It defines which interfaces share the same broadcast domain and MAC table.
- **EVC** (Ethernet Virtual Connection) — the L2 service sold by a SP to a customer. It defines the endpoints of a Layer 2 circuit.
- **C-Tag** (Customer Tag) — the inner 802.1q tag. Any VLAN 1–4094 from the customer's space.
- **S-Tag** (Service Tag) — the outer 802.1ad tag. Assigned by the SP to identify the customer. Encapsulates all of that customer's C-Tags.
- **PBN** (Provider Bridge Network) — the entire SP Layer 2 fabric.
- **PEB** (Provider Edge Bridge) — the SP edge device. Pushes/pops S-Tags on customer-facing ports.
- **S-VLAN Bridge** — an interior SP device that only examines and switches based on the S-Tag.
- **Customer ports** — PEB ports facing the customer. S-Tags are applied or removed here.
- **Network ports** — interior SP ports that carry double-tagged frames without modification.
- **IRB** (Integrated Routing and Bridging) — a logical interface that gives a bridge domain an IP address, enabling the router to act as the default gateway for hosts in that domain.

---

### 802.1q

The standard VLAN tagging protocol. Inserts a 4-byte tag into the Ethernet frame.

- VLAN ID field is **12 bits** → only **4094 usable VLANs** (0 and 4095 are reserved)
- This creates problems for service providers: Not enough vlans to support all customers, issues with vlan overlap,  STP compatibility issues while incorporating multiple domains, and needing to learn all customer MAC addresses

---

### 802.1ad — Q-in-Q (Provider Bridging)

802.1ad stacks a second VLAN tag (S-Tag) on top of the customer's existing tag (C-Tag). The SP assigns each customer a unique S-Tag, so customer VLAN spaces are fully isolated and can overlap without conflict.

- The SP **core** devices only need to learn and switch on the S-Tag — they are never exposed to customer C-Tags

**TPID reference:**
- **TPID** (Tag Protocol Identifier) —  A 16-bit field in an Ethernet frame header that identifies the frame as being VLAN-tagged and indicates the location of the tag information.

| TPID | Standard | Tag Type |
|------|----------|---------|
| `0x8100` | 802.1q | Customer (C-Tag) or single-tagged |
| `0x88A8` | 802.1ad | Provider (S-Tag) |

**Tag operations:**

| Operation | Description |
|-----------|-------------|
| **Push** | Add S-Tag to the frame (ingress PEB, customer-facing port) |
| **Pop** | Remove S-Tag from the frame (egress PEB, customer-facing port) |
| **Swap** | Replace one S-Tag with another (inter-provider handoff) |

Compound operations for multi-provider scenarios: pop-pop, pop-swap, swap-swap, push-push

---

### Provider Bridge Configuration

**IVL (Independent VLAN Learning)** is the simplest Q-in-Q deployment. The PEB uses access ports on the customer-facing side and the S-Tag as the bridge VLAN on network-facing ports. Each customer MAC address is learned and associated with their S-Tag VLAN.

**PEB interface config:**

```
# customer-facing (access)
set interfaces ae1 unit 0 family bridge interface-mode access
set interfaces ae1 unit 0 family bridge vlan-id 1001

# network-facing (trunk)
set interfaces ge-0/0/3 unit 0 family bridge interface-mode trunk
set interfaces ge-0/0/3 unit 0 family bridge vlan-id 1001


set bridge-domains Cust1 vlan-id 1001
```

**Tunneling specific customer VLANs through the core** using `inner-vlan-id-list`:

```
ge-0/0/4 {
    flexible-vlan-tagging;
    encapsulation flexible-ethernet-services;
    unit 0 {
        vlan-id 2002;
        family bridge {
            interface-mode trunk;
            inner-vlan-id-list 200-201;
        }
    }
}
```

---

### Virtual Switching

When multiple customers need to tunnel the same C-Tag VLANs across the provider core, IVL breaks down — the MAC tables from different customers would collide. **Virtual switching** solves this by giving each customer their own isolated L2 forwarding domain, similar to how VRFs partition routing tables.

Each customer gets a `virtual-switch` routing instance with its own bridge domains and MAC table.

```
routing-instances {
    customer1 {
        instance-type virtual-switch;
        interface ge-0/0/1.0;
        bridge-domains {
            bd-100 {
                vlan-id 100;
                interface ge-0/0/1.0;
            }
            bd-200 {
                vlan-id 200;
                interface ge-0/0/1.0;
            }
        }
    }
    customer2 {
        instance-type virtual-switch;
        interface ge-0/0/2.0;
        bridge-domains {
            bd-100 {
                vlan-id 100;            # same VLAN ID, isolated from customer1
                interface ge-0/0/2.0;
            }
        }
    }
}
```

---

### Integrated Routing and Bridging (IRB)

An IRB interface is a logical Layer 3 interface attached to a bridge domain. It allows the router to participate in the bridged domain as a default gateway — the router can both bridge between ports in the domain and route traffic in and out of it.

- IRB interfaces are numbered to match their bridge domain (e.g., `irb.10` for VLAN 10)
- Configured under `interfaces irb`
- Associated with a bridge domain via `routing-interface`

```
interfaces {
    irb {
        unit 10 {
            family inet {
                address 192.168.10.1/24;
            }
        }
    }
}

bridge-domains {
    customer-vlan10 {
        vlan-id 10;
        routing-interface irb.10;
    }
}
```

Hosts in the bridge domain use `192.168.10.1` as their default gateway. Traffic destined for other subnets is routed by the IRB.

---

### S-Tag Rewrite (VLAN Translation)

When two providers collaborate on a cross-provider EVC, S-Tag values must be translated at the handoff point — each provider uses their own S-Tag numbering and the tags must be swapped at the interconnect.

This operation is **bidirectional** — configured once, applies in both directions.

```
ge-0/0/1 {
    flexible-vlan-tagging;
    encapsulation flexible-ethernet-services;
    unit 0 {
        family bridge {
            interface-mode trunk;
            vlan-id-list 2002;          # must use vlan-id-list, not vlan-id
            vlan-rewrite {
                translate 3001 2002;    # translate their S-Tag to ours
            }
        }
    }
}
```

> Note: S-Tag rewrite requires `vlan-id-list` on the interface, not `vlan-id`. Using `vlan-id` will cause the config to silently not work.

---

### Monitoring and Troubleshooting

```
show bridge mac-table
show bridge mac-table instance <name>
show bridge statistics
show bridge domain
show interfaces ge-0/0/0 detail
```

**Sample `show bridge mac-table` output:**

```
root@Core> show bridge mac-table
MAC flags  (S -static MAC, D -dynamic MAC, L -locally learned, R -Remote PE MAC)
Routing instance : default-switch
 Bridging domain : customer1, VLAN : 1001
   MAC                 MAC      Logical
   address             flags    interface
   00:50:00:00:0a:00   D        ge-0/0/3.0
   00:50:00:00:0b:00   D        ge-0/0/2.0
```

---

## Spanning-Tree Protocol (STP)

STP prevents Layer 2 loops and broadcast storms by building a loop-free logical topology across a bridged network. It does this by electing a root bridge and then selectively blocking redundant paths.

---

### Terms

- **BPDU** (Bridge Protocol Data Unit) — the control frames STP uses to exchange topology information
  - **Configuration BPDU** — used for normal root bridge election and topology maintenance
  - **TCN BPDU** (Topology Change Notification) — sent when a port transitions to notify the fabric to flush MAC tables
- **Bridge ID** — the unique identifier for a bridge: priority (2 bytes) + MAC address (6 bytes). Lower = more preferred.
- **Root Bridge** — the bridge with the lowest Bridge ID. All other bridges calculate their shortest path to it.
- **Root Port** — the single port on each non-root bridge that provides the best path to the root bridge.
- **Designated Port** — the port on each network segment that is closest to the root. Responsible for forwarding toward the root on that segment.
- **Port Cost** — metric used to find the lowest-cost path to the root, based on link speed.

**Default port costs (802.1D):**

| Link Speed | STP Cost |
|-----------|---------|
| 10 Mbps | 100 |
| 100 Mbps | 19 |
| 1 Gbps | 4 |
| 10 Gbps | 2 |

> Junos uses different default costs (e.g., 1 GE = 20,000) under 802.1w (RSTP). The values differ by revision — know that higher speed = lower cost.

---

### STP Port States

Classic STP transitions through five states. The Forward Delay timer (default **15 seconds**) controls how long a port spends in Listening and Learning.

| State | Sends/Receives BPDUs | Learns MACs | Forwards Data | Notes |
|-------|---------------------|-------------|---------------|-------|
| **Disabled** | No | No | No | Interface is admin down |
| **Blocking** | Receives only | No | No | Initial state; listening for a better path |
| **Listening** | Yes | No | No | Participates in election; not yet forwarding |
| **Learning** | Yes | Yes | No | Building MAC table before forwarding |
| **Forwarding** | Yes | Yes | Yes | Normal operation |

Transition path: **Blocking → Listening → Learning → Forwarding** (each step takes ~15 seconds with default timers)

---

### STP Timers

| Timer | Default | Purpose |
|-------|---------|---------|
| **Hello** | 2 seconds | How often the root bridge sends Configuration BPDUs |
| **Forward Delay** | 15 seconds | Time spent in Listening and Learning states each |
| **Max Age** | 20 seconds | How long a BPDU is kept before being considered stale |

> Total worst-case convergence time with defaults: **~50 seconds** (20s Max Age + 15s Listening + 15s Learning). This is why RSTP was developed.

---

### STP Topology Election

**Step 1 — Elect the Root Bridge:**
- Lowest Bridge ID wins (default priority: **32768**)
- Best practice: manually set the intended root to priority **4096** and the backup to **8192**

**Step 2 — Select Root Ports:**
- Each non-root bridge selects the port with the lowest **path cost** to the root
- Tiebreakers (in order): lowest upstream Bridge ID → lowest upstream port priority → lowest upstream port number

**Step 3 — Select Designated Ports:**
- On each segment, the bridge with the lowest path cost to the root wins the designated port
- Tiebreaker: lowest Bridge ID

**Step 4 — Block all other ports:**
- Ports that are neither root ports nor designated ports are placed in **Blocking** state

---

### RSTP (Rapid Spanning Tree — 802.1w)

Junos uses **RSTP by default**. RSTP dramatically improves convergence by replacing the timer-based state machine with an explicit proposal/agreement handshake between adjacent bridges.

**RSTP port states (simplified from five to three):**

| RSTP State | Equivalent STP States |
|------------|----------------------|
| **Discarding** | Disabled + Blocking + Listening |
| **Learning** | Learning |
| **Forwarding** | Forwarding |

**RSTP port roles (new roles not in classic STP):**

| Role | Description |
|------|-------------|
| **Root** | Best path to root bridge. Same as STP root port. |
| **Designated** | Best port on the segment toward root. Same as STP designated port. |
| **Alternate** | Backup to the root port. Immediately takes over if the root port fails (no waiting). |
| **Backup** | Backup to a designated port on the same shared segment (same bridge, two ports on same LAN). Rare. |
| **Disabled** | Port is administratively or operationally down. |

> **Key RSTP improvement:** Alternate ports provide near-instant failover — they are pre-computed backup paths held in Discarding state. When the root port fails, the alternate port transitions to Forwarding without waiting for timers.

**Edge ports:** Ports connected to end hosts (not switches) are configured as **edge ports**. They skip the STP election entirely and immediately enter Forwarding state — no 30-second wait. If an edge port receives a BPDU, it automatically loses its edge status and begins participating in STP normally.

---

### STP Security

- **BPDU Guard** — if an edge port receives a BPDU, the port is immediately disabled. Protects against someone plugging a rogue switch into an access port.
- **Root Guard** — prevents a port from ever becoming a root port, even if a superior BPDU is received. Protects the current root bridge from being displaced.
- **Loop Guard** — if a non-designated port stops receiving BPDUs (due to a unidirectional link failure), Loop Guard puts the port in a loop-inconsistent state instead of transitioning to Forwarding.

---

### STP Configuration

```
# Set bridge priority (must be a multiple of 4096)
set protocols rstp bridge-priority 4096

# Configure an edge port (connected to an end host)
set protocols rstp interface ge-0/0/10 edge

# Enable BPDU guard on an edge port
set protocols rstp interface ge-0/0/10 no-root-port

# Enable root guard on a port
set protocols rstp interface ge-0/0/1 no-root-port

# Enable loop guard globally
set protocols rstp no-root-port
```

---

### Operational Commands

```
show spanning-tree bridge
show spanning-tree bridge detail
show spanning-tree interface
show spanning-tree interface detail
show spanning-tree statistics
show spanning-tree statistics interface ge-0/0/1
```

**Sample `show spanning-tree bridge` output:**

```
STP bridge parameters:
  Routing instance name: default
  Context ID: 0
  Enabled protocol: RSTP
    Root ID: 32768.50:00:00:01:00:00
    Hello time: 2, Maximum age: 20, Forward delay: 15
    This bridge is the root
  Port       Interface     State          Role
  128:1      ge-0/0/1.0    Forwarding     Designated
  128:2      ge-0/0/2.0    Forwarding     Designated
  128:3      ge-0/0/3.0    Discarding     Alternate
```

---

## Quick Reference

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
