---
title: TCP friends
url: https://lwn.net/Articles/511254/
date: "August 15, 2012"
category: Networking
author: "By Jonathan Corbet August 15, 2012"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
August 15, 2012

One of the many advantages of the TCP network protocol is that the process at one end of a connection need not have any idea of where the other side is. A process could be talking with a peer on the other side of the world, in the same town, or, indeed, on the same machine. That last case may be irrelevant to the processes involved, but it can be important for performance-sensitive users. A new patch from Google seems likely to speed that case up in the near future. 

A buffer full of data sent on the network does not travel alone. Instead, the TCP layer must split that buffer into reasonably-sized packets, prepend a set of TCP headers to it, and, possibly, calculate a checksum. The packets are then passed to the IP layer, which throws its own headers onto the beginning of the buffer, finds a suitable network interface, and hands the result off to that interface for transmission. At the receiving end the process is reversed: the IP and TCP headers are stripped, checksums are compared, and the data is merged back into a seamless stream for the receiving process. 

It is all a fair amount of work, but it allows the two processes to communicate without having to worry about all that happens in between. But, if the two processes are on the same physical machine, much of that work is not really necessary. The bulk of the overhead in the network stack is there to ensure that packets do not get lost on their way to the destination, that the data does not get corrupted in transit, and that nothing gets forgotten or reordered. Most of these perils do not threaten data that never leaves the originating system, so much of the work done by the networking stack is entirely wasted in this case. 

That much has been understood by developers for many years, of course. That is why many programs have been written specifically to use Unix-domain sockets when communicating with local peers. Unix-domain sockets ("pipes") provide the same sort of stream abstraction, but, since they do not communicate between systems, they avoid all of the overhead added by a full network stack. So faster communications between local processes is possible now, but it must be coded explicitly in any program that wishes to use it. 

What if local TCP communications could be accelerated to the point that they are competitive with Unix-domain sockets? That is the objective of [this patch](<https://lwn.net/Articles/511079/>) from Bruce Curtis. The idea is simple enough to explain: when both endpoints of a TCP connection are on the same machine, the two sockets are marked as being "friends" in the kernel. Data written to such a socket will be immediately queued for reading on the friend socket, bypassing the network stack entirely. The TCP, IP, and loopback device layers are simply shorted out. The actual patch, naturally enough, is rather more complicated than this simple description would suggest; friend sockets must still behave like TCP sockets to the point that applications cannot tell the difference, so friend-handling tweaks must be applied to many places in the TCP stack. 

One would hope that this approach would yield local networking speeds that are at least close to competitive with those achieved using Unix-domain sockets. Interestingly, Bruce's patch not only achieves that, but it actually does better than Unix-domain sockets in almost every benchmark he ran. "Better" means both higher data transmission rates and lower latencies on round-trip tests. Bruce does not go into why that is; perhaps the amount of attention that has gone into scalability in the networking stack pays off in his 16-core testing environment. 

There is one important test for which Bruce posted no results: does the TCP friends patch make things any slower for non-local connections where the stack bypass cannot be used? Some of the network stack hot paths can be sensitive to even small changes, so one can imagine that the networking developers will want some assurance that the non-bypass case will not be penalized if this patch goes in. There are various other [little issues](<https://lwn.net/Articles/511403/>) that need to be dealt with, but this patch looks like it is on track for merging in the relatively near future. 

If it is merged, the result should be faster local communications between processes without the need for special-case code using Unix-domain sockets. It could also be most useful on systems hosting containerized guests where cross-container communications are needed; one suspects that Google's use case looks somewhat like that. In the end, it is hard to argue against a patch that can speed local communications by as much as a factor of five, so chances are this change will go into the mainline before too long.
