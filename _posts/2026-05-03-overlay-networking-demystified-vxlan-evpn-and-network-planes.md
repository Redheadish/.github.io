---
layout: post
title: "Overlay Networking Demystified: VXLAN, EVPN and Network Planes"
categories: [networking]
tags: [vxlan, evpn, bgp, networking, data-plane, control-plane, overlay]
lang: en
---

Modern networking often becomes confusing because several ideas are mixed together: device architecture, forwarding tables, tunnelling, encapsulation, underlay networks, overlay networks, and control-plane protocols.

A common shortcut is:

```text
VXLAN = data plane
```

This is useful, but not fully precise.

A better explanation is:

> **VXLAN is a data-plane encapsulation and forwarding mechanism. EVPN is commonly used as the control plane for scalable VXLAN overlays.**

VXLAN defines how Ethernet frames are encapsulated and forwarded over an IP underlay. EVPN commonly distributes overlay reachability information: MAC addresses, IP addresses, VTEP locations, VNI membership, and IP prefixes.

## 1. The Three Network Planes

A network device can be understood as a system with three logical planes: the control plane, the data plane, and the management plane.

### Control Plane

The control plane builds network knowledge. It decides what routes exist, which paths are valid, which MAC addresses are known, and how the topology should work.

Examples include OSPF, BGP, STP, ARP/ND processing, LISP mapping, and the logic that builds routing and forwarding tables.

### Data Plane

The data plane forwards real traffic. It uses existing tables and applies forwarding decisions at high speed.

This is where Layer 2/Layer 3 forwarding, ACL enforcement, QoS, VXLAN encapsulation, and VXLAN decapsulation usually happen.

### Management Plane

The management plane allows humans or systems to manage the device. This includes SSH, CLI, Web UI, SNMP, Syslog, NetFlow/IPFIX, NETCONF, RESTCONF, and APIs.

```text
Control plane  -> builds knowledge and tables
Data plane     -> uses tables and forwards packets
Management     -> provides access, monitoring and configuration
```

This separation is logical, not always perfectly physical. In real devices, some functions overlap. MAC learning may happen in hardware, EVPN may populate MAC tables through BGP, and ACLs may be configured through management or control logic but enforced in the data plane.

## 2. Tables Used by Switches and Routers

A switch or router does not analyse every packet from zero. It builds tables in advance and then checks incoming traffic against them.

A MAC/CAM table maps a MAC address and VLAN to a physical port:

```text
MAC + VLAN -> port
```

This is used for Layer 2 switching.

A RIB, or Routing Information Base, is the logical routing table built from routing protocols and static routes:

```text
IP prefix -> candidate route
```

A FIB, or Forwarding Information Base, is the optimised forwarding table used by the data plane:

```text
destination prefix -> next hop / outgoing interface / action
```

It is more accurate to say “destination prefix” than just “IP address”, because routing works with prefixes such as `10.10.20.0/24`.

An ARP table maps IPv4 addresses to MAC addresses in a local Layer 2 segment. IPv6 uses Neighbor Discovery instead.

An ACL/TCAM entry defines conditions and actions:

```text
condition -> permit / deny / action
```

On many switches, these rules are applied in hardware using TCAM.

## 3. CPU, ASIC and FPGA

The CPU is flexible. It can run routing protocols, management services, BGP sessions, SSH, SNMP, calculations, and system processes.

However, forwarding every packet through the CPU would be too slow for high-performance networking. That is why modern switches and routers use specialised forwarding hardware.

An ASIC is designed for very fast and efficient packet forwarding. It is commonly used for switching, routing, ACLs, QoS, ECMP, and VXLAN offload.

An FPGA is also hardware-based, but it can be reprogrammed. It is useful for specialised processing and prototyping, although it is usually more complex and less efficient than a fixed ASIC.

A simplified data-plane pipeline looks like this:

```text
Ingress -> Parsing -> Lookup -> Decision/Action -> Egress
```

For example:

- A frame enters a port.
- The ASIC parses Ethernet, IP, TCP/UDP, and VXLAN headers.
- The ASIC checks MAC, FIB, and ACL tables.
- The ASIC decides whether to forward, drop, rewrite, encapsulate, or decapsulate.
- The packet leaves through the egress port.

## 4. Mapping Protocols to Planes

Different protocols belong to different planes, although some have mixed roles.

OSPF, BGP, and STP are mainly control-plane protocols. They build routing information, topology knowledge, or loop-free Layer 2 logic.

