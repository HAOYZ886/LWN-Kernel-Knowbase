---
title: "Support for Intel's Linear Address Masking"
url: https://lwn.net/Articles/902094/
date: "July 25, 2022"
category: "Architectures-x86"
author: "By Jonathan Corbet July 25, 2022"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
July 25, 2022

A 64-bit pointer can address a lot of memory — far more than just about any application could ever need. As a result, there are bits within that pointer that are not really needed to address memory, and which might be put to other needs. Storing a few bits of metadata within a pointer is a common enough use case that multiple architectures are adding support for it at the hardware level. Intel is no exception; support for its "Linear Address Masking" (LAM) feature has been slowly making its way toward the mainline kernel. 

CPUs can support this metadata by simply masking off the relevant bits before dereferencing a pointer. Naturally, every CPU vendor has managed to support this feature differently. Arm's [top-byte ignore](<https://en.wikichip.org/wiki/arm/tbi>) feature allows the most-significant byte of the address to be used for non-pointing purposes; it has been supported by the Linux kernel since 5.4 came out in 2019. AMD's "upper address ignore" feature, instead, only allows the seven topmost bits to be used in this way; support for this feature was [proposed earlier this year](<https://lwn.net/Articles/888914/>) but has not yet been accepted. 

One of the roadblocks in the AMD case is that this feature would allow the creation of valid user-space pointers that have the most-significant bit set. In current kernels, only kernel-space addresses have that bit set, and an unknown amount of low-level code depends on that distinction. The consequences of confusing user-space and kernel-space addresses could be severe and contribute to the ongoing CVE-number shortage, so developers are nervous about any feature that could cause such confusion to happen. Quite a bit of code would likely have to be audited to create any level of confidence that allowing user-space addresses with that bit set would not open up a whole set of security holes. 

Intel's LAM feature offers two modes, both of which are different from anybody else's: 

  * `LAM_U57` allows six bits of metadata in bits 62 to 57\. 
  * `LAM_U48` allows 15 bits of metadata in bits 62 to 48\. 

It's worth noting that neither of these modes allows bit 63 (the most-significant bit) to be used for this purpose, so LAM avoids the pitfall that has created trouble for AMD. 

Support for LAM is added by [this patch set](<https://lwn.net/ml/linux-kernel/20220712231328.5294-1-kirill.shutemov@linux.intel.com/>) from Kirill Shutemov. Since LAM must be enabled in the CPU by privileged code, the patch set introduces a new API in the form of two new [`arch_prctl()`](<https://man7.org/linux/man-pages/man2/arch_prctl.2.html>) commands that are designed to be able to support any CPU's pointer-metadata mechanism. The first, `ARCH_ENABLE_TAGGED_ADDR`, enables the use of LAM for the current process; it takes an integer argument indicating how many bits of data the process wishes to store in pointers and selects the mode (from the above set) that holds at least that many. 

Programs trying to use this feature need to know where they can store their metadata within a pointer; this needs to happen in a general way if such programs are to be portable across architectures. The second `arch_prctl()` operation, `ARCH_GET_UNTAG_MASK`, returns a 64-bit value with bits set to indicate the available space. The patch set also adds a line to each process's `arch_status` file in `/proc` indicating the effective mask. 

The LAM patches are, for the most part, uncontroversial; the LAM feature is seen as being better designed than AMD's equivalent. That said, there are some ongoing concerns about the `LAM_U48` mode in particular that may prevent it from being supported in Linux anytime soon. 

Linux has supported [five-level page tables](<https://lwn.net/Articles/717293/>) on x86 systems since the 4.11 release in 2017. On a system with five-level page tables enabled, 57 bits of address space are available to running processes. The kernel will not normally map memory into the upper nine bits of that address space, which is the part added by the fifth page-table level, out of fear of breaking applications; among other things, some programs may be storing metadata in those bits even without hardware support. More care must be taken when applying this trick since the metadata bits must always be masked out before dereferencing a pointer, but it is possible. Programs that do this would obviously break, though, if those bits became necessary to address memory within their address space. 

To avoid this kind of problem, the kernel will only map memory into the upper part of the address space if the application explicitly asks for it in an [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) call. It's a rare application that will need to do that, and should be a relatively easy thing to add in cases where it's necessary. 

The `LAM_U48` mode, which uses 15 pointer bits and only leaves 48 significant bits for the actual address, is clearly inconsistent with any attempt to use a 57-bit address space. One might argue that any programmer who tries to use both together deserves the resulting explosions, but it is better to simply not provide useless footguns whenever possible. Since the kernel already plays a role in the use of both modes, it is well placed to ensure that they are not used together. 

In Shutemov's patch set, the enabling of `LAM_U48` is relegated to a set of optional patches at the end; if they are left out, the kernel will only support the `LAM_U57` mode, which is certainly one way to solve the problem. If the patches are included, instead, then user space must choose which of the two features it will use (if either); the mode that is selected first wins. If `LAM_U48` is enabled, the ability to map memory in the upper nine bits of the address space will be permanently removed from that process. But if the process has already mapped memory there when it tries to enable `LAM_U48`, the attempt will fail. 

It seems like a reasonable solution that would make all of the functionality available and let processes choose which they will actually use, but developers remain concerned about the `LAM_U48` mode. Alexander Potapenko [suggested](<https://lwn.net/ml/linux-kernel/CAG_fn=WR3s3UMh76+bibN0nUpZk9AS_M18=oxP+pc_vtqKt34A@mail.gmail.com/>) that distributors would want to remove this mode if it makes it into the mainline, but that it would become harder to do so over time as other changes land on top of it. Dave Hansen, one of the x86 maintainers, [said](<https://lwn.net/ml/linux-kernel/c5dc0415-8c2d-8c9c-5bdc-824c267aa960@intel.com/>) that he would not merge `LAM_U48` immediately, but would consider doing so in the future. 

So, while there does not seem to be much to impede the adoption of LAM in general, it is not clear that _all_ of the LAM patches will be merged anytime soon. If there are people with use cases for `LAM_U48` out there, this might be a good time to make those use cases known; otherwise they may find that the feature is unavailable to them.
