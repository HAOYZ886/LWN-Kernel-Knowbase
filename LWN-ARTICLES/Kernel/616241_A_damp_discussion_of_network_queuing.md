---
title: A damp discussion of network queuing
url: https://lwn.net/Articles/616241/
date: "October 15, 2014"
category: "Networking-Bufferbloat"
author: "By Jonathan Corbet October 15, 2014 Linux
Plumbers Conference"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
October 15, 2014

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2014-Linux_Plumbers_Conference>)

Very few presenters at technical conferences come equipped with gallons of water and a small inflatable swimming pool to contain it. But that is just how Stephen Hemminger showed up at the 2014 [Linux Plumbers Conference](<http://www.linuxplumbersconf.org/2014/>). Stephen was there to talk about the current state of the fight against bufferbloat; while there was some good news to share, the sad fact is that, in a number of areas, we are still all wet. 

When Jim Gettys discovered and named bufferbloat some years ago, he had stumbled onto a set of problems that networking developers had been aware of for years. But nobody quite understood the extent of the problem or how it could affect everyday networking. In short, bufferbloat comes about when one or more players in the networking pipeline buffer far more data than they should. The user-visible results can include degraded download speeds, uploading not working at all, and high latencies — all with no packet loss. 

The latency issue is problematic at a number of levels. If you are trying to provide remote display service, 15ms latencies will ruin the usability of the system. At 100ms delay, voice over IP protocols cease to work well. Users will generally hit the "reload" button (or give up) on a web page load after about one second. Bufferbloat can create latencies far longer than that. But it's the lack of packet loss that is the real problem; without dropped packets, [![\[Steve Hemminger\]](https://static.lwn.net/images/conf/2014/lce-lpc/SteveHemminger1-sm.jpg)](<https://lwn.net/Articles/616281/>) the TCP congestion control algorithms cannot do their job. So the networking stack keeps trying to send more data when the proper response is to slow down and let the queues drain. 

So who is to blame for the bufferbloat problem? One possible response, Stephen said, was to blame Linux. After all, Windows XP limited TCP connections to 64KB of outstanding data; there is not much buffering happening there. Windows 7, instead, added a rate limiter that would throttle all connections. Android has done something similar, adding a limit on the size of the receive window for any connection. Linux developers tend not to be enamored with artificial limits, so Linux users get to experience the full pain of the bufferbloat problem. 

An alternative, he said, is to blame the customer. That is why Internet service providers like Comcast went through a period of blaming its biggest customers for its networking problems. Comcast went as far as capping bandwidth use and charging extra in some cases. But the real problem was not those customers, it was bufferbloat in their internal network. 

#### Getting wet

At this point the aquatic games began. Stephen put together a set of demonstrations where a network queue was represented by an inverted plastic bottle. The bottle could hold a fair amount of water (packets), but there are limits on how quickly the water can drain out. So if water arrives more quickly than the bottle can drain, the bottle begins to fill. If the [![\[Steve Hemminger\]](https://static.lwn.net/images/conf/2014/lce-lpc/SteveHemminger2-sm.jpg)](<https://lwn.net/Articles/616287/>) bottle is quite full, a drop of water added at the top will take a long time to reach the opening and exit the bottle — especially if the bottle is large. Bufferbloat, thus, was represented as bottlebloat. 

In the real world, network queuing systems are more complicated than a single bottle, though. The default queuing discipline in Linux employs three parallel bottles of varying sizes; one for bulk traffic, one for high-priority traffic, and one for everything else ("normal" traffic). Almost all traffic goes through the normal bottle; SSH can use the high-priority queue, while Dropbox and BitTorrent use the bulk queue. It was an OK idea for its time, Stephen said, but it does not work on today's net. Those three bottles do nothing to prevent excess buffering. 

The first attempt to come up with a smarter solution was the RED queue management algorithm. RED was represented by poking a bunch of holes into a (red, naturally) bottle. Once the water level in the bottle goes above the holes, water escapes through those holes and is lost; that corresponds to dropping packets in the real world. Rather than dropping packets, RED can set the explicit congestion notification (ECN) bit in the TCP header, notifying the receiver that it needs to reduce the size of the receive window to slow down the connection. It's a nice idea, but the net broke it. Routers will drop packets with ECN set, or, worse, simply reset the bit. As a result, Linux "will play the game," Stephen said, but only if the other side initiates it. The networking developers just do not want to deal with complaints about dropped connections. 

A different approach is called "hierarchical token bucket"; it looks like a bunch of small bottles all connected in parallel. Each type of traffic gets its own bottle (queue), and packets are dispatched equally from all queues. The problem with this mechanism is that it requires a great deal of configuration to work well. That might be manageable on a server with a static workload, but it is not useful on desktop systems. 

An alternative is stochastic fair queuing (SFQ). The same set of small bottles is used, but each network connection is assigned to its own bottle by way of a hash function. No configuration is required. SFQ can make things work better, but it is not a full solution to the bufferbloat problem; it was the state of the art in the Linux kernel about five years ago. 

In an attempt to come up with a smarter solution, Kathie Nichols and Van Jacobson created the "Controlled Delay" or CoDel algorithm. CoDel looks somewhat like RED, in that it starts to drop packets when buffers get too full. But CoDel works by looking at the packets at the head of the queue — those which have been through the entire queue and are about to be transmitted. If they have been in the queue for too long, they are simply [![\[Steve Hemminger\]](https://static.lwn.net/images/conf/2014/lce-lpc/SteveHemminger3-sm.jpg)](<https://lwn.net/Articles/616288/>) dropped. The general idea is to try to maintain a maximum delay in the queue of 5ms (by default). CoDel has a number of good properties, Stephen said; it drops packets quickly (allowing TCP congestion control algorithms to do their thing) and maintains reasonable latencies. The "fq_codel" algorithm adds an SFQ dispatching mechanism in front of the CoDel queue, maintaining fairness between network connections. 

Stephen noted that replacing a queue with something like fq_codel is a good thing, but one should remember that there are a lot of queues in a typical system. Only the one with the smallest hole (the slowest outgoing link) matters in the end, since that's where the packets will accumulate. 

After a discussion of how most network benchmarking utilities look at the wrong thing (one should examine upload speed, download speed, and latency simultaneously), he put up a set of plots showing how the network responds to load with the various queuing mechanisms. The results clearly showed the CoDel solves the bufferbloat problem well, and that fq_codel does even better. 

So, are we there yet? As noted, there are a lot of queues in a typical network path, and not all of them have been addressed. Different techniques are needed at different levels. For excessive buffering at the socket layer, for example, [TCP small queues](<https://lwn.net/Articles/507065/>) can be used. A bigger problem is Internet service providers, which tend to have large amounts of legacy equipment in their racks. There is not a lot the networking developers can do about that. Still, it helps to be running the best software locally. So Stephen encouraged everybody to run a command like: 

```
sysctl -w net.core.default_qdisc=fq_codel
```

That will cause fq_codel to be used for all future connections (up to the next reboot). Unfortunately, the default queuing discipline cannot be changed, since it will certainly disturb some user's workload somewhere. 

#### The good, the bad, and the ugly

Stephen concluded by saying that there are good, bad, and ugly parts to bufferbloat and the efforts to solve it. On the good side, the industry is aware of the problem. Bufferbloat is routinely talked about at IETF meetings, and researchers are working on it. Perhaps best of all, the solutions are all open source. In some cases (CoDel for example), open-source publication was deliberately chosen to forestall the adoption of patent-encumbered techniques. 

On the bad side, there is a lot of legacy equipment and software out there. Original equipment manufacturers, Stephen said, are focused on cost, not on queue management details. So a lot of equipment out there — especially consumer-level equipment — is bad and will stay that way for some years yet. There are also issues with backbone congestion, but they tend to be more political than technical in nature. 

The ugly part is wireless networking, which has a bunch of unique buffering problems of its own. Packet aggregation, for example, can help with bandwidth, but creates latency problems. Wireless systems are mostly using proprietary software and are never updated. Standards bodies are starting to pay attention, Stephen said, but a solution in this area is distant. 

Even with the bad and ugly parts, though, the message was mostly positive, if a bit damp. Quite a bit of good work has been done to address the ~~bottle~~ bufferbloat problem, and that work has shown up first as free software. Bufferbloat will be with us for a while yet, but solutions are far closer than they were a few years ago.
