---
title: "The congestion-notification conflict"
url: https://lwn.net/Articles/783673/
date: "March 22, 2019"
category: "Networking-Congestion control"
author: "By Jonathan Corbet March 22, 2019"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 22, 2019

Most of the time, the dreary work of writing protocol standards at organizations like the IETF and beyond happens in the background, with most of us being blissfully unaware of what is happening. Recently, though, a disagreement over protocols for congestion notification and latency reduction has come to a head in a somewhat messy conflict. The outcome of this discussion may well affect how well the Internet of the future works — and whether Linux systems can remain first-class citizens of that net. 

Network congestion is a fact of life; when it occurs, the only useful response is to get senders of traffic to slow down. Many governments place traffic signals on the on-ramps to major highways in congestion-prone areas in an attempt to limit traffic entering and to keep things flowing. Network traffic can benefit from similar controls, but the placement of traffic signals at every entry point to the net is impractical. So network protocols must rely on other types of signals to learn when they should reduce their transmission rate. 

Protocols like TCP, unfortunately, were not designed with such signals, so congestion-control algorithms have been built to use the one signal that is always reliably delivered: dropped packets. But dropping packets on the floor can make things worse (it forces the data to be transmitted again), it introduces delays and, by the time it happens, congestion is already occurring. It would be better to inform senders of congestion in a less heavy-handed manner, before that congestion becomes a problem. 

#### ECN and its discontents

