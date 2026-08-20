---
title: Let them run CAKE
url: https://lwn.net/Articles/758353/
date: "June 27, 2018"
category: Networking
author: "By Jonathan Corbet June 27, 2018"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
June 27, 2018

While there has been quite a bit of work on various aspects of networking performance, including bufferbloat reduction, queue management, and more, much of that work has been oriented toward the needs of high-end users. But there is more to the Internet than data centers and high-speed links. A large number of Internet-connected devices can be found behind consumer-level routers on relatively slow broadband links. For some time, a group of developers has been working on the "Common Applications Kept Enhanced" (CAKE) queuing discipline, which is aimed directly at the needs of those users. 

Home networks face a number of challenges not found in many commercial settings. Bufferbloat can cause significant latencies, but can often be difficult to address by end users. The links themselves are relatively slow, and they are often highly asymmetric — download speeds can be an order of magnitude higher than upload speeds. The result of all this can be significant domestic tension when, for example, one household member is pushing a large Git tree while the other is engaged in a high-stakes raid. Given the special features of home networks, it would seem to make sense to tune the behavior of the network stack to match. That is where CAKE is meant to come in. 

#### CAKE ingredients

The CAKE patches, posted by Toke Høiland-Jørgensen (though the principal author is said to be Jonathan Morton), are an attempt to better meet the needs of home-network users; the [18th revision of this patch set](<https://lwn.net/Articles/756171/>) was posted at the end of May. CAKE takes the form of a queuing discipline, meaning that it sits between the higher-level protocol code and the network interface and decides which packets to dispatch at any given time. It has four different components designed to make things work on home links. 

The first of those is a rate-based bandwidth shaper. LWN readers are relatively likely to have a home router running a recent kernel and entirely free drivers; on such systems, the problems with bufferbloat have mostly been solved. Others, though, are not so lucky. There is often buffering to be found within proprietary drivers or the hardware itself that cannot be fixed just by installing a current OpenWrt release. And, in any case, there may be buffering problems in external components — a cable modem, for example — that are not under the user's control at all. 

In such cases, sending too much data through the link at any given time will almost certainly lead to excessive buffering and the resulting latency problems. There is still, though, one way to avoid such problems: don't send data faster than the upstream link can carry it. That is where the bandwidth shaper comes in. It regulates outgoing traffic to cap it at just below the bandwidth that the link can handle, preventing buffering at that link. In essence, it takes control of buffering away from the downstream components, solving bufferbloat problems in settings where the code itself cannot be changed. 

One potential problem with this kind of shaping is that it can, if configured conservatively, waste a portion of the available bandwidth. The shaper goes to great lengths to try to account for all of the overhead that will be applied to packets over the link (including things like DOCSIS framing), with the result that, it is claimed, the speed limit can be configured to over 99% of the actual link speed. What seems to be missing, though, is an automatic way to determine what the link speed actually is. 

For the parts of the system that _are_ under the control of the networking code, some sort of queue-management algorithm is needed to prevent the overfilling of buffers. CAKE includes a variant of the [FQ-CoDel algorithm](<https://tools.ietf.org/html/draft-ietf-aqm-fq-codel-06>) (called "Cobalt") for that purpose. Among other things, FQ-CoDel performs packet scheduling to ensure fairness between the various flows (connections) that are transmitting at any given time. Cobalt adds to that, though, in that it can also ensure fairness between the various hosts that are sending packets through the router. If host H1 has a single connection running, while H2 has four, FQ-CoDel will allocate 20% of the available bandwidth to each, with the result that H2 is able to use 80% of the total. Cobalt, instead, will give 50% to the flow from H1, then allocate 12.5% to each of the flows from H2. This feature can help to ensure that all devices on the network get reasonable access to the available bandwidth. 

The [differentiated services](<https://en.wikipedia.org/wiki/Differentiated_services>) (or "DiffServ") specification uses a field in the IP packet header to classify the data contained within that packet. Some packets can be marked as being high priority or latency sensitive, while others might be low-priority bulk traffic. CAKE implements DiffServ in the bandwidth shaper with a small number of priority-ordered queues. The highest-priority queues are serviced first, but only to a point; the latency-sensitive queue is given a maximum of 25% of the available bandwidth, for example. If a given queue does not use all of its allocation, that bandwidth is naturally made available to the other classes of service. 

This approach to DiffServ is meant to enable priority handling for traffic like video conferencing without letting an abusive host use DiffServ to crowd out all other users. There are several different mappings of DiffServ classes to priorities available in CAKE. 

The last major component of CAKE is ACK filtering. A stream of data flowing in one direction over a TCP connection will generate a corresponding stream of acknowledgment (ACK) packets heading the other way. The ACK traffic is much smaller than the actual data, but it can still reach problematic levels on asymmetric links like those found in many home links. Much of that data will be redundant: if an ACK packet for the first 10,000 bytes is immediately followed by an ACK for the first 20,000 bytes, the first can often be dropped with no ill effect. 

Since CAKE maintains per-flow queues of packets, it is relatively easy for it to tell when a newly queued ACK packet makes an earlier one redundant. Some care must be taken, though, to only drop ACK packets that contain no other data, or bad things will happen. The ACK filtering also will not touch packets that contain unknown headers; that is an attempt to avoid protocol ossification that could break future extensions. 

#### The baking process

CAKE first [appeared](<https://lwn.net/Articles/752777/>) on the netdev list in April; since then it has been, to put it politely, subjected to a great deal of discussion ranging from serious technical criticism to the inevitable [requirement](<https://lwn.net/Articles/758450/>) to put variable declarations in "reverse Christmas tree" ordering. It is probably fair to say that many developers would have given up on getting this code merged by now. Høiland-Jørgensen appears to be a persistent and good-humored developer, though, with the result that this patch set is now up to version 18 and is, with luck, close to being merged. 

Early on, networking developer Eric Dumazet's [reaction](<https://lwn.net/Articles/758447/>) was: ""Oh my god. Cake became a monster"". He questioned a number of things, with [special attention](<https://lwn.net/Articles/758448/>) for the ACK filter which, he said, should be in the TCP stack itself if it exists at all. He has since then [let it be known](<https://lwn.net/Articles/758449/>) that he will indeed be adding ACK filtering to TCP as a whole. Many other issues raised by Dumazet have been fixed in subsequent versions of the patch set. 

One other significant objection has to do with how the host-based shaping works. When network address translation (NAT) is in use — as it often is on home networks — a queuing discipline loses the information about where packets originally came from. To get around that, the CAKE patches reach into the netfilter subsystem to obtain that information and keep it with packets as they pass through the system. This is seen as a layering violation, and it makes CAKE dependent on the netfilter connection-tracking mechanism. Better solutions have not been offered, though, so that feature remains in the current patch set. 

One slice of CAKE that remains lacking is good documentation on how to actually use it. The intent is to create something that is ""simple enough that even an ISP can configure it"", but it is still not entirely straightforward. The [CAKE page](<https://www.bufferbloat.net/projects/codel/wiki/Cake/>) on bufferbloat.net has some information on how to get started, including instructions on how to get it running on an OpenWrt system. Detailed information on CAKE, including performance numbers, can be found in [this paper [PDF]](<https://arxiv.org/pdf/1804.07617.pdf>). With luck, the baking process will be finished soon and we will all be able to have CAKE.
