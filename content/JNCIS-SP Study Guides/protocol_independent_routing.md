---
title: Protocol-Independent Routing for JNCIS-SP
date: 2026-04-15
tags: [juniper, routing, networking]
---

## Protocol-Independent Routing

Protocol-independent routing features work regardless of which dynamic routing protocol is running. This covers how Junos selects routes when multiple sources compete, how to define static and summary routes, how to filter unwanted prefixes, and how to carve the routing table into separate instances for policy-based forwarding and VPNs.

---

### Route Preferences

When multiple protocols learn a route to the same destination, Junos uses **preference** (administrative distance) to pick the winner. Lower value wins.

| Protocol / Route Type | Default Preference |
|-----------------------|-------------------|
| Direct (connected) | 0 |
| Local | 0 |
| Static | 5 |
| OSPF Internal | 10 |
| IS-IS Level 1 Internal | 15 |
| IS-IS Level 2 Internal | 18 |
| RIP | 100 |
| Aggregate / Generated | 130 |
| OSPF AS External | 150 |
| IS-IS Level 1 External | 160 |
| IS-IS Level 2 External | 165 |
| BGP (internal and external) | 170 |

Preference can be manually overridden on static routes and via routing policy on dynamic routes.

> **Exam tip:** Static routes (preference 5) beat all dynamic protocols by default. If you want a static to act as a backup to an OSPF route (preference 10), set the static preference to something higher than 10.

---

### Static Routes

All static route configuration lives under `routing-options`. A valid next-hop is required.

**Next-hop types:**

| Next-hop | Behavior |
|----------|---------|
| IP address | Forward to the specified next-hop |
| `reject` | Drop packet and send **ICMP unreachable** to source |
| `discard` | Drop packet **silently** — no ICMP |
| `next-table` | Pass lookup to another routing table |

**Key options:**

- `qualified-next-hop` — define multiple next-hops for the same prefix, each with its own preference. Used for floating static routes (primary + backup).
- `no-readvertise` — prevents the route from being picked up by routing protocols via redistribution.
- `preference` — override the default preference of 5.
- `resolve` — allow the next-hop to be resolved recursively through the routing table (useful when the next-hop is not directly connected).

```
routing-options {
    static {
        route 0.0.0.0/0 {
            next-hop 192.168.1.1;
            no-readvertise;
        }
        route 10.0.0.0/8 {
            next-hop 10.1.1.1;
            qualified-next-hop 10.2.2.2 {
                preference 10;
            }
        }
    }
    rib inet6.0 {
        static {
            route ::/0 next-hop 2001:db8::1;
        }
    }
}
```

> **Floating static route:** The primary next-hop uses default preference 5. The `qualified-next-hop` uses a higher preference value, so it only becomes active if the primary disappears. Useful for backup paths behind a dynamic protocol.

---

### Aggregate Routes

Aggregate routes are manually defined summary prefixes that become active when at least one **contributing route** (a more specific prefix within the aggregate) is active in the routing table.

- Default preference: **130**
- Default next-hop: **reject** (drop + ICMP unreachable)
- Used to reduce the number of routes advertised to peers and to hide internal instability
- The aggregate prefix itself is what gets advertised — contributing routes are not

```
routing-options {
    aggregate {
        defaults {
            community 1:888;
        }
        route 172.29.0.0/22;
        route 172.25.0.0/16 {
            community 1:999;
            discard;
        }
    }
}
```

```
show route 172.29.0.0/22 exact detail
```
Shows the aggregate route and which contributing prefixes are making it active.

> **discard vs reject on aggregate routes:** `reject` (the default) sends ICMP unreachable back to the source when a packet hits the aggregate but no more-specific route exists. `discard` silently drops it. Use `discard` when you don't want to reveal to senders that the destination doesn't exist.

---

### Generated Routes

Generated routes are nearly identical to aggregate routes with one key difference: instead of defaulting to a `reject` next-hop, a generated route **inherits the next-hop of its primary contributing route**.

- **Primary contributing route** = the active contributing route with the lowest preference value; ties broken by lowest prefix length, then numerically lowest prefix
- Commonly used as a **route of last resort** — a generated default (`0.0.0.0/0`) that only activates when a specific condition (a contributing route) is present in the table

```
routing-options {
    generate {
        route 0.0.0.0/0 {
            policy contrib-routes-exist;
        }
    }
}
```

Both aggregate and generated routes are matched in routing policy with:
```
from protocol aggregate
```

```
show route hidden
```
Shows routes that exist in the table but are inactive (e.g., a generated route whose contributing route disappeared).

---

### Martian Addresses

Martian addresses are prefixes that Junos silently ignores and never installs in the RIB, regardless of the source. They represent invalid or reserved address space.

**Default IPv4 martians:** `0.0.0.0/8`, `127.0.0.0/8`, `192.0.0.0/24`, `240.0.0.0/4` (plus their more-specifics)

