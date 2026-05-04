---
layout: post
title: "Overlay Networking Demystified: VXLAN, EVPN, and Network Planes"
categories: [networking]
tags: [networks, vxlan, evpn, bgp, mp-bgp, overlay, underlay, data-plane, control-plane, evpn-vxlan, data-center]
lang: en
---

Modern networking often appears complex because several different concepts are mixed together: forwarding, routing, encapsulation, control-plane protocols, virtualisation, address planning, and operational design.

A common shortcut is:

```text
VXLAN = data plane
```

VXLAN should be understood more precisely as a **data-plane encapsulation mechanism**. It defines how Ethernet frames are carried over an IP network. However, VXLAN alone does not define how endpoints are discovered, how remote MAC addresses are learned, how IP-to-MAC information is distributed, or how forwarding state is exchanged across a large fabric.

A better mental model is:

```text
VXLAN = overlay encapsulation over UDP/IP transport
EVPN  = reachability learning and signalling
BGP   = distribution mechanism for EVPN routes
```

> **VXLAN defines how traffic is carried. EVPN defines where it should go. BGP distributes that knowledge across the fabric.**

This article explains VXLAN/EVPN from the ground up: network planes, forwarding tables, underlay/overlay design, VXLAN encapsulation, MTU, EVPN route types, MP-BGP, packet flow, IRB, anycast gateway, ARP suppression, route reflectors, Open vSwitch labs, troubleshooting, failure scenarios, and practical configuration structure.

---

## Reading Map

If you are new to VXLAN/EVPN, read Sections 1–14 first.

If you are mainly interested in packet flow, read Sections 18–22.

If you are preparing for troubleshooting, focus on Sections 31–32.

If you want practical deployment structure, read Sections 34–36.

---

## Summary: The Core Mental Model

VXLAN acts as a data-plane encapsulation mechanism, carrying Ethernet frames within UDP/IP for flexible Layer 2 connectivity over routed IP networks. EVPN provides the control-plane intelligence, typically using MP-BGP with the `l2vpn evpn` address family to distribute MAC, IP, VTEP, VNI, Ethernet Segment, and IP-prefix reachability information.

Together, VXLAN and EVPN create a scalable architecture that separates the physical underlay from the logical overlay. This supports modern data-centre workloads, workload mobility, multi-tenancy, distributed routing, and more controlled address learning than traditional flood-and-learn Layer 2 designs.

Two practical points are especially important in real deployments:

```text
MTU must be planned before VXLAN works reliably.
Distributed Anycast Gateway is what makes local routing and workload mobility practical.
```

Because VXLAN adds extra headers, the underlay MTU must be carefully planned to avoid fragmentation or packet loss. For a 1500-byte inner frame with IPv4-based VXLAN encapsulation, this commonly means at least about 1550 bytes of underlay MTU, with many data-centre fabrics using 1600, 9000, or another consistent jumbo MTU value.

Without correct MTU, the overlay may form but traffic can fail. Without distributed anycast gateway design, VXLAN may provide Layer 2 extension, but it will not deliver the full operational benefit of a modern EVPN fabric.

---

## Who This Article Is For

This article is intended for:

- learners who already understand basic routing, switching, VLANs, and IP addressing;
- junior network engineers trying to understand data-centre overlays;
- system, cloud, and infrastructure engineers who need a practical mental model of VXLAN/EVPN;
- anyone who has heard “VXLAN is the data plane” and wants to understand what that really means.

The goal is not to memorise vendor syntax first. The goal is to understand the architecture.

---

## 1. The Problem VXLAN/EVPN Solves

Traditional Layer 2 networks have several limitations:

- VLAN space is limited to 4096 possible VLAN IDs, with fewer usable in practice because some values are reserved.
- Large broadcast domains are difficult to control.
- Spanning Tree Protocol may block links to prevent loops.
- Multi-tenant segmentation becomes difficult at scale.
- Workload mobility can be operationally painful.
- Extending Layer 2 across a data-centre fabric creates complexity.
- Troubleshooting becomes harder when topology, addressing, and forwarding state are not clearly separated.

VXLAN/EVPN addresses these problems by separating the physical transport network from the logical tenant network.

The modern model is:

```text
Underlay = routed IP fabric
Overlay  = logical networks carried over the underlay
```

The underlay provides IP reachability between VTEPs.

The overlay provides virtual Layer 2 and Layer 3 services.

This separation is one of the most important ideas in modern data-centre networking.

---

## 2. Network Planes: Control, Data, and Management

A network device can be understood through three logical planes:

```text
Control plane  -> builds knowledge
Data plane     -> forwards traffic
Management     -> configuration and visibility
```

### Control Plane

The control plane builds network knowledge. It decides which routes exist, which paths are valid, and how reachability information is distributed.

Examples:

- OSPF
- IS-IS
- BGP
- MP-BGP
- EVPN
- STP
- ARP/ND control logic

