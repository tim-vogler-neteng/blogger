---
title: JNCIP-SP ALL THE VPNS!
date: 2026-04-29
draft: true
tags:
  - Juniper
  - JNCIP-SP
---
Exam Objectives
Layer 3 VPNs
	Describe the concepts, operation, or functionality of Layer 3 VPNs:
		Traffic flow—control and data planes
		Full mesh versus hub-and-spoke topology
		VPN-IPv4 addressing
		Route distinguishers
		Route targets
		Route distribution
		Site of origin
		Sham links
		Virtual routing and forwarding (VRF) table-label
		Next-generation multicast virtual private networks (MVPNs)
		Flow of control and data traffic in a MVPN
		Layer 3 VPN scaling
		IPv6 Layer 3 VPNs
		Layer 3 VPN Internet access options
	Given a scenario, demonstrate knowledge of how to configure or monitor the components of Layer 3 VPNs.
	Describe Junos OS support for carrier-of-carriers or inter-provider VPN models.

Layer 2 VPNs
	Describe the concepts, operation, or functionality of BGP Layer 2 VPNs:
		Traffic flow—control and data planes
		Forwarding tables
		Connection mapping
		Layer 2 VPN network layer reachability information (NLRI)
		Route distinguishers
		Route targets
		Layer 2 VPN scaling
		Describe the concepts, operation, or functionality of LDP Layer 2 circuits:
	Traffic flow—control and data planes
		Virtual circuit label
		Autodiscovery (AD)
		Layer 2 interworking
	Describe the concepts, operation, or functionality of virtual private LAN service (VPLS)
		Traffic flow—control and data planes
		BGP VPLS label distribution
		LDP VPLS label distribution
		Route targets
		VPLS multihoming
		Site IDs
	Describe the concepts, operation, or functionality of EVPN:
		Lets dive into some EVPN!!!!!!!
		According to KNOX, why EVPN Wins 
			1. Operates at L2 or L3 VPN
			2. Why choose EVPN?
				1. Active-Active Load balancing 
					1. LACP without McLag
				2. MAC Learning = Reduces Bum traffic
				3. Layer2 & Layer3 VPNs can be simultaneuously be deployed
				4. Anycast Default gateways 
				5. Data
		1. Glossary
			1. EVI - EVPN Instance - Per customer collection of devices/interfaces participating in a shared EVPN configuration 
			2. ES - Ethernet Segment  The link or links connecting to a particular end device. The ES advertised across the EVI. 
			3. ESI - Ethernet Segment Identifier - Need to be identical for any links on a ES. 
			   Need to be unique in the EVI. 10 Bytes long 
				1. All zeros - single home segment 
				2. All Fs - also reserved. 
		2. NLRIs Overview - Network Layer Reachability Information
			1. Overview
				1. Type 1 
					1. Type 1A - Electing a designated forwarder. 
					2. Type 1B - Edge devices agree on a the same value for an ESI
				2. Type 2 - MAC address advertisements and the label to use to reach it 
				3. Type 3 - Broadcast traffic - advertise a label to use when sending a BUM traffic to this device. 
				4. Type 4 - During active-active advertise traffic with an additional bit saying Im not the designated forwarder. That way when the designated forwarder does receive it, it doesnt forward it back to the original source. 
				5. Type 5 - IP prefix advertisements 
			2. Type 2 NLRI deep dive 
				1. Advertises reachability info for a specific MAC addresses. 
				2. To do so it advertises
					1. Route Distinguisher
					2. ESI 
					3. VLAN ID  802.1Q 
					4. Learned MAC address
					5.  IP Address - Optional 
					6. Inner Label
					7. Next-hop IP addresses
					8. Communities - Route Target
			3. Type 3 NLRI deep dive
				1. PE devices advertise their eligibility to receive BUM traffic. 
					1. VLANs anybody? But only sending this traffic to devices that have hosts in the specific VLAN can also be accomplished with standard provider bridging. Lipstick on a pig 
				2. Format
					1. Route Distinguisher
					2. Vlan ID
					3. PMSI -Provider multicast servicing interface 
					4. Next-hop address loopback
					5. Communities - VRF Target. 
			4. Type 4 NLRI - Used in Active-Active setups
				1. Two devices advertise if they are multi-homing 
				2. Format
					1. Route Distinguisher
					2. ESI - Same for all PE routers performing the mulithoming
					3.  Loopback IP address of PE
					4. Communitiies - ES-Import Route Target
				3. Only the routers participating in the same ESI care about these advertisements, 
				4. One is elected Designated Forwarder and the other becomes Non Forwarder
			5. Type 1 NLRI
				1. Type 1a 
					1. Format
						1. Route distinguisher
						2. ESI
						3. VLan ID
						4. Inner Lable
						5. Loopback IP address of PE
						6. Communities
							1. Split Horizon bit - If youre in the same ESI as me and you recieve it do not forward it
				2. Aliasing - 
					1. Advertise an aliased inner label that all members of the ESI will accept and know what to do with. 
		3. Learning MAC address
			1. IRB - Internal Routing and Bridging interface. Vitrual interface on the PE with a configured IP adress
			2. Customer configures the IRB as their default gateway.
			3. When the customer sends an ARP for the default gateway the PE learns the mac address of the customer and advertises it all around. 
			4. Use the same default gateway on all sites to facilitate equipment re-location. No change needed  
				1. Dynamic DG Discovery or
				2. Better to just statically configure the same mac address for all gateways 
		Traffic flow—control and data planes
		Media access control (MAC) learning and distribution
		Ethernet VPN (EVPN) multihoming
		BGP EVPN label distribution
Given a scenario, demonstrate knowledge of how to configure, monitor, or troubleshoot Layer 2 VPNs:
	BGP Layer 2 VPNs
	LDP Layer 2 circuits
	EVPNs
	VPLS