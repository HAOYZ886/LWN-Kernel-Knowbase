---
title: Increasing the TCP initial congestion window
url: https://lwn.net/Articles/427104/
date: "February 9, 2011"
category: "Networking-Congestion control"
author: "By Jonathan Corbet February 9, 2011"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
February 9, 2011

The TCP slow start algorithm, initially developed by Van Jacobson, was one of the crucial protocol tweaks which made TCP/IP actually work on the Internet. Slow start works by limiting the amount of data which can be in flight over a new connection and ramping the transmission speed up slowly until the carrying capacity of the connection is found. In this way, TCP is able to adapt to the actual conditions on the net and avoid overloading routers with more data than can be accommodated. A key part of slow start is the initial congestion window, which puts an upper bound on how much data can be in flight at the very beginning of a conversation. 

That window has been capped by [RFC 3390](<http://tools.ietf.org/html/rfc3390>) at four segments (just over 4KB) for the better part of a decade. In the meantime, connection speeds have increased and the amount of data sent over a given connection has grown despite the fact that connections live for shorter periods of time. As a result, many connections never ramp up to their full speed before they are closed, so the four-segment limit is now seen as a bottleneck which increases the latency of a typical connection considerably. That is one reason why contemporary browsers use many connections in parallel, despite the fact that the HTTP specification says that a maximum of two connections should be used. 

Some developers at Google have been [agitating](<http://research.google.com/pubs/pub36640.html>) for an increase in the initial congestion window for a while; in July 2010 they posted [an IETF draft](<http://tools.ietf.org/html/draft-hkchu-tcpm-initcwnd-01>) pushing for this change and describing the motivation behind it. Evidently Google has run some large-scale tests and found that, by increasing the initial congestion window, user-visible latencies can be reduced by 10% without creating congestion problems on the net. They thus recommend that the window be increased to 10 segments; the draft suggests that 16 might actually be a better value, but more testing is required. 

David Miller has posted [a patch](<https://lwn.net/Articles/426883/>) increasing the window to 10; that patch has not been merged into the mainline, so one assumes it's meant for 2.6.39. 

Interestingly, Google's tests worked with a number of operating systems, but not with Linux, which uses a relatively small initial _receive_ window of 6KB. Most other systems, it seems, use 64KB instead. Without a receive window at least as large as the congestion window, a larger initial congestion window will have little effect. That problem will be fixed in 2.6.38, thanks to [a patch from Nandita Dukkipati](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=356f039822b8d802138f7121c80d2a9286976dbd>) raising the initial receive window to 10 segments.