In VXLAN/EVPN, the control plane tells devices where remote MAC addresses, IP addresses, VNIs, Ethernet Segments, and prefixes are located.

### Data Plane

The data plane forwards actual traffic. It uses precomputed tables and applies forwarding decisions at high speed, often in ASIC hardware on modern switches.

Examples:

- Layer 2 switching
- Layer 3 routing
- ACL enforcement
- QoS
- VXLAN encapsulation
- VXLAN decapsulation

VXLAN belongs here as an encapsulation and forwarding mechanism.

### Management Plane

The management plane allows humans and systems to configure, monitor, and automate the device.

Examples:

- SSH
- CLI
- SNMP
- Syslog
- NETCONF
- RESTCONF
- APIs
- streaming telemetry

In VXLAN/EVPN fabrics, a useful practical distinction is:

```text
EVPN builds and distributes forwarding knowledge.
VXLAN uses that knowledge to encapsulate and forward traffic.
```

---

## 3. Forwarding Tables

Network devices do not analyse every packet from zero. They use prebuilt tables.

| Table | Purpose | Simplified meaning |
|---|---|---|
| MAC/CAM table | Layer 2 forwarding | `MAC + VLAN -> port` |
| RIB | Logical routing table | `IP prefix -> candidate route` |
| FIB | Data-plane forwarding table | `destination prefix -> next hop / interface` |
| ARP / ND | IP-to-MAC mapping | `IP address -> MAC address` |
| TCAM | Hardware match/action rules | `condition -> action` |

It is more accurate to say “destination prefix” than simply “IP address”, because routing works with prefixes such as:

```text
10.10.20.0/24 -> next hop
```

In VXLAN/EVPN, additional overlay-related forwarding entries appear:

```text
Remote MAC/IP -> remote VTEP
VNI -> bridge domain / tenant segment
VRF -> routing context
L3 VNI -> routed tenant context
```

This is why VXLAN/EVPN is not just “a tunnel”. It creates a structured relationship between Layer 2 forwarding, Layer 3 routing, and overlay reachability.

---

## 4. Hardware Reality: The Data-Plane Pipeline

In modern switches, forwarding usually happens in ASIC hardware.

A simplified pipeline looks like this:

```text
Ingress
  ↓
Header parsing
  ↓
Lookup: MAC / FIB / ACL / VXLAN
  ↓
Decision: forward / drop / rewrite / encapsulate / decapsulate
  ↓
Egress
```

For VXLAN, the device may need to:

- identify the local VLAN or bridge domain;
- map it to a VNI;
- find the remote VTEP;
- add VXLAN, UDP, IP, and outer Ethernet headers;
- send the packet through the underlay.

In software labs, this may be handled by Linux or Open vSwitch. In production data-centre switches, VXLAN support must be efficient in hardware, otherwise the overlay can become a performance bottleneck.

---

## 5. Underlay and Overlay

The underlay is the physical or routed IP network.

The overlay is the virtual network built on top.

```text
             Underlay: IP Fabric

        Spine-1          Spine-2
          /  \            /  \
         /    \          /    \
      Leaf-1  Leaf-2  Leaf-3  Leaf-4


             Overlay: VXLAN

      VNI 100 ---------------- VNI 100
      VNI 200 ---------------- VNI 200
```

The underlay should provide:

- IP reachability between VTEPs;
- ECMP support;
- stable routing;
- consistent MTU;
- fast convergence;
- predictable latency.

The overlay provides:

- tenant segmentation;
- Layer 2 extension where needed;
- Layer 3 routing between segments;
- workload mobility;
- virtual network abstraction.

A weak underlay will break the overlay, no matter how elegant the EVPN design is.

---

## 6. Design Note: Leaf-Spine and IP Discipline

VXLAN/EVPN is commonly deployed on top of a leaf-spine architecture.

In this model:

```text
Leaf switches  -> connect servers, hypervisors, storage, firewalls, service nodes
Spine switches -> provide high-speed routed transport between leaves
```

This design supports:

- predictable paths;
- ECMP load balancing;
- horizontal scalability;
- clean routed underlay design;
- simpler failure isolation than large flat Layer 2 networks.

However, leaf-spine and VXLAN do not remove the need for disciplined IP planning.

A mature design should include:

```text
documented prefixes
stable loopback addressing
clear VTEP addressing
separate management, infrastructure, and tenant networks
consistent routing policy
planned summarisation where possible
controlled address ownership
```

This is similar to the discipline used in larger provider or registry-aware environments: address space should not be random, undocumented, or reused without control.

VXLAN/EVPN makes IP design more important, not less important. If addressing, loopbacks, MTU, or routing policy are messy in the underlay, the overlay may fail in ways that are difficult to diagnose.

---

## 7. VXLAN Fundamentals

VXLAN encapsulates an original Ethernet frame inside UDP/IP.

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

Or as a packet stack:

```text
[Outer Ethernet]
[Outer IP: VTEP -> VTEP]
[UDP: destination port 4789]
[VXLAN header: VNI]
[Inner Ethernet frame]
[Inner payload]
```

