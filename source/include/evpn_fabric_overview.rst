EVPN VXLAN overview
===================

Ethernet Virtual Private Network (EVPN) is a standard technology that
can be used for scaling layer 2 networks that is becoming increasingly
popular. It combines a layer 3 (Equal-Cost Multi-Path)
ECMP "underlay" fabric with VXLAN "overlays" that stretch VLANs between
switches.

EVPN is typically used with a spine/leaf network architecture:

.. figure:: _static/spine-leaf.png
   :alt: Spine/leaf network architecture
   :class: no-scaled-link

This type of multipath network is resilient to failures of
individual links or devices. A standard layer 2 (Ethernet) network
cannot achieve this without introducing forwarding loops or disabling
links and/or switches (e.g. Spanning Tree Protocol (STP)).

Often the leaf switches will be paired, with MLAG or similar technology
used to connect servers to both switches in a pair.

Border Gateway Protocol (BGP) is used in the control plane of both the
underlay and the overlay to exchange connectivity between the switches.
BGP is a proven, widely used protocol that underpins exchange of routing
information on the Internet. The underlay uses standard BGP, while the
overlay uses Multi Protocol BGP (MP-BGP) to exchange MAC addresses and
other information between Virtual Tunnel Endpoints (VTEPs).

This network architecture is undoubtedly more complex than the standard
layer 2 networks we have generally used in the past. It's easiest
to build up the picture in layers.

Underlay IP links
-----------------

Each leaf switch has a layer 3 (IP) /31 point to point connection to
each spine switch. Typically we would divide up a supernet (e.g. /24)
into multiple /31 subnets. Doing this puts the spine-leaf links into
layer 3 mode.

Example leaf interface config on Dell OS10:

::

   !
   interface ethernet1/1/1
    no shutdown
    no switchport
    ip address 172.0.0.0/31

Example spine interface config on Dell OS10:

::

   !
   interface ethernet1/1/1
    no shutdown
    no switchport
    ip address 172.0.0.1/31

One of the implications of this is that each switch and the hosts
attached to it have become an isolated layer 2 (Ethernet) network.

Another implication is that each switch only has layer 3 (IP)
connectivity to other neighbouring switches.

On Dell leaf switches with Virtual Link Trunking (VLT, aka MLAG), the
inter-switch link between leaf switch pairs is also configured with an
IP point to point link. This ends up getting used more than you might
expect.

BGP underlay
------------

In order to stitch together these individual point to point IP fabric
links, we use a BGP control plane to exchange routing information
between the switches. This allows each switch to reach (via L3) not only
its immediate neighbours, but any of their neighbours (and so on). Wait,
we could build an Internet out of this...

For the BGP underlay, each switch establishes a BGP session with each of
its immediate neighbours.

Example on Dell OS10 from a Leaf:

::

   # show ip bgp summary 
   BGP router identifier 172.1.0.0 local AS number 65001
   Neighbor                                      AS          MsgRcvd     MsgSent     Up/Down           State/Pfx      
   172.0.0.1                                   65101         22834       22842       1w:6d:18:55:29    34 

The neighbour's IP address (172.0.0.1) is the fabric link partner's IP.

The BGP router identifier (172.1.0.0) is unique to each switch, and
should be a separate IP range from the fabric links. On Dell OS10
switches this is assigned to a loopback device as a /32 IP address.

The Autonomous System (AS) number (spine: 65101, leaf: 65001) may be
assigned to multiple devices. Dell provides various different reference
configurations which use a single shared AS, or multiple AS. It's not
clear to me why you would choose one or another approach.

One thing that may not be immediately obvious is that BGP within an AS
is internal BGP (iBGP), whereas between different AS it is external BGP
(eBGP).

BGP neighbour info on a Dell OS10 system:

