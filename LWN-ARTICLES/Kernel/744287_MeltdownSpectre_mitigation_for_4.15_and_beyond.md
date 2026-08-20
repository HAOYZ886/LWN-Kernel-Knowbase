---
title: Meltdown/Spectre mitigation for 4.15 and beyond
url: https://lwn.net/Articles/744287/
date: "January 15, 2018"
category: "Security-Meltdown and Spectre"
author: "By Jonathan Corbet January 15, 2018"
---

By **Jonathan Corbet**  
January 15, 2018

While some aspects of the kernel's defenses against the Meltdown and Spectre vulnerabilities were more-or-less in place when the problems were disclosed on January 3, others were less fully formed. Additionally, many of the mitigations (especially for the two Spectre variants) had not been seen in public prior to the disclosure, meaning that there was a lot of scope for discussion once they came out. Many of those discussions are slowing down, and the kernel's initial response has mostly come into focus. The 4.15 kernel will include a broad set of mitigations, while some others will have to wait for later; read on for details on where things stand. 

[This article from January 5](<https://lwn.net/Articles/743265/>) gives an overview of the defenses for all three vulnerability variants. That material will not be repeated here, so those who have not read it may want to take a quick look before proceeding. 

#### Variant 1

On its surface, the mitigation for Spectre variant 1 (speculative bounds-check bypass) hasn't changed much. In [the latest patch set](<https://lwn.net/Articles/744257/>) from Dan Williams, the proposed `nospec_array_ptr()` macro has been renamed to just `array_ptr()`: 

```
array_ptr(array, index, size)
```

Its function remains the same: it returns a pointer that is either within the given `array` or NULL and prevents the processor from speculating with values that are outside the array. The implementation of this macro has been the subject of some debate, though. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The initial implementation used the Intel-blessed mechanism of inserting an `lfence` instruction as a barrier to prevent speculation past the bounds check. But barriers are relatively expensive, so this approach generated a fair amount of concern about its performance impacts, though few actual measurements were posted. In response, a different approach, which appears to have originated with Alexei Starovoitov, is being explored. It takes a different tack; rather than disabling speculation, it tries to ensure that any speculation that does occur remains within the bounds of the array being accessed. 

The trick is to AND the pointer value with a mask which is generated in the following way, given a constant `size` and a possibly hostile `index`: 

```
mask = ~(long)(index | (size - 1 - index)) >> (BITS_PER_LONG - 1);
```

If `index` is larger than `size`, the subtraction at the core of the macro will generate a negative number. Putting in the index again with an OR operation also ensures that the sign bit will be set for the largest `index` values that might otherwise cause the subtraction to underflow back to a positive number. The subsequent right-shift by `BITS_PER_LONG-1` has the effect of replicating the sign bit through the entire mask, yielding a mask that is either all zeros or all ones; the latter case happens when the `index` is too large. Finally, the "`~`" at the beginning inverts all the bits. The result: a mask that is all ones for a valid index, all zeroes otherwise. (Note that there is an x86 implementation of this computation that comes down to two instructions). 

The key point here is that, if the processor speculates a load from the array with a given `index` value, it will speculate the mask generation from the _same_ value. That should ensure that the mask is appropriate to the `index` used and cause the right thing to happen when the mask is ANDed with the pointer value, heading off any attempts to force speculative loads outside of the bounds of the array. There seems to be a high level of confidence that the processor will not speculate on any of the data values used in the masking operation — speculation is almost entirely limited to _control_ decisions, not data values. Normal speculation and reordering can continue, though, retaining the performance of the code overall. 

This would appear to be an optimal solution to the problem. It seems that some developers are not yet fully comfortable with this approach, though; they [worry](<https://lwn.net/Articles/744349/>) that there is still room for the processor to mis-speculate the calculation of the mask, perhaps abetted by optimizations done by the compiler. The fact that the processor vendors have not given any assurances to the contrary gives weight to those concerns. Linus Torvalds, instead, [believes](<https://lwn.net/Articles/744350/>) that the masking approach is actually safer than using barriers. Even so, some would like to stick with the barrier-based approach. The current patches, as posted, offer both approaches, controlled by a configuration option. 

The other significant problem — finding the places where this macro needs to be used — remains unsolved. The current patch set leaves out most of the locations that had been protected in previous versions, since a number of them proved to be controversial. As of this writing, the variant-1 defenses have not yet found their way into the mainline, but that could yet change in this rather atypical development cycle. 

#### Variant 2

Variant 2 (poisoning of the branch prediction buffer) is primarily protected against using the "retpoline" mechanism, which replaces indirect jumps and calls with a dance that defeats speculation. This mechanism was merged into the mainline for the 4.15-rc8 release — a late date indeed for a change of this magnitude — and there are still a few small pieces missing. Given the short time involved and the number of questions needing answers, though, it would not have been possible to get this work done any sooner. 

There were various discussions about implementation details, and ongoing uncertainty over whether retpolines are a sufficient protection against variant 2 on Intel Skylake processors. The problem on Skylake has to do with another data structure internal to the processor: the return stack buffer (RSB). Normally, this buffer is used to predict the address used in a "return" instruction, but there are situations where this buffer can run out of entries. That generally happens when the call stack is made deeper without the processor knowing about it; just about any sort of context switch can cause that to happen, for example. On a Skylake processor, an RSB underflow will cause a fallback to the branch prediction buffer instead, turning any "return" into a possible attack point. 

It may also be possible, on some other processors, for user space to populate the RSB with hostile values, once again enabling the wrong kind of speculation. The answer in either case is the same: stuff the RSB full of well-known values in places (like context switches) where things could go wrong. The RSB-stuffing patches have been circulating for a while; they have not yet been merged but that should happen in the near future. 

One other issue with retpolines remains somewhat unresolved, though: using them requires support from the compiler, and almost nobody has a compiler with that support available. Support for GCC was only [posted](<https://gcc.gnu.org/ml/gcc-patches/2018-01/msg00422.html>) by H.J. Lu on January 7; those patches were then subjected to a fair amount of ... discussion ... on the details that threatened to delay their merging indefinitely. Richard Biener finally jumped in to [request](<https://gcc.gnu.org/ml/gcc-patches/2018-01/msg00888.html>) that the process be expedited a bit: 

And I'd also like people not to bikeshed too much on this given we're in the situation of having exploitable kernels around for which we need a cooperating compiler. So during the time we bikeshed this (rather than reviewing the actual patches) we have to "backport" the current non-upstream state anyway to deliver fixed kernels to our customer. 

That seems to have been enough to at least bring about agreement that this feature would be requested with the `-mindirect-branch=thunk-extern` compiler option. The GCC developers did force a name change for the retpoline thunk itself, though, breaking the existing kernel patches and making the compiler (when released) incompatible with the version that distributors have been using to create fixed kernels thus far. If that change sticks, it will require [more 4.15 patches](<https://lwn.net/Articles/744303/>) in the immediate future. 

Meanwhile, the IBRS feature is being added to the microcode for some processors to defend against variant-2 attacks, but the degree to which the kernel will use it is still unclear. Setting the IBRS bit in a model-specific register acts as a sort of barrier, preventing bad values placed in the branch prediction buffer from being used when speculating the execution of code in the kernel. IBRS is generally considered inferior to retpolines because it has a much higher performance impact, though that cost is lower on the newest CPUs. An extensive mailing-list discussion made it clear that few people truly understand how IBRS is meant to work or when it should be used. A [rather frustrated series of questions](<https://lwn.net/Articles/744301/>) from Thomas Gleixner elicited some answers, but only after a considerable amount of contradictory information had been passed around. 

Work on IBRS seems to have slowed for now, though, perhaps because retpolines are now seen as being good enough for Skylake processors — for now, at least. As Gleixner [put it](<https://lwn.net/Articles/744302/>), the IBRS question can now be resolved in a non-emergency mode: 

The further RSB vs. IBRS discussion has to be settled in the way we normally work. We need full documentation, proper working micro code and actual comparisons of the two approaches vs. performance, coverage of attack vectors and code complexity/ugliness. 

The remaining [concern](<https://lwn.net/Articles/744352/>) on Skylake processors would appear to be system-management interrupts (SMIs), which can cause unprotected code to be run in kernel context. There does not appear to be a consensus that SMIs are exploitable in the real world, though, and no known proofs of the concept. Still, David Woodhouse has [stated his intent](<https://lwn.net/Articles/744353/>) to eventually have Skylake processors use IBRS by default, with retpolines as a boot-time option. But, as he [pointed out](<https://lwn.net/Articles/744354/>), this outcome has been slowed by the lack of anybody pushing the IBRS patches forward at an acceptable rate. 4.15 looks set to release without IBRS support, but it will almost certainly show up in the relatively near future. 

#### Variant 3

Variant 3 (the "Meltdown" vulnerability) allows a user-space process to read the contents of kernel memory on a vulnerable system. The defense against this problem is [kernel page-table isolation](<https://lwn.net/Articles/741878/>) (KPTI), which has been developed in public since early November. It was merged for the 4.15-rc5 and has remained mostly unchanged since then — if one doesn't count a rather large number of bug fixes. Such a fundamental memory-management change was never going to be without glitches, but they are being found and dealt with, one at a time. 

The biggest upcoming change to KPTI is certainly the ability to control its use on a per-process basis. KPTI is an expensive mitigation, with overheads of 30% or more reported for some specific workloads (though most workloads will not see an impact of that magnitude). The `nopti` command-line option can be used to disable KPTI entirely, but there are likely to be settings where an administrator wishes to exempt specific performance-critical processes from KPTI while retaining that protection for the system as a whole. Willy Tarreau has been working on [a patch set](<https://lwn.net/Articles/744292/>) to provide that capability, but there are some remaining differences of opinion on how it should work. 

Tarreau's patch set adds a couple of new operations to the `ptrace()` system call: `ARCH_DISABLE_PTI_NOW` and `ARCH_DISABLE_PTI_NEXT`. The first immediately disables KPTI for the calling process, while the latter merely sets a flag that causes KPTI to be disabled for the process after it makes a call to `execve()`. The `CAP_SYS_RAWIO` capability is required to be able to disable KPTI. There is also a sysctl knob (`/proc/sys/vm/pti_adjust`) that can be used to disable these operations, either temporarily or permanently. 

Many aspects of this interface have been discussed without a whole lot of conclusions. The current proposal works at the process level, for example; it is not possible for different threads within a process to have a different KPTI state. Some developers, though, think that thread-level control makes more sense. Another point of discussion was whether both the "now" and "next" modes are needed, but there is, naturally, disagreement over which of the two should go. Linus Torvalds was [adamant](<https://lwn.net/Articles/744297/>) that the "next" mode is the right one, because the natural place to disable KPTI is in an external wrapper program: 

Processes should never say "I'm so important that I'm disabling PTI". That's crazy talk, and wrong. It's wrong for all the usual reasons - everybody always thinks that _their_ own work is so important and bug-free, and that things like PTI are about protecting all those other incompetent people. 

Instead, he said, the decision to disable KPTI should be made by an external program run by the administrator. As might be expected for this group, the first use case for such a wrapper would be a `nopti` wrapper that could be used to run kernel builds without KPTI. 

Andy Lutomirski has [proposed](<https://lwn.net/Articles/744298/>) that a new capability (`CAP_DISABLE_PTI`) should control access to this functionality rather than `CAP_SYS_RAWIO`. That would make a lot of the existing privilege checks just work without the need to add a bunch of new infrastructure. The idea is somewhat controversial, though, and it's not clear whether it will make it into the final version of this feature. 

All told, there are a number of unresolved issues around how per-process KPTI control should work, even though everybody involved seems to agree that the feature itself should exist. The 4.15 kernel will be released without the per-process KPTI feature, and it would be surprising to see it get into 4.16 as well. 

#### In conclusion

After all of this work, it would appear that the 4.15 kernel will be released with fairly complete Meltdown and Spectre protection, though a number of sharp edges are sure to remain. But, quoting Gleixner again, the time has come to slow down a bit: 

Surely we all know there is room for improvements, but we also have reached a state where the remaining issues are not longer to be treated in full emergency and panic mode. We're good now, but not perfect. [...] 

We all are exhausted and at our limits and I think we can agree that having the most problematic stuff covered is the right point to calm down and put the heads back on the chickens. Take a break and have a few drinks at least over the weekend! 

Those of us who know Gleixner can be fairly well assured that he will have taken his own advice. 

All told, this set of vulnerabilities has been an intense death march for a number of kernel developers, most (or all) of whom were not informed of the problems until months after their discovery. Many of them were doing this work as part of their normal job, but others jumped in just because the work needed to be done. All of them were working to address issues that were not of their making in any way. As a result of their effort, Linux systems are reasonably well protected from these problems. We are all very much in their debt.