Key concepts:

- **VTEP** — VXLAN Tunnel Endpoint. It performs encapsulation and decapsulation.
- **VNI** — VXLAN Network Identifier. It is a 24-bit segment identifier.
- **BUM traffic** — Broadcast, Unknown unicast, and Multicast traffic.
- **Underlay** — IP transport network.
- **Overlay** — virtual network created over the underlay.

---

## 8. MTU and Encapsulation Overhead

MTU is one of the most important practical details in VXLAN design.

VXLAN adds extra headers to the original Ethernet frame:

```text
Outer Ethernet
Outer IP
UDP
VXLAN
Original Ethernet frame
```

For IPv4-based VXLAN, the additional overhead is commonly about **50 bytes**. Depending on VLAN tagging and platform details, this is often discussed as **50–54 bytes**. With IPv6 as the outer transport, the overhead is higher.

The exact overhead depends on whether outer VLAN tags, IPv6 transport, or additional encapsulation are used. Therefore, MTU should be calculated for the actual platform and design, not copied blindly from a generic example.

This means that if hosts send 1500-byte frames, the underlay must be able to carry the larger encapsulated packet.

A simplified rule is:

```text
inner frame + VXLAN overhead <= underlay MTU
```

In practice, this often means:

```text
1500-byte inner traffic
+ ~50 bytes VXLAN overhead
= at least ~1550 bytes underlay MTU
```

Many data-centre fabrics therefore use an underlay MTU of 1600, 9000, or another carefully planned jumbo MTU value.

The key point is not simply “turn on jumbo frames everywhere”. The key point is:

```text
Every underlay link between VTEPs must be able to carry the encapsulated packet safely.
```

If MTU is wrong, VXLAN may fail in confusing ways:

- small pings work, but real applications fail;
- some TCP sessions hang;
- large packets are dropped;
- fragmentation appears;
- traffic works in one direction but not reliably in another.

MTU should be tested between VTEP loopbacks before assuming that the overlay is healthy.

---

## 9. Why VXLAN Uses UDP

VXLAN uses UDP rather than TCP for several reasons.

### 1. Avoiding TCP-over-TCP problems

If an inner TCP flow were carried inside an outer TCP tunnel, both layers could retransmit and reduce speed independently. This can lead to inefficient behaviour.

### 2. ECMP hashing

VXLAN can vary the UDP source port so the underlay can distribute flows across multiple equal-cost paths.

```text
Different inner flows
        ↓
Different UDP source ports
        ↓
Different ECMP hash results
        ↓
Better load distribution
```

### 3. Hardware efficiency

UDP/VXLAN headers are simple and predictable for switching ASICs and software dataplanes to parse.

### 4. One-to-many traffic patterns

BUM traffic may require replication. UDP fits this model better than connection-oriented TCP.

Important point:

> VXLAN does not provide reliability. It carries traffic. If the inner application uses TCP, TCP handles reliability.

---

## 10. VLAN and VNI

VLAN and VNI look similar, but they are not the same thing.

| Feature | VLAN | VNI |
|---|---|---|
| Identifier size | 12 bits | 24 bits |
| Approximate scale | ~4096 IDs | ~16 million segments |
| Scope | local Layer 2 domain / trunk | overlay-wide segment |
| Purpose | local segmentation | scalable overlay segmentation |

A local VLAN can be mapped to a VNI:

```text
Leaf-1: VLAN 100 -> VNI 5000
Leaf-2: VLAN 300 -> VNI 5000
```

The local VLAN IDs are different, but the overlay segment is the same because both map to VNI 5000.

This is useful because local access-layer design does not need to match everywhere, while the overlay remains consistent.

---

## 11. VXLAN Operation Models

VXLAN can operate in several ways.

| Model | VTEP discovery | BUM handling | Scalability |
|---|---|---|---|
| Static VXLAN | manual configuration | ingress replication | low |
| Multicast VXLAN | multicast group membership | underlay multicast | medium |
| EVPN/VXLAN | BGP EVPN control plane | controlled flood lists and reachability | high |

### Static VXLAN

Static VXLAN is useful for labs. Remote VTEPs are manually configured.

When BUM traffic appears, the ingress VTEP replicates traffic to configured remote VTEPs.

This is called:

```text
ingress replication
head-end replication
```

It is simple, but not scalable.

### Multicast VXLAN

In multicast VXLAN, each VNI maps to a multicast group. The underlay replicates BUM traffic.

This avoids manual flood lists but requires multicast in the underlay, which can increase operational complexity.

### EVPN/VXLAN

EVPN/VXLAN uses a control plane to distribute reachability information. This is the most common modern data-centre model.

---

## 12. EVPN: Control Plane for VXLAN

EVPN is not limited to VXLAN. It was originally specified for BGP MPLS-based Ethernet VPNs and later became widely used with Network Virtualization Overlays such as VXLAN.

In this article, EVPN is discussed mainly in the EVPN/VXLAN data-centre context.

