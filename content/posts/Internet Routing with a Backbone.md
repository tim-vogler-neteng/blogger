---
title: "Post Title Here"
date: 2026-04-29
draft: true
---
Nearing midnight, oncall is paged; a customer is seeing a severe spike in latency between their cloud instances in Singapore and one of our regions in the Mid-West. Use atlas probes to test inbound from provider to our region and an mtr outbound to those IPs. Both directions are via a single provider and some loss is carying on from  Egress is via transit connections in Chicago and ingress is via connections into the backbone network in Los Angeles. Egress is easy*, raise local preference on different provider for the affected networks. (Local preference adjustment for an entire AS, especially a large cloud provider, is an awfully big hammer and shouldn't be done without understanding how much traffic will shift). With Ingress there is no simple knob to spin as you don't have access to the source or any of the transit devices it crosses. Pop quiz hot-shot! What do you do?

Well let's dig into it. 








TTL Expired


Most of the networks I've worked on have been island networks. I'm not sure if thats an official term or just something I've heard offhand. An island network is an autonomous system with many sites but no interconnection between them. All traffic is routed to the internet including traffic destined for other islands in the AS. This design has its pro's and cons. 

The Pro's: 
	Simpler routing. **Ding ding ding**
		Reason for this is any route adjustments are local to the island for just the networks originating from that particular island. 
		Provider A is suffering loss on traffic to one of the major cloud providers. Depref the inbound routes and prepend your advertisements. Now watch your traffic move away from the provider and hopefully more performant Provider B. 

The Con's 
	Less control over your routing. 
		Your traffic between sites is at the whim of the Internet Gods. 
	More pricey. 
		One 
