---
title: "Kernel Summit: Network summit summary"
url: https://lwn.net/Articles/94565/
date: "July 21, 2004"
category: "IPv6; Networking-Networking summits"
author: 
---

This article is part of LWN's [2004 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2004/>).   
---  
One week prior to the kernel summit, a small group got together in Oregon for a two-day networking summit. By all accounts from those who were present, the summit was a focused and effective event. Stephen Hemminger summarized the results for the kernel group; he also maintains [a web site](<http://developer.osdl.org/shemminger/ns2004/>) with materials from the networking summit. 

The networking folks have a long "to do" list, but few of the items on that list call for major changes. In the IPv6 area, there is a need for better load balancing and use of equivalent routes, and multicast routing still needs to be done (in a cleaner manner than it was done for IPv4). There is also still a major lack of useful performance data for the IPv6 stack; the developers have little idea of whether they perform well or not. There are plans in the works to support mobile IPv6, a protocol which lets you plug in anywhere and communicate over a tunnel back to your ISP. The tunneling code needs work in general to communicate tunnel status back to user space, and to provide a more generic tunneling capability. The networking layer increasingly needs to work with long timeouts, which will drive a push toward using 64-bit timers in the kernel. 

There is also, we were warned, a big reshuffle coming for the network portion of the kernel source tree. 

In the security area, there are plans to drop CIPE altogether and use the OpenVPN standard. The crypto support layer needs improvement, including crypto hardware support and an asynchronous mode suited to use in bottom half handlers. Netfilter needs some performance work, perhaps extending to the creation of a "grand unified flow cache" which would minimize the number of passes required on each packet. Nonlinear SKBs do not fit well with the netfilter way of looking at things, and performance suffers accordingly. 

Things are coming together in the area of 802.11 stacks; Jeff Garzik's new development tree centered around HostAP will help in that area. 

UDP fragmentation suffers from an "ID wrapping" problem which could lead to security issues; it needs to be fixed. [RDMA](<http://www.rdmaconsortium.org/home>) is an area of increasing interest; believe it or not, the networking developers think that using TCP offload capabilities on network adaptors may be appropriate for this case. Some cards also offer "receive collation offload," essentially the reassembly of incoming TCP streams; the Linux stack may eventually take advantage of that. Asynchronous I/O support was discussed; it may be added for the transmit side, but few people see a need for it when reading from network sockets. There is continued work in TCP congestion control; a few algorithms have been added to the kernel, but nobody has yet figured out which ones actually perform well. 

Future work may include the implementation of [TCP BIC](<http://www.csc.ncsu.edu/faculty/rhee/export/bitcp/>) 1.1. The format of nonlinear SKBs may change; the various fragments will be stored in the same sort of "scatterlist" used elsewhere in the kernel; this will require a driver API change, however. More IPSec work is needed. [MPLS](<http://www.mplsrc.com/>) support is on the list. And there is talk of moving away from the route cache. 

[>> Next: Asynchronous I/O](<https://lwn.net/Articles/94566/>).