Without EVPN, VXLAN often relies on flood-and-learn behaviour:

```text
Unknown destination
  ↓
Flood
  ↓
Response
  ↓
Learn
```

With EVPN, reachability is advertised through the control plane:

```text
MAC/IP reachability
  ↓
MP-BGP EVPN
  ↓
Remote VTEPs learn where hosts are
```

EVPN distributes:

- MAC address reachability;
- optional IP address reachability;
- VTEP next-hop information;
- VNI membership;
- IP prefixes for Layer 3 services;
- Ethernet Segment information for multihoming.

The practical model is:

```text
VXLAN = how traffic is transported
EVPN  = how destinations are learned
BGP   = how that information is exchanged
```

---

## 13. MP-BGP and the L2VPN EVPN Address Family

EVPN routes are typically carried using MP-BGP.

This matters because BGP is not only used for Internet routing. BGP can carry different types of reachability information through address families.

For EVPN/VXLAN, the important address family is commonly:

```text
address-family l2vpn evpn
```

This allows BGP to carry EVPN Network Layer Reachability Information rather than ordinary IPv4 or IPv6 unicast routes.

In simplified terms:

```text
BGP for IPv4 unicast -> carries IP prefixes
BGP for EVPN         -> carries MACs, MAC/IPs, VNIs, Ethernet Segments, prefixes
```

This is why saying “EVPN uses BGP” is correct, but a more precise statement is:

```text
EVPN commonly uses MP-BGP with the L2VPN EVPN address family.
```

---

## 14. EVPN Route Types

EVPN uses different route types for different purposes.

The table below lists the main EVPN route types commonly discussed in EVPN/VXLAN data-centre designs. The EVPN registry contains additional route types for multicast and other extensions.

| Route Type | Name | Purpose |
|---|---|---|
| Type 1 | Ethernet Auto-Discovery | multihoming, aliasing, fast convergence |
| Type 2 | MAC/IP Advertisement | host MAC/IP reachability |
| Type 3 | Inclusive Multicast Ethernet Tag | VNI membership and BUM flood-list construction |
| Type 4 | Ethernet Segment | Ethernet Segment discovery and DF election |
| Type 5 | IP Prefix | Layer 3 prefix advertisement |

A useful memory shortcut is:

```text
Type 2 = host reachability
Type 3 = VNI membership and BUM replication/flood-list construction
Type 5 = routed prefixes
```

Route Type 2 and Type 3 are fundamental for Layer 2 overlay behaviour.

Route Type 5 becomes important when EVPN is used for routed Layer 3 services.

---

## 15. Deep Dive: Route Type 2

Route Type 2 advertises that a MAC address, and optionally an IP address, is reachable behind a specific VTEP.

Simplified:

```text
For VNI X:
MAC address M
optional IP address I
is reachable via VTEP V
```

Conceptually:

```text
MAC/IP -> remote VTEP
```

Once a leaf receives this information, it can install forwarding state:

```text
If destination MAC = M
encapsulate in VXLAN
send to VTEP V
```

This reduces unnecessary unknown-unicast flooding.

Type 2 routes are also important for ARP/ND suppression and workload mobility.

---

## 16. Deep Dive: Route Type 3

Route Type 3 advertises VNI membership.

It helps VTEPs understand which other VTEPs participate in a given VNI.

This is important for BUM traffic:

```text
Broadcast
Unknown unicast
Multicast
```

A simplified view:

```text
VTEP A advertises:
"I participate in VNI 10010."

Other VTEPs use this information to build flood lists for that VNI.
```

Type 3 does not advertise individual hosts like Type 2 does. It helps build the overlay replication domain for a VNI.

---

## 17. Deep Dive: Route Type 5

Route Type 5 advertises IP prefixes.

```text
Type 2 = host reachability
Type 5 = prefix reachability
```

Example:

```text
Type 2:
MAC/IP -> VTEP

Type 5:
IP prefix -> VTEP / VRF context
```

Route Type 5 is important for Layer 3 EVPN/VXLAN, VRFs, IRB, and inter-subnet routing.

It allows EVPN/VXLAN to become more than a Layer 2 extension mechanism. It becomes a scalable Layer 3 overlay architecture.

---

## 18. Packet Walk: Same Subnet, Known Destination

Assume Server A and Server B are in the same overlay segment but connected to different leaf switches.

```text
Server A sends Ethernet frame to Server B
        ↓
Leaf-1 receives the frame
        ↓
Leaf-1 maps local VLAN to VNI
        ↓
Leaf-1 checks EVPN/MAC table
        ↓
Server B MAC is known behind Leaf-2 VTEP
        ↓
Leaf-1 encapsulates original frame in VXLAN
        ↓
Underlay routes outer IP packet to Leaf-2 VTEP
        ↓
Leaf-2 decapsulates
        ↓
Leaf-2 forwards original Ethernet frame to Server B
```

The underlay does not learn Server A or Server B MAC addresses.

The underlay only forwards IP traffic between VTEP addresses.