SNMP belongs to the management plane. It is used for monitoring and sometimes configuration, but it does not forward user traffic.

VXLAN is a data-plane encapsulation mechanism. It defines how Ethernet frames are wrapped inside UDP/IP packets.

EVPN is commonly used as a control-plane solution for VXLAN overlay networks. It uses MP-BGP to tell VTEPs where MAC addresses, IP addresses, VNI membership, VTEP reachability, and prefixes are located.

LISP has both control-plane and data-plane parts. It has mapping logic, such as EID to RLOC, and also encapsulation.

## 5. From GRE to VXLAN and Geneve

VXLAN is easier to understand as part of the evolution of overlay tunnelling.

The basic idea is simple: if the transport network should not directly carry a certain logical network, we can wrap one packet inside another and carry it across the transport network.

- **GRE** introduced the general idea of tunnelling: one packet can be carried inside another IP packet.
- **NVGRE** adapted this idea for network virtualisation in multi-tenant data centres.
- **VXLAN** became a dominant approach for Layer 2 overlays in data centres.
- **Geneve** is a newer and more flexible encapsulation format with extensible metadata options.

The main idea is always the same:

```text
network over network
```

## 6. What VXLAN Does

VXLAN is often described as “Ethernet over IP”, but more precisely it encapsulates Ethernet frames inside UDP/IP so that they can travel across a routed IP network.

The encapsulation looks like this:

```text
Original Ethernet frame
        ↓
VXLAN header with VNI
        ↓
UDP header, destination port 4789
        ↓
Outer IP header: source VTEP -> destination VTEP
        ↓
Outer Ethernet header for the underlay link
```

To the underlay network, this is just UDP/IP traffic between VTEPs.

To the overlay network, it looks like an extended Layer 2 segment.

A VTEP, or VXLAN Tunnel Endpoint, is the device that performs VXLAN encapsulation and decapsulation. It can be a leaf switch, hypervisor, or software switch.

A VNI, or VXLAN Network Identifier, identifies the virtual Layer 2 segment. It is 24 bits long, which gives about 16 million possible VXLAN segments.

The underlay is the physical or routed IP network below.

The overlay is the virtual network built on top of it.

BUM traffic means Broadcast, Unknown unicast, and Multicast. This traffic usually requires replication or special handling, because the destination is not always known as a single unicast endpoint.

## 7. Underlay and Overlay

The key VXLAN architecture idea is this:

```text
The underlay should be a simple routed IP fabric.
The overlay carries the virtual Layer 2 or Layer 3 networks.
```

The underlay does not need to know the MAC addresses of every virtual machine. Its job is to provide IP reachability between VTEPs.

Example:

```text
Underlay:
Leaf-1 ---- Spine(s) ---- Leaf-2
IP routing, loopbacks, ECMP, BGP/OSPF/ISIS

Overlay:
VNI 100 = virtual Layer 2 segment
Server A and Server B believe they are in the same L2 network
```

If Server A sends an ARP broadcast, the ingress VTEP encapsulates that Ethernet frame into VXLAN and sends it to other VTEPs in the same VNI. The remote VTEP removes the outer headers and releases the original Ethernet frame into the local segment.

## 8. VLAN and VNI

VLAN and VNI look similar, but they are not the same thing.

| Feature | VLAN | VNI |
|---|---|---|
| Size | 12 bits, about 4096 VLANs | 24 bits, about 16 million VXLAN segments |
| Scope | Usually local to a Layer 2 domain or trunk | Overlay-wide between VTEPs |
| Purpose | Layer 2 segmentation | Overlay segmentation |

A local VLAN can be mapped to a VNI.

Example:

```text
Leaf-1: VLAN 100 -> VNI 5000
Leaf-2: VLAN 300 -> VNI 5000
```

The local VLANs are different, but the overlay segment is the same because both are mapped to VNI 5000.

This is one of the reasons VXLAN is useful in data centres: it separates local VLAN numbering from overlay segmentation.

## 9. Static VXLAN

Static VXLAN is the simplest lab mode.

The engineer manually configures remote VTEP addresses. When BUM traffic appears, the ingress VTEP creates multiple VXLAN copies and sends them as unicast packets to all configured remote VTEPs.

This is called ingress replication or head-end replication.

The advantage is simplicity. It is easy to understand, useful in small labs, and does not require multicast in the underlay.

The disadvantage is scalability. Every new VTEP must be configured on other VTEPs. BUM traffic is replicated by the ingress VTEP, and MAC learning remains mostly flood-and-learn.

