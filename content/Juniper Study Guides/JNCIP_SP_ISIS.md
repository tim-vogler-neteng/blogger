---
title: JNCIP-SP IS-IS
date: 2026-04-29
draft: true
---
Exam Objectives
	Describe the concepts, operation, or functionality of IS-IS
		IS-IS areas/levels and operations
			**Mesh Group** - When link-state PDUs are being flooded throughout an area, each router within a mesh group receives only a single copy of a link-state PDU instead of receiving one copy from each neighbor, thus *minimizing the overhead* associated with the flooding of link-state PDUs.
		Label-switched path (LSP) flooding through an IS-IS multi- area network
		Designated intermediate system (DIS) operation
		SPF algorithm
		Metrics, including wide metrics
		Route summarization and route leaking
		Given a scenario, demonstrate knowledge of how to configure or monitor single-area or multi-area IS-IS
	Implement IS-IS routing policy
Reviewing ISIS
	ES-IS = Level 0 
	IS-IS same area = Level 1 
	IS-IS Inter Area = Level 2 
	IS-IS(Exterior AS) = Level 3 
	Rules:
		Level 1 routers can only form adjacencies with other Level 1 routers in the *same* area
		Level 2 routers can only form adjacencies with other level 2 routers but area doesn't matter. 
		Each level has its own LSDB 
			Level 1 routes are transferred to the Level 2 DB
			Level 2 routes are not transferred to the level 1 DB
		Attached IS Router - Router connected to another area advertises itself as an Attached IS
			Other devices in the area create default routes pointed at the Attached IS
			Lot like a Total NSSA in OSPF 
	For Message types view [[]]
			