---

## 19. Packet Walk: Unknown Unicast

Unknown unicast happens when the destination MAC is not known.

Without EVPN:

```text
Unknown MAC
  ↓
Flood to all VTEPs in VNI
  ↓
Correct destination responds
  ↓
MAC learning happens
```

With EVPN:

```text
Remote MAC already advertised via Type 2
  ↓
Ingress VTEP knows destination VTEP
  ↓
No unknown-unicast flooding needed for that MAC
```

EVPN reduces flooding, but it does not eliminate all Ethernet behaviour.

Broadcast, ARP/ND, and some discovery traffic may still exist depending on the design.

---

## 20. Inter-Subnet Routing and Anycast Gateway

EVPN/VXLAN is not only about Layer 2 extension. It also supports Layer 3 routing between subnets.

One of the most important production features is the **distributed anycast gateway**.

In this design, the same default gateway IP address and the same gateway MAC address are configured on multiple leaf switches.

Simplified:

```text
Leaf-1 gateway for VLAN 10: 192.168.10.1
Leaf-2 gateway for VLAN 10: 192.168.10.1
Leaf-3 gateway for VLAN 10: 192.168.10.1
```

To the server, the default gateway stays the same.

To the fabric, each leaf can route traffic locally.

This is powerful because a workload can move from one rack or leaf to another without changing its default gateway configuration.

Packet flow:

```text
Host sends traffic to another subnet
        ↓
Host sends frame to its default gateway
        ↓
Local leaf acts as the gateway
        ↓
Leaf performs routing / IRB lookup
        ↓
Leaf uses EVPN information to find the destination
        ↓
Traffic is VXLAN-encapsulated if the destination is remote
```

Benefits:

- local routing on the nearest leaf;
- less traffic tromboning through central routers;
- better workload mobility;
- consistent gateway behaviour across racks;
- better scale for east-west data-centre traffic.

This is one of the reasons EVPN/VXLAN is not just “Layer 2 over IP”. With distributed anycast gateway and IRB, it becomes a scalable Layer 2 and Layer 3 fabric.

---

## 21. Symmetric vs Asymmetric IRB

IRB means Integrated Routing and Bridging.

There are two common models.

| Model | Description | Scalability |
|---|---|---|
| Asymmetric IRB | routing mainly on the ingress leaf switch | less scalable |
| Symmetric IRB | routing on both ingress and egress leaf switches using L3 VNI | more scalable |

Symmetric IRB is commonly used in modern EVPN/VXLAN fabrics because it provides better scale and consistency.

A simplified symmetric IRB flow:

```text
Ingress leaf switch:
bridge -> route -> encapsulate into L3 VNI

Egress leaf switch:
decapsulate -> route/bridge -> deliver to destination
```

This separates tenant routing context from individual Layer 2 segments.

---

## 22. ARP and ND Suppression

In traditional Layer 2 networks, ARP and Neighbor Discovery can create broadcast or multicast traffic.

In EVPN/VXLAN, the control plane may already know the mapping:

```text
IP -> MAC -> VTEP
```

With ARP/ND suppression, a VTEP can answer some ARP or ND requests locally instead of flooding them across the overlay.

Benefits:

- less broadcast;
- better scalability;
- faster address resolution;
- reduced BUM traffic.

Important limitation:

> ARP/ND suppression depends on correct control-plane information. If the EVPN database is wrong or incomplete, troubleshooting can become confusing.

---

## 23. Control-Plane Scaling: Route Reflectors and eBGP Designs

In small labs, every VTEP may peer with every other VTEP.

This full-mesh model does not scale well.

In many iBGP-based EVPN designs, BGP route reflectors are used to simplify control-plane peering.

```text
Leaf-1  \
Leaf-2   -> Route Reflector -> EVPN routes distributed
Leaf-3  /
```

Important point:

> Route reflectors distribute control-plane information. They do not forward VXLAN data-plane traffic.

Data traffic still flows directly between VTEPs through the underlay.

A route reflector is typically an **iBGP scaling mechanism**, meaning the EVPN speakers usually belong to the same BGP AS for the overlay control plane.

There is also another common model: **eBGP EVPN leaf-spine design**.

In an eBGP EVPN design, leaves and spines may use different AS numbers, and spines act as BGP peers rather than classical iBGP route reflectors.

A clean mental separation is:

```text
iBGP EVPN overlay:
  route reflectors are commonly used

eBGP EVPN fabric:
  spines are usually eBGP peers, not traditional route reflectors
```

Both approaches can be valid. The important thing is not to mix the terminology carelessly.

Route reflectors, when used, scale the EVPN control plane.

They do not encapsulate traffic, decapsulate traffic, or sit in the VXLAN data path.

---

## 24. ECMP and Flow Distribution

VXLAN fabrics usually use ECMP in the underlay.

VXLAN uses UDP source-port variation to help the underlay hash different flows across different paths.

