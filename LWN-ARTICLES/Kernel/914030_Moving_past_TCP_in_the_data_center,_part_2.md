---
title: "Moving past TCP in the data center, part 2"
url: https://lwn.net/Articles/914030/
date: "November 9, 2022"
category: "Networking-Protocols"
author: "By Jake Edge November 9, 2022 Netdev"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
November 9, 2022

* * *

[Netdev](<https://lwn.net/Archives/ConferenceIndex/#Netdev-2022>)

At the end of [our earlier article](<https://lwn.net/Articles/913260/>) on John Ousterhout's talk at [Netdev 0x16](<https://netdevconf.info/0x16/>), he had concluded that TCP was unsuitable for data-center environments for a variety of reasons. He also argued that there was no way to repair TCP so that it could serve the needs of data-center networking. In order for software to be able to use the full potential of today's networking hardware, TCP needs to be replaced with a protocol that is different in almost every way, he said. The second half of the talk covered the [Homa transport protocol](<https://homa-transport.atlassian.net/wiki/spaces/HOMA/overview>) that he and others at Stanford have been working on as a possible replacement for TCP in the data center. 

The Homa project set out to design a protocol from scratch that would be ideal for the needs of data-center networking today. It turned out to be different from TCP in each of the five aspects that he had covered in the first half of the talk; the choices made by Homa for those work well together to "produce a really really high-performance data-center protocol". But, he stressed, Homa is not suitable for wide-area networks (WANs); it is only for data centers. 

#### Homa overview

To start with, Homa is message-based, rather than byte-stream-based, which means that the "dispatchable units" are recorded in the protocol. Each message is made up of multiple packets on the wire that get assembled into a full message for application processing. This solves the thread load-balancing problems that TCP has, because multiple threads can safely read from a single socket; it will also allow network interface cards (NICs) to directly dispatch messages to threads in the future should NICs ever gain that support. 

Homa is connectionless; its fundamental unit is a remote procedure call (RPC), which consists of two messages: a request and a response. "The notion of 'round trip' is explicit in the Homa protocol." RPCs are independent with no ordering guarantees between them; multiple RPCs can be initiated and can finish in any order. There is no long-lived connection state stored by Homa; once an RPC completes, all of its state is removed. There is a small amount (roughly 200 bytes) of state stored for each peer host, for IP routing information and the like, however. There is also no connection setup overhead and a single socket can be used to send any number of RPCs to any number of peers. Homa ensures that an RPC either completes or an error is returned, so application-level timers are not needed. 

Congestion control in Homa is receiver-driven, which has major advantages over the sender-driven congestion control used by TCP. Ousterhout described how flow control works in Homa. When a message needs to be sent to a receiver, the sender can transmit a few "unscheduled packets" immediately, but additional "scheduled packets" must each wait for an explicit "grant" from the receiver. The idea is that there are enough unscheduled packets to cover the round-trip time between the hosts; if there is no congestion, the sender will have received some grants by the time it has sent all of the unscheduled packets. That allows sending at the hardware line speed. 

A receiver can choose not to send grants if it detects congestion in its top-of-rack (TOR) switch; it can pause or slow the grants until that condition subsides. It can also prioritize messages by sending or denying grants based on the remaining size of the message. Having the message size available in the protocol is helpful because "message sizes allow us to predict the future". Once a single packet of a message is received, the receiver knows how many unscheduled packets are coming, how many scheduled packets remain after that, and it can quickly decide on a strategy to minimize the congestion that could result from all of the different in-progress messages it is receiving. It can react much more quickly than TCP can. 

Homa also takes advantage of the priority queues that modern switches have; each egress port in those switches typically has 8-16 priority queues. It uses the queues to implement the "shortest remaining processing time" (SRPT) algorithm; when a receiver has multiple messages in progress, it allows the shorter messages to use the higher-priority queues on the switch. In addition, the queues provide a way to achieve a "much better tradeoff between throughput and latency". 

"Buffers are funny; you can't live with them, but you can't live without them either." Some amount of buffering is needed at the switch to ensure that link bandwidth is not wasted if the higher-priority senders stop sending for any reason. Throughput is maintained by having lower-priority packets queue up and be sent when there is a lull from a higher-priority sender; Homa can then send grants to other senders to get them ramped up. This "overcommitment" of granting more messages than can be handled, coupled with the prioritized buffering leads to higher throughput with low latency on the important, shorter messages, he said. 

One might think that long messages could potentially suffer from starvation on a loaded network. He said that he tried to produce that behavior in Homa so that he could study it, but he found it really hard to do; he can create starvation scenarios, but had to "contort the benchmarks" to do it. But, in order to avoid starvation, Homa takes a small portion of the receiver's bandwidth (typically 5-10%) and uses it for the oldest message, rather than the shortest message as strict SRPT would dictate. That guarantees that those messages make progress; eventually their remaining size gets small enough to be prioritized with the other small messages. 

Homa does not rely on in-order packet delivery; packets can arrive in any order and the receivers will sort them as needed. In practice, the packets arrive nearly in order anyway, he said, so it is not computationally expensive to do the reordering. He believes that Homa eliminates core-congestion problems in data centers, unless there is simply too much traffic overall, because packets can take different paths through the core fabric. That leads to better load balancing in the fabric as well as across CPU cores on the receiving hosts. 

#### Replacing TCP?

It is hard to imagine a standard more entrenched than the TCP protocol is, Ousterhout said, so "I undertake this with full realization that I may be out of my mind". Based on the results he has seen from Homa, he has set a personal mission to figure out a way for Homa to take over a substantial portion of TCP's traffic in the data center. Either that, or learn why it is not possible; "I'm going to keep going until I hit a roadblock that I simply can't solve". 

The first step to doing that is to have a production-quality implementation of Homa "that people can actually use". He is "maybe a little bit of a freak among academics", he said, because he loves to write code. A few years ago, he set out to write a Linux kernel driver for Homa; he "knew nothing about the Linux kernel", but now has a [working driver for Homa](<https://github.com/PlatformLab/HomaModule>). 

The Homa module runs on Linux 5.17 and 5.18 and is not meant as a "research prototype", which is a term that he does not like. A research prototype is something that does not work, "but you can somehow write a paper about it and make claims about it, even though it doesn't actually really work". The Homa module is nearing production quality at this point, he said; the only way to find and fix the remaining bugs is to start running it in production. 

In terms of performance, Homa "completely dominates" TCP and [Data Center TCP](<https://www.rfc-editor.org/rfc/rfc8257>) (DCTCP) for all workloads and all message sizes, he said. He gave a few sample benchmarks that showed 3-7x improvements in short-message latency at the 50th percentile, and 19-72x improvements in the 99th percentile (the "tail latency"). More information about the kernel driver for Homa can be found in his [paper](<https://www.usenix.org/system/files/atc21-ousterhout.pdf>) from the 2021 USENIX annual technical conference. 

#### Applications

The biggest problem he sees with his mission is the huge number of applications that directly use TCP via the socket interface. Homa's message-based API is different and he has been unable to find a way to "slide it in under the existing TCP sockets interface". But it is not realistic to plan to convert even a substantial fraction of those applications. Many, perhaps most, of the applications that directly use TCP sockets also need to work over the WAN, where Homa is not a good choice; others really do not need the performance boost that Homa brings. The applications that will really benefit from Homa are the newer data-center applications that mostly use one of a handful of RPC frameworks. 

Adding Homa support to RPC frameworks would allow applications that use them to switch to Homa with a small configuration change, instead of a major code change; much like applications can change the server name they use, they would be able to choose Homa instead of TCP. He has [work in progress](<https://github.com/PlatformLab/grpc_homa>) on adding Homa support to [gRPC](<https://grpc.io/>). The C++ gRPC integration is working now, though without support for encryption, while the Java gRPC support for Homa is "embryonic" but currently underway. 

Working with gRPC has made him "very sad", however, because it is "unimaginably slow". The round-trip latency for gRPC on TCP is 90µs, with two-thirds of that time spent in the gRPC layers on the client and server (30µs each). If the goal is to get to 5µs round trips, it is pretty clear it cannot be done using gRPC, he said. With Homa, gRPC is roughly twice as fast, even in the gRPC layers on the endpoints, which he does not really have an explanation for. But even that is far from the goal, so he believes a "super lightweight" framework needs to be developed eventually. 

Even with Homa, though, performance is still an order of magnitude off from what the hardware is capable of, he said. "Network speeds are increasing at this amazing rate, CPU speeds aren't changing much." Software simply cannot keep up and he sees no solution to that problem. So software implementations of transport protocols no longer make sense; those protocols need to move into the NICs. 

#### Moving to user space

Some people will say that you simply need to get "those terrible operating systems" out of the way and move everything to user space; that will help some, he said, but it is not enough. In the [RAMCloud storage project](<https://ramcloud.atlassian.net/wiki/spaces/RAM/overview>), Stanford researchers implemented Homa in user space and were able to achieve 99th percentile round-trip latencies of 14µs, while the tail latency for Homa on Linux is 100µs. 

The primary culprit for high tail latencies is software overhead and, in particular, load-balancing overhead. A single core cannot handle more than 10Gbps but as soon as you split the job over multiple cores, there is an inherent performance penalty just for doing the split. Beyond that, there is the problem of hot spots, where too much work ends up on certain cores while others are idle or nearly so; those can cause spikes of high latency. 

He has not measured it, but his theory is that the inherent slowdown from splitting up the job to multiple cores is due to cache interference. Lots of cache-coherency traffic coming from the split reduces the performance when compared to doing the processing on a single core. It is not just his Homa implementation that sees a problem of this magnitude; he looked at Google's [Snap/Pony](<https://storage.googleapis.com/pub-tools-public-publication-data/pdf/36f0f9b41e969a00d75da7693571e988996c9f4c.pdf>) system, which sees a 4-6x efficiency loss when moving from single to multiple cores. He said that the rule of thumb is that if you need to go beyond a single core, a second or third core is not enough; "you actually have to go to four cores before you actually get any performance improvement over one core". 

If you look at some of the numbers from various papers, Ousterhout said, you might conclude that moving the protocol processing to user space is a good idea. But, all of those user-space implementations are "basically just research prototypes and they are way oversimplified"; they do not do a lot of the processing that any production system would require. They are also measured under the most ideal conditions, with either no load balancing or perfect by-hand partitioning, handling only short messages ("that's easy"), and so on. 

But Snap is a production-quality user-space implementation that can be compared; it is roughly 2x better than Homa in the Linux kernel. If you look at the number of cores needed to drive 80Gbps (an 80% loaded 100Gbps network) bidirectionally, Snap requires 9-14 cores, while Linux Homa requires 17\. So user space is better, but not on the scale needed to hit the goals. 

#### To the NIC

The only alternative that he sees is to move the protocol processing of the transport layer into the NIC. Applications would access the NIC directly, bypassing the kernel, via a message-based interface; "the notion of a packet never gets to software". Other features like load balancing by choosing an idle application thread, virtualization, and encryption should be added to the NICs as well. 

The architecture for these NICs is not going to be easy to figure out, he said; they need to process packets at line rate, while being programmable to support multiple protocols. It is also important that the programs for those protocols can still be open source, Ousterhout said. None of the existing "smart NIC" architectures is adequate, he thinks, so it makes for an interesting problem for computer architects to solve. 

Homa is "not without its controversies"; there have been several papers that have claimed to find problems with Homa. There are also some alternatives being proposed in other papers, but "all of these papers have pretty significant flaws in them, unfortunately". They have used Homa in unrealistic configurations or hobbled it in some fashion in his opinion; while he does not "want to get into a food fight" about them, he has gathered some of his responses on the [Homa wiki](<https://homa-transport.atlassian.net/wiki/spaces/HOMA/overview#Related-Work>). 

But there is a bigger "meta question" that needs to be answered: "do applications actually care about harnessing the full network performance?" Today, we are "stuck in a no-chicken-no-egg cycle", there are no applications that require the full hardware capabilities, because there would be no way to run them. So today's applications make do with the performance provided by today's network stacks and there is no real incentive to make the infrastructure faster, because there is no one who is "screaming for it". 

So, he wondered, if we make the networking "as blindingly fast as I think we can do", will new applications appear that people have not even thought about today because the capability is not available? He would be interested in hearing about any applications that would be dramatically improved by these performance improvements. As an academic, he does not actually need to have a market; he can build something and hope that the applications come along later, but that "would be a terrible way to do a startup". 

He concluded by reiterating the main points of what he thinks needs to be done if applications are going to take advantage of the amazing advances in networking speeds. A new transport protocol is needed; he is obviously a Homa advocate but would be happy to discuss other possibilities for that. Beyond that, a new lightweight RPC framework will be important, and, ultimately, whatever transport protocol is used will need to move down into the NIC. Ousterhout's keynote laid out a vision of a fairly radical reworking of the state of data-center networking today; it will be interesting to see how it plays out over the coming years.
