---
title: The CHOKe packet scheduler
url: https://lwn.net/Articles/422477/
date: "January 11, 2011"
category: "Networking-Congestion control"
author: "By Jonathan Corbet January 11, 2011"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
January 11, 2011

A packet on the network typically passes through several machines on the way from its source to its destination. One of those machines (or, more correctly, one of the outbound links from one of those machines) will be the limiting factor on how many packets can traverse that path in a given period of time. If a system tries to send too many packets through the limiting link, the packet queue on the router attached to that link will grow. A growing queue affects other users of that router and will eventually hit its limits, causing packet loss. 

The TCP protocol has, for many years, included congestion control algorithms which attempt to determine the carrying capacity of a path and to avoid exceeding that capacity. These algorithms have successfully prevented a repeat of the meltdowns which plagued the early Internet. But congestion control isn't working as well as it should, for a few reasons. Some TCP implementations are more dutiful than others when it comes to congestion control. An increasing amount of traffic on the net uses other protocols (UDP in particular) which do not have congestion control built into them. Excessive queue sizes in routers ("[bufferbloat](<https://gettys.wordpress.com/bufferbloat-faq/>)") can also disguise congestion problems until it is too late. All of these problems are motivating a search for better ways of controlling congestion. 

The key signal for congestion control on the net is dropped packets; TCP will continue to ramp up its transmit rate until the occasional lost packet makes it clear that the limit has been hit. So the way for a router in the middle of the network to tell a specific sender that it's transmitting too much data is to drop some of that data on the floor. The idea is simple in concept, but it can be harder in practice for a simple reason: network routers can deal with many thousands of packets every second; they cannot afford to spend significant amounts of time on any one of those packets. 

An obvious way for a router to schedule packets would be to maintain a queue for every flow (source/destination pair with port numbers) through the system. Packets could then be dequeued and transmitted with absolute fairness, and, any time the queue for a flow gets too long, packets could be dropped from that queue. Implementing this algorithm would require some complex data structures and a fair amount of processor time, though, so it is not an option for a router which handles a significant amount of traffic. 

An alternative is the [CHOKe algorithm [PS]](<http://www.stanford.edu/~balaji/papers/choke_info.ps>); CHOKe stands either for "CHOose and Kill" or "CHOose and Keep," depending on one's attitude toward the problem. Stephen Hemminger has recently [posted](<https://lwn.net/Articles/422481/>) a CHOKe implementation for Linux, so this seems like a good time to look at how this algorithm works. 

CHOKe is intended for points where multiple flows come together - routers and bridges, primarily. The idea behind CHOKe is to keep the length of transmit queues under control and to penalize flows with excessive traffic while avoiding the need to maintain any sort of per-flow state. To that end, the packet queuing algorithm works essentially like the following. When a packet arrives for a given outbound link, the CHOKe code will: 

  * Calculate a moving average of the length of the queue. The algorithm includes a parameter for the period over which the average is calculated; a longer period will allow longer load spikes before the algorithm starts CHOKing traffic. 

  * If the average queue length is below a minimum watermark, there is no problem with congestion, so the packet will simply be queued and the job is done. 

  * If the queue length is above the minimum, the CHOKe algorithm picks a random packet from the queue. If that packet belongs to the same flow as the packet under consideration, _both_ packets will be dropped. When a randomly-picked packet comes from the same flow, chances are good that packets from that flow occupy a substantial amount of queue space, so that flow is likely to be a source of the problem. 

  * If the packets are from different flows, but the queue length is above an administrator-set maximum, then the new packet (only) will be dropped. 

  * In the final case, the algorithm calculates a probability that the packet will be dropped, even though the maximum queue length has not been reached. The probability grows as the queue length increases, but, by default, it remains low - about 2% at the maximum. Thus, for mid-length queues, the algorithm will occasionally send a signal to a transmitter that it should back off a bit, but most packets will be queued normally. 

The key feature of CHOKe - the one which distinguishes it from RED (from which it is derived) - is the check against a random packet in the queue. That is a heuristic mechanism for identifying problematic flows without actually having to track what each flow is doing. Experience, as reported in the CHOKe paper, suggest that it works pretty well. 

An important factor in successful use of CHOKe in the real world will be careful selection of the controlling parameters: the minimum and maximum queue lengths, average period, and drop probability. In particular, there is mounting evidence (thanks to the efforts by Jim Gettys) that overly long queues lead to all kinds of pathological network behavior, and could even threaten a net collapse at some point. Use of algorithms like CHOKe, combined with reasonably-sized queues, could help keep the Internet working well into the future.
