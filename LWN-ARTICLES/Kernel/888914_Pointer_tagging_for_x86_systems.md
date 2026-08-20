---
title: Pointer tagging for x86 systems
url: https://lwn.net/Articles/888914/
date: "March 28, 2022"
category: "Architectures-x86"
author: "By Jonathan Corbet March 28, 2022"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
March 28, 2022

Pointers are a fact of life for developers working in numerous languages. It is often convenient to be able to associate a small amount — a few bits at most — of ancillary information with a pointer. This can often be done within the pointer value itself with some careful masking and shifting. CPU manufacturers have been adding ways to support the addition of this sort of "tag" to pointers; the most recent may be AMD's "upper address ignore" (UAI) feature, support for which was recently [posted](<https://lwn.net/ml/linux-kernel/20220310111545.10852-1-bharata@amd.com/>) by Bharata B Rao. This feature has an uncertain future in Linux, though, as the result of a fundamental design decision. 

On a 64-bit system, a pointer is, naturally, 64 bits wide. But the CPU does not actually need all of those bits to dereference an address stored in a pointer. There are no systems (yet) that require — or can provide — all of the memory that can be addressed by 64 bits, meaning that there are ranges of address space that do not map to physical memory. Normally, user-space addresses start at (or near) zero and increase from there; that means that the highest-order bits will be zero even with the largest possible addresses. As a result, it can be possible to use those high-order bits to store other types of information. 

There are numerous use cases for stashing metadata into those unused bits. Memory allocators could use that space to track different memory pools, for example, or for garbage collection. Database management systems have their own uses for that space. Applications can implement this sort of tagging now, but it must be done with care; an address with extra bits set is no longer a valid pointer, so that metadata must be masked out before dereferencing that pointer or passing it into code that does not understand the tagging scheme. That is error-prone and may slow down the application. 

To make life easier for the developers of this sort of application, CPU manufacturers have been adding the ability for the processor to simply ignore the non-address bits in an address value. Naturally, every manufacturer has invented its own way of supporting this feature. The AMD version, UAI, specifically allows the uppermost seven bits of an address to be used for ancillary data. 

If accepted, AMD's implementation of this feature would not be the first; support for the Arm "top-byte ignore" feature was merged for the 5.4 kernel in 2019. At that time, a set of [`prctl()`](<https://man7.org/linux/man-pages/man2/prctl.2.html>) commands was added to control the use of this feature. Top-byte ignore can be enabled with: 

```
int prctl(PR_SET_TAGGED_ADDR_CTRL, PR_TAGGED_ADDR_ENABLE, 0, 0, 0);
```

This interface was designed around Arm's implementation, which makes eight bits available for tag data. The AMD implementation only allows for seven bits, meaning that applications wanting to use tagged addresses will need a way to discover how many bits are available. So Rao's patch set starts with [a patch from Kirill Shutemov](<https://lwn.net/ml/linux-kernel/20220310111545.10852-2-bharata@amd.com/>) (intended to add support for a similar Intel feature, more about that below) adding two new parameters to the above `prctl()` call, both of which are integer pointers. The first of those is for the caller to specify how many bits they would like to use for pointer metadata; the kernel will update that value to reflect the number of bits that are actually available. The second pointer tells the kernel where to store the number of bits to right-shift a pointer value to obtain the tag data. 

The subsequent patches then implement support for UAI in the Linux kernel. 

The idea is simple enough, and this feature already exists for the Arm architecture, but the UAI patches have still run into pushback, for a number of reasons. Perhaps the most fundamental of those is that UAI allows the most-significant bit of the address to be used by user space. In current systems, only kernel-space addresses have that bit set. Turning on UAI would allow user space to create pointer values that _look_ like kernel addresses, but which would actually be valid user-space pointers. Those pointers can, of course, be passed into the kernel via system calls where, in the absence of due care, they might be interpreted as kernel-space addresses. The consequences of such confusion would not be good, and the possibility of it happening is relatively high. 

This mechanism could probably be made to work safely, but, as Andy Lutomirski [said](<https://lwn.net/ml/linux-kernel/6a5076ad-405e-4e5e-af55-fe2a6b01467d@www.fastmail.com/>): ""A lot of auditing of existing code would be needed to make it safe"". Even more auditing would be required, of course, to keep it safe in a rapidly evolving kernel. It sounds like a recipe for ongoing security problems, which is why Thomas Gleixner [said](<https://lwn.net/ml/linux-kernel/87fsnac3pb.ffs@tglx/>) that ""there is no justification for the bit 63 abuse"". He suggested that AMD should rework the feature in its processors to disallow that bit in address tags; he did not say that this problem would block the merging of UAI, but the meaning was reasonably clear. 

Another problem that Lutomirski pointed out is that UAI is not specific to any running context; once it is enabled, it is turned on for the entire CPU. That, too, could lead to unpleasant surprises, so he suggested that the kernel would need to make the UAI settings process-local, even if it slows down context switches considerably. 

Finally, there is the issue of Intel's similar feature, called "Linear Address Masking" (LAM). It does not have the most-significant-bit issue that UAI has, and it is managed as part of the process context. It supports two modes, with either six or 15 bits being made available for ancillary data; the 15-bit mode only works if five-level page tables are not in use. LAM has been around for a while, and [support patches](<https://lwn.net/ml/linux-kernel/20210205151631.43511-1-kirill.shutemov%40linux.intel.com/>) were posted (by Shutemov) in early 2021. That work seems to have stalled after that posting, but can be expected to come back at some point. 

Rao's UAI patch set deliberately keeps the AMD implementation entirely separate from the proposed LAM implementation, even though the two are doing essentially the same thing. That led recently appointed x86 co-maintainer Dave Hansen to [object](<https://lwn.net/ml/linux-kernel/81b6f618-05bc-f7d0-5461-4c3f0ca42d3f@intel.com/>): ""We'll have one x86 implementation of address bit masking. Both the Intel and AMD implementations will feed into a shared implementation"". So this is something that would certainly need to be fixed before this work could be considered for mainline merging. 

The other issues are tied to the design of the hardware, though, and will be rather harder to fix in kernel code. For these reasons, the sentiment among kernel developers seems to be that LAM is a better-designed implementation of pointer tagging and should perhaps be what all x86 systems use. In the above-linked message, Lutomirski concluded: 

> I believe it's possible for a high-quality kernel UAI implementation to exist, but, as above, I think it would be slow, and it might be quite complex and fragile. Are we sure that it's worth supporting it? 

A better solution, he suggested, would be for AMD to go back to the drawing board and create its own implementation of LAM instead. 

In the early days of Linux, kernel developers had to adapt to whatever the hardware manufacturers put out; the alternative was to not have hardware to run on at all. In 2022, though, those developers feel more confident in their ability to reject support for hardware features that, for whatever reason, they feel do not fit in well with the design of the system. If AMD is unable to get support for UAI into the kernel (it's worth noting that Rao [hasn't given up yet](<https://lwn.net/ml/linux-kernel/b0861376-e628-06bd-713e-8837e0dc9d0b@amd.com/>)), UAI is likely to go mostly unused and developers needing pointer tagging may gravitate toward competing CPUs. According to Gleixner (linked above), AMD was told about the problems with its implementation some time ago; the company may yet have reason to wish it had listened.
