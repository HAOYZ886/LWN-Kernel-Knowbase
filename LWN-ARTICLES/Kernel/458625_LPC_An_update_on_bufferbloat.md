---
title: "LPC: An update on bufferbloat"
url: https://lwn.net/Articles/458625/
date: "September 13, 2011"
category: "Networking-Bufferbloat"
author: "By Jonathan Corbet September 13, 2011"
---

By **Jonathan Corbet**  
September 13, 2011

Approximately one year after describing bufferbloat to the world and starting his campaign to remedy the problem, Jim Gettys traveled to the 2011 Linux Plumbers Conference to update the audience on the current state of affairs. A lot of work is being done to address the bufferbloat problem, but even more remains to be done. 

"Bufferbloat" is the problem of excessive buffering used at all layers of the network, from applications down to the hardware itself. Large buffers can create obvious latency problems (try uploading a large file from a home network while somebody else is playing a fast-paced network game and you'll be able to measure the latency from the screams of frustration in the other room), but the real issue is deeper than that. Excessive buffering wrecks [![\[Jim Gettys\]](https://static.lwn.net/images/conf/2011/lpc/JimGettys3-sm.jpg)](<https://lwn.net/Articles/458631/>) the control loop that enables implementations to maximize throughput without causing excessive congestion on the net. The experience of the late 1980's showed how bad a congestion-based collapse of the net can be; the idea that bufferbloat might bring those days back is frightening to many. 

The initial source of the problem, Jim said, was the myth that dropping packets is a bad thing to do combined with the fact that it is no longer possible to buy memory in small amounts. The truth of the matter is that the timely dropping of packets is essential; that is how the network signals to transmitters that they are sending too much data. The problem is complicated with the use of the [bandwidth-delay product](<https://secure.wikimedia.org/wikipedia/en/wiki/Bandwidth-delay_product>) to size buffers. Nobody really knows what either the bandwidth or the delay are for a typical network connection. Networks vary widely; wireless networks can be made to vary considerably just by moving across the room. In this environment, he said, no static buffer size can ever be correct, but that is exactly what is being used at many levels. 

As a result, things are beginning to break. Protocols that cannot handle much in the way of delay or loss - DNS, ARP, DHCP, VOIP, or games, for example - are beginning to suffer. A large proportion of broadband links, Jim said, are "just busted." The edge of the net is broken, but the problem is more widespread than that; Jim fears that bloat can be found everywhere. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

If static buffer sizes cannot work, buffers must be sized dynamically. The [RED](<http://icir.org/floyd/red.html>) protocol is meant to do that sizing, but it suffers from one little problem: it doesn't actually work. The problem, Jim said, is that the protocol knows about the size of a given buffer, but it knows nothing about how quickly that buffer is draining. Even so, it can improve the situation in some situations. But it requires quite a bit of tuning to work right, so a lot of service providers simply do not bother. Efforts to create an improved version of RED are underway, but the results are not yet available. 

A real solution to bufferbloat will have to be deployed across the entire net. There are some things that can be done now; Jim has spent a lot of time tweaking his home router to squeeze out excessive buffering. The result, he said, involved throwing away a bit of bandwidth, but the resulting network is a lot nicer to use. Some of the fixes are fairly straightforward; Ethernet buffering, for example, should be proportional to the link speed. Ring buffers used by network adapters should be reviewed and reduced; he found himself wondering why a typical adapter uses the same size for the transmit and receive buffers. There is also an extension to the [DOCSIS](<https://secure.wikimedia.org/wikipedia/en/wiki/DOCSIS>) standard in the works to allow ISPs to remotely tweak the amount of buffering employed in cable modems. 

A complete solution requires more than that, though. There are a lot of hidden buffers out there in unexpected places; many of them will be hard to find. Developers need to start thinking about buffers in terms of time, not in terms of bytes or packets. And we'll need active queue management in all devices and hosts; the only problem is that nobody really knows which queue management algorithm will actually solve the problem. Steve Hemminger noted that there are no good multi-threaded queue-management algorithms out there. 

#### CeroWRT

Jim yielded to Dave Taht, who talked about the [CeroWRT](<http://www.bufferbloat.net/projects/cerowrt>) router distribution. Dave pointed out that, even when we figure out how to tackle bufferbloat, we have a small problem: actually getting those fixes to manufacturers and, eventually, users. A number of popular routers are currently shipping with 2.6.16 kernels; it is, he said, the classic embedded Linux problem. 

One router distribution that is doing a better job of keeping up with the mainline is [OpenWRT](<http://openwrt.org/>). Appropriately, CeroWRT is based on OpenWRT; its purpose is to complement the [debloat-testing](<https://lwn.net/Articles/429943/>) kernel tree and provide a platform for real-world testing of bufferbloat fixes. The goals behind [![\[Dave Täht\]](https://static.lwn.net/images/conf/2011/lpc/DaveTaht-sm.jpg)](<https://lwn.net/Articles/458632/>) CeroWRT are to always be within a release or two of the mainline kernel, provide reproducible results for network testing, and to be reliable enough for everyday use while being sufficiently experimental to accept new stuff. 

There is a lot of new stuff in CeroWRT. It has fixes to the packet aggregation code used in wireless drivers that can, in its own right, be a source of latency. The length of the transmit queues used in network interfaces has been reduced to eight packets - significantly smaller than the default values, which can be as high as 1000. That change alone is enough, Dave said, to get quality-of-service processing working properly and, he thinks, to push the real buffering bottleneck to the receive side of the equation. CeroWRT runs a tickless kernel, and enables protocol extensions like explicit congestion notification (ECN), selective acknowledgments (SACK), and duplicate SACK (DSACK) by default. A number of speedups have also been applied to the core netfilter code. 

CeroWRT also includes a lot of interesting software, including just about every network testing tool the developers could get their hands on. Six TCP congestion algorithms are available, with [Westwood](<http://www.cs.ucla.edu/NRL/hpi/tcpw/>) used by default. Netem (a network emulator package) has been put in to allow the simulation of packet loss and delay. There is a bind9 DNS server with an extra-easy DNSSEC setup. Various mesh networking protocols are supported. A lot of data collection and tracing infrastructure has been added from the [web10g project](<http://www.web10g.org/>), but Dave has not yet found a real use for the data. 

All told, CeroWRT looks like a useful tool for validating work done in the fight against bufferbloat. It has not yet reached its 1.0 release, though; there are still some loose ends to tie and some problems to be fixed. For now, it only works on the Netgear WNDR3700v2 router - chosen for its open hardware and relatively large amount of flash storage. CeroWRT should be ready for general use before too long; fixing the bufferbloat problem is likely to take rather longer. 

[Your editor would like to thank LWN's subscribers for supporting his travel to LPC 2011.]
