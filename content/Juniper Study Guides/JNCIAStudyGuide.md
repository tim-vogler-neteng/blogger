---
title: JNCIA Study Guide
date: 2026-03-17
weight: 1
tags:
  - juniper
  - jncia
  - networking
---

**Last Updated: 3/17/26**

**Table of Contents**

1. [**Networking Fundamentals**](#networking-fundamentals)  
2. [**Junos OS Fundamentals**](#junos-os-fundamentals)  
3. [**User Interfaces**](#user-interfaces)  
4. [**Configuration Basics**](#configuration-basics)  
5. [**Operational Monitoring and Maintenance**](#operational-monitoring-and-maintenance)  
6. [**Routing Fundamentals**](#routing-fundamentals)   
7. [**Routing Policy and Firewall Filters**](#routing-policy-and-firewall-filters)   
8. [**Glossary**](#glossary)

### **Networking Fundamentals**  {#networking-fundamentals}

* #### **Function of routers and switches** 

  * Routers use l3 information to forward packets between networks  
  * Switches use l2 info to forward packets on the lan 

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

[image1]: <data:image/png;base64,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>

[image2]: <data:image/png;base64,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>