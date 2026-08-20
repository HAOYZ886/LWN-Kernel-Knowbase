---
title: "Increasing the range of address-space layout randomization"
url: https://lwn.net/Articles/667790/
date: "December 16, 2015"
category: "Security-Address-space layout randomization"
author: "By Jonathan Corbet December 16, 2015"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
December 16, 2015

Many attempts to exploit vulnerabilities in computing systems depend on the ability to locate a specific data structure or piece of code in a running process. Attempts that proceed with an incorrect address will typically fail or, even better, cause a crash that brings attention to the exploit attempt. For this reason, most contemporary systems try to obscure the layout of running processes. To a great extent this effort relies on not leaking information from those processes, or from the kernel, but it is also important to avoid letting those locations be predictable in the first place. The mechanisms used in Linux toward this end are seen by many as woefully inadequate; a current patch set is unlikely to change that impression greatly, but it might just succeed in increasing security on some systems. 

Address space layout randomization (ASLR) introduces some randomness into the locations where a process's code and data segments are placed within that process's address space. Two processes running the same program should, under ASLR, see that program placed in different virtual locations. The Linux kernel [first implemented ASLR](<https://lwn.net/Articles/121845/>) back in 2005. Even then, that implementation was seen as being, at best, a partial solution to the problem. 

ASLR works by calculating a random offset to be added to the return value from every `mmap()` call. Since `mmap()` is used to place most data and code segments, that causes this offset to apply to most of the process's address space (one exception is the stack, which has a separate offset). On 32-bit systems, the offset is an eight-bit value; the offset is interpreted in pages, so, on these systems, memory areas from `mmap()` will have a random offset between zero and 255 pages. Note that the lowest bits of the address of any mapping are _not_ random, since mappings are page-aligned. 

This offset will be sufficient to frustrate (almost all the time) a simple exploit that tries a single, hard-coded address. It is somewhat less effective, though, if the exploit is able to retry with varying offsets of its own. Only 256 attempts are required to explore the entire range of potential offsets; an exploit running locally can probably work through the entire set in well less than a second, especially if it is able to avoid causing process crashes in its attempts. ASLR, thus, does not provide a great deal of additional security against such threats. 

The problem is that there is only so much address space to play with, especially on 32-bit systems. Shifting the heap and stack areas around will reduce the space between them (traditional layout schemes having been designed to maximize that space); that, in turn, can cause programs needing to do huge allocations to fail. Increasing the amount of randomness in the layout offset may not cause problems in a lot of systems, but a mechanism that goes into everybody's kernel has to be implemented conservatively. Thus, the eight-bit value used on 32-bit systems (64-bit systems have more space to play with, so a 28-bit offset is used on x86-64, for example). 

One might think that 32-bit systems are on the way out, but there are still a lot of them — including a lot of Android devices. When the "Stagefright" vulnerability, which affects Android systems, was disclosed in July, Mark Brand at Google [set out to write an exploit for it](<http://googleprojectzero.blogspot.com/2015/09/stagefrightened.html>). He duly found that ASLR was not a significant obstacle in the face of a brute-force attack; as he put it: ""I knew that ASLR on 32-bit was always a bit shaky; but I didn’t think it was this broken."" An easily exploitable vulnerability on vast numbers of Android devices is a bit of a nightmare scenario; it would be nice if the kernel had ways to mitigate exploits in such situations. ASLR, as it turns out, is, in its current form, not really one of those ways. 

The interesting thing about Android, though, is that it is a relatively controlled environment. It may not always need the same level of generality that a kernel-for-everybody must provide. If it is known that Android systems will not be making huge allocations (and that should be fairly well known), then it may well be safe to increase the randomness of the ASLR mechanism, making brute-force attacks harder to carry out. But current kernels offer no way to increase the amount of randomness used with ASLR. 

It would not be out of character for the Android developers to simply patch a higher degree of randomness into the kernel shipped with the Android distribution. But, sometimes, they try to get a general solution into the upstream kernel instead. That is what is happening with [Daniel Cashman's patch](<https://lwn.net/Articles/667582/>) to provide an customizable random offset range for ASLR. 

This patch set replaces the compile-time ASLR offset range with a pair of new sysctl knobs called `/proc/sys/vm/mmap_rnd_bits` and `/proc/sys/vm/mmap_rnd_compat_bits`. The first covers normal processes, while the second applies to those running in the compatibility mode — 32-bit processes running on a 64-bit kernel, for example. The default value matches the value in current kernels, so users will see no change unless they (or their distributor) explicitly make a change. 

As the names suggest, these knobs control the number of address-space bits used for the ASLR random offset. Each architecture sets minimum and maximum values that make sense, given the available address space. On x86, the value may range between eight and 16 bits (for 32-bit) or 28 and 32 (for 64-bit). The limits for ARM are rather more complicated, depending on the specific subarchitecture and the page size in use. In all cases, though, this patch makes it possible to increase the number of bits used for the random offset. Each additional bit doubles the space that must be searched, making an exploit slower and more likely to be detected. If the Android developers are able to use this feature to increase ASLR randomness, the next Stagefright-like vulnerability will, hopefully, be harder to exploit. 

This patch set has been through a number of revisions (six thus far with at least one more expected) based on comments. Those comments, though, are concerned with how the patch can be improved; there does not seem to be any real discomfort with the idea in general. So Linux kernels could offer more random address-space layout randomization in the relatively near future.
