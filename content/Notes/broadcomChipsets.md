---
title: Broadcom Switch Chipset Families
date: 2026-04-29
tags:
  - networking
  - hardware
  - broadcom
---

# Broadcom Switch Chipset Families

## 1. Tomahawk Series — *Max Bandwidth*

**Focus:** Maximum throughput, low latency, and high radix (port density). These are the "packet pushers" used in the Spine and Core of hyperscale data centers.

### Tomahawk 5 (TH5)
The first to hit **51.2 Tbps**. It introduced "Cognitive Routing" to handle the bursty nature of AI traffic and supports **64 ports of 800GbE**.

### Tomahawk 6 (TH6)
Hit production volume in March 2026. Doubles capacity to **102.4 Tbps on a single die**.

- **Key Stat:** Supports 512 ports of 200GbE or 1,024 ports of 100GbE.
- **AI Impact:** Enables a "flat" 128,000 GPU cluster using only two switch tiers, drastically reducing the number of optics and hops required.

---

## 2. Trident Series — *More Features*

**Focus:** Feature richness, deep programmability, and enterprise-grade flexibility. Typically found in Top-of-Rack and Leaf switches.

### Trident 4 (TD4)
Introduced a compiler-programmable architecture using NPL, allowing network engineers to add new protocols or telemetry features without changing hardware.

### Trident 5-X12 *(Current Flagship — 16 Tbps)*
- **NetGNT Engine:** An on-chip Neural Network inference engine that detects congestion patterns and security threats (e.g., DDoS) in real-time at line rate.
- **Connectivity:** Bridges the gap to the spine by supporting 800G uplinks

---

## 3. Jericho Series — *"The Distance & Depth Champion"*

**Focus:** Deep buffers and massive routing tables. Designed for Edge or DCI  routers where traffic may need buffering during congestion or must travel long distances.

### Jericho 3-AI
Designed to compete with InfiniBand in AI backends. Provides **scheduled fabrics** to ensure zero-packet-loss for **RDMA** workloads.

### Jericho 4 *(Sampling for 2026 production)*
- **Deep Buffers:** Uses **High Bandwidth Memory (HBM)** — the same technology found in GPUs — to buffer massive amounts of data.
- **DCI Champion:** Optimized for "Scale-Across," connecting AI clusters that are geographically separated (**up to 100 km**) while maintaining a lossless environment.

---

## Comparison Table — Current Flagships

| Family | Latest Model | Throughput | Primary Strength | Typical Deployment |
|--------|-------------|------------|-----------------|-------------------|
| Tomahawk | Tomahawk 6 | 102.4 Tbps | Raw Bandwidth / Density | Spine, AI Fabric Core |
| Trident | Trident 5-X12 | 16.0 Tbps | Programmability / ML Telemetry | ToR, Enterprise Leaf |
| Jericho | Jericho 4 | 51.2 Tbps | Deep Buffers / HBM / DCI | Internet Edge, DCI |