**Default IPv6 martians:** loopback, link-local, and RFC 2373 reserved space

Additional prefixes can be added — useful for blocking bogons or RFC 1918 space from being installed via a routing protocol:

```
routing-options {
    martians {
        192.168.0.0/16 orlonger allow;
    }
}
```

Note the `allow` keyword — you can also use martians to **permit** a prefix that would otherwise be blocked by default.

```
show route martians table inet.0
```

---

### Routing Instances

By default, Junos creates the **master** routing instance, which contains `inet.0` (IPv4), `inet6.0` (IPv6), and related tables. User-defined routing instances partition the router into separate forwarding domains — each with its own RIB, interfaces, and protocol configuration.

**Instance types:**

| Type | Use Case |
|------|---------|
| `forwarding` | Filter-based forwarding (FBF). No routing protocol support. Lightweight — just a separate forwarding table. |
| `virtual-router` | Full independent routing instance. Runs its own protocols. Similar to a VRF but for the router itself. |
| `vrf` | Layer 3 VPN. Used in MPLS L3VPN deployments. Has import/export route distinguishers and route targets. |
| `vpls` | Virtual Private LAN Service — L2VPN over MPLS. |
| `l2vpn` | Layer 2 VPN circuits (point-to-point). |

```
show route instance
show route instance <name> detail
ping 172.26.25.1 routing-instances my-instance
```

**RIB Groups** allow routes from one routing table to be shared into another. Commonly used with FBF so that forwarding instances have access to the interface routes from `inet.0` (so they have valid next-hops):

```
routing-options {
    interface-routes {
        rib-group inet my-rib-group;
    }
    rib-groups {
        my-rib-group {
            import-rib [ inet.0 ISP-A.inet.0 ISP-B.inet.0 ];
        }
    }
}
```

- `import-rib` — the list of tables that will receive the shared routes (first entry is the primary/source table)
- `import-policy` — optional filter to control which routes are shared

---

### Load Balancing

Junos supports two load balancing modes for equal-cost paths:

- **Per-flow (default)** — traffic flows (identified by the 5-tuple) are consistently forwarded out the same interface. Prevents packet reordering. Preferred for most deployments.
- **Per-packet** — round-robin across all equal-cost interfaces regardless of flow. Can cause out-of-order delivery but maximizes link utilization.

**Default hash (5-tuple):** source IP, destination IP, IP protocol, source port, destination port.

Load balancing requires a routing policy applied to the forwarding table:

```
policy-options {
    policy-statement lb-policy {
        then {
            load-balance per-packet;
        }
    }
}

routing-options {
    forwarding-table {
        export lb-policy;
    }
}
```

> **BGP load balancing:** By default, BGP installs only a single best path per prefix. To load-balance across multiple equal-cost iBGP paths, you must also configure `multipath` under BGP:
> ```
> set protocols bgp multipath
> ```

---

### Filter-Based Forwarding (FBF)

FBF makes forwarding decisions based on criteria other than destination address — source IP, protocol, ingress interface, DSCP, etc. It uses a firewall filter to classify traffic and steer matching packets into a specific routing instance, effectively doing policy-based routing.

**Four-step configuration:**

**1. Create a firewall filter** that matches traffic and assigns a routing instance:

```
firewall {
    family inet {
        filter fbf-filter {
            term match-subnet-A {
                from {
                    source-address { 172.25.0.0/24; }
                }
                then {
                    routing-instance ISP-A;
                }
            }
            term match-subnet-B {
                from {
                    source-address { 172.20.20.0/24; }
                }
                then {
                    routing-instance ISP-B;
                }
            }
            term default {
                then accept;
            }
        }
    }
}
```

**2. Apply the filter to the ingress interface** (traffic is classified as it enters the router):

```
set interfaces ge-0/0/0 unit 0 family inet filter input fbf-filter
```

**3. Create `forwarding` routing instances** with the desired exit paths:

```
routing-instances {
    ISP-A {
        instance-type forwarding;
        routing-options {
            static {
                route 0.0.0.0/0 next-hop 172.20.0.2;
            }
        }
    }
    ISP-B {
        instance-type forwarding;
        routing-options {
            static {
                route 0.0.0.0/0 next-hop 172.20.10.2;
            }
        }
    }
}
```

**4. Create a RIB group** to share interface routes from `inet.0` into the forwarding instances (so they can resolve next-hops):

```
routing-options {
    interface-routes {
        rib-group inet fbf-rib-group;
    }
    rib-groups {
        fbf-rib-group {
            import-rib [ inet.0 ISP-A.inet.0 ISP-B.inet.0 ];
        }
    }
}
```

> **Why the RIB group is required:** Forwarding instances don't automatically inherit interface routes from `inet.0`. Without the RIB group, the forwarding instances have no next-hops and traffic blackholes.

---

## Quick Reference

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