::

   # show ip bgp neighbors 
   BGP neighbor is 172.0.0.1, remote AS 65101, local AS 65001   external link
     
     BGP version 4, remote router ID 172.1.0.1
     BGP state ESTABLISHED, in this state for 1 weeks 6 days 19:04:49
     Last read 01:18:01 seconds
     Hold time is 180, keepalive interval is 60 seconds
     Configured hold time is 180, keepalive interval is 60 seconds
     Fall-over disabled
     
     Received 22845 messages
       1 opens, 0 notifications, 15 updates
       22829 keepalives, 0 route refresh requests
     Sent 22854 messages
       1 opens, 0 notifications, 17 updates
       22836 keepalives, 0 route refresh requests
     Minimum time between advertisement runs is 30 seconds
     Minimum time before advertisements start is 0 seconds
     
     Capabilities received from neighbor for IPv4 Unicast:
       MULTIPROTO_EXT(1)
       ROUTE_REFRESH(2)
       CISCO_ROUTE_REFRESH(128)
       4_OCTET_AS(65)
     Capabilities advertised to neighbor for IPv4 Unicast:
       MULTIPROTO_EXT(1)
       ROUTE_REFRESH(2)
       CISCO_ROUTE_REFRESH(128)
       4_OCTET_AS(65)
     Prefixes accepted 34, Prefixes advertised 36
     Connections established 1; dropped 0
     Last reset never
     For address family: IPv4 Unicast
       Allow local AS number 0 times in AS-PATH attribute
      Prefixes ignored due to:
       Martian address  0, Our own AS in AS-PATH 0
       Invalid Nexthop  0, Invalid AS-PATH length 0
       Wellknown community 0, Locally originated 0
     
     Local host: 172.0.0.0, Local port: 179
     Foreign host: 172.0.0.1, Foreign port: 44058

We're looking for a BGP state of ESTABLISHED.

Here is a route table on Dell OS10:

::

   # show ip bgp
   BGP local RIB : Routes to be Added , Replaced , Withdrawn 
   BGP local router ID is 172.1.0.0
   Status codes: s suppressed, S stale, d dampened, h history, * valid, > best
   Path source: I - internal, a - aggregate, c - confed-external,
   r - redistributed/network, S - stale
   Origin codes: i - IGP, e - EGP, ? - incomplete
             Network                                 Next Hop                                Metric         LocPrf         Weight         Path           
   *         172.0.0.0/31                            172.0.0.1                               0              100            0              65001 ?
   *         172.0.0.0/31                            172.2.0.5                               0              100            0              65001 ?
   *>r       172.0.0.0/31                            0.0.0.0                                 0              100            32768           ?

At this point it should be possible to ping the fabric IP address of any
switch in the network.

On Dell OS this is configured as follows:

::

   router bgp 65001
    router-id 172.1.0.0
    !
    address-family ipv4 unicast
     redistribute connected
    !
    neighbor 172.0.0.1
     remote-as 65101
     no shutdown
     !
     address-family ipv4 unicast
      no sender-side-loop-detection
    !

BGP-EVPN overlay
~~~~~~~~~~~~~~~~

The MP-BGP overlay is used to share VXLAN connectivity information
between switches.

On Dell OS10 (from a leaf):

::

   # show ip bgp l2vpn evpn summary 
   BGP router identifier 172.1.0.0 local AS number 65001
   Neighbor                                      AS          MsgRcvd     MsgSent     Up/Down           State/Pfx      
   172.1.0.1                                     65101       29100       33582       1w:6d:19:15:50    295

This may appear similar to the underlay BGP summary, however here the
neighbours are using the per-switch BGP router ID. This IP is now
reachable across the IP fabric. Again, each switch establishes a session
with its immediate neighbours.

BGP neighbour info on a Dell OS10 system:

::

   # show ip bgp l2vpn evpn neighbors 
   BGP neighbor is 172.1.0.1, remote AS 65101, local AS 65001   external link
     
     BGP version 4, remote router ID 172.1.0.1
     BGP state ESTABLISHED, in this state for 1 weeks 6 days 21:59:35
     Last read 00:11:56 seconds
     Hold time is 180, keepalive interval is 60 seconds
     Configured hold time is 180, keepalive interval is 60 seconds
     Fall-over disabled
     EBGP multihop enabled, multihop TTL set to 4
     
     Received 39322 messages
       2 opens, 2 notifications, 21181 updates
       18137 keepalives, 0 route refresh requests
     Sent 32041 messages
       5 opens, 0 notifications, 12303 updates
       19733 keepalives, 0 route refresh requests
     Minimum time between advertisement runs is 30 seconds
     Minimum time before advertisements start is 0 seconds
     
     Prefixes accepted 270, Prefixes advertised 163
     Connections established 2; dropped 2
     Closed by neighbor sent 1 weeks 6 days 21:59:50 ago
     Local host: 172.1.0.0, Local port: 41483
     Foreign host: 172.1.0.1, Foreign port: 179

