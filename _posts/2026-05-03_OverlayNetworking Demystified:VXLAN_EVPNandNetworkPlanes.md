from pathlib import Path
from html import escape

html = """<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Network Planes and VXLAN/EVPN: A Clear Explanation</title>
    <style>
        :root {
            --bg: #f7f7f5;
            --paper: #ffffff;
            --text: #1f2933;
            --muted: #5f6b7a;
            --border: #d8dee6;
            --accent: #8b1e3f;
            --code-bg: #f1f3f5;
        }

        body {
            margin: 0;
            background: var(--bg);
            color: var(--text);
            font-family: Arial, Helvetica, sans-serif;
            line-height: 1.7;
        }

        main {
            max-width: 980px;
            margin: 40px auto;
            padding: 48px;
            background: var(--paper);
            border: 1px solid var(--border);
            border-radius: 18px;
            box-shadow: 0 18px 45px rgba(0, 0, 0, 0.06);
        }

        h1 {
            font-size: 2.2rem;
            line-height: 1.2;
            margin-bottom: 12px;
            color: #111827;
        }

        h2 {
            margin-top: 42px;
            padding-top: 22px;
            border-top: 1px solid var(--border);
            color: #111827;
        }

        h3 {
            margin-top: 24px;
            color: #243447;
        }

        p {
            margin: 14px 0;
        }

        .lead {
            font-size: 1.08rem;
            color: var(--muted);
        }

        .note {
            border-left: 5px solid var(--accent);
            background: #fff7fa;
            padding: 16px 18px;
            margin: 22px 0;
            border-radius: 10px;
        }

        pre {
            background: var(--code-bg);
            padding: 16px;
            border-radius: 10px;
            overflow-x: auto;
            white-space: pre-wrap;
            border: 1px solid var(--border);
        }

        code {
            background: var(--code-bg);
            padding: 2px 6px;
            border-radius: 5px;
        }

        ul {
            padding-left: 24px;
        }

        li {
            margin: 6px 0;
        }

        table {
            width: 100%;
            border-collapse: collapse;
            margin: 22px 0;
        }

        th, td {
            border: 1px solid var(--border);
            padding: 12px;
            text-align: left;
        }

        th {
            background: #f3f4f6;
        }

        .summary {
            background: #f8fafc;
            border: 1px solid var(--border);
            border-radius: 12px;
            padding: 18px;
            margin: 22px 0;
        }

        footer {
            margin-top: 48px;
            color: var(--muted);
            font-size: 0.95rem;
        }

        @media (max-width: 700px) {
            main {
                margin: 0;
                padding: 24px;
                border-radius: 0;
                border-left: 0;
                border-right: 0;
            }

            h1 {
                font-size: 1.7rem;
            }
        }
    </style>
</head>
<body>
<main>

<h1>Network Planes and VXLAN/EVPN: A Clear Explanation</h1>

<p class="lead">
Modern networking often becomes confusing because several ideas are mixed together:
device architecture, forwarding tables, tunnelling, encapsulation, underlay networks,
overlay networks, and control-plane protocols.
</p>

<div class="note">
<p>A common shortcut is:</p>
<pre>VXLAN = data plane</pre>
<p>This is useful, but not fully precise.</p>
<p>A better explanation is:</p>
<p><strong>VXLAN is a data-plane encapsulation and forwarding mechanism. EVPN is commonly used as the control plane for scalable VXLAN overlays.</strong></p>
</div>

<p>
VXLAN defines how Ethernet frames are encapsulated and forwarded over an IP underlay.
EVPN commonly distributes overlay reachability information: MAC addresses, IP addresses,
VTEP locations, VNI membership, and IP prefixes.
</p>

<h2>1. The Three Network Planes</h2>

<p>
A network device can be understood as a system with three logical planes:
the control plane, the data plane, and the management plane.
</p>

<h3>Control Plane</h3>
<p>
The control plane builds network knowledge. It decides what routes exist, which paths are valid,
which MAC addresses are known, and how the topology should work.
</p>
<p>
Examples include OSPF, BGP, STP, ARP/ND processing, LISP mapping, and the logic that builds
routing and forwarding tables.
</p>

<h3>Data Plane</h3>
<p>
The data plane forwards real traffic. It uses existing tables and applies forwarding decisions
at high speed.
</p>
<p>
This is where Layer 2/Layer 3 forwarding, ACL enforcement, QoS, VXLAN encapsulation,
and VXLAN decapsulation usually happen.
</p>

<h3>Management Plane</h3>
<p>
The management plane allows humans or systems to manage the device. This includes SSH, CLI,
Web UI, SNMP, Syslog, NetFlow/IPFIX, NETCONF, RESTCONF, and APIs.
</p>

<div class="summary">
<p><strong>In short:</strong></p>
<pre>Control plane  -> builds knowledge and tables
Data plane     -> uses tables and forwards packets
Management     -> provides access, monitoring and configuration</pre>
</div>

<p>
This separation is logical, not always perfectly physical. In real devices, some functions overlap.
MAC learning may happen in hardware, EVPN may populate MAC tables through BGP, and ACLs may be
configured through management or control logic but enforced in the data plane.
</p>

<h2>2. Tables Used by Switches and Routers</h2>

<p>
A switch or router does not analyse every packet from zero. It builds tables in advance and then
checks incoming traffic against them.
</p>

<p>A MAC/CAM table maps a MAC address and VLAN to a physical port:</p>
<pre>MAC + VLAN -> port</pre>

<p>This is used for Layer 2 switching.</p>

<p>A RIB, or Routing Information Base, is the logical routing table built from routing protocols and static routes:</p>
<pre>IP prefix -> candidate route</pre>

<p>A FIB, or Forwarding Information Base, is the optimised forwarding table used by the data plane:</p>
<pre>destination prefix -> next hop / outgoing interface / action</pre>

<p>
It is more accurate to say “destination prefix” than just “IP address”, because routing works with
prefixes such as <code>10.10.20.0/24</code>.
</p>

<p>
An ARP table maps IPv4 addresses to MAC addresses in a local Layer 2 segment.
IPv6 uses Neighbor Discovery instead.
</p>

<p>An ACL/TCAM entry defines conditions and actions:</p>
<pre>condition -> permit / deny / action</pre>

<p>On many switches, these rules are applied in hardware using TCAM.</p>

<h2>3. CPU, ASIC and FPGA</h2>

<p>
The CPU is flexible. It can run routing protocols, management services, BGP sessions, SSH, SNMP,
calculations, and system processes.
</p>

<p>
However, forwarding every packet through the CPU would be too slow for high-performance networking.
That is why modern switches and routers use specialised forwarding hardware.
</p>

<p>
An ASIC is designed for very fast and efficient packet forwarding. It is commonly used for switching,
routing, ACLs, QoS, ECMP, and VXLAN offload.
</p>

<p>
An FPGA is also hardware-based, but it can be reprogrammed. It is useful for specialised processing
and prototyping, although it is usually more complex and less efficient than a fixed ASIC.
</p>

<p>A simplified data-plane pipeline looks like this:</p>
<pre>Ingress -> Parsing -> Lookup -> Decision/Action -> Egress</pre>

<p>For example:</p>
<ul>
<li>A frame enters a port.</li>
<li>The ASIC parses Ethernet, IP, TCP/UDP, and VXLAN headers.</li>
<li>The ASIC checks MAC, FIB, and ACL tables.</li>
<li>The ASIC decides whether to forward, drop, rewrite, encapsulate, or decapsulate.</li>
<li>The packet leaves through the egress port.</li>
</ul>

<h2>4. Mapping Protocols to Planes</h2>

<p>Different protocols belong to different planes, although some have mixed roles.</p>

<p>
OSPF, BGP, and STP are mainly control-plane protocols. They build routing information,
topology knowledge, or loop-free Layer 2 logic.
</p>

<p>
SNMP belongs to the management plane. It is used for monitoring and sometimes configuration,
but it does not forward user traffic.
</p>

<p>
VXLAN is a data-plane encapsulation mechanism. It defines how Ethernet frames are wrapped
inside UDP/IP packets.
</p>

<p>
EVPN is commonly used as a control-plane solution for VXLAN overlay networks. It uses MP-BGP
to tell VTEPs where MAC addresses, IP addresses, VNI membership, VTEP reachability, and prefixes
are located.
</p>

<p>
LISP has both control-plane and data-plane parts. It has mapping logic, such as EID to RLOC,
and also encapsulation.
</p>

<h2>5. From GRE to VXLAN and Geneve</h2>

<p>VXLAN is easier to understand as part of the evolution of overlay tunnelling.</p>

<p>
The basic idea is simple: if the transport network should not directly carry a certain logical network,
we can wrap one packet inside another and carry it across the transport network.
</p>

<ul>
<li><strong>GRE</strong> introduced the general idea of tunnelling: one packet can be carried inside another IP packet.</li>
<li><strong>NVGRE</strong> adapted this idea for network virtualisation in multi-tenant data centres.</li>
<li><strong>VXLAN</strong> became a dominant approach for Layer 2 overlays in data centres.</li>
<li><strong>Geneve</strong> is a newer and more flexible encapsulation format with extensible metadata options.</li>
</ul>

<div class="summary">
<p><strong>The main idea is always the same:</strong></p>
<pre>network over network</pre>
</div>

<h2>6. What VXLAN Does</h2>

<p>
VXLAN is often described as “Ethernet over IP”, but more precisely it encapsulates Ethernet frames
inside UDP/IP so that they can travel across a routed IP network.
</p>

<p>The encapsulation looks like this:</p>
<pre>Original Ethernet frame
        ↓
VXLAN header with VNI
        ↓
UDP header, destination port 4789
        ↓
Outer IP header: source VTEP -> destination VTEP
        ↓
Outer Ethernet header for the underlay link</pre>

<p>To the underlay network, this is just UDP/IP traffic between VTEPs.</p>
<p>To the overlay network, it looks like an extended Layer 2 segment.</p>

<p>
A VTEP, or VXLAN Tunnel Endpoint, is the device that performs VXLAN encapsulation and decapsulation.
It can be a leaf switch, hypervisor, or software switch.
</p>

<p>
A VNI, or VXLAN Network Identifier, identifies the virtual Layer 2 segment. It is 24 bits long,
which gives about 16 million possible VXLAN segments.
</p>

<p>The underlay is the physical or routed IP network below.</p>
<p>The overlay is the virtual network built on top of it.</p>

<p>
BUM traffic means Broadcast, Unknown unicast, and Multicast. This traffic usually requires replication
or special handling, because the destination is not always known as a single unicast endpoint.
</p>

<h2>7. Underlay and Overlay</h2>

<p>The key VXLAN architecture idea is this:</p>

<div class="summary">
<p>The underlay should be a simple routed IP fabric.</p>
<p>The overlay carries the virtual Layer 2 or Layer 3 networks.</p>
</div>

<p>
The underlay does not need to know the MAC addresses of every virtual machine.
Its job is to provide IP reachability between VTEPs.
</p>

<p>Example:</p>
<pre>Underlay:
Leaf-1 ---- Spine(s) ---- Leaf-2
IP routing, loopbacks, ECMP, BGP/OSPF/ISIS

Overlay:
VNI 100 = virtual Layer 2 segment
Server A and Server B believe they are in the same L2 network</pre>

<p>
If Server A sends an ARP broadcast, the ingress VTEP encapsulates that Ethernet frame into VXLAN
and sends it to other VTEPs in the same VNI. The remote VTEP removes the outer headers and releases
the original Ethernet frame into the local segment.
</p>

<h2>8. VLAN and VNI</h2>

<p>VLAN and VNI look similar, but they are not the same thing.</p>

<table>
<tr>
<th>Feature</th>
<th>VLAN</th>
<th>VNI</th>
</tr>
<tr>
<td>Size</td>
<td>12 bits, about 4096 VLANs</td>
<td>24 bits, about 16 million VXLAN segments</td>
</tr>
<tr>
<td>Scope</td>
<td>Usually local to a Layer 2 domain or trunk</td>
<td>Overlay-wide between VTEPs</td>
</tr>
<tr>
<td>Purpose</td>
<td>Layer 2 segmentation</td>
<td>Overlay segmentation</td>
</tr>
</table>

<p>A local VLAN can be mapped to a VNI.</p>

<p>Example:</p>
<pre>Leaf-1: VLAN 100 -> VNI 5000
Leaf-2: VLAN 300 -> VNI 5000</pre>

<p>
The local VLANs are different, but the overlay segment is the same because both are mapped to VNI 5000.
</p>

<p>
This is one of the reasons VXLAN is useful in data centres: it separates local VLAN numbering
from overlay segmentation.
</p>

<h2>9. Static VXLAN</h2>

<p>Static VXLAN is the simplest lab mode.</p>

<p>
The engineer manually configures remote VTEP addresses. When BUM traffic appears, the ingress VTEP
creates multiple VXLAN copies and sends them as unicast packets to all configured remote VTEPs.
</p>

<p>This is called ingress replication or head-end replication.</p>

<p>
The advantage is simplicity. It is easy to understand, useful in small labs, and does not require
multicast in the underlay.
</p>

<p>
The disadvantage is scalability. Every new VTEP must be configured on other VTEPs.
BUM traffic is replicated by the ingress VTEP, and MAC learning remains mostly flood-and-learn.
</p>

<p>Static VXLAN is good for learning. It is not ideal for large modern data-centre fabrics.</p>

<h2>10. Multicast VXLAN</h2>

<p>In multicast VXLAN, each VNI is mapped to a multicast group.</p>

<p>
VTEPs that belong to that VNI join the multicast group. When BUM traffic appears, the ingress VTEP
sends one VXLAN packet to the multicast address, and the multicast-capable underlay replicates it
to the correct VTEPs.
</p>

<p>This removes the need to manually configure all remote VTEPs in a flood list.</p>

<p>
However, it requires multicast in the underlay, usually with protocols and logic such as PIM, RP,
or Anycast RP. This can make operations more complex.
</p>

<p>
Multicast VXLAN also explains why VXLAN uses UDP rather than TCP: some traffic patterns are one-to-many.
</p>

<p>In many modern data centres, EVPN is preferred instead of relying on underlay multicast.</p>

<h2>11. EVPN/VXLAN</h2>

<p>EVPN/VXLAN is the most important modern design.</p>

<div class="summary">
<pre>VXLAN = how to encapsulate and forward traffic
EVPN  = how VTEPs learn who is where
BGP   = transport for EVPN information</pre>
</div>

<p>
With EVPN, VTEPs can exchange information about MAC addresses, IP addresses, VTEP locations,
VNI membership, and IP prefixes.
</p>

<p>This means the network does not need to rely only on flooding or static configuration.</p>

<h2>12. EVPN Route Types</h2>

<p>EVPN uses different route types for different purposes.</p>

<h3>Route Type 1 — Ethernet Auto-Discovery</h3>
<p>Used for multihoming, fast convergence, aliasing, and Ethernet Segment logic.</p>

<h3>Route Type 2 — MAC/IP Advertisement</h3>
<p>Advertises that a MAC address, and optionally an IP address, is reachable behind a specific VTEP.</p>

<h3>Route Type 3 — Inclusive Multicast Ethernet Tag</h3>
<p>Advertises VTEP participation in a VNI and helps build flood lists for BUM traffic.</p>

<h3>Route Type 4 — Ethernet Segment</h3>
<p>Used for Ethernet Segment discovery and Designated Forwarder election in multihoming.</p>

<h3>Route Type 5 — IP Prefix</h3>
<p>
Advertises IP prefixes through EVPN. This is important for Layer 3 EVPN/VXLAN, VRFs, IRB,
and inter-subnet routing.
</p>

<p>Route Type 5 is not about MAC addresses. It is about IP prefixes.</p>

<h2>13. What EVPN Improves</h2>

<p>EVPN improves VXLAN in several important ways.</p>

<ul>
<li>It provides automatic VTEP discovery instead of manually configured flood lists.</li>
<li>It distributes MAC/IP reachability through BGP instead of relying only on flood-and-learn.</li>
<li>It reduces unknown-unicast flooding because remote MAC information can be known in advance.</li>
<li>It supports multihoming, fast convergence, mass withdrawal, and aliasing.</li>
<li>It supports anycast gateway, where the same default gateway can exist on multiple leaf switches.</li>
<li>It supports Layer 3 EVPN through Route Type 5 and IP VRFs.</li>
</ul>

<p>
However, EVPN does not remove all flooding. Broadcast and some initial discovery traffic still exist.
EVPN reduces and controls flooding, but it does not magically remove all Ethernet behaviour.
</p>

<h2>14. Open vSwitch and VXLAN</h2>

<p>
Open vSwitch is useful for VXLAN labs because it is a software switch that can perform VXLAN framing
and act as a dataplane.
</p>

<p>However, OVS by itself is not a full EVPN control plane.</p>

<p>
For EVPN scenarios, an external BGP/EVPN component or controller is usually needed, such as FRR,
GoBGP, OVN, or OpenStack Neutron.
</p>

<p>
A useful nuance: OVS supports VXLAN packet framing, but not the multicast aspects of VXLAN.
A workaround is to pre-provision MAC-to-IP mappings manually or through a controller.
</p>

<h2>15. Final Summary</h2>

<p>
VXLAN is an overlay technology that encapsulates Ethernet frames inside UDP/IP. It allows virtual
Layer 2 networks to run over a routed IP underlay.
</p>

<p>
A VTEP performs encapsulation and decapsulation. A VNI identifies the virtual segment. The underlay
provides IP reachability between VTEPs. The overlay provides the virtual network seen by hosts or
virtual machines.
</p>

<p>Static VXLAN uses manually configured VTEPs.</p>
<p>Multicast VXLAN uses multicast groups for BUM replication.</p>
<p>EVPN/VXLAN uses BGP EVPN as a common control plane to distribute MAC, IP, VTEP, VNI membership, and IP-prefix information.</p>

<div class="summary">
<p><strong>The most important idea is:</strong></p>
<pre>VXLAN = data-plane encapsulation and forwarding
EVPN  = commonly used control-plane learning/signalling for scalable VXLAN overlays
BGP   = protocol that carries EVPN routes</pre>
</div>

<p>
So the phrase “VXLAN = data plane” is useful, but incomplete.
</p>

<p>A more accurate statement is:</p>

<p><strong>
VXLAN defines how Ethernet frames are encapsulated and forwarded over an IP underlay.
EVPN is commonly used as the control plane that tells VTEPs where MAC addresses, IP addresses,
VNI membership, VTEP reachability, and prefixes are located.
</strong></p>

<footer>
<p>Prepared as a clear technical article on network planes, VXLAN, and EVPN.</p>
</footer>

</main>
</body>
</html>
"""

path = Path("/mnt/data/vxlan-evpn-full-article.html")
path.write_text(html, encoding="utf-8")
path.as_posix()
