---
title: "Partial address-space mirroring"
url: https://lwn.net/Articles/684866/
date: "April 27, 2016"
category: Memory management
author: "By Jonathan Corbet April 27, 2016 LSFMM 2016"
---

By **Jonathan Corbet**  
April 27, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

A feature found in some systems designed for high availability is memory mirroring: providing two copies of data stored in main memory so that said data can be recovered should something happen to one of the copies. But, as Taku Izumi noted during the memory-management track of the 2016 Linux Storage, Filesystem, and Memory-Management Summit, as memory sizes grow larger, the cost of providing that mirror grows as well. So there is interest in building systems that only mirror part of the physical address space. It quickly became clear, though, that the memory-management developers have strong doubts about the wisdom of such an arrangement. 

This feature, Taku said, is managed by the low-level BIOS; it can be configured there or by using the `efibootmgr` command. The amount of memory to be mirrored can be set there, and read from the EFI memory map by the kernel at boot time. Unlike fully mirrored memory, partially mirrored memory requires support from the kernel. The idea is to improve fault tolerance by using mirrored memory for the kernel and its data, while putting user space in single-copy [![\[Taku Izumi\]](https://static.lwn.net/images/conf/2016/lsfmm/TakuIzumi-sm.jpg)](<https://lwn.net/Articles/684873/>) memory. Everything that is not mirrored would be placed into the `ZONE_MOVABLE` zone, so that kernel memory would not be allocated there. 

By default, in such a system, one would want user-space memory to be kept out of `ZONE_NORMAL`, since that's where the mirrored memory is. To that end, a new `__GFP_NONMIRROR` allocation flag would be added; it would be part of `GFP_HIGHUSER_MOVABLE`. But, occasionally, there might be critical user data that should go into mirrored memory. That could be obtained via a new `MADV_MIRROR` flag to the `madvise()` system call. 

Kirill Shutemov objected that `madvise()` is the wrong interface to use; placement in mirrored memory would be mandatory, while `madvise()` is, as its name suggests, an advisory system call. Rik van Riel asked why we would want to put user-space memory into the mirrored range; the answer seems to be to enable the running of a particularly important virtual machine with mirroring. The problem is that, once you try to put a user-space program into that range, everything has to go there, including shared libraries. Making all that work properly could get a little messy. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

On the other hand, it was pointed out that computers exist to run applications. If a particular application is so important that it needs a computer with (expensive) mirrored memory, why not protect that application, too? Aneesh Kumar said that one has to start somewhere, and that protecting the kernel is the first step. Protection can be expanded from there. 

There was some talk about preventing user space from exhausting the mirrored range; perhaps requesting mirrored memory should be a privileged operation. It's also not clear what the kernel should do if mirrored memory runs out; should it fall back to non-mirrored memory? The conclusion seemed to be that falling back would remove the reliability guarantee that mirrored memory is meant to provide, so it should not be done. Instead, if possible, the range of memory that is mirrored should be expanded. 

It was Michal Hocko who raised the key objection to this scheme, though: it threatens to bring back all of the low-memory problems that, the developers had thought, we had finally left behind. On 32-bit systems, only a portion of the physical address space is directly addressable by the kernel; that portion is called "low memory." Kernel data structures, as a rule, can only be placed in the low-memory region. That has led to many problems over the years where the system runs out of low memory and finds itself crippled, even though quite a bit of memory is available in general. 64-bit systems do not have this problem, since they can map the entire address range. 

By creating a new zone that must contain all kernel memory, partial memory mirroring recreates the low-memory problem. It will place hard limits on the amount of user-space memory that can be allocated, leading, Michal said, to out-of-memory situations when, in fact, lots of memory is free. Rik added that experience has shown that the ratio of non-kernel memory to kernel-addressable memory should not go much above 4:1; after that, problems start to develop. 

Returning to the user-space side, Rik said that it would be necessary to place some user-space data in mirrored memory. If the init process dies due to memory corruption, for example, the fact that the kernel is protected will provide little comfort. Then the C library probably needs to live there, and probably no end of other things. In the end, Michal said, the obvious conclusion is that one should simply mirror the entire address space. 

Mike Kravetz suggested that mirrored memory could be an opt-out resource rather than opt-in, but Kirill pointed out that an application that opts out would likely end up placing important libraries in unmirrored memory. Those would have to be somehow upgraded later on when another application needs them. Mel Gorman said that, in the end, nobody would volunteer to opt out; as Linus [noted](<https://lwn.net/Articles/682769/>) recently, few developers think that their application is not important. 

Mel went on to say that partial mirroring is simply the wrong approach; if a system needs that level of reliability, he said, it should just mirror all of memory. Trying to work around that requirement is trading a potential future problem (memory corruption) for a definite problem now (kernel-memory issues). Beyond that, we can't pretend that user space can make mirroring decisions correctly. Security issues remain; even if requesting mirrored memory directly is a privileged operation, an unprivileged process will still be able to force the exhaustion of mirrored memory. There is no privilege separation in this scheme; it promises the ability to protect specific applications, but is unable to deliver on it. The result will be worse than a false hope; it will create a system that is fundamentally fragile. 

Andrea Arcangeli pointed out that some of this fragility is already inherent in the `ZONE_MOVABLE` memory zone. Mel agreed, saying that `ZONE_MOVABLE` is a curse that should never have been used the way it is. As a result, he said, features like memory hotplug are fundamentally broken and systems are more fragile than they should be. But, he said, if you see a car crash, you don't normally drive in to join it; the same approach should be taken here. 

At this point, time ran out, and it became clear that the conversation was circling around on itself. But it was also clear that the memory-management developers think little of the partial-mirroring idea and would rather not see code added to the kernel to support it.
