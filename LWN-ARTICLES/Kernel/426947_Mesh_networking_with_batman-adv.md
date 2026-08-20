---
title: "Mesh networking with batman-adv"
url: https://lwn.net/Articles/426947/
date: "February 8, 2011"
category: "Networking-Protocols"
author: "By Jonathan Corbet February 8, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
February 8, 2011

Your editor has recently seen two keynote presentations on two continents which, using two very different styles, conveyed the same message: the centralization of the Internet and the services built on it has given governments far too much control. Both speakers called for an urgent effort to decentralize the net at all levels, including the transport level. An Internet without centralized telecommunications infrastructure can be hard to envision; when people try the term that usually comes out is "mesh networking." As it happens, the kernel has a mesh networking implementation which made the move from the staging tree into the mainline proper in 2.6.38. 

Mesh networking, as its name implies, is meant to work via a large number of short-haul connections without any sort of centralized control. A proper mesh network should configure itself dynamically, responding to the addition and removal of nodes and changes in connectivity. In a well-functioning mesh, networking "just happens" without high-level coordination; such a net should be quite hard to disrupt. What the kernel offers now falls somewhat short of that ideal, but it is a good demonstration of how hard mesh networking can be. 

The "Better Approach To Mobile Ad-hoc Networking" (BATMAN) protocol is described in [this draft RFC](<http://tools.ietf.org/html/draft-wunderlich-openmesh-manet-routing-00>). A BATMAN mesh is made up of a set of "originators" which communicate via network interfaces - normal wireless interfaces, for example. Every so often, each originator sends out an "originator message" (OGM) as a broadcast to all of its neighbors to tell the world that it exists. Each neighbor is supposed to note the presence of the originator and forward the message onward via a broadcast of its own. Thus, over time, all nodes in the mesh should see the OGM, possibly via multiple paths, and thus each node will know (1) that it can reach the originator, and (2) which of its neighbors has the best path to that originator. Each node maintains a routing table listing every other node it has ever heard of and the best neighbor by which to reach each one. 

This protocol has the advantage of building and maintaining the routing tables on the fly; no central coordination is needed. It should also find near-optimal routes to each. If a node goes away, the routing tables will reconfigure themselves to function in its absence. There is also no node in the network which has a complete view of how the mesh is built; nodes only know who is out there and the best next hop. This lack of knowledge should add to the security and robustness of the mesh. 

Nodes with a connection to the regular Internet can set a bit in their OGMs to advertise that fact; that allows others without such a connection to route packets to and from the rest of the world. 

The original BATMAN protocol uses UDP for the OGM messages. That design allows routing to be handled with the normal kernel routing tables, but it also imposes a couple of unfortunate constraints: nodes must obtain an IP address from somewhere before joining the mesh, and the protocol is tied to IPv4. The [BATMAN-adv](<http://www.open-mesh.org/wiki/batman-adv>) protocol found in the Linux kernel has changed a few things to get around these problems, making it a rather more flexible solution. BATMAN-adv works entirely at the link layer, exchanging non-UDP OGMs directly with neighboring nodes. The routing table is maintained within a special virtual network device, which makes all nodes on the mesh appear to be directly connected via that virtual interface. Thus the system can join the mesh before it has a network address, and any protocol can be run over the mesh. 

BATMAN-adv removes some of the limitations found in BATMAN, but readers who have gotten this far will likely be thinking of the limitations that remain. The flooding of broadcast OGMs through the net can only scale so far before a significant amount of bandwidth is consumed by network overhead. The protocol trims OGMs which are obviously not of interest - those which describe a route which is known to be worse than others, for example - but the OGM traffic will still be significant if the mesh gets large. The routing tables will also grow, since every node must keep track of every other node in existence. The overhead for these tables is probably manageable for a mesh of 1,000 nodes; it is probably hopeless for 1,000,000 nodes. Mobile devices - which are targeted by this protocol - are especially likely to suffer as the table gets larger. 

Security is also a concern in this kind of network. Simple bandwidth-consuming denial of service attacks would seem relatively straightforward. Sending bogus OGMs could cause the size of routing tables to explode or disrupt the routing within the mesh. A more clever attack could force traffic to route through a hostile node, enabling man-in-the-middle exploits. And so on. The draft RFC quickly mentions some of these issues, but it seems clear that security has not been a major design goal. 

So it would seem clear that BATMAN-adv, while interesting, is not the solution to the problem of an overly-centralized network. It could be a useful way to extend connectivity through a building or small neighborhood, but it is not meant to operate on a large scale or in an overtly hostile environment. The bigger problem is a hard one to solve, to say the least. The experience gained with protocols like BATMAN-adv may will prove valuable in the search for that solution, but there is clearly some work to be done still.