```text
Flow A -> UDP source port X -> path 1
Flow B -> UDP source port Y -> path 2
Flow C -> UDP source port Z -> path 3
```

This improves bandwidth utilisation and allows the fabric to scale horizontally.

Without good ECMP behaviour, a fabric may have multiple available paths but still use them inefficiently.

---

## 25. Failure Domains

One advantage of VXLAN/EVPN is that it helps limit failure domains.

Traditional large Layer 2 networks may propagate problems widely. A routed underlay contains failures better because routing boundaries are clearer.

VXLAN/EVPN improves stability by separating:

```text
physical transport problems
from
logical tenant segmentation
```

However, the overlay depends on the underlay.

If underlay reachability to VTEPs fails, overlay traffic fails too.

---

## 26. MAC Mobility

In virtualised environments, workloads can move between hosts or leaf switches.

EVPN supports MAC mobility by updating the advertised location of a MAC address.

When a host moves:

```text
New leaf advertises updated MAC/IP route
Old path is withdrawn or replaced
Remote VTEPs update forwarding state
Traffic follows the new location
```

This prevents traffic from being sent to the wrong VTEP.

MAC mobility is useful, but excessive MAC movement can also indicate a loop, misconfiguration, or unstable workload placement.

---

## 27. EVPN Multihoming

EVPN supports multihoming, where a device is connected to multiple leaf switches.

Simplified:

```text
      Leaf-1
        |
      Server
        |
      Leaf-2
```

In real deployments, this may use Ethernet Segments.

Benefits:

- redundancy;
- active-active forwarding;
- load balancing;
- fast convergence.

EVPN uses Route Type 1 and Route Type 4 to support Ethernet Segment discovery, aliasing, mass withdrawal, and Designated Forwarder election.

---

## 28. Loop Prevention and Split Horizon

Traditional Layer 2 networks often rely on STP to prevent loops.

VXLAN/EVPN uses different mechanisms.

One important rule is split horizon:

```text
Traffic received from a VXLAN tunnel
should not be forwarded back into another VXLAN tunnel
for the same segment.
```

This helps prevent loops without blocking physical links like STP.

EVPN multihoming also uses additional mechanisms to prevent duplicate forwarding and loops in Ethernet Segment designs.

---

## 29. Flood-and-Learn vs Control-Plane Learning

| Model | How learning happens | Scaling |
|---|---|---|
| Flood-and-learn VXLAN | unknown traffic is flooded and learned from data plane | weaker |
| EVPN/VXLAN | MAC/IP reachability is advertised through BGP EVPN | stronger |

This is one of the main reasons EVPN is preferred for larger VXLAN deployments.

EVPN does not make Ethernet magically disappear. It makes Ethernet behaviour more controlled and more visible through the control plane.

---

## 30. Security Considerations

VXLAN/EVPN fabrics introduce security considerations.

VXLAN does not provide encryption or authentication by itself. It provides encapsulation and segmentation. If confidentiality or strong protection is required, additional controls such as MACsec, IPsec, strict underlay filtering, control-plane protection, and proper tenant isolation must be considered.

Important areas:

- BGP session protection;
- control-plane filtering;
- tenant segmentation through VNIs and VRFs;
- ACL enforcement in the data plane;
- protection of VTEP loopbacks;
- route-target import/export policy;
- monitoring for unexpected MAC movement;
- management-plane access control;
- logging and telemetry.

Segmentation is powerful, but it must be designed and enforced carefully.

A VNI is not a security boundary by itself unless the surrounding control-plane, VRF, ACL, and operational policies are correct.

---

## 31. Observability and Troubleshooting

Troubleshooting VXLAN/EVPN requires checking both the control plane and the data plane.

Control-plane checks:

```text
Are BGP EVPN sessions established?
Are Type 2 routes present?
Are Type 3 routes present?
Are Type 5 routes present?
Are VNIs advertised?
Are route targets imported/exported correctly?
```

Data-plane checks:

```text
Is the VTEP reachable?
Is the MAC installed?
Is VXLAN encapsulation happening?
Is the MTU correct?
Are ACLs dropping traffic?
Is the correct VRF used?
```

Useful operational commands often include:

```text
show bgp l2vpn evpn
show evpn route
show mac address-table
show nve peers
show nve vni
show forwarding route
show ip route
show interface counters
```

Exact commands depend on the platform.

The important habit is to avoid guessing. Check control plane first, then data plane, then policy.

---

## 32. How VXLAN/EVPN Fails in Real Networks

Many VXLAN/EVPN failures come from a mismatch between what the control plane believes and what the data plane can actually forward.

| Symptom | Possible Cause |
|---|---|
| EVPN routes are present, but traffic does not pass | MTU issue, ACL drop, missing NVE/VTEP state |
| VTEPs are reachable, but remote MACs are missing | Type 2 routes are not advertised or imported |
| BUM traffic behaves incorrectly | Type 3 routes, flood-list, or ingress replication issue |
| Inter-subnet routing fails | VRF, L3 VNI, route target, or IRB issue |
| Traffic blackholes after VM movement | stale MAC mobility state or slow withdrawal |
| One tenant leaks into another | incorrect VNI/VRF/route-target policy |
| Some flows work, others fail | ECMP hashing, MTU, asymmetric path, or firewall policy |
| Ping works, application fails | MTU, TCP MSS, ACL, or service path issue |

