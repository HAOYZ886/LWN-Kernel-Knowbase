---
title: Checksum offloads and protocol ossification
url: https://lwn.net/Articles/667059/
date: "December 8, 2015"
category: "Networking-Protocols"
author: "By Jonathan Corbet December 8, 2015"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
December 8, 2015

Given the processing requirements for high-speed networking, it is not surprising that there is interest in offloading some of that work to dedicated hardware. Linux has always carefully limited the support provided for such offloading, though; it has been just over ten years since support for TCP offload engines was [definitively blocked](<https://lwn.net/Articles/148697/>) from entering the Linux network stack. That rejection was driven by a number of concerns, with a reluctance to entrust network-protocol processing to closed-source, unextendable, unfixable software being near the top of the list. Nearly ten years later, offload engines are again the topic of fierce discussion. The hardware has changed, but the concerns have not; indeed, some of the problems being worked around now show why those concerns were valid in the first place. 

#### Encapsulation and ossification

Recent years have seen a large increase in the use of tunneling protocols, especially those that use UDP as a mechanism to encapsulate and transport packets for other protocols. Tunneling has a lot of attractive features; it allows the creation of virtual private networks on the large scale, and virtual overlay networks to support virtualized systems on the smaller scale. The choice of UDP as the transport protocol may not seem entirely obvious until one takes an important fact into account: of all the protocols out there, only two, TCP and UDP, are widely supported by network routers and protocol offload engines in network interfaces. Tunneling protocols use UDP because they have to if they are to get the performance they need. 

Full-protocol offloading, as was discussed in 2005, is generally not present in current hardware; if nothing else, the lack of Linux support for that feature made it unappealing for hardware manufacturers. But that does not mean that modern network cards lack offload support; it is just more specialized. Most reasonable hardware can handle the calculation of packet checksums (a relatively expensive operation on the host), packet segmentation, and more. These cards also support performance-enhancing techniques like [receive-side scaling](<https://www.kernel.org/doc/Documentation/networking/scaling.txt>) and [equal-cost multipath routing](<https://en.wikipedia.org/wiki/Equal-cost_multi-path_routing>) — but only for a small subset of protocols. 

Advanced hardware support has some obvious appeal, but it also has the effect of setting protocols into stone — a process that is becoming known as "protocol ossification." We increasingly find ourselves on an Internet that can only manage TCP and UDP, and relatively unchanging versions of TCP and UDP at that. A great deal of the work behind the development of the [multipath TCP protocol](<https://lwn.net/Articles/544399/>) has gone into making it look like regular TCP so that it can be routed efficiently and not be blocked by middleboxes that don't understand it. The [Stateless Transport Tunneling protocol](<https://tools.ietf.org/html/draft-davie-stt-01>) takes things further by disguising a stateless protocol behind (what looks like) TCP packets. We have slowly optimized ourselves into a situation where the development and deployment of new protocols (or even significant enhancements to existing protocols) is increasingly difficult; even well defined protocols like [SCTP](<https://en.wikipedia.org/wiki/Stream_Control_Transmission_Protocol>) and [DCCP](<https://lwn.net/Articles/149756/>) are hard to deploy in real-world settings. 

#### Tunneling offload

With regard to tunneling in particular, some hardware has, over the years, grown support for some tunneling protocols. The kernel now has support for interfaces that can offload some aspects of [VXLAN](<https://en.wikipedia.org/wiki/Virtual_Extensible_LAN>), for example. But that support doesn't extend to other tunneling protocols, even VXLAN extensions like [VXLAN-GPE](<https://datatracker.ietf.org/doc/draft-quinn-vxlan-gpe/>). It is an attempt to expand this support that has raised some opposition in the networking community. 

Anjali Singhai Jain, representing Intel, recently posted [a patch set](<https://lwn.net/Articles/667067/>) meant to generalize the kernel's support for UDP tunnels and, in particular, add offload support for the [GENEVE](<https://tools.ietf.org/html/draft-gross-geneve-00>) tunneling protocol to the i40e driver. To that end, Anjali removed the VXLAN-specific low-level network device operations, replacing them with more generic versions that would have the actual encapsulation type passed into them. Nobody will mourn the VXLAN-specific code, but replacing that code with multiplexer code that handles a multitude of tunneling protocols was not seen as helping the cause. 

One might wonder how offload comes into play with tunneling, given that the whole point is hiding a network packet in another packet's payload where the network interface shouldn't care about it. Once again, it comes down to checksum generation and verification. Packets tunneled within another protocol still carry their own checksums, and those checksums must be valid. If the hardware cannot generate or verify that internal checksum, then the host system must do it. One way to enable checksum offloading is for the hardware to understand both the tunneling protocol and the protocol being tunneled so that it can manage the checksums for both. That has been, for the most part, the approach that has been taken by hardware manufacturers thus far. 

The networking developers, though, do not want to see a world where each network interface has support for its own special subset of tunneling protocols, and where the kernel has to cope with all of them. They are, instead, developing a simpler, protocol-independent mechanism by which the hardware can support any protocol with checksum offloading. Those who want the details can look at [this paper by Tom Herbert [PDF]](<http://people.netfilter.org/pablo/netdev0.1/papers/UDP-Encapsulation-in-Linux.pdf>) and [this patch set](<https://lwn.net/Articles/666178/>). The executive summary of the technique for the rest of us would be: 

  * For packet transmission, things are straightforward: the hardware accepts, with each packet, a range of data to be checksummed and the location within the packet where the checksum should be written. The tunneling protocol can set that range to indicate the encapsulated packet and its own checksum field, and everything just works. (Note that, in situations like this, the outer UDP checksum is just left as zero, since it is redundant anyway). 

  * For packet reception, verifying the inner checksum requires an awareness of the tunneling protocol itself. Rather than bury such awareness in the hardware, though, the networking developers would rather that the interface simply calculate a checksum for the packet as a whole. It is then a relatively cheap operation for the kernel to "subtract out" the portion of the checksum corresponding to the outer headers and arrive at the correct inner checksum. Hardware that operates in this way can support any tunneling protocol without a lot of protocol-specific code at the driver level. There have also been [suggestions](<https://lwn.net/Articles/667163/>) that Berkeley Packet Filter (BPF) programs could be used for more complicated processing. 

Given that the kernel developers want to go in this direction, it is not surprising that Tom [pushed back against Anjali's patches](<https://lwn.net/Articles/667071/>), saying that they encourage manufacturers to implement more protocol-specific awareness into their interfaces rather than implementing the protocol-independent mechanism he would like to see. Networking maintainer David Miller was [quick to state](<https://lwn.net/Articles/667072/>) that he would not be merging code that takes things in the wrong direction. 

#### What happens next?

The problem with that position, of course, is that cards with, for example, GENEVE support are shipping now, and users will want to make use of them. At best, blocking support for that hardware condemns those users to slower tunneling performance. A worse scenario, [put forward](<https://lwn.net/Articles/667073/>) by Alexander Duyck, is that users would move to out-of-tree binary drivers or user-space protocol stacks — though, it must be said, the rejection of TCP offload engine support ten years ago created no such stampede. In any case, keeping users from getting the most out of their hardware has never really been the Linux way, so it's not surprising that David did eventually [moderate his position](<https://lwn.net/Articles/667075/>) a bit: 

Pushing back is different from blocking entirely. 

That means I'm going to be very difficult and make a lot of noise until I see the message has seeped in. 

It doesn't mean that I won't allow a means to use existing hardware offloads. You'll just have to bear with me, be patient, and survive my tantrum on this matter. 

He would appear to be serious about waiting until he sees that the message has been received, though. 

Meanwhile, Anjali's current patches seem unlikely to go in without some changes, even after David's tantrum has been duly survived. The attempt to generalize low-level tunneling support in the networking stack will probably have to be dropped in favor of a set of GENEVE-specific device operations. Doing things that way limits the temptation to expand support to other tunneling protocols; it also makes it easy for the tunneling software itself to determine whether hardware support is available. Making those changes will be straightforward enough; convincing the networking maintainer that the hardware designers have heard his complaint may take a little longer. 

An attempt to push hardware designers in a different direction may seem a bit like throwing Linux's weight around. But it is an important exercise, not only for the long-term maintenance of the kernel, but also for the net as a whole. If hardware can be made to support important offloading features with less protocol awareness then, maybe, the problem of protocol ossification can be softened up a bit. The net must certainly evolve in the coming years; adding flexibility to our networking systems is one of the most important ways of making that possible.
