---
title: Kernel support for SYN packet fingerprinting
url: https://lwn.net/Articles/644906/
date: "May 20, 2015"
category: Networking
author: "By Jake Edge May 20, 2015"
---

By **Jake Edge**  
May 20, 2015

The initial packet of a TCP connection (i.e. the [`SYN` packet](<http://en.wikipedia.org/wiki/Transmission_Control_Protocol#CONNECTION-ESTABLISHMENT>)) contains information that can be used to detect attributes of the remote system through [TCP/IP fingerprinting](<http://en.wikipedia.org/wiki/TCP/IP_stack_fingerprinting>). But that data is contained in the headers of the packets, which means it is only accessible to the kernel. A patch set that was recently merged into the net-next tree would change that to allow user-space servers to request the header information on connections they have accepted. 

Eric Munson started the conversation when he [posted a patch](<https://lwn.net/Articles/642919/>) that would allow a program to request that the `SYN` packets be saved by using `setsockopt()` on a listening socket. The `SYN` headers could then be retrieved, once, via a `getsockopt()` call on the socket returned by `accept()`. That would allow user space to examine the TCP and IP headers to identify (or at least narrow down) the operating system of the remote host that made the connection. 

Munson's patch simply stored the `SKB` (i.e. `struct sk_buff`) that contained the `SYN` packet, which could be rather large (up to 4KB), as Eric Dumazet [pointed out](<https://lwn.net/Articles/645192/>). For millions of client connections, that memory can add up, he said. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Instead, Dumazet suggested, a 2012 [patch from Tom Herbert](<https://patchwork.ozlabs.org/patch/146034/>) (or one based on that) should be used. That code has been used internally at Google for around two years, he said, without any problems handling large numbers of simultaneous connections. Instead of storing the `SKB`, it allocates space just for the headers with `kmalloc()`—usually less than 128 bytes per connection. 

When Herbert posted his patch, there were concerns about adding eight bytes to each `SKB` for a ""very fringe feature"" (in the [words](<http://permalink.gmane.org/gmane.linux.network/223611>) of network maintainer David Miller). Herbert's original patch also stored the `SKB` like Munson's does. The patch was never merged, but Dumazet modified it to `kmalloc()` space for the headers and it was put into production at Google. 

Munson was not particularly tied to his implementation; he [said](<https://lwn.net/Articles/645127/>) that he was happy to back Dumazet's patch if it met his needs. That [patch](<https://lwn.net/Articles/645128/>) was posted on May 3\. It adds two new socket options that are used to request and retrieve the `SYN` headers. Servers request that the kernel save the headers by calling `setsockopt()` with `TCP_SAVE_SYN` either before or after the `listen()` call; the kernel will save the headers for subsequent connection requests. IP and TCP headers can be retrieved, once, by calling `getsockopt()` with `TCP_SAVED_SYN` on the socket returned from `accept()`. 

Michael Kerrisk [complained](<https://lwn.net/Articles/645129/>) that the option names were too similar, while also asking about how the interface would be used. Dumazet disagreed about the names, but [provided a test program](<https://lwn.net/Articles/645130/>) used by Google to demonstrate how the new options work for user space. 

Andy Lutomirski [wondered](<https://lwn.net/Articles/645132/>) if too much information was being returned to user space with Munson's patch. It turned out that Ethernet headers were also being returned, which Munson [agreed](<https://lwn.net/Articles/645133/>) was probably not needed. John Heffner [asked](<https://lwn.net/Articles/645134/>) a related question: ""Are there conditions where, for security purposes, you don't want an application to have access to the raw SYNs?"" Dumazet [indicated](<https://lwn.net/Articles/645135/>) that it was believed to be safe to provide the IP and TCP headers. 

The patch was [applied](<https://lwn.net/Articles/645136/>) by Miller on May 5, though he noted that the behavior when a too-small buffer was passed to `getsockopt()` should be rethought. The original patch simply copied as much data as it could into the user-space buffer, but that gave no indication that the `SYN` headers were not complete. Miller suggested that it should return an error and indicate the proper length so that the program could allocate more space if needed. Munson subsequently [posted a patch](<https://lwn.net/Articles/645137/>) to do just that. 

The feature seems like it will be useful; it appears that it already has been for Google. It is interesting to note that the company has been collecting these fingerprints on (at least) some portion of its vast server farm, though it is not clear what it is doing with all of that information. Soon, though, others will be able to do so too—once 4.2 is released.