Again, we're looking for a state of ESTABLISHED. At Habrok we saw the
BGP session getting to ESTABLISHED, then sometimes flapping after 3
minutes. This is the default hold time, and would happen when a large
BGP update occurred, due to an MTU blackhole on the network path (the
inter-switch link).

So far we have not configured any VXLANs to share information about.
Let's fix that.

VXLANs
------

If we return to our mental model of each switch as an isolated layer 2
Ethernet network, consider connecting up those isolated networks with a
series of overlay networks, such that a host in VLAN A on switch 1 again
has direct connectivity to a host in VLAN A on switch 2. We can do this
using VXLANs. These overlays, or tunnels, are used to encapsulate a
layer 2 packet within a VXLAN UDP packet. This allows the packet to
traverse a network with only layer 3 connectivity, such as our underlay
fabric.

We must create a VXLAN network on each switch that maps to a VLAN.

On a Dell OS10 system, here is one such VXLAN network:

::

   # show virtual-network 10016
   Codes: DP - MAC-learn Dataplane, CP - MAC-learn Controlplane, UUD - Unknown-Unicast-Drop
   Virtual Network: 10016
      Members:
         VLAN 16: port-channel1000
      VxLAN Virtual Network Identifier: 10016
         Source Interface: loopback0(172.2.0.0)
         Remote-VTEPs (flood-list):

In this case we have VXLAN VNI 10016, which maps to VLAN 16. The source
interface is loopback0, which we have configured with a /32 IP address
for the VTEP. In an MLAG scenario , this IP address is shared between
each leaf switch pair. This IP address is used as the source and
destination for the outer VXLAN UDP packet.

Currently, there are no remote VTEPs.

EVIs
----

EVPN Instances (EVIs) are the missing link between the EVPN BGP control
plane and the VXLAN networks - they define which VXLAN networks will be
shared via EVPN BGP, and with which switches.

::

   # show evpn evi 10016
     
   EVI : 10016, State : up
     Bridge-Domain       : Virtual-Network 10016, VNI 10016
     Route-Distinguisher : 1:172.2.0.0:10016
     Route-Targets       : 0:65001:10016 both, 0:65101:10016 import
     Inclusive Multicast : 172.2.0.1
     IRB                 : Disabled

On Dell OS10 switches there is an "auto evi" mode, which automatically
adds an EVI for each VXLAN. However this doesn't work with the multiple
AS topology used at Habrok.

The route distinguisher (RD) is an ID for routes shared by this switch.
The Route Targets (RT) are AS numbers of other switches. Routes can be
exported, imported, or both. Inclusive multicast defines the list of
VTEPs to be included in a multicast group for BUM traffic. IRB is
Integrated Routing and Bridging (IRB), which we'll get onto.

Now that we have an EVI configured for our VXLAN, we now see EVPN
"routes" for MAC addresses:

::

   *         Route distinguisher: 172.23.62.133:10016 VNI:10016
   [2]:[0]:[48]:[16:7f:06:fb:02:47]:[0]:[0.0.0.0]/280          172.23.62.133                 0              100            0              65103 65005 ?

The most common type of route is type 2, and this defines MAC address
routes. Each EVI shares MAC addresses in its local MAC table for the
VLAN with other EVPN switches, avoiding the "flood and learn" behaviour
of a static VXLAN configuration. This means that a MAC address lookup on
switch A will now potentially include remote VTEPs, as well as local
interfaces.

Integrated Routing and Bridging (IRB)
-------------------------------------

IRB can be used to perform routing in a distributed manner, across the
fabric. Typically, each leaf switch is configured as a router, and will
route on ingress to the destination VXLAN.

Resources
---------

This explainer series by nullzero is very helpful in building up the
details in the picture. Here's the first part:
https://www.nullzero.co.uk/aruba-aos-cx-evpn-vxlan/
