# Enterprise IPv6 Network Topology & Infrastructure Design

Welcome to the documentation for my enterprise IPv6 network design project! This repository contains the complete GNS3 topology, configuration files, and verification tests for an end-to-end IPv6 infrastructure featuring high availability, dynamic routing, and automated host addressing.

## 📌 Project Overview & Rationale

Modern enterprise environments are rapidly transitioning toward IPv6-only or dual-stack architectures to overcome IPv4 address exhaustion and streamline end-to-end connectivity. 

The primary goal of this project was to construct a resilient, redundant, and scalable IPv6 Enterprise Network. The design focuses on four main pillars:
1. Dynamic Addressing Flexibility: Combining both DHCPv6 (Stateful/Relay) and SLAAC (Stateless) to simulate real-world host provisioning scenarios.
2. First-Hop & Link Redundancy: Utilizing HSRPv2 and LACP (EtherChannel) at the distribution/access layers to eliminate single points of failure.
3. Core Mesh Routing: Running OSPFv3 across a multi-path core topology for optimal path selection and rapid fault recovery.
4. Subnetting Precision: Structuring clean /64 subnets (2001:DB8::/32 space) to enforce standard IPv6 aggregation and NDP capabilities.

---

##  Topology Architecture & Breakdown

Below is the complete network topology diagram:

<img width="1221" height="582" alt="image" src="https://github.com/user-attachments/assets/1f0e1f36-64e3-4b2d-acd1-107ed2af08c2" />


### 1. Host Addressing Layer
* VLAN 10 (DHCPv6 Domain): 2001:DB8:1:100::/64  
  Uses DHCPv6 Stateful allocation. A Cisco Router configured as PC1 is utilized to act as a full DHCPv6 client. MUL1 acts as the DHCPv6 Relay Agent to forward requests across the core to the central server pool (2001:DB8:3:100::/64).
* VLAN 20 (SLAAC Domain): 2001:DB8:2:200::/64  
  Hosts auto-configure their IPv6 address and default gateway via Router Advertisements (RA) and Neighbor Discovery Protocol (NDP).

### 2. High Availability & Access Integration
* LACP (Link Aggregation): Configured between MUL2 and MUL3 to aggregate link bandwidth and provide Layer 2 redundancy.
* HSRPv2 for IPv6: Configured on the sub-interfaces/VLAN interfaces of MUL2 and MUL3 using virtual Link-Local addresses (FE80::) to ensure uninterrupted default gateway availability for endpoints in VLAN 20.

### 3. Core Mesh & Dynamic Routing
* OSPFv3 Core (R1-R6, MUL1-MUL4):  
  All core links run OSPFv3 Area 0. Link-Local addresses (FE80::) are leveraged for neighbor adjacencies. Multi-path topology ensures sub-second convergence if any single transit link fails.

---

##  Protocols Implemented

* IPv6: Base network layer protocols across all devices and hosts.
* OSPFv3: Core interior gateway routing protocol for IPv6 reachability.
* HSRPv2: Gateway redundancy for client subnets.
* LACP (802.3ad): Link bundling and trunking redundancy between distribution nodes.
* DHCPv6 Server & Relay: Stateful IPv6 allocation and relay agent integration.
* SLAAC: Stateless auto-configuration using ICMPv6 Router Advertisements.
* NDP (ICMPv6): Replaces IPv4 ARP for neighbor resolution and router discovery.
* Inter-VLAN Routing: Communication between isolated VLAN segments over trunk links.

---

##  Verification & Proof of Concept

To validate the deployment, several verification steps and operational tests were performed.

### 1. Core Routing Adjacency (OSPFv3)
Verifying that all routers in the core mesh have formed full adjacencies and built complete Link-State Databases (LSDB).

Command:
R4#show ipv6 ospf neighbor

<img width="1371" height="352" alt="image" src="https://github.com/user-attachments/assets/3b70bfa9-b76a-4b3b-b112-443b47c2fad8" />

---

### 2. DHCPv6 Stateful Allocation & Relay Test
Checking that PC1 (Router operating as IPv6 host) successfully requests and obtains an IPv6 address from the central DHCPv6 pool via the MUL1 relay agent.

Commands:
PC1# show ipv6 interface brief
PC1# show ipv6 dhcp interface

<img width="1102" height="835" alt="image" src="https://github.com/user-attachments/assets/4223dd26-2a08-418d-b011-b3b2b3fbb592" />


---

### 3. SLAAC Address Generation & Default Gateway
Verifying that VPCS hosts in VLAN 20 (PC2-PC5) auto-assign their global unicast addresses based on the prefix advertised by the gateway.

Command:
PC3> show ipv6

<img width="956" height="561" alt="image" src="https://github.com/user-attachments/assets/65c6ddc3-c77c-48f0-b493-9ee368b21fba" />


---

### 4. End-to-End Reachability (PC to Server Ping & Traceroute)
Testing full end-to-end connectivity across the core network from a client in VLAN 20 to Server 2 (2001:DB8:3:100::11).

Commands:
PC5> ping 2001:DB8:3:100::11
PC5> trace 2001:DB8:3:100::11

<img width="1002" height="617" alt="image" src="https://github.com/user-attachments/assets/7a582fe4-b365-4ba9-9112-bed18f3e9de2" />

### 5. First-Hop Redundancy Verification (HSRPv2 for IPv6)
To validate High Availability (HA) and default gateway redundancy for endpoints in VLAN 20, HSRPv2 state verification and a simulated failover test were conducted.

#### A. Active/Standby State Verification
Verifying that MUL2 is properly elected as the **Active** gateway and MUL3 as the **Standby** gateway, sharing the Virtual Link-Local IPv6 Address (`FE80::20`).

Commands:
MUL2# show standby brief

<img width="1220" height="207" alt="image" src="https://github.com/user-attachments/assets/47263496-1699-4764-b911-325b19919345" />


MUL3# show standby brief

<img width="1212" height="213" alt="image" src="https://github.com/user-attachments/assets/2b059099-88d4-4993-b522-10c33e04c34a" />


---

##  Troubleshooting Scenario Solved

During deployment, a key issue was identified and resolved regarding host IPv6 address allocation:

### Issue: VPCS Host Using SLAAC Instead of DHCPv6
* Symptom: Host PC1 in VLAN 10 was taking a SLAAC address instead of obtaining an IPv6 address from the DHCPv6 server, and it was unclear why.
* Root Cause: Upon checking the host version, it was discovered that VPCS nodes in GNS3 only support SLAAC auto-configuration natively and lack commands/support to pull stateful DHCPv6 addresses.
* Fix: Added a Cisco Router to VLAN 10 to act as PC1, configured it as an IPv6 host, and enabled DHCPv6 client functionality on its interface. This successfully allowed PC1 to obtain an IPv6 address via the DHCPv6 Server and Relay Agent.