Static VXLAN is good for learning. It is not ideal for large modern data-centre fabrics.

## 10. Multicast VXLAN

In multicast VXLAN, each VNI is mapped to a multicast group.

VTEPs that belong to that VNI join the multicast group. When BUM traffic appears, the ingress VTEP sends one VXLAN packet to the multicast address, and the multicast-capable underlay replicates it to the correct VTEPs.

This removes the need to manually configure all remote VTEPs in a flood list.

However, it requires multicast in the underlay, usually with protocols and logic such as PIM, RP, or Anycast RP. This can make operations more complex.

Multicast VXLAN also explains why VXLAN uses UDP rather than TCP: some traffic patterns are one-to-many.

In many modern data centres, EVPN is preferred instead of relying on underlay multicast.

## 11. EVPN/VXLAN

EVPN/VXLAN is the most important modern design.

```text
VXLAN = how to encapsulate and forward traffic
EVPN  = how VTEPs learn who is where
BGP   = transport for EVPN information
```

With EVPN, VTEPs can exchange information about MAC addresses, IP addresses, VTEP locations, VNI membership, and IP prefixes.

This means the network does not need to rely only on flooding or static configuration.

## 12. EVPN Route Types

EVPN uses different route types for different purposes.

### Route Type 1 — Ethernet Auto-Discovery

Used for multihoming, fast convergence, aliasing, and Ethernet Segment logic.

### Route Type 2 — MAC/IP Advertisement

Advertises that a MAC address, and optionally an IP address, is reachable behind a specific VTEP.

### Route Type 3 — Inclusive Multicast Ethernet Tag

Advertises VTEP participation in a VNI and helps build flood lists for BUM traffic.

### Route Type 4 — Ethernet Segment

Used for Ethernet Segment discovery and Designated Forwarder election in multihoming.

### Route Type 5 — IP Prefix

Advertises IP prefixes through EVPN. This is important for Layer 3 EVPN/VXLAN, VRFs, IRB, and inter-subnet routing.

Route Type 5 is not about MAC addresses. It is about IP prefixes.

## 13. What EVPN Improves

EVPN improves VXLAN in several important ways.

- It provides automatic VTEP discovery instead of manually configured flood lists.
- It distributes MAC/IP reachability through BGP instead of relying only on flood-and-learn.
- It reduces unknown-unicast flooding because remote MAC information can be known in advance.
- It supports multihoming, fast convergence, mass withdrawal, and aliasing.
- It supports anycast gateway, where the same default gateway can exist on multiple leaf switches.
- It supports Layer 3 EVPN through Route Type 5 and IP VRFs.

However, EVPN does not remove all flooding. Broadcast and some initial discovery traffic still exist. EVPN reduces and controls flooding, but it does not magically remove all Ethernet behaviour.

## 14. Open vSwitch and VXLAN

Open vSwitch is useful for VXLAN labs because it is a software switch that can perform VXLAN framing and act as a dataplane.

However, OVS by itself is not a full EVPN control plane.

For EVPN scenarios, an external BGP/EVPN component or controller is usually needed, such as FRR, GoBGP, OVN, or OpenStack Neutron.

A useful nuance: OVS supports VXLAN packet framing, but not the multicast aspects of VXLAN. A workaround is to pre-provision MAC-to-IP mappings manually or through a controller.

## 15. Final Summary

VXLAN is an overlay technology that encapsulates Ethernet frames inside UDP/IP. It allows virtual Layer 2 networks to run over a routed IP underlay.

A VTEP performs encapsulation and decapsulation. A VNI identifies the virtual segment. The underlay provides IP reachability between VTEPs. The overlay provides the virtual network seen by hosts or virtual machines.

Static VXLAN uses manually configured VTEPs.

Multicast VXLAN uses multicast groups for BUM replication.

EVPN/VXLAN uses BGP EVPN as a common control plane to distribute MAC, IP, VTEP, VNI membership, and IP-prefix information.

The most important idea is:

```text
VXLAN = data-plane encapsulation and forwarding
EVPN  = commonly used control-plane learning/signalling for scalable VXLAN overlays
BGP   = protocol that carries EVPN routes
```

So the phrase “VXLAN = data plane” is useful, but incomplete.

A more accurate statement is:

> **VXLAN defines how Ethernet frames are encapsulated and forwarded over an IP underlay. EVPN is commonly used as the control plane that tells VTEPs where MAC addresses, IP addresses, VNI membership, VTEP reachability, and prefixes are located.**
