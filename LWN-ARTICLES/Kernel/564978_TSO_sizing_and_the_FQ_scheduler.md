---
title: TSO sizing and the FQ scheduler
url: https://lwn.net/Articles/564978/
date: "August 28, 2013"
category: Networking
author: "By Jonathan Corbet August 28, 2013"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
August 28, 2013

One might think that, by now, we would have a pretty good idea of how to optimally manage data streams with the TCP protocol. In truth, there still seems to be substantial room for improvement. Larger problems like [bufferbloat](<http://bufferbloat.net/>) have received a lot of attention recently, but there is ongoing work in other aspects of real-world networking as well. A couple of patches posted recently by Eric Dumazet show the kind of work that is being done. 

#### TSO sizing

TCP segmentation offload (TSO) is a hardware-assisted technique for improving the performance of outbound data streams. As a stream of data (a "flow") is transmitted, it must be broken up into smaller segments, each of which fits into one packet. A network interface that supports TSO can accept a large buffer of data and do this segmentation work within the hardware. That reduces the number of operations performed and interrupts taken by the host CPU, making the transmission process more efficient. The use of techniques like TSO makes it possible for Linux systems to run high-end network interfaces at their full speed. 

A potential problem with TSO is that it can cause the interface to dump a large number of packets associated with a single stream onto the wire in a short period of time. In other words, data transmission can be bursty, especially if the overall flow rate for the connection is not all that high. All of those packets will just end up sitting in a buffer somewhere, contributing to bufferbloat and increasing the chances that some of those packets will be dropped. If those packets were transmitted at a more steady pace, the stress on the net as a whole would be reduced and throughput could well increase. 

The simple [TSO automatic sizing patch](<https://lwn.net/Articles/564979/>) posted by Eric (with a Cc to Van Jacobson at a new google.com address) tries to spread out transmissions in just that way. There are two changes involved, the first of which is to make intelligent choices about how much data should be handed to the interface in a single TSO transmission. Current kernels will provide a maximally-sized buffer — usually 64KB — to be transmitted all at once. With the automatic sizing patch, that buffer size is reduced to an amount that, it is estimated, will take roughly 1ms to transmit at the current flow rate. In this way, each transmission will produce a smaller burst of data if the flow rate is not high. 

The other piece of the puzzle is called "TCP pacing"; it is a TCP implementation change intended to set the pace at which packets are transmitted to approximately match the pace at which they can get through the network. The existing TCP flow control mechanisms tell each endpoint how much data it is allowed to transmit, but they don't provide a time period over which that transmission should be spread, so, once again, the result tends to be bursts of packets. TCP pacing ensures that packets are transmitted at a rate that doesn't cause them to pile up in buffers somewhere between the two endpoints. It can, of course, also be used to restrict the data rate of a given flow to something lower than what the network could handle, but that is not the objective of this patch. 

Interestingly, the patch does not actually implement pacing, which would add some significant complexity to the TCP stack — code that does not really need more complexity. All it does is to calculate the rate that should be used, in the hope that some other level of the stack can then enforce that rate. For now, that other part would appear to be [the new "fair queue" packet scheduler](<https://lwn.net/Articles/564825/>). 

#### The FQ scheduler

A packet scheduler is charged with organizing the flow of packets through the network stack to meet a set of policy objectives. The kernel has quite a few of them, including CBQ for fancy class-based routing, [CHOKe](<https://lwn.net/Articles/422477/>) for routers, and a couple of variants on the [CoDel](<https://lwn.net/Articles/496509/>) queue management algorithm. FQ joins this list as a relatively simple scheduler designed to implement fair access across large numbers of flows with local endpoints while keeping buffer sizes down; it also happens to implement TCP pacing. 

FQ keeps track of every flow it sees passing through the system. To do so, it calculates an eight-bit hash based on the socket associated with the flow, then uses the result as an index into an array of red-black trees. The data structure is designed, according to Eric, to scale well up to millions of concurrent flows. A number of parameters are associated with each flow, including its current transmission quota and, optionally, the time at which the next packet can be transmitted. 

That transmission time is used to implement the TCP pacing support. If a given socket has a pace specified for it, FQ will calculate how far the packets should be spaced in time to conform to that pace. If a flow's next transmission time is in the future, that flow is added to another red-black tree with the transmission time used as the key; that tree, thus, allows the kernel to track delayed flows and quickly find the one whose next packet is due to go out the soonest. A single timer is then used, if needed, to ensure that said packet is transmitted at the right time. 

The scheduler maintains two linked lists of active flows, the "new" and "old" lists. When a flow is first encountered, it is placed on the new list. The packet dispatcher services flows on the new list first; once a flow uses up its quota, that flow is moved to the old list. The idea here appears to be to give preferential treatment to new, short-lived connections — a DNS lookup or HTTP "GET" command, for example — and not let those connections be buried underneath larger, longer-lasting flows. Eventually the scheduler works its way through all active flows, sending a quota of data from each; then the process starts over. 

There are a number of additional details, of course. There are limits on the amount of data queued for each flow, as well as a limit on the amount of data buffered within the scheduler as a whole; any packet that would exceed one of those limits is dropped. A special "internal" queue exists for high-priority traffic, allowing it to reach the wire more quickly. And so on. 

One other detail is garbage collection. One problem with this kind of flow tracking is that nothing tells the scheduler when a particular flow is shut down; indeed, nothing _can_ tell the scheduler for flows without local endpoints or for non-connection-oriented protocols. So the scheduler must figure out on its own when it can stop tracking any given flow. One way to do that would be to drop the flow as soon as there are no packets associated with it, but that would cause some thrashing as the queues empty and refill; it is better to keep flow data around for a little while in anticipation of more traffic. FQ handles this by putting idle flows into a special "detached" state, off the lists of active flows. Whenever a new flow is added, a pass is made over the associated red-black tree to clean out flows that have been detached for a sufficiently long time — three seconds in the current patch. 

The result, says Eric, is fair scheduling of packets from any number of flows with nice spacing in situations where pacing is being used. Given that the comments so far have been mostly concerned with issues like white space, it seems unlikely that anybody is going to disagree with its merging. So TCP pacing and the FQ scheduler seem like candidates for the mainline in the near future — though the upcoming 3.12 cycle may be just a bit too near at this point.
