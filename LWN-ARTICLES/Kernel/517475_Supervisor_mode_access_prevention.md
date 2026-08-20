---
title: Supervisor mode access prevention
url: https://lwn.net/Articles/517475/
date: "September 26, 2012"
category: "Security-Security technologies"
author: "By Jonathan Corbet September 26, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
September 26, 2012

Operating system designers and hardware designers tend to put a lot of thought into how the kernel can be protected from user-space processes. The security of the system as a whole depends on that protection. But there can also be value in protecting user space from the kernel. The Linux kernel will soon have support for a new Intel processor feature intended to make that possible. 

Under anything but the strangest (out of tree) memory configurations, the kernel's memory is always mapped, so user-space code could conceivably read and modify it. But the page protections are set to disallow that access; any attempt by user space to examine or modify the kernel's part of the address space will result in a segmentation violation (`SIGSEGV`) signal. Access in the other direction is rather less controlled: when the processor is in kernel mode, it has full access to any address that is valid in the page tables. Or nearly full access; the processor will still not normally allow writes to read-only memory, but that check can be disabled when the need arises. 

Intel's new "Supervisor Mode Access Prevention" (SMAP) feature changes that situation; those wanting the details can find them starting on page 408 of [this reference manual [PDF]](<http://software.intel.com/sites/default/files/319433-014.pdf>). This extension defines a new SMAP bit in the CR4 control register; when that bit is set, any attempt to access user-space memory while running in a privileged mode will lead to a page fault. Linux support for this feature has been [posted](<https://lwn.net/Articles/517251/>) by H. Peter Anvin to generally positive reviews; it could show up in the mainline as early as 3.7. 

Naturally, there are times when the kernel needs to work with user-space memory. To that end, Intel has defined a separate "AC" flag that controls the SMAP feature. If the AC flag is set, SMAP protection is in force; otherwise access to user-space memory is allowed. Two new instructions (STAC and CLAC) are provided to manipulate that flag relatively quickly. Unsurprisingly, much of Peter's patch set is concerned with adding STAC and CLAC instructions in the right places. User-space access functions (`get_user()`, for example, or `copy_to_user()`) clearly need to have user-space access enabled. Other places include transitions between kernel and user mode, futex operations, floating-point unit state saving, and so on. Signal handling, as usual, has special requirements; Peter had to make some significant changes to allow signal delivery to happen without excessive overhead. 

Speaking of overhead, support for this feature will clearly have its costs. User-space access functions tend to be expanded inline, so there will be a lot of STAC and CLAC instructions spread around the kernel. The "[alternatives](<https://lwn.net/Articles/164121/>)" mechanism is used to patch them out if the SMAP feature is not in use (either not supported by the kernel or disabled with the `nosmap` boot flag), but the kernel will grow a little regardless. The STAC and CLAC instructions also require a little time to execute. Thus far, no benchmarks have been posted to quantify what the cost is; one assumes that it is small but not nonexistent. 

The kernel will treat SMAP violations like it treats any other bad pointer access: the result will be an oops. 

One might well ask what the value of this protection is, given that the kernel can turn it off at will. The answer is that it can block a whole class of exploits where the kernel is fooled into reading from (or writing to) user-space memory by mistake. The set of [null pointer vulnerabilities](<https://lwn.net/Articles/342330/>) exposed a few years ago is one obvious example. There have been many situations where an attacker has found a way to get the kernel to use a bad pointer, while the cases where the attacker could execute arbitrary code in kernel space (before exploiting the bad pointer) have been far less common. SMAP should block the more common attacks nicely. 

The other benefit, of course, is simply finding kernel bugs. Driver writers (should) know that they cannot dereference user-space pointers directly from the kernel, but code that does so tends to work on some architectures anyway. With SMAP enabled, that kind of mistake will be found and fixed earlier, before the bad code is shipped in a mainline kernel. As is so often the case, there is real value in having the system enforce the rules that developers are supposed to be following. 

Linus [liked the patch set](<https://lwn.net/Articles/517476/>) and nobody else has complained, so the changes have found their way into the "tip" tree. That makes it quite likely that we will see them again quite soon, probably once the 3.7 merge window opens. It will take a little longer, though, to get processors that support this feature; SMAP is set to first appear in the [Haswell](<https://en.wikipedia.org/wiki/Haswell_%28microarchitecture%29>) line, which should start shipping in 2013. But, once the hardware is available, Linux will be able to take advantage of this new feature.
