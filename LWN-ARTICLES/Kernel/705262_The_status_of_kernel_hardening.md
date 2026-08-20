---
title: The status of kernel hardening
url: https://lwn.net/Articles/705262/
date: "November 2, 2016"
category: "Security-Security technologies"
author: "By Jonathan Corbet November 2, 2016 2016 Kernel Summit"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
November 2, 2016

* * *

[2016 Kernel Summit](<https://lwn.net/Articles/KernelSummit2016/>)

At the [2015 Kernel Summit](<https://lwn.net/Articles/662219/>), Kees Cook said, he talked mostly about the things that the community _could_ be doing to improve the security of the kernel. In 2016, instead, he was there to talk about what had actually been done. Kernel hardening, he reminded the group, is not about access control or fixing bugs. Instead, it is about the kernel protecting itself, eliminating classes of exploits, and reducing its attack surface. There is still a lot to be done in this area, but the picture is better than it was one year ago. 

One area of progress is in the [integration of GCC plugins](<https://lwn.net/Articles/691102/>) into the build system. The plugins in the kernel now are mostly examples, but there will be more interesting ones coming in the future. Plugins are currently supported for the x86, arm, and arm64 architectures; he would like to see that list grow, but he needs help from the architecture maintainers to validate the changes. Plugins are also not yet used for routine kernel compile testing, since it is hard to get the relevant sites to install the needed dependencies. 

Linus asked how much plugins would slow the kernel build process; linux-next maintainer Stephen Rothwell also expressed interest in that question, noting that "some of us do compiles all day." Kees responded that there hadn't been a lot of benchmarking done, but that the cost was "not negligible." It is, though, an important part of protecting the kernel. 

#### Probabilistic protections

The kernel has adopted a number of probabilistic protections over the last year. These protections only work if the attacker doesn't know something about the system. They include kernel address-space layout randomization (KASLR) and stack protection. Probabilistic protections can be defeated if the information leaks out, but they are still effective and worth doing. 

One improvement is in the randomization of the kernel text base; it was added to arm64 in the 4.6 release and MIPS in 4.7. But the text base is only the beginning, more memory areas need to be randomized. One possibility is to randomize the kernel's link order at boot time. That would be a lot of work, but it would mean that an attacker would need more than a single information leak to defeat the whole thing. 

Linus said that randomization can be a pain for debugging; it is not fun to track down a problem that only happens in one boot out of every 300 or so. Al Viro worried that changing the link order would also change the order in which the kernel's initialization calls are made, with unpredictable [![\[Kees Cook\]](https://static.lwn.net/images/conf/2016/ks/KeesCook-sm.jpg)](<https://lwn.net/Articles/705268/>) effects. Kees responded that this particular change isn't coming anytime soon. Andi Kleen suggested just doing the link randomization and dropping KASLR altogether; the kernel's addresses tend to leak via all kinds of paths anyway. Linus responded that, while the address leaks are being plugged over time, KASLR does indeed work poorly against local attackers, but it is more useful against remote attackers. 

Kees went on to say that the kernel got KASLR for its memory areas in 4.8 for the x86_64 architecture. 

Work is being done on free-list randomization, which makes the layout of the heap less predictable. Perhaps more controversial is struct layout randomization. That cannot be done in a general way without causing all kinds of problems, but there is one place where it is especially useful: structs consisting of only function pointers. Such structs are one of the most prized targets for attackers, and the kernel has a lot of them. A GCC plugin can be used to detect these structures and randomize their order. In general, the kernel shouldn't care about that ordering, and changing it should not have performance effects. 

Linus was not entirely convinced; he said that most people are running distributor kernels, so the specific ordering used will always be available to an attacker. The value, Kees responded, is forcing attackers to identify specific kernel builds; that is "excruciating" for them. It greatly expands the number of settings their exploit has to work in. 

#### Deterministic protections

While probabilistic protections only work if some key data remains secret, Kees said, deterministic protections work all the time. These include things like read-only memory; if memory is read-only, it is always protected from being changed. Bounds checking to head off overflows is another form of deterministic protection. 

One useful protection is the `CONFIG_DEBUG_RODATA` configuration option which, Kees said, is badly named. It ensures that executable memory is not writable anywhere in the kernel; it should be mandatory on all systems that support it. It is turned on by default on the x86 architecture as of 4.6, and will be for arm64 as of 4.9. 

Another important protection is protection of user space against access by the processor when it is running in a privileged mode. By far the most common way to exploit the kernel, he said, is to get the kernel to execute code that has been placed somewhere in user-space memory. If the kernel cannot access that memory, such exploits will not work. Processor vendors have worked to provide such protections using technologies like SMAP and SMEP (on x86) and PAN (on ARM), but there is a problem: such protections are not widely available yet. There are no Xeon processors with ~~SMEP~~ SMAP on the market; PAN was added to the ARMv8.1 specification, but no hardware is shipping yet. 

So, he said, the kernel needs emulation of those features instead; it is, he said, a fundamental need. Linus replied, though, that he hates the emulation patches with a passion. And, he said, it is not necessary, in that the kernel's support for SMEP protects systems that lack SMEP too. That is because it forces all kernel paths that access user-space memory to be verified, preventing accidental accesses. So, he said, the emulation does not buy much. Kees disagreed, saying that the emulation can protect systems that will not have hardware protection for a few years yet. 

Work is being done on [hardened usercopy](<https://lwn.net/Articles/695991/>), which performs sanity checking on operations that copy data to and from user space. The current patch set contains about 1/3 of the PaX USERCOPY protections, which is a start. Next steps include segregating the slab caches; objects that are exposed to user space should be stored apart from those that are purely internal to the kernel. The problem here is to find a clean way to deal with exceptions. An `inode` object, for example, should not be copyable to or from user space, but there can be reasons to copy the file name stored within that structure. The PaX code does such copies by way of the stack, which is generally seen as being the wrong approach; Kees said that a more maintainable API for exceptions is needed. Linus added that this kind of problem is exactly why he has never seriously considered merging the grsecurity patch set; it's full of "this kind of craziness." 

Memory wiping is useful, in that it can block information leaks and some types of use-after-free exploits. The slab allocator can do poisoning of memory, but not zeroing, which would be nice to add. After Linus asked, Kees said that the advantage of zeroing is that the kernel often needs to allocate zeroed pages; if freed memory has already been zeroed, those allocations can be optimized. A problem with zeroing is that some objects are allocated and freed so often that the performance hit becomes prohibitive, so there needs to be a way to make exceptions. There is a GCC plugin out there to do stack clearing, which is worth looking at. 

"Constification" — making unchanging data constant — can protect against some types of exploits. The lowest-hanging fruit here is structs full of function pointers; the "constify" GCC plugin tries to make those `const` by default. As of 4.6, the kernel can make data read-only after initialization, but that feature is not yet widely used in the kernel. There would be value in identifying "write-rarely" data that would be read-only most of the time, and only made writable during explicit updates. 

Kees's final topic was [reference-count hardening](<https://lwn.net/Articles/693038/>). If an attacker is able to force a reference count to overflow, a use-after-free exploit is usually not far away. Most of these attacks can be blocked if atomic variables can be kept from overflowing. The hardening patches out there will kill the responsible process when an overflow is detected, and the counter involved is permanently blocked at a high value. In this way, an exploit is downgraded to a denial-of-service situation. 

[Kees's slides](<http://outflux.net/slides/2016/ks/kspp.pdf>) are available for the curious.