The [explicit congestion notification (ECN)](<https://tools.ietf.org/html/rfc3168>) protocol, standardized in 2001, was an attempt to improve the situation by informing senders of congestion without dropping packets. ECN repurposed two bits in the IP header; for reasons that will become clear below, it is worth looking at how those bits are interpreted: 

> 00| Transport is not ECN-capable  
> ---|---  
> 01| ECN-capable ECT(1)  
> 10| ECN-capable ECT(0)  
> 11| Congestion experienced  
  
The ECT(0) and ECT(1) values have the same meaning; they indicate that ECN is supported at least somewhere on the route between two endpoints. In practice all implementations use ECT(0); attempts to give a separate meaning to ECT(1) have not gained traction. ECN famously broke the Internet because many routers would drop packets with either of those bits set; that delayed its adoption for years. 

ECN has improved the situation, but not enough; it suffers from a couple of significant problems. One is that a "congestion experienced" signal still arrives too late; congestion is already happening and the router is pleading for help. It is also still a heavy hammer; the RFC requires that a congestion-experienced signal be treated as if a packet had been dropped, so congestion-control algorithms respond by severely reducing their transmission rates, then working back up. That can reduce the throughput of a connection (and increase its latency) more than is needed to resolve the problem. 

As networks get faster, the demands for lower latencies grow, and as bufferbloat-reduction efforts reduce the amount of queue space available on routers, congestion control needs to become a bit more nuanced. There is widespread agreement on that point. How that nuance should happen is a matter of rather less agreement. 

#### L4S

One attempt at improving congestion control has been developed, mostly slowly and mostly in private, by various industry players; it is called [Low Latency, Low Loss, Scalable Throughput](<https://tools.ietf.org/html/draft-ietf-tsvwg-l4s-arch-03>), or L4S. The core idea seems to be to replace ECN with a more flexible signal built into a higher-level protocol; [data center TCP](<https://tools.ietf.org/html/rfc8257>) (DCTCP) is one example. DCTCP acknowledgment packets can include information on how much queuing space is available; senders can use the information to keep the queue full without overflowing it. Linux has supported DCTCP since the 4.1 release. 

The problem with something like DCTCP is that it must work with the active queue-management algorithms running on all of the routers between the two endpoints. Those algorithms see _all_ traffic passing through the router, not just the DCTCP traffic. The proponents of L4S seem to want a sort of privileged treatment for suitably clueful protocols so that they can get low-latency treatment through the router without having to contend with what the L4S draft terms "classic TCP". 

To bring that about, L4S redefines the ECT(1) value described above to indicate "this packet is using better congestion notification". Routers would then create two separate queues; a fast one for the L4S traffic, and a slower one for the "classic" traffic. That differentiation can, on its own, raise some eyebrows, but the queue-management algorithm needs to be evaluated as a whole to see what its broader effects, including on fairness, would really be. 

DCTCP is not seen as being entirely safe for use outside of protected environments like data centers. For wider deployment, the intent has been to create a new TCP congestion-control algorithm called "TCP Prague". The L4S portion would then be implemented with a queue-management algorithm called "DUALPI", so named because it maintains the two independent queues described above. Both of these modules have been vaporware until recently: a [repository with TCP Prague](<https://github.com/L4STeam/tcp-prague/commit/f16165de39ec3c107c65550a3bbba898eb836943>) showed up on March 12 (no attempt has yet been made to submit it for the mainline), and DUALPI was [posted to the netdev list](<https://lwn.net/ml/netdev/20190311151455.3436-1-olga@albisser.org/>) on March 11\. 

#### SCE

The alternative, pushed by longtime bufferbloat fighters Jonathan Morton and Dave Täht, along with UDP creator David Reed, is called [some congestion experienced](<https://github.com/dtaht/bufferbloat-rfcs/blob/master/sce/draft-morton-taht-tsvwg-sce.txt>), or SCE. It is a rather simpler proposal, intended to provide a "congestion is imminent" signal that, once again, is less heavy handed; it places a higher priority on compatibility with existing TCP congestion-control implementations, though. 

SCE also makes use of the ECT(1) value to encode the "some congestion" signal. The full congestion-experienced value would retain its current meaning, with protocols expected to treat it as being equivalent to a dropped packet. The SCE signal, instead, should be interpreted this way: 

New SCE-aware receivers and transport protocols SHOULD interpret the SCE codepoint as an indication of mild congestion, and respond accordingly by applying send rates intermediate between those resulting from a continuous sequence of ECT codepoints, and those resulting from a CE codepoint. The ratio of ECT and SCE codepoints received indicates the relative severity of such congestion, such that 100% SCE is very close to the threshold of CE marking, 100% ECT indicates that the bottleneck link may not be fully utilised, and a 1:1 balance of ECT and SCE codepoints indicates that the present send rate is a good match to the bottleneck link. 

The code implementing SCE is also quite new; it showed up in the [fq_codel_fast repository](<https://github.com/dtaht/fq_codel_fast/commits/master>) on March 14.. It's worth noting that this proposal does not give intermediate routers a way of knowing whether either endpoint is capable of responding to SCE signals or not. There is, perhaps, an implicit assumption that, once SCE is supported by Linux and FreeBSD, it will quickly become omnipresent. 

#### Which is better?

These two proposals are clearly incompatible with each other; each places its own interpretation on the ECT(1) value and would be confused by the other. The SCE side argues that its use of that value is fully compatible with existing deployments, while the L4S proposal turns it over to private use by suitably anointed protocols that are not compatible with existing congestion-control algorithms. L4S proponents argue that the dual-queue architecture is necessary to achieve their latency objectives; SCE seems more focused on fixing the endpoints. 

It looks like a fairly typical battle between a protocol pushed by the largest Internet service providers, and one with a rather more grass-roots origin. There is, however, another important thing to know about L4S: Alcatel-Lucent [claims a patent](<https://datatracker.ietf.org/ipr/2952/>) on the dual-queue algorithm. The company has generously offered to make that patent available under "fair, reasonable, and non-discriminatory" terms; such terms are, of course, highly discriminatory against free software implementations. They make it impossible to merge the affected code into a GPL-licensed kernel. 

As is the case with many patents, the quality of this one is not universally recognized. Bob Briscoe, one of the developers of L4S, [claims loudly](<https://lwn.net/ml/bufferbloat/7e49b551-22e5-5d54-2a1c-69f53983d7e5@bobbriscoe.net/>) that there is prior art for the claims in the Alcatel-Lucent patent and that it should never have been issued. The patent unfortunately exists, though; as long as Alcatel-Lucent continues to claim it, the code cannot become part of the Linux kernel. If L4S becomes the IETF-anointed standard, and if industry adoption follows, Linux could find itself out in the cold. 

The disagreement over these protocols reflects a difference in approach between developers and their associated industries. It is subject to all of the usual technical and political maneuverings; the process could be unpleasant to watch as it plays out. One could argue that the Linux community could happily let it play out and simply merge the winner; one might also argue that SCE better matches the values that have shaped our network stack in general. The assertion of that patent, though, raises the stakes considerably; it would not be good for Linux to find itself unable to play with other high-performance network stacks. As long as the patent remains, the technical choice is easy.
