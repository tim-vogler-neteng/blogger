---
title: JNCIA Reference Guide
date: 2026-07-03
weight: 1
tags:
  - juniper
  - jncia
  - networking
---

**Last Updated: 3/17/26**

**Note to Test Takers:** This document is a **Summary Reference**, not a replacement for a comprehensive course and hands on experience in a lab. I recommend the CBT Nugget course as Knox is great explaining networking concepts with right amount of enthusiasm. 

**Table of Contents**

1. [**Networking Fundamentals**](#networking-fundamentals)  
2. [**Junos OS Fundamentals**](#junos-os-fundamentals)  
3. [**User Interfaces**](#user-interfaces)  
4. [**Configuration Basics**](#configuration-basics)  
5. [**Operational Monitoring and Maintenance**](#operational-monitoring-and-maintenance)  
6. [**Routing Fundamentals**](#routing-fundamentals)   
7. [**Routing Policy and Firewall Filters**](#routing-policy-and-firewall-filters)   
8. [**Glossary**](#glossary)
9. [**Lab Recomendations**](#lab-recomendations)

### **Networking Fundamentals**  {#networking-fundamentals}

* #### **Function of routers and switches** 

  * Routers use L3 information to forward packets between networks  
  * Switches use L2 info to forward packets on the lan 

* #### **Ethernet networks** 

  * Major concept here is Mac addresses  
    * Physical address made up of 48 bits and displayed using hexadecimal format  
    * Broadcast address is ffff.ffff.ffff  
  * Uses mac addresses to forward ethernet frame  
  * Ethernet header \+ trailing checksum

  ![][image1]

    * Preamble \- Tells the receiving side that a frame is coming and allows synchronization   
    * SFD \- Start Frame Delimiter \- Signals the D-MAC is next   
    * Dest MAC \- MAC address of the frames destination  
    * SRC MAC \- MAC address of the frame sender  
    * Type \- Defines the type of protocol found inside the frame. IE v4 vs v6   
    * Data \+ padding \- The frame payload and optional padding to get it to a minimum of 46 bytes in this field.   
    * FCS \- Frame Check Sequence \- Contains a 32 bit CRC which checks for corrupted data

* #### **Layer 2 addressing, including address resolution Layer 3 / IP addressing including subnet masks** 

  * ARP \- Address resolution protocol acts at layer2 and is a process for mapping mac addresses to IP addresses. 

* #### **IPv4 Fundamentals** 

  * 32 bit addresses   
  * The data unit at layer 3 is called a packet  
  * Packet header: 

![][image2]

* #### **IPv6 Fundamentals** 

  * No broadcast traffic   
    * Anycast used instead  
  * Made up of 128 bits   
  * 8 groups of 4 hex characters   
    * For each group you can eliminate leading zeros You can also remove 0 groups that are in order one time using double colon   
    * 2001:0FA7:0000:0000:00E2:0000:0000:BEEF   
    * Becomes \- 2001:FA7::E2:0:0:BEEF   
  * Every interface requires a Link local address used for communications on the subnet that the host is connected to   
    * will not be forwarded by the router   
    * Not guaranteed to be unique  
      * DAD \- (Deduplicate Address Detection) \- check if its unique  
    * Assigned from fe80::/10 generally by stateless address autoconfiguration   
    * Takes the above prefix, adds some padding and the mac address to automatically configure the link local   
  * Routable addresses are assigned from 2000::/3  
  * Fragmentation only happens at the source node  
  * Header was designed to be simpler and easier to process. 

* #### **Subnetting and supernetting**

  * Subnetting is a skill that requires practice but does not require any special knowledge. Practice, practice, practice   
  * Supernetting  
    * Radix trees used to evaluate prefixes for route filters (follow up on this)

* #### **Longest match routing** 

  *  An algorithm used by IP routers to select an entry from a routing table. The router uses the longest match to determine the egress interface and the address of the next device to which to send a packet   
  * When routing to 192.168.1.10 and you have 192.168.1.0/28 and 192.168.1.0/24 in the routing table the router will use 192.168.1.0/28

* #### **CoS** 

  * Class of service allows you to divide traffic into classes and offer various levels of throughput and packet loss when congestion occurs. 

* #### **Connection-oriented vs. connectionless protocols**  

  * ##### **TCP \- Connection oriented**

    * Uses the three-way handshake to set up a session  
    * Syn, syn-ack, ack  
    * Receiving side responds with acks after receiving segments (AKA tcp data frame)   
      * Window size is beyond the scope of JNCIA  
    * Guarantees delivery

  * ##### **UDP \- Connectionless**

    * Ideal for real time communication and streaming media  
    * Fast which is what is needed  
    * If a packet is dropped it ends up being noise in the stream. IE degraded video for streaming but does not cause a failure 

### **Junos OS Fundamentals**  {#junos-os-fundamentals}

* #### **Software architecture**

  * Each process operates in its own protected memory space   
  * Two benefits of the disaggregated Junos OS   
    * Platform drivers and forwarding engine are removed from the control plane to increase performance   
    * The Architecture facilitates programmability through provisioning the control plane, the data path, and the platform APIs   
  * Junos release types  
    * R1 \- first widely distributed version  
    * R2, R3 \- maintenance releases  
  * Junos version breakdown M.nZb.s  
    * M \- major release  
    * n \- minor release  
    * Z \- type  
    * b \- build number   
    * s \- spin number  

* #### **Control and forwarding planes** 

  * There’s a rate limiter configured by default between the control and forwarding planes   
  * Control traffic is given higher priority than exception traffic if the link is congested   
  *  fxp1, em1, or similar (vs em0 and fxp0 which are oob mgmt)

* #### **Routing Engine** 

  *  Maintains Routing table   
  *  Maintains Forwarding table   
  *  Control/maintain chassis   
  *  Manages the PFE   
  *  Provides CLI or web interface 

* #### **Packet Forwarding Engine** 

  * Implement services   
    * Policing, Stateless FW filters, QOS   
  * Uses the L2 and L3 forwarding tables to pass traffic  
  * Transit traffic processing   
  * When a packet arrives and does not match an entry in the forwarding table the PFE drops the packet and sends a destination unreachable icmp reply 

* #### **Exception traffic** 

  * Destined for the local system   
  * Needs an icmp response 

### **User Interfaces**  {#user-interfaces}

* #### **CLI modes** 

  *  \> user mode   
  *  % shell mode   
    * reached by default when root user connects to the device   
    * *cli* brings the user back to the user mode   
  *  \# configure mode 

* #### **CLI navigation** 

  *  Ctrl \+ a – beginning of line   
  *  Ctrl \+ e – end of line   
  *  Ctrl \+ d – delete character under cursor   
  *  Ctrl \+ w – delete word left of cursor  
  *  Ctrl \+ k – delete everything right of cursor  
  * edit “level” \-\> to go to the specific spot of the config hierarchy  
  * up “number” \-\> to move up in the config hierarchy

* #### **CLI Help** 

  * *help topic interfaces address* \-\> Written documentation detailing how to configure interface addresses  
  * *help reference interfaces address* \-\> provides the syntax to configure this  
  * *help apropos snmp* \-\> all commands with “snmp” in them 

* #### **Filtering output** 

  * Useful Pipe commands  
    * | match  \-   show all lines of output with the given string  
    * | find      \-   start output at first instance of string and then everything               
                        afterwards  
    * | count	   \-   show how many lines in the given command  
      * *root@network-hub-1\> show interfaces terse | count*  
      * *Count: 49 lines*  
    * | last	   \-   display the last X amount of output  
    * | except \-   anything but the specified string  
* Active versus candidate configuration   
  * Active configuration is the config that the device is using   
  * Candidate config is where changes are made while in config mode  
    *  Once committed it becomes the active configuration  
* Reverting to previous configurations   
  * *root@network-hub-1\# rollback 1*   — Loads the previous active config into the candidate config   
* Modifying, managing, and saving configuration files Viewing, comparing, and loading configuration files   
  * *root@network-hub-1\# show | compare rollback 0*   
    * Compares current candidate config to current active config  
* J-Web (core/common functionality only)   
  * Same authentication as cli   
  * *set system services web management http* (or https this required for j-web to work)   
  * System identity sub page   
    * Configurable: Hostname, root password, dns servers, domain name 

### **Configuration Basics**  {#configuration-basics}

* #### **Factory-default state** 

  *  Can revert to the factory default with:   
    * *load factory-default*   
    * *set system root-authentication plain-text-password*   
    * *commit*   
* Initial configuration   
* User accounts   
  * Member of a single login class  
* Login classes   
  * A named container that groups together a set of one or more permission flags   
  * Four predefined classes  
    * Super-user \- all permissions   
    * operator \- clear, network, reset, trace, view   
    * Read-only \- view only   
    * unauthorized \- no permissions 

* #### **User authentication methods** 

  * Local database  
    * Name and password individually for each user  
  * Radius and TACACS+   
    * Can be mapped to locally defined template users  
    * Radius uses udp and encrypts the pass  
    * TACACS+ uses tcp and encrypts everything  
  * *show system authentication-order*  
    * Goes through the order trying one after the other even on rejects  
    * If local authentication is not in the authentication order it is only used if there was no response from the other options

* #### **Interface types and properties** 

  * fpc \- flexible pic concentrator  
  * pic \- port interface concentrator  
  * Port \#  
  * ge-{{fpc}}/{{pic}}/{{port\#}}  \- ge-0/0/0  
  * When multiple IPs are on an interface belonging to the same subnet you can use *preferred* to set the ip you want to respond for the interface 

* #### **Configuration groups** 

  * Use pipe command *| display inheritance* to show config with any config inherited from config groups included  
  * Allows you to separate common config from interface specific config   
* Additional initial configuration elements, such as NTP, SNMP, and syslog   
  * SNMP:   
    * MIB   
      * Used to define managed objects on a network device  
      * Designed in a hierarchical tree structure  
      * Standard or enterprise specific  
* Configuration archival 

* #### **Logging and tracing** 

  * By default messages are saved to /var/log including traceoptions

* #### **Rescue configuration** 

  * Recommended to contain the minimal configuration needed to allow connectivity  
  * save active config as rescue using:   
  * *request system configuration rescue save*   
  * *rollback rescue*   
  * *commit* 

* #### **Interface Configurations** 

  * All configuration directly under the ge-0/0/0 hierarchy is considered physical configuration. (MTU, lag interface, speed, duplex, encap, etc)  
  * All configuration under the unit \# is considered logical configuration  
    * family inet, family ethernet-switching, aka the protocol on the interface

### **Operational Monitoring and Maintenance**  {#operational-monitoring-and-maintenance}

* **Show commands**   
  * *show chassis routing-engine* \- shows the RE’s temp, cpu util, memory util, serial, and uptime   
  * *show system alarms*  \- displays current alarms   
  * *show interfaces {{ name }}*   
  * *show interfaces terse* \- shows interfaces and port status along with protocol and IP if it has one   
  * *show interfaces {{ name }} extensive*   
    * Shows errors, physical counters   
  * *Monitor interface {{ name }}*  
    * Shows realtime info on packet and byte counters  
    * Error and alarms   
  * *monitor interface traffic*  – to see traffic on all interfaces in real time   
* Interface statistics and errors 

* #### **Network tools, such as ping, traceroute, telnet, SSH, etc**

  * ping sends continues icmp messages to specified destination  
  * *monitor traffic*  \- captures traffic headed to the RE   
    * Can save these using write-file option and then open in wireshark  
  * traceroute \-  
    * Transmits UDP Packets  
    * Receives ICMP time-exceeded packets

* #### **Junos OS installation and upgrades** 

  * request system software add {{ image name }}   – to upgrade  
  * Need to reboot the device afterwards  
  * Unified in-service software upgrade ISSU  
    * Enables you to upgrade between two different Junos OS releases with no disruption on the control plane   
    * Only supported on dual RE platforms   
    * Require Nonstop active routing (NSR)  
      * Basically runs the routing daemons on the backup RE  
    * Step 1: Enable GRES and NSR and verify the re’s are synced  
    * Step 2: Download the image   
    * Step 3: *request system software in-service-upgrade* \- on the primary RE  
* Storage operations  
  * show system storage  – make sure there’s space for another image  
  * request system storage cleanup  
  * request system zeroize  \- clears config along with all logs   
    * Add media option to sanitize all storage on the device  
* Powering on and shutting down Junos devices   
* Root password recovery   
  * Requires a console connection  
  * Can be disabled with: *set system ports console insecure*  
  * Steps for password recovery  
    * Step 1: Reboot the system   
      * Press space bar when prompted   
      * Enter *boot \-s* to access single user mode   
    * Step 2: Enter recovery when prompted to go into recovery mode  
    * Step 3: Configure root password   
    * Step 4: *commit and-quit* and reboot when prompted

### **Routing Fundamentals**  {#routing-fundamentals}

* Traffic forwarding concepts 

* #### **Routing tables** 

  * Common tables  
    * inet.0 \- used for ipv4 unicast routes  
    * inet.1 \- used for multicast forwarding cache  
    * inet.4 \- used for Multicast BGP routes for rpf checking  
    * inet.3 \- used for mpls path info  
    * inet.4 \- used for MSDP route entries  
    * inet6.0 \- used for ipv6 unicast routes  
* Routing versus forwarding tables 

* #### **Route preference**

  * Juniper’s way of saying administrative distance   
  *  Used to differentiate routes learned from different protocols  
  * Values  
    * Direct	: 0   
    * Local	: 0  
    * Static	: 5  
    * Ospf	: 10   
    * RIP	: 100  
    * BGP	: 170  
* Routing instances \- A collection of routing tables, interfaces, and routing protocol parameters. The set of interfaces belongs to the routing tables, and the routing protocol parameters control the information in the routing tables.

* #### **Static routing** 

  * Need a valid next hop  
    * Ip address of interface on neighboring router  
    * Egress port   
    * bit bucket (reject/discard)  
  * Qualified next hop function  
    * If primary becomes unavailable use defined next hop with a higher preference value  
* Advantages of and use cases for dynamic routing protocols  
  * Less administrative overhead  
  * Dynamically route around failures

* #### **OSPF**

  * Link state protocol  
    * Faster reconvergence  
    * Support larger networks  
    * Less susceptible to insufficient routing info than distant vector  
  * Main objectives of a link state protocol  
    * Reliably flood link-state info to neighbors  
    * Create a complete database of the network  
    * Calculate the best path to each destination  
  * LSAs (Link State Advertisements)  
    *   
  * LSDB (Link-State Database)  
    * Stores the LSAs as a series of records  
  * Areas  
    * Uses Areas to incorporate hierarchy and enable scalability  
    * Software can summarize routing info from an OSPF area and pass it to the rest of the network   
    * Each OSPF router maintains a separate LSDB for each area its a part of  
      * LSDB is identical for all participating routers in an area  
    * All areas must connect to area 0   
    * All data traffic between areas, must transit the backbone area  
  * Neighbor Adjacency States  
    * Attempt  
    * Down  
    * Exchange  
    * ExStart  
    * Full \- up and running   
    * Init  
    * Loading   
    * 2-way  
  * Display Commands  
    * *show route protocol ospf* 

* #### **IPv6 routing**

  * Enabling on interface using family inet6  
    * Once enabled the link-local ip is configured and the interface will now process IPv6 traffic  
  * To configure a v6 static route  
    * *set routing-options rib inet6.0 static route 2001::0/20 next-hop 2001::1*  
    * Need to include the inet6.0 routing table for v6   
    * You can use a link local address as the next-hop but will need to include the interface as well   
  * Ospfv3 or Ospf for v6   
    * Fundamentally no different than vanilla ospf   
    * Config is the same syntax as vanilla ospf just being done under protocol ospf3 vs protocol ospf 

### **Routing Policy and Firewall Filters**  {#routing-policy-and-firewall-filters}

* #### **Default routing policies** 

  * ##### **OSPF**

    * Import \- Accept all OSPF routes and import into the inet.0 routing table  
    * Export \- Reject everything. (The protocol uses flooding to announce local routes and any learned routes.)

  * ##### **BGP**

    * Import \- Accept all received BGP IPv4 routes learned from configured neighbors and import into the inet.0 routing table. Accept all received BGP IPv6 routes learned from configured neighbors and import into the inet6.0 routing table.  
    * Export \- Readvertise all active BGP routes to all BGP speakers, while following protocol-specific rules that prohibit one IBGP speaker from readvertising routes learned from another IBGP speaker, unless it is functioning as a route reflector.  
* Import and export policies   
* Routing policy flow   
* Effect of policies on routes and routing tables   
* Policy structure and terms   
* Policy match criteria, match types, and actions 

* #### **Firewall filter concepts** 

  * aka acls  
  * Stateless filter  
    * Does not detect connections  
    * Looks at each packet  
    * Needs to be applied on egress and ingress   
  * Stateful   
    * Keeps state   
    * Only needs to permit traffic in one direction  
* Filter structure and terms   
  * Default action of firewall filters is discard  
  * The order of terms in a filter is important  
* Filter match criteria and actions   
  * Match Conditions for firewall filters  
    * Numeric range, Address, Bit-field   
  * Terminating Actions  
    * accept \- accept the packet and continue the in/out processing   
    * discard \- silently drop the packet without responding to source   
    * reject \- Causes the system to discard the packet and send an icmp message back to the source address  
  * Other Actions  
    * next term \- causes junos os to evaluate the next term  
      * Can be used to set a policer or dscp value and then continue to evaluate the traffic in the rest of the filter  
    * action modifiers \-  count, log, syslog, policer, forwarding-class  
* Firewall operational commands  
  * *show firewall counter filter {{filter\_name}} {{counter\_name}}*  
* Policing   
  * Rate limiting   
  * Work with firewall filters to stop ddos attacks

* #### **Unicast reverse-path-forwarding (RPF)**

  * Don’t run it on ports where you don’t need it because it eats up control plane resources  
  * Strict mode \- The packet is not accepted when either:  
    * The packet has a source address that does not match a prefix in the routing table.  
    * The interface does not expect to receive a packet with this source address prefix.  
  * Loose mode \- The packet is not accepted when the packet has a source address that does not match a prefix in the routing table.

### **Glossary** {#glossary}

* #### **Junos Architecture & Operations**

  * **RE (Routing Engine):** The "brain" of the device. It handles the control plane, running the Junos OS, managing routing tables, and controlling the user interface (CLI).  
  * **PFE (Packet Forwarding Engine):** The "brawn" of the device. It handles the data plane, performing hardware-based packet switching, filtering, and queuing.  
  * **Transit Traffic:** Traffic that enters one port and exits another. This is handled entirely by the PFE.  
  * **Exception Traffic:** Traffic destined for the device itself (e.g., SSH, BGP updates, or ICMP). This is passed from the PFE to the RE for processing.  
  * **Active vs. Candidate Configuration:** Junos uses a "check-out" system. You edit a **Candidate** config; it does not take effect until you **Commit** it to become the **Active** config.

* #### **Interface Terminology**

  * **FPC (Flexible PIC Concentrator):** A physical slot or card in a chassis (e.g., `ge-0/x/x`).  
  * **MAC:** Typically referring to the mac address, the layer 2 address used to for frames in a broadcast domain.   
  * **Logical Unit:** In Junos, you must define a logical unit (usually `unit 0`) even for physical interfaces to assign an IP address.  
  * **Family:** Defines the protocol stack on an interface. Common types include `inet` (IPv4), `inet6` (IPv6), and `mpls`.

* #### **Routing & Protocols**

  * **RT (Routing Table):** The master list of all paths learned by the RE (e.g., `inet.0` for IPv4).  
  * **FT (Forwarding Table):** A streamlined version of the RT sent down to the PFE for high-speed lookups.  
  * **Route Preference:** Juniper's term for Administrative Distance. Lower numbers are more preferred (e.g., Direct \= 0, OSPF \= 10, BGP \= 170).  
  * **Routing Policy:** Used to control the flow of routing information into or out of the Routing Table.
* 
### **Lab Recomendation**s {#lab-recomendations}
* I can't stress the importance enough of having a lab environment to run through these different concepts. Yes you can memorize enough material to pass the exam but then its just a piece of paper. Or, these days, just a digital stamp you can add to your LinkedIn profile. 
* The most important aspect is when you try to build a topology and it **doesn't** work. Then you're pushing show commands and pouring through the config to find the one little thing you missed. And that there is Network Engineering in a nutshell. 
* My gear
	* When I first passed the JNCIA ages ago I ran a couple of switches and an ancient J series firewall. It got the job done but was far from ideal, takes up too much space, and is far too loud. 
	* Now I run an EVE-NG bare metal installation on a Dell Precision 5810 I bought off Craigslist for ~$150. Started with 56GB of ram and thats more than plenty for the JNCIA. 
	* With that said, getting access to https://jlabs.juniper.net/vlabs/ is your best bet. There's a ton of great labs already laid out and is a huge resource for learning how things work. 



Good Luck!
  
  
  
  
[image1]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAACjwAAADwCAYAAAC+eV85AABjkElEQVR4XuzdwatsXXof5m+Q/0BI7aEHHghEe2awgghtywhHEw006DgkkDbEGkh4IERAg0ioM9NEljwIoeMEIVsDDYQt49BqE+LG6o4JiJY8a6lBxrLcYGTsBuMoceBGvyuWvPTed+29q+rcXafqPD94+L57atXeu3ade+uttd7a9cknIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIi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>

[image2]: <data:image/png;base64,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>