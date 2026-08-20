---
title: "Kernel Summit 2005: Report from the networking summit"
url: https://lwn.net/Articles/144272/
date: "July 19, 2005"
category: "Networking-Networking summits"
author: 
---

From LWN's [2005 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2005/>).   
---  
A separate summit for Linux networking hackers was held in Montreal immediately prior to the Kernel Summit; sixteen developers were in attendance. Jamal Hadi Salim briefly presented the results from the discussion there. The theme of the meeting was taken from a comment by Alexey Kuznetsov found in `net/ipv4/tcp_input.c`: 

`/* Old crap is replaced with new one. 8)`

It seems that much old crap is being replaced in the networking subsystem. Briefly: 

  * Changes are in store in the IPSec/XFRM area; the current configuration interface is considered to be too inflexible and unfriendly. A proposal for a replacement is circulating. 

  * Thomas Graf is reworking the networking configuration options. He also has a new netlink library and a netconfig tool in the prototype stage. 

  * David Miller continues to work on his "super TSO" code - trying to improve the performance of TCP segmentation offload in real-world situations. 

  * The venerable socket buffer ("SKB") structure has come under fire for being too bloated; an SKB now can occupy 256 bytes on 64-bit systems. This structure will begin to shrink in 2.6.14, with a target size of 192 or even 128 bytes. 

  * Herbert Xu is working on the crypto-API interface, with the goal of adding asynchronous support. The existing synchronous interface will be maintained, as will, hopefully, performance. Also added will be support for multiple implementations of the same algorithm; it was not immediately clear why. The first user of the new code will be IPSec. 

  * Stephen Hemminger is working on a number of projects. One is the [pluggable TCP congestion avoidance modules](<http://lwn.net/Articles/128681/>) patch, which has been merged into the mainline. Improving port number randomization for TCP is another project. There is a project to reduce the size of the networking stack for embedded systems. The `CONFIG_EMBEDDED` flag will be used to this effect, though that approach is not popular with all developers (who see that flag as being somewhat overloaded). 

Another item on the list is remote DMA (RDMA). Several issues must be resolved for RDMA support in Linux (see [this Kernel Page article](<http://lwn.net/Articles/133649/>)), and there is little consensus that RDMA should be supported at all. In the end, though, RDMA exists, some people will want to use it, so Linux will probably support it in some way. The developers are currently waiting for code. 

It appears that a similar attitude is slowly developing toward TCP offload engines - something which the Linux developers have always refused to support. TOE is not going away, and some sort of support might yet be merged. 

  * Mention was made (in other contexts as well) of the "ostra" tool being developed by Arnaldo Carvalho de Melo. Ostra is a debugging tool based on "sparse"; it is being used with the networking code. Arnaldo has not yet reached a point of releasing this code, however. 

  * Linux is now IPv6 ready! We're sure you are glad to know that. IPv6 work continues, however; among other things, current development is focusing on mobile IPv6, policy routing, and various types of tunneling functionality. 

  * James Morris is working on the integration of SELinux and networking. Integration with the netfilter code is at the top of the list, with IPSec not too far behind. Once that's done, there are cluster filesystems to worry about. There are plans to use the "trusted platform module" chip for secure networking in the future. 

David Miller concluded with a "state of the networking union" address. The summary is simple: "we are in good shape, we kick ass, more good stuff is coming."
