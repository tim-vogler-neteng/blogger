---
title: Broadcom Switch Chipset Families
date: 2026-04-29
tags:
  - networking
  - hardware
  - broadcom
---

# Broadcom Switch Chipset Families

Broadcom dominates merchant silicon for data center and carrier switching. Their three main ASIC families, Tomahawk, Trident, and Jericho, each make different tradeoffs between bandwidth, feature depth, and buffer size. Most Arista, Cisco Nexus, and Juniper QFX/PTX platforms run one of these under the hood.

---

## 1. Tomahawk Series

**Design philosophy:** Maximum port density and throughput at the cost of feature depth. These chips use cut-through forwarding, carry shallow on-chip buffers (~50–100 MB), and support little to no L3 routing table depth. The trade-off is intentional; at spine and AI fabric scale, you want wire-rate forwarding with predictable low latency, not a large TCAM.

### Tomahawk 5 (TH5)
First chip in the family to reach 51.2 Tbps with 64 ports of 800GbE. Notable for introducing credit-based scheduling to manage incast congestion, a problem specific to AI training workloads where hundreds of GPUs complete a collective operation simultaneously and flood the network at the same instant.

### Tomahawk 6 (TH6)
Reached production volume in early 2026. Doubles capacity to 102.4 Tbps on a single die, using a 3nm process node.

- Supports 512 × 200GbE or 1,024 × 100GbE in breakout configurations.
- At this density, a two-tier (spine-leaf) fabric can interconnect ~128,000 GPUs without needing a third tier, which reduces total fiber count and hop count significantly. Whether that's worth the cost of deploying TH6-based spines depends heavily on cluster size.

---

## 2. Trident Series

**Design philosophy:** Feature richness over raw bandwidth. Trident chips carry larger TCAMs, richer ACL support, and more flexible telemetry pipelines. They're the right choice when you need per-flow visibility, complex policy enforcement, or protocol flexibility at the access or leaf layer.

### Trident 4 (TD4)
Introduced a programmable pipeline using NPL, a P4-like language that lets you modify forwarding behavior (add new encapsulations, custom telemetry headers, experimental protocols) without a hardware respin. This was a meaningful shift from fixed-function pipelines.

### Trident 5-X12 (16 Tbps)
- **NetGNT Engine:** An on-chip neural network inference block that classifies traffic and detects congestion patterns or anomalous flows (e.g., volumetric DDoS) at line rate. The inference runs in the datapath, not on a separate CPU, so it doesn't add latency.
- Supports 800G uplinks to spine, which allows a leaf to be physically connected to a TH5/TH6 spine without a speed mismatch at the uplink.

---

## 3. Jericho Series

**Design philosophy:** Deep buffers and large routing tables. Where Tomahawk sacrifices buffer depth for speed, Jericho inverts that. It's designed for environments where packets may need to queue for milliseconds (WAN congestion, DCI links with variable RTT) and where the FIB needs to hold full Internet routing tables.

### Jericho 3-AI
Targeted at lossless AI backend fabrics, the role that InfiniBand has traditionally filled. Provides scheduled fabric support to guarantee in order delivery for RDMA traffic, where a single dropped or reordered packet forces a full retransmit and stalls a GPU collective operation.

### Jericho 4 *(Sampling, targeting 2026 production)*
- **HBM Buffers:** Uses High Bandwidth Memory stacked on the package, the same DRAM technology used in GPUs, to provide much deeper buffers than SRAM based designs. This matters at DCI or internet edge roles where link utilization can spike and you need to absorb bursts without dropping.
- Targets Scale Across use cases: connecting geographically separated AI clusters (up to ~100 km) over a lossless routed fabric, where latency and reorder sensitivity of RDMA traffic otherwise makes Ethernet a poor fit.

---

## Comparison Table — Current Flagships

| Family | Latest Model | Throughput | Buffer Depth | Primary Use Case |
|--------|-------------|------------|--------------|-----------------|
| Tomahawk | Tomahawk 6 | 102.4 Tbps | Shallow (~50–100 MB SRAM) | Spine, AI fabric core |
| Trident | Trident 5-X12 | 16.0 Tbps | Moderate | ToR, enterprise leaf |
| Jericho | Jericho 4 | 51.2 Tbps | Deep (HBM) | Internet edge, DCI, lossless AI fabric |