This is why VXLAN/EVPN troubleshooting must be systematic.

A useful sequence is:

```text
1. Is the underlay healthy?
2. Are VTEP loopbacks reachable?
3. Are BGP EVPN sessions established?
4. Are the required route types present?
5. Are route targets correct?
6. Is forwarding state installed?
7. Is MTU correct?
8. Are ACLs or firewalls dropping traffic?
```

---

## 33. Open vSwitch and VXLAN

Open vSwitch is useful for VXLAN labs because it can create VXLAN ports and act as a software data plane.

However, OVS by itself should not be understood as a complete EVPN/VXLAN fabric solution.

VXLAN encapsulation is a data-plane function.

EVPN reachability signalling requires a control-plane component.

In practical labs, OVS is commonly used in one of these ways:

- with statically configured VXLAN tunnels;
- controlled by higher-level systems such as OVN or OpenStack Neutron;
- combined with an external BGP/EVPN component such as FRRouting or GoBGP.

The important distinction is:

```text
OVS can handle VXLAN forwarding.
EVPN route exchange normally comes from another control-plane component.
```

So, OVS is excellent for understanding VXLAN mechanics, but a full EVPN/VXLAN architecture requires more than just creating a VXLAN interface.

---

## 34. Example Configuration: FRRouting + Linux VXLAN

The following example is illustrative. It shows the general idea, not a universal production template.

This example uses an **iBGP EVPN overlay with a Route Reflector**.

Assume:

```text
Leaf-1 VTEP loopback: 10.0.0.1
Leaf-2 VTEP loopback: 10.0.0.2
Route Reflector: 10.0.0.254
Overlay BGP AS: 65000
VNI: 10010
Local VLAN / bridge domain: 10
```

This example focuses on the overlay control-plane structure.

The underlay is assumed to already provide IP reachability between VTEP loopbacks.

In other words, before EVPN/VXLAN can work, this must already be true:

```text
Leaf-1 can reach 10.0.0.2
Leaf-1 can reach 10.0.0.254
Leaf-2 can reach 10.0.0.1
Leaf-2 can reach 10.0.0.254
```

### Linux VXLAN and bridge example

```bash
# Create a bridge for the local segment
ip link add br10 type bridge
ip link set br10 up

# Create a VXLAN interface
ip link add vxlan10010 type vxlan id 10010 local 10.0.0.1 dstport 4789 nolearning

# Attach VXLAN interface to bridge
ip link set vxlan10010 master br10
ip link set vxlan10010 up

# Attach local server-facing interface to bridge
ip link set eth1 master br10
ip link set eth1 up
```

The `nolearning` option is commonly used in EVPN-style labs because remote MAC learning should come from the control plane rather than ordinary VXLAN flood-and-learn behaviour.

However, creating a Linux VXLAN interface does not automatically create a complete EVPN fabric.

The EVPN control plane still needs a routing component such as FRRouting, and the exact Linux bridge/VXLAN integration may vary depending on the distribution, kernel, FRR version, and lab design.

### FRR BGP EVPN example on Leaf-1

```text
router bgp 65000
 bgp router-id 10.0.0.1
 no bgp default ipv4-unicast

 neighbor SPINES peer-group
 neighbor SPINES remote-as 65000
 neighbor SPINES update-source lo
 neighbor 10.0.0.254 peer-group SPINES

 address-family l2vpn evpn
  neighbor SPINES activate
  advertise-all-vni
 exit-address-family
```

This simplified FRR example enables the L2VPN EVPN address family and advertises local VNIs.

Because this is an iBGP EVPN overlay example, the leaf and the route reflector use the same BGP AS.

In a real design, you must also ensure:

- underlay routing works;
- VTEP loopbacks are reachable;
- bridge/VXLAN interfaces exist;
- route targets and route distinguishers are correct;
- import/export policy is appropriate;
- kernel, FRR, and interface state are consistent;
- MTU is correct across the entire underlay path.

The key idea is:

```text
Linux bridge / VXLAN interface = data-plane structure
FRR BGP EVPN                  = control-plane signalling
Underlay routing              = IP transport between VTEPs
```

---

## 35. Example Configuration: Cisco NX-OS VXLAN EVPN Skeleton

The following is a simplified Cisco Nexus-style skeleton. It is not a complete production configuration, but it shows the main building blocks.

```text
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay

nv overlay evpn
```

### VLAN to VNI mapping

```text
vlan 10
  vn-segment 10010
```

### NVE interface

```text
interface nve1
  no shutdown
  source-interface loopback1
  member vni 10010
    ingress-replication protocol bgp
```

### BGP EVPN address family

