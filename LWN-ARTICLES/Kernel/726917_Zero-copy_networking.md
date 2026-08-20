---
title: "Zero-copy networking"
url: https://lwn.net/Articles/726917/
date: "July 3, 2017"
category: "Networking-Performance"
author: "By Jonathan Corbet July 3, 2017"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
July 3, 2017

In many performance-oriented settings, the number of times that data is copied puts an upper limit on how fast things can go. As a result, zero-copy algorithms have long been of interest, even though the benefits achieved in practice tend to be disappointing. Networking is often performance-sensitive and is definitely dominated by the copying of data, so an interest in zero-copy algorithms in networking comes naturally. A set of patches under review makes that capability available, in some settings at least. 

When a process transmits a buffer of data, the kernel must format that data into a packet with all of the necessary headers and checksums. Once upon a time, this formatting required copying the data into a single kernel-space buffer. Network hardware has long since gained the ability to do scatter/gather I/O and, with techniques like [TCP segmentation offloading](<https://lwn.net/Articles/9123/>), the ability to generate packets from a buffer of data. So support for zero-copy operations has been available at the hardware level for some time. 

On the software side, the contents of a file can be transmitted without copying them through user space using the [`sendfile()`](<http://man7.org/linux/man-pages/man2/sendfile.2.html>) system call. That works well when transmitting static data that is in the page cache, but it cannot be used to transmit data that does not come directly from a file. If, as is often the case, the data to be transmitted is the result of some sort of computation — the application of a template in a content-management system, for example — `sendfile()` cannot be used, and zero-copy operation is not available. 

The [`MSG_ZEROCOPY` patch set](<https://lwn.net/Articles/726353/>) from Willem de Bruijn is an attempt to make zero-copy transmission available in such settings. Making use of it will, naturally, require some changes in user space, though. 

Requesting zero-copy operation is a two-step process. Once a socket has been established, the process must call `setsockopt()` to set the new `SOCK_ZEROCOPY` option. Then a zero-copy transmission can be made with a call like: 

```
status = send(socket, buffer, length, MSG_ZEROCOPY);
```

One might wonder why the `SOCK_ZEROCOPY` step is required. It comes down to a classic API mistake: the `[send()](<http://man7.org/linux/man-pages/man2/send.2.html>)` system call doesn't check for unknown flag values. The two-step ritual is thus needed to avoid breaking any programs that might have been accidentally setting `MSG_ZEROCOPY` for years and getting away with it. 

If all goes well, a transmission with `MSG_ZEROCOPY` will lock the given buffer into memory and start the transmission process. Transmission will almost certainly not be complete by the time that `send()` returns, so the process must take care to not touch the data in the buffer while the operation is in progress. That immediately raises a question: how does the process know when the data has been sent and the buffer can be used again? The answer is that the zero-copy mechanism will place a notification message in the error queue associated with the socket. That notification can be read with something like: 

```
status = recvmsg(socket, &message, MSG_ERRORQUEUE);
```

The socket can be polled for an error status, of course. When an "error" packet originating from `SO_EE_ORIGIN_ZEROCOPY` shows up, it can be examined to determine the status of the operation, including whether the transmission succeeded and whether it was able to run in the zero-copy mode. These status packets contain a sequence number that can be used to associate them with the operation they refer to; the fifth zero-copy `send()` call will generate a status packet with a sequence number of five. These status packets can be coalesced in the kernel, so a single packet can report on the status of multiple operations. 

The mechanism is designed to allow traditional and zero-copy operations to be freely intermixed. The overhead associated with setting up a zero-copy transmission (locking pages into memory and such) is significant, so it makes little sense to do it for small transmissions where there is little data to copy in the first place. Indeed, the kernel might decide to use copying for a small operation even if `MSG_ZEROCOPY` is requested but, in that case, it must still go to the extra effort of generating the status packet. So the developers of truly performance-oriented programs will want to take care to only request zero-copy behavior for large buffers; just where the cutoff should be is not entirely clear, though. 

Sometimes, zero-copy operation is not possible regardless of the buffer size. For example, if the network interface cannot generate checksums, the kernel will have to perform a pass over the data to do that calculation itself; at that point, copying the data as well is nearly free. Anytime that the kernel must transform the data — when IPSec is being used to encrypt the data, for example — it cannot do zero-copy transmission. But, for most straightforward transmission cases, zero-copy operation should be possible. 

Readers might be wondering why the patch does not support zero-copy reception; while the patch set itself does not address this question, it is possible to make an educated guess. Reading is inherently harder because it is not generally known where a packet is headed when the network interface receives it. In particular, the interface itself, which must place the packet somewhere, is probably not in a position to know that a specific buffer should be used. So incoming packets end up in a pile and the kernel sorts them out afterward. Fancier interfaces have a fair amount of programmability, to the point that zero-copy reception is not entirely infeasible, but it remains a more complex problem. For many common use cases (web servers, for example), transmission is the more important problem anyway. 

As was noted in the introduction, the benefits from zero-copy operation are often less than one might hope. Copying is expensive, but the setup required to avoid a copy operation also has its costs. In this case, the author claims that a simple benchmark ([`netperf`](<https://github.com/HewlettPackard/netperf>) blasting out data) runs 39% faster, while a more realistic production workload sees a 5-8% improvement. So the benefit for real-world systems is not huge, but it may well be enough to be worth going for on highly-loaded systems that transmit a lot of data. 

The patch set is in its fourth revision as of this writing, and the rate of change has slowed considerably. There do not appear to be any fundamental objections to its inclusion at this point. For those wanting more details, [this paper [PDF]](<https://netdevconf.org/2.1/papers/netdev.pdf>) by De Bruijn is worth a read.
