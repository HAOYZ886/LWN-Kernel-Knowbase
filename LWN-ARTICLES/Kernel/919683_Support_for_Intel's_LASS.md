---
title: "Support for Intel's LASS"
url: https://lwn.net/Articles/919683/
date: "January 13, 2023"
category: "Architectures-x86; Security-Meltdown and Spectre"
author: "By Jonathan Corbet January 13, 2023"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
January 13, 2023

Speculative-execution vulnerabilities come about when the CPU, while executing speculatively, is able to access memory that would otherwise be denied to it. Most of these vulnerabilities would go away if the CPU were always constrained by the established memory protections. An obvious way to fix these problems would be to make CPUs behave that way, but doing that without destroying performance is not an easy task. So, instead, Intel has developed a feature called "linear address-space separation" (LASS) to paper over parts of the problem; Yian Chen has posted [a patch set](<https://lwn.net/ml/linux-kernel/20230110055204.3227669-1-yian.chen@intel.com/>) adding support for this feature. 

Speculative execution happens when the CPU is unable to complete an instruction because it needs data that is not resident in the CPU's caches. Rather than just wait for that data to be fetched from RAM, the CPU will make a guess as to its value and continue running in the speculative mode. If the guess turns out to be correct — which happens surprisingly often — the CPU will have avoided a stall and will be ahead of the game; otherwise, the work that was done speculatively is thrown out and the computation restarts. 

This technique is crucial for getting reasonable performance out of current CPUs, but it turns out to have a security cost: speculative execution is allowed to access data that would be denied to code running normally. A CPU will be able to speculatively read data, despite permissions denying that access in the page tables, without generating a fault. That data is never made available to the running process, but accessing it can create state changes (such as loading data into the cache) that can be detected by a hostile program and used to exfiltrate data that should not be readable. In response, kernel developers have adopted a number of techniques, including address-space isolation and preemptive cache clearing, to block these attacks, but those mitigations can have a substantial performance cost. 

LASS partially addresses the speculative-execution problem by wiring some address-space-management policy into the hardware. A look at, for example, the [Linux x86-64 address-space layout](<https://www.kernel.org/doc/html/latest/x86/x86_64/mm.html>) shows that all kernel-space addresses begin with `0xffff`. More to the point, they all have the highest-order (sign) bit set, while all user-space addresses have that bit clear. Linux is not the only kernel to partition the 64-bit address space in this way. LASS uses this convention (and, indeed, requires it) to provide some hardware-based address-space isolation. 

Specifically, when LASS is enabled, the CPU will intercept any user-mode reference to an address with the sign bit set, or any kernel-mode access with that bit clear. In other words, it prevents either mode from accessing addresses that, according to the sign bit, belong to the other mode. Crucially, this policy is applied early in the execution of an instruction. Normal page protections can only be read (and, thus, enforced) by traversing through the page-table hierarchy, which produces timing and cache artifacts. LASS can trap a forbidden access simply by looking at the address, without any reference to the page tables, yielding constant timing and avoiding any internal state changes. And this test is easily performed during speculative execution as well. 

Of course, adding a new protection mechanism like this requires adaptation in the kernel, which must disable LASS when it legitimately needs to access user-space memory. Most of the infrastructure needed to handle this is already in place, since [supervisor-mode access prevention](<https://lwn.net/Articles/517475/>) must be handled in a similar way. There is a problem, though, with the [vsyscall](<https://lwn.net/Articles/446528/>) mechanism, which is a virtual system-call implementation. The vsyscall area is hardwired to be placed between the virtual addresses `ffffffffff600000` and `ffffffffff601000`. Since the sign bit is set in those addresses, LASS will block accesses from user mode, preventing vsyscalls from working. LASS is thus mutually exclusive with vsyscalls; if one is enabled, the other must be disabled. Vsyscalls have long since been replaced by the vDSO, but there may be old versions of the C library out there that still use them. If LASS support is merged, distributors will have to decide which feature to enable by default. 

LASS should be able to protect against speculative attacks where user space is attempting to extract information from the kernel — Meltdown-based attacks in particular. It may not directly block most Spectre-based attacks, which generally involve speculative execution entirely in kernel space, but it may still be good enough to block the cache-based covert channels used to get information out of the kernel. The actual degree of protection isn't specified in the patches, though, leading Dave Hansen to [ask for more information](<https://lwn.net/ml/linux-kernel/8ac54f52-5bf0-bf6c-2473-7f0cf2a1a957@intel.com/>): 

> LASS seemed really cool when we were reeling from Meltdown. It would *obviously* have been a godsend five years ago. But, it's less clear what role it plays today and how important it is. 

If LASS can allow some of the more expensive Meltdown and Spectre mitigations to be turned off without compromising security, it seems worth having. But, for now, nobody has said publicly which mitigations, if any, are rendered unnecessary by LASS. 

In any case, it is not possible to buy a CPU that supports LASS now; it will be necessary to wait until processors from the "Sierra Forest" line become available. Once those CPUs get out to where they can be tested, the value of LASS will, hopefully, become more clear. Until then, the development community will have to do its best to decide whether a partial fix to speculative-execution problems is better than the current state of affairs.