```text
router bgp 65000
  router-id 10.0.0.1

  neighbor 10.0.0.254
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community extended
```

In this simplified example, Leaf-1 and the route reflector are in the same BGP AS, which matches a typical iBGP EVPN route-reflector design.

### Optional SVI / anycast gateway concept

```text
fabric forwarding anycast-gateway-mac 0000.1111.2222

interface Vlan10
  no shutdown
  vrf member TENANT-A
  ip address 192.168.10.1/24
  fabric forwarding mode anycast-gateway
```

In real Cisco NX-OS deployments, additional configuration is commonly required:

- loopbacks;
- underlay routing;
- VRFs;
- L3 VNIs;
- route targets;
- route distinguishers;
- anycast gateway MAC;
- NVE peer reachability;
- BGP policies;
- platform-specific feature requirements.

---

## 36. How to Read These Configurations

Do not memorise syntax first. Understand the structure.

Every VXLAN/EVPN configuration usually needs these building blocks:

```text
1. Underlay routing
2. VTEP loopback reachability
3. VLAN / bridge-domain to VNI mapping
4. NVE / VXLAN interface
5. BGP EVPN address family
6. Route targets / import-export policy
7. Optional VRF and L3 VNI
8. Anycast gateway for distributed routing
9. MTU planning
10. Observability and troubleshooting commands
```

If one of these is missing, the fabric may partially work or fail in confusing ways.

---

## 37. Common Misunderstandings

### “VXLAN is just a bigger VLAN”

Not exactly.

VXLAN provides a larger segment identifier space, but it also changes the network model. Instead of extending Layer 2 directly across the physical network, VXLAN carries logical segments over a routed IP underlay.

### “EVPN replaces VXLAN”

No.

EVPN does not replace VXLAN. EVPN provides control-plane signalling. VXLAN provides encapsulation and transport.

### “BGP means Internet routing only”

No.

BGP is also widely used inside data centres as a control-plane protocol. In EVPN/VXLAN, BGP carries EVPN route information between VTEPs or through route reflectors.

### “The underlay does not matter”

Wrong.

The overlay depends completely on the underlay. If VTEP loopbacks are unreachable, MTU is wrong, ECMP is broken, or routing is unstable, the VXLAN overlay will also fail.

### “VXLAN/EVPN removes all flooding”

No.

EVPN greatly reduces unnecessary flooding by advertising MAC/IP reachability through the control plane, but some BUM traffic may still exist depending on the design and workload behaviour.

### “A VNI automatically means security”

No.

A VNI is a segmentation construct. Security still depends on correct VRF design, route-target policy, ACLs, firewalling, management-plane protection, and operational controls.

### “Anycast Gateway is optional in every design”

Not exactly.

A simple Layer 2 extension lab can work without distributed anycast gateway. But in a modern EVPN/VXLAN fabric with distributed inter-subnet routing and workload mobility, anycast gateway is a key design component.

---

## 38. Final Summary

VXLAN is an overlay technology that encapsulates Ethernet frames inside UDP/IP. It allows virtual Layer 2 and Layer 3 networks to run over a routed IP underlay.

EVPN is the control plane commonly used with VXLAN. It distributes MAC, IP, VTEP, VNI, Ethernet Segment, and IP-prefix reachability information.

BGP, more precisely MP-BGP with the L2VPN EVPN address family, is commonly used to carry EVPN routes.

The most important model is:

```text
VXLAN answers: how is traffic transported?
EVPN answers: where is the destination?
BGP answers: how is that knowledge exchanged?
```

VXLAN alone is not a complete scalable fabric.

VXLAN + EVPN creates a control-plane-driven overlay system suitable for modern data-centre networks.

The real understanding is not “VXLAN is a tunnel.”



```text
A reliable overlay depends on:
clean underlay routing,
correct MTU,
stable VTEP reachability,
accurate EVPN control-plane state,
clear VNI/VRF design,
distributed gateway behaviour where needed,
and disciplined operations.
```

---

## Further Reading

- RFC 7348 — Virtual eXtensible Local Area Network (VXLAN): <https://datatracker.ietf.org/doc/html/rfc7348>
- RFC 7432 — BGP MPLS-Based Ethernet VPN (EVPN): <https://datatracker.ietf.org/doc/html/rfc7432>
- RFC 8365 — A Network Virtualization Overlay Solution Using EVPN: <https://datatracker.ietf.org/doc/html/rfc8365>
- RFC 9135 — Integrated Routing and Bridging in EVPN: <https://datatracker.ietf.org/doc/html/rfc9135>
- RFC 9136 — IP Prefix Advertisement in EVPN: <https://datatracker.ietf.org/doc/html/rfc9136>
- FRRouting EVPN documentation: <https://docs.frrouting.org/en/latest/evpn.html>
- Linux VXLAN documentation: <https://docs.kernel.org/networking/vxlan.html>
- Open vSwitch VXLAN documentation: <https://docs.openvswitch.org/en/latest/faq/vxlan/>
