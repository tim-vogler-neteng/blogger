---
title: JNCIP-SP COS and Multicast
date: 2026-04-29
draft: true
---
Exam Objectives
	COS
	Describe the concepts, operation, or functionality of Junos OS CoS:
		CoS processing on Junos OS devices
		CoS header fields
		Forwarding classes
		Classification
		Packet loss priority
		Policers
		Schedulers
		Drop profiles
		Rewrite rules
	Given a scenario, demonstrate knowledge of how to configure or monitor CoS.
	Multicast
	Describe the concepts, operation, or functionality of IP multicast:
		Components of IP multicast, including multicast addressing
		IP multicast traffic flow
		Any-source multicast (ASM) versus source-specific multicast (SSM)
		Reverse path forwarding (RPF)—concept and operation
		Internet Group Management Protocol (IGMP)
			A protocol to manage multicast groups on the lan
			Four devices want to stream video FOO lets send it to a multicast group for those that want to listen vs 4 of the same identical stream
			Main Message types
			- **Membership Query:** Sent by the router to discover which multicast groups have members on the local network.
				- **General Query**:Asks if _any_ group has listeners.
				- **Group-Specific Query**: Sent after a "Leave" to check if _anyone else_ is still listening to that specific group.
			- **Membership Report (Join):** Sent by a host when it wants to join a group, or in response to a router's query to say, _"Yep, I'm still here."_
			- **Leave Group (IGMPv2/v3):** Sent by a host to proactively tell the router it's done with the stream.
			- 
		Protocol Independent Multicast (PIM) dense mode and sparse mode
		Rendezvous point (RP)—concept, operation, discovery, election
		Source-specific multicast (SSM)—requirements, benefits, address ranges
		Anycast rendezvous point (RP)
	Given a scenario, demonstrate knowledge of how to configure or monitor IGMP, PIM dense mode, or PIM sparse mode (including SSM):
	Implement IP multicast routing policy