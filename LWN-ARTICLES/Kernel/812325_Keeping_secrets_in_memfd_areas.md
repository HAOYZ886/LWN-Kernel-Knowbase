---
title: Keeping secrets in memfd areas
url: https://lwn.net/Articles/812325/
date: "February 14, 2020"
category: "Memfd; Memory management-Address-space isolation; System calls-memfd secret"
author: "By Jonathan Corbet February 14, 2020"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
February 14, 2020

Back in November 2019, Mike Rapoport [made the case](<https://lwn.net/Articles/803823/>) that there is too much address-space sharing in Linux systems. This sharing can be convenient and good for performance, but in an era of advanced attacks and hardware vulnerabilities it also facilitates security problems. At that time, he proposed a number of possible changes in general terms; he has now come back with [a patch](<https://lwn.net/ml/linux-kernel/20200130162340.GA14232@rapoport-lnx/>) implementing a couple of address-space isolation options for the [memfd](<https://lwn.net/Articles/593918/>) mechanism. This work demonstrates the sort of features we may be seeing, but some of the hard work has been left for the future. 

Sharing of address spaces comes about in a number of ways. Linux has traditionally mapped the kernel's address space into every user-space process; doing so improves performance in a number of ways. This sharing was thought to be secure for years, since the mapping doesn't allow user space to actually access that memory. The Meltdown and Spectre hardware bugs, though, rendered this sharing insecure; thus [kernel page-table isolation](<https://lwn.net/Articles/741878/>) was merged to break that sharing. 

Another form of sharing takes place in the processor's memory caches; once again, hardware vulnerabilities can expose data cached in this shared area. Then there is the matter of the kernel's direct map: a large mapping (in kernel space) that contains all of physical memory. This mapping makes life easy for the kernel, but it also means that _all_ user-space memory is shared with the kernel. In other words, an attacker with even a limited ability to run code in the kernel context may have easy access to all memory in the system. Once again, in an era of speculative-execution bugs, that is not necessarily a good thing. 

The memfd subsystem wasn't designed for address-space isolation; indeed, its initial purpose was as a sort of interprocess communication mechanism. It does, however, provide a way to create a memory region attached to a file descriptor with specific characteristics; a memfd can be "sealed", for example, so that a recipient knows that it will not be changed. Rapoport decided that it would be a good foundation on which to build a "secret memory" feature. 

Actually creating an isolated memory area requires passing a new flag to `[memfd_create()](<http://man7.org/linux/man-pages/man2/memfd_create.2.html>)` called `MFD_SECRET`. That, however, doesn't describe _how_ this secrecy should be implemented. There are a number of options that offer varying levels of security and performance degradation, so the user has to make a decision. The available options, as implemented in the patch, could easily have been specified directly to `memfd_create()` with their own flags, but Rapoport decided to require the use of a separate `[ioctl()](<http://man7.org/linux/man-pages/man2/ioctl.2.html>)` call instead. Until the secrecy mode has been specified with this call, the user cannot map the memfd, and thus cannot actually make use of it. 

There are two modes implemented so far; the first of them, `MFD_SECRET_EXCLUSIVE`, does a number of things to hide the memory attached to the memfd from prying eyes. That memory is marked as being unevictable, for example, so it will never be flushed out to swap. The effect is similar to calling `[mlock()](<http://man7.org/linux/man-pages/man2/mlock.2.html>)`, but with a couple of differences: pages are not actually allocated until they are faulted in, and the limit on the number of locked pages appears to be (perhaps by mistake) implemented separately from the limits imposed by `mlock()`. There is also no way to unlock pages except by destroying the memfd, which requires unmapping it and closing its file descriptor. 

The other thing done by `MFD_SECRET_EXCLUSIVE` is to remove the pages used by the memfd from the kernel's direct map, making it inaccessible from kernel space. The problem with this is that the direct map is normally set up using huge pages, which makes accessing it far more efficient. Removing individual (small) pages forces huge pages to be broken apart into lots of small pages, slowing the system for everybody. The current code (admittedly a proof of concept) allocates each page independently when it is faulted in, which seems likely to maximize the damage done to the direct mapping. That will need to change before this feature could be seriously considered for merging. 

The other mode, `MFD_SECRET_UNCACHED` does everything `MFD_SECRET_EXCLUSIVE` does, but also causes the memory to be mapped with caching disabled. That will prevent its contents from ever living in the processor's memory caches, rendering it inaccessible to exploits that use any of a number of hardware vulnerabilities. It also makes access to that memory far slower in general, to the point that it may seem inaccessible to the intended user as well. For small amounts of infrequently accessed data (cryptographic keys, for example) it may be a useful option, though. 

In its current form, the feature only allows one mode to be selected. In truth, though, `MFD_SECRET_UNCACHED` is a strict superset of `MFD_SECRET_EXCLUSIVE`, so that is not currently a problem. Rapoport suggests that this whole API could change in the future, with an alternative being ""something like 'secrecy level' from 'a bit more secret than normally' to 'do your best even at the expense of performance'"". 

Part of the purpose behind this posting was to get comments on the proposed API, but those have not been forthcoming so far. This may be one of those projects that has to advance further — and get closer to being merge-ready — before developers will take notice. But at least the work itself is not a secret anymore, so interested users can start to think about whether it meets their needs or not.
