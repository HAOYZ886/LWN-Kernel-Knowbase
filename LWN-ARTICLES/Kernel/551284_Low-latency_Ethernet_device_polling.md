---
title: "Low-latency Ethernet device polling"
url: https://lwn.net/Articles/551284/
date: "May 21, 2013"
category: Networking
author: "By Jonathan Corbet May 21, 2013"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 21, 2013

Linux is generally considered to have one of the most fully featured and fast networking stacks available. But there are always users who are not happy with what's available and who want to replace it with something more closely tuned for their specific needs. One such group consists of people with extreme low latency requirements, where each incoming packet must be responded to as quickly as possible. High-frequency trading systems fall into this category, but there are others as well. This class of user is sometimes tempted to short out the kernel's networking stack altogether in favor of a purely user-space (or purely hardware-based) implementation, but that has problems of its own. A relatively small patch to the networking subsystem might just be able to remove that temptation for at least some of these users. 

Network interfaces, like most reasonable peripheral devices, are capable of interrupting the CPU whenever a packet arrives. But even a moderately busy interface can handle hundreds or thousands of packets per second; per-packet interrupts would quickly overwhelm the processor with interrupt-handling work, leaving little time for getting useful tasks done. So most interface drivers will disable the per-packet interrupt when the traffic level is high enough and, with cooperation from the core networking stack, occasionally poll the device for new packets. There are a number of advantages to doing things this way: vast numbers of interrupts can be avoided, incoming packets can be more efficiently processed in batches, and, if packets must be dropped in response to load, they can be discarded in the interface before they ever hit the network stack. Polling is thus a win for almost all situations where there is any significant amount of traffic at all. 

Extreme low-latency users see things differently, though. The time between a packet's arrival and the next poll is just the sort of latency that they are trying to avoid. Re-enabling interrupts is not a workable solution, though; interrupts, too, are a source of latency. Thus the drive for user-space solutions where an application can simply poll the interface for new packets whenever it is prepared to handle new messages. 

Eliezer Tamir has posted an alternative solution in the form of the [low-latency Ethernet device polling patch set](<https://lwn.net/Articles/551179/>). With this patch, an application can enable polling for new packets directly in the device driver, with the result that those packets will quickly find their way into the network stack. 

The patch adds a new member to the `net_device_ops` structure: 

```
int (*ndo_ll_poll)(struct napi_struct *dev);
```

This function should cause the driver to check the interface for new packets and flush them into the network stack if they exist; it should not block. The return value is the number of packets it pushed into the stack, or zero if no packets were available. Other return values include `LL_FLUSH_BUSY`, indicating that ongoing activity prevented the processing of packets (the inability to take a lock would be an example) or `LL_FLUSH_FAILED`, indicating some sort of error. The latter value will cause polling to stop; `LL_FLUSH_BUSY`, instead, appears to be entirely ignored. 

Within the networking stack, the `ndo_ll_poll()` function will be called whenever polling the interface seems like the right thing to do. One obvious case is in response to the `poll()` system call. Sockets marked as non-blocking will only poll once; otherwise polling will continue until some packets destined for the relevant socket find their way into the networking stack, up until the maximum time controlled by the `ip_low_latency_poll` sysctl knob. The default value for that knob is zero (meaning that the interface will only be polled once), but the "recommended value" is 50µs. The end result is that, if unprocessed packets exist when `poll()` is called (or arrive shortly thereafter), they will be flushed into the stack and made available immediately, with no need to wait for the stack itself to get around to polling the interface. 

Another patch in the series adds another call site in the TCP code. If a `read()` is issued on an established TCP connection and no data is ready for return to user space, the driver will be polled to see if some data can be pushed into the system. So there is no need for a separate `poll()` call to get polling on a TCP socket. 

This patch set makes polling easy to use by applications; once it is configured into the kernel, no application changes are needed at all. On the other hand, the lack of application control means that _every_ `poll()` or TCP `read()` will go into the polling code and, potentially, busy-wait for as long as the `ip_low_latency_poll` knob allows. It is not hard to imagine that, on many latency-sensitive systems, the hard response-time requirements really only apply to some connections, while others have no such requirements. Polling on those less-stringent sockets could, conceivably, create new latency problems on the sockets that the user really cares about. So, while no reviewer has called for it yet, it would not be surprising to see the addition of a `setsockopt()` operation to enable or disable polling for specific sockets before this code is merged. 

It almost certainly will be merged at some point; networking maintainer Dave Miller [responded](<https://lwn.net/Articles/551294/>) to an earlier posting with ""I just wanted to say that I like this work a lot."" There are still details to be worked out and, presumably, a few more rounds of review to be done, so low-latency sockets may not be ready for the 3.11 merge window. But it would be surprising if this work took much longer than that to get into the mainline kernel.
