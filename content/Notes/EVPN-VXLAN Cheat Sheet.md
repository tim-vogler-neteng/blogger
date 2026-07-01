---
title: Post Title Here
date: 2026-06-07
draft: true
---

I've been dragged kicking and screaming into the world of EVPN-VXLAN. So, I'm begrudgingly writing some stuff down because it helps me process and remember it. 

EVPN NLRI is a term used when describing the specific format of the MP-BGP NLRIs that carry EVPN information. 

EVPN ROUTE TYPEs

| Route Type | Name                | Use                                                                  |
| ---------- | ------------------- | -------------------------------------------------------------------- |
| 1          | Ethernet AD per ESI | Mac mass withdraws<br>Split-horizon using IP@<br>Split-horizon label |
| 2          | Ethernet AD per EVI | For load balancing                                                   |
| 3          | MAC route           | Advertising a MAC address via BGP                                    |
| 4          |                     |                                                                      |
| 5          |                     |                                                                      |
| 6          |                     |                                                                      |
| 7          |                     |                                                                      |
| 8          |                     |                                                                      |
