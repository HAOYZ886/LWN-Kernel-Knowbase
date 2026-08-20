---
title: "Netconf discussions, part 1"
url: https://lwn.net/Articles/674943/
date: "February 10, 2016"
category: "Networking-Networking summits"
author: "By Nathan Willis February 10, 2016 Netdev/Netconf"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Nathan Willis**  
February 10, 2016

* * *

[Netdev/Netconf](<https://lwn.net/Archives/ConferenceByYear/#2016-Netdev>)

For two days prior to the [Netdev 1.1](<http://www.netdevconf.org/1.1/>) conference in Seville, Spain, kernel networking developers gathered for [Netconf](<http://vger.kernel.org/netconf2016.html>), an informal summit to discuss recent issues face to face and to hash out approaches to take for upcoming work. What follows is a recap of how those discussions progressed on the first day of the event; an account of the second day is forthcoming. 

#### Checksums

The first issue on the table was checksum offloading, in particular with respect to tunneling protocols. Since we first [looked at](<https://lwn.net/Articles/667059/>) checksum offloading in December, an updated [patch set](<https://lwn.net/Articles/671457/ >) has been posted. Checksums allow the kernel to verify that a packet was unaltered during transit; computing them is a task that it would be nice to offload to hardware. For tunneling there are two checksums to cope with: inner checksums cover a packet encapsulated within another protocol (the tunneled payload) while outer checksums cover the entire packet, including the payload and its wrapper. 

In short, Local Checksum Offloading (LCO) allows the kernel to compute the outer checksum of an outgoing packet in software, while offloading the job of calculating the inner checksum to the hardware device. This is a bit of clever sleight-of-hand; it can be done efficiently since the outer checksum is merely the checksum of the outer header--a substantially smaller set of bytes, which the kernel is already processing in memory--added to the inner checksum in one's-complement arithmetic. For incoming packets, most network interfaces are capable of offloading only the outer checksum but, again, the kernel can derive the inner checksum relatively easily by processing only the outer header. 

Several factors continue to make checksum offloading a convoluted issue from the kernel's perspective, though. As Alex Duyck explained, some newer networking devices are designed to compute the checksum of the innermost _recognized_ packet type. If the inner packet is [VXLAN](<https://en.wikipedia.org/wiki/Virtual_Extensible_LAN>), the device will compute the inner checksum; if not, the device will compute the outer checksum instead. And additional work is required to get checksum offloading to play nicely with TCP segmentation offload (TSO), where the network hardware splits up packets before sending them out over the outgoing interface. Thus, the device performing the hardware checksum computations must be told the proper offsets at which to begin the checksum calculations. 

In addition to the implementation details, which at this point seem to be relatively well-understood, the need to improve the documentation of the checksum-offloading features was raised. As Jesse Brandeburg pointed out based on his experiences at Intel, it can be rather difficult for the authors of new device drivers to make sense out of all of the flags in use. Tom Herbert commented that it is probably incorrect to call many of the calculations involved "checksums" to begin with, since they are, in fact, cyclic redundancy checks (CRCs). It does not look like _that_ terminology change, however, should be expected any time soon. 

#### IPv4 containers on IPv6 hosts

Next, Thomas Graf proposed creating a new socket type that would enable an IPv6-only system to support containerized applications that are, internally, IPv4-aware only. The idea would be that a containerized application explicitly asking for an IPv4 address would be given a socket that was transparently bound to an external IPv6 address. Doing so would do away with the need to set up an IPv4 routing table and other overhead--which is required, currently, whenever an application explicitly asks for an IPv4 address on an IPv6 system. 

Not everyone was persuaded that the approach was a worthwhile idea; Shrijeet Mukherjee, for example, asked whether setting up a 6to4 tunnel instead would suffice. The end goal, Graf said, was to make containerized applications as lightweight as possible, thus making life simpler for systems that have to cope with ancient containers that are no longer updated (or whose developers refuse to transition to proper IPv6 support). There was enough interest to convince Graf to pursue the idea further, though, and he agreed to work on a patch and submit it for comment. 

#### SCTP

Eric Dumazet asked the group how important it was to support the [Stream Control Transmission Protocol](<https://en.wikipedia.org/wiki/Stream_Control_Transmission_Protocol>) (SCTP)--the implication being that SCTP is rarely used and, therefore, not worth expending excess effort on. Networking subsystem maintainer David Miller agreed that it was difficult to know how many users there were of the SCTP code (as is true of other protocols) and floated the idea of asking Linux distributions to gather anonymized usage statistics--with end-user permission--much as they already collect statistics on package installations. If such information were available, he said, it could lead to further investment in networking features; it is difficult to get traction on work like SCTP support unless that work is funded. Anecdotally, many SCTP users seem to use proprietary, out-of-tree SCTP stacks, so this would seem to be an area where there is a need for more investment. 

#### Virtual routing and forwarding

David Ahern presented an update on Virtual Routing and Forwarding (VRF), a feature used to create a virtual layer-3 (IP) routing domain in the network stack. Basic support for IPv4 VRFs was added in kernel 4.3, and IPv6 support in kernel 4.4. Several usability issues remain, however. Creating the routing-table rules required for the VRF is still a cumbersome process. Miller has already rejected a patch that would have allowed a VRF driver to automatically create "simple" routing rules; Ahern noted that there were other possible solutions, such as defining a `vrf` subcommand for the `ip` utility. 

A more serious usability problem is that every time a VRF's enslaved network devices (that is, the individual interfaces that are combined into the VRF) are brought down and then back up, they lose their IPv6 addresses. The cycling is required, since it is used to flush the neighbor cache and routes. But, technically, flushing the entire interface is not required--only wiping the cache and route data. Ahern said that a patch was being developed to perform more of a "pause/resume" or "soft down" operation. Miller noted, however, that the problem is not limited to VRF; ultimately one would like to be able to cycle an interface without losing IPv6 addresses. Arriving at a fix for that underlying problem is going to take considerable effort, Miller said, but he encouraged Ahern to proceed with the VRF patch anyway, "and let's see what happens." 

Ahern also listed several features missing from VRF, starting with the ability to run tasks in a VRF context. Control groups seem like the right approach for implementing that feature, but Tejun Heo objected to that idea when Ahern sent [an RFC](<https://lwn.net/Articles/670190/>) in January. Miller noted that Heo has spent quite a bit of time cleaning up control groups and is likely to balk at uses that violate the new model he is moving toward. Miller said he would attempt to smooth the path to acceptance in that regard, but suggested that Ahern look at alternative solutions in case he is unable to persuade Heo. 

Ahern also noted that the VRF developers would like more netfilter hooks on transmit and receive paths, and would like to be able to bind a socket to an enslaved device. Finally, he noted that there is a major roadblock to using VRF with [switchdev](<https://www.kernel.org/doc/Documentation/networking/switchdev.txt>): currently, switchdev disables layer-3 offload on a system if _any_ IP rules are installed in the Forwarding Information Base (FIB). That makes it impossible to use VRF with switchdev's hardware-offloading capabilities. He suggested that the "overly cautious" ban on IP rules be relaxed, perhaps allowing rules for non-hardware (i.e., virtual) ports or allowing the "simple" rules needed for VRF. He conceded, though, that it could be challenging to find a solution. 

#### Header (un)alignment

Herbert then discussed how to approach byte-alignment problems. It is a fundamental issue that the headers on a network packet do not always arrive conveniently aligned on four-byte boundaries as the kernel would prefer, thus decreasing performance. In some cases, the misalignment is entirely predictable--for example, in tunneling, when the outer IP header is stripped off, whatever is inside will be offset by two bytes. While several approaches to alleviating the performance impact have been raised (shifting headers with `memmove()`, for example), they do not attack the underlying issue head-on. 

Miller advocated taking a moderate, fix-the-major-cases approach rather than attempting an attack on the root of the problem that could prove disruptive. On Sparc systems, for instance, unaligned headers trigger scores of kernel warnings. While it might be possible to wrap every access on Sparc in a `get_unaligned()` macro, he said, that solution would certainly be rejected by Linus. In the end, Duyck posted [a short patch](<http://marc.info/?l=linux-netdev&m=145502729402027&w=2>) that will correct unaligned headers in the Generic Routing Encapsulation (GRE) code, which Herbert signed off on. 

#### Documentation

Jeff Kirsher proposed refreshing the skeleton driver (the "outline" that developers are intended to use when writing a new network device driver), which has not been updated to show new features like offloading. In addition, he suggested that the documentation could be improved in several areas--code comments most obviously, but perhaps more papers as well, like Herbert's 2015 white paper on checksum offloading. He also asked about adding per-queue statistics. Miller did not find the idea appealing, since network interface dumps are "enormous" already. But he noted that perhaps "extended statistics" could be made available with filter bits; there are several other cases where more statistics need to be exposed, such as network interface cards (NICs) that include built-in switches. 

#### TCP performance

Dumazet discussed progress on the new TCP congestion algorithm in use at Google. Though it is working well enough so far, there is concern over the fact that it is not currently possible to tell if a packet was dropped because of a bottleneck or because of the action of a firewall (or some other interfering device) somewhere along the route. He also reported that work was still underway exploring random packet spraying (RPS), which gives up attempting to force all TCP packets in a stream to follow the exact same path. The idea of using RPS was first floated after [a paper [PDF]](<https://engineering.purdue.edu/~ychu/publications/infocom13_pktspray.pdf>) on the subject was published by researchers at Purdue University. The challenge is how to reorder TCP packets that arrive on different paths; using flow labels was discussed briefly, but there may be no simple solution. 

#### Wireless

Johannes Berg gave a brief report on recent work on the wireless side. At present, major effort is being directed at receive side scaling (RSS), he said, splitting up everything serialized in order to make it parallel. Currently, the effort is focused on implementing RSS on a single TCP stream; the developers would like to support aggregating TCP streams, but real world experience seems to indicate that most common wireless devices (e.g., mobile phones) will use only one stream at a time anyway. 

Miller asked if anything is "getting in the way," to which Berg replied that the wireless developers would like to have more transmit queues, to better support [multi-user MIMO](<https://en.wikipedia.org/wiki/Multi-user_MIMO>). That feature allows a wireless access point to simultaneously stream to different devices via separate antennas (on the same frequency band), which works if the receiving devices are physically far apart, and provides better overall throughput. Modern access points have as many as eight antennas--all sharing a single network interface. The kernel, therefore, needs to be able to manage multiple transmit queues to keep the antennas filled. Work is underway on supporting the feature, although only on Intel hardware so far. 

#### Statistics

Jamal Hadi Salim closed out the first day by proposing a new message type for netlink that could be used to subscribe to periodic status updates. The use case is recording statistics in high volume (say, once every one to five seconds) but, specifically, statistics that are intended to profile system performance. Collecting data in this fashion is different from subscribing to other netlink updates, where one generally wants to hear about an event immediately. There did not seem to be any serious objection to the idea, so a new "stats" message would appear to be a possibility. 

With that, the discussion wound down for day one. 

_[The author would like to thank the Netconf and Netdev organizers for travel assistance to Seville.]_
