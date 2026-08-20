---
title: Supporting Intel MPX in Linux
url: https://lwn.net/Articles/582712/
date: "January 29, 2014"
category: "Security-Security technologies"
author: "By Jonathan Corbet January 29, 2014"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
January 29, 2014

Buffer overflows have long been a source of serious bugs and security problems at all levels of the software stack. Much work has been done over the years to eliminate unsafe library functions, add stack-integrity checking and more, but buffer overflow bugs still happen with great regularity. A recently posted kernel patch is one of the final steps toward the availability of a new tool that should help to make buffer overflow problems more uncommon: Intel's upcoming "MPX" hardware feature. 

MPX is, at its core, a hardware-assisted mechanism for performing bounds checking on pointer accesses. The hardware, following direction from software, maintains a table of pointers in use and the range of accessible memory (the "bounds") associated with each. Whenever a pointer is dereferenced, special instructions can be used to ensure that the program is accessing memory within the range specified for that particular pointer. These instructions are meant to be fast, allowing bounds checking to be performed on production systems with a minimal performance impact. 

As one might expect, quite a bit of supporting software work is needed to make this feature work, since the hardware cannot, on its own, have any idea of what the proper bounds for any given pointer would be. The first step in this direction is to add support to the GCC compiler. [Support for MPX in GCC](<http://gcc.gnu.org/wiki/Intel%20MPX%20support%20in%20the%20GCC%20compiler>) is well advanced, and should be considered for merging into the repository trunk sometime in the near future. 

When a file is compiled with the new `-fmpx` flag, GCC will generate code to make use of the MPX feature. That involves tracking every pointer created by the program and the associated bounds; any time that a new pointer is created, it must be inserted into the bounds table for checking. Tracking of bounds must follow casts and pointer arithmetic; there is also a mechanism for "narrowing" a set of bounds when a pointer to an object within another object (a specific structure field, say) is created. The function-call interface is changed so that when a pointer is passed to a function, the appropriate bounds are passed with it. Pointers returned from functions also carry bounds information. 

With that infrastructure in place, it becomes possible to protect a program against out-of-bounds memory accesses. To that end, whenever a pointer is dereferenced, the appropriate instructions are generated to perform a bounds check first. See [Documentation/x86/intel_mpx.txt](<https://lwn.net/Articles/582739/>), included with the kernel patch set (described below), for details on how code generation changes. In brief: the new `bndcl` and `bndcu` instructions check a pointer reference against the lower and upper limits, respectively. If those instructions succeed, the pointer is known to be within the allowed range. 

The next step is to prepare the C library for bounds checking. At a minimum, that means building the library with `-fmpx`, but there is more to it than that. Any library function that creates an object (`malloc()`, say) needs to return the proper bounds along with the pointer to the object itself. In the end, the C library will be the source for a large portion of the bounds information used within an application. The bulk of the work for the GNU C library (glibc) is evidently done and committed to the glibc git repository. Instrumentation of other libraries would also be desirable, of course, but the C library is the obvious starting point. 

Then there is the matter of getting the necessary support code into the kernel; Qiaowei Ren has recently posted [a patch set](<https://lwn.net/Articles/582473/>) to do just that. Part of the patch set is necessarily management overhead: allowing applications to set up bounds tables, removing bounds tables when the memory they refer to is unmapped, and so on. But much of the work is oriented around the user-space interface to the MPX feature. 

The first step is to add two new `prctl()` options: `PR_MPX_INIT` and `PR_MPX_RELEASE`. The first of those sets up MPX checking and turns on the feature, while the second cleans everything up. Applications can thus explicitly control pointer bounds checking, but that is not expected. Instead, the system runtime will probably turn on MPX as part of application startup, before the application itself begins to run. Current discussion on the linux-kernel list suggests that it may be possible to do the entire setup and teardown job within the user-space runtime code, making these `prctl()` calls unnecessary, so they may not actually find their way into the mainline kernel. 

When a bounds violation is detected, the processor will trap into the kernel. The kernel, in turn, will turn the trap into a `SIGSEGV` signal to be delivered to the application, similar to other types of memory access errors. Applications that look at the `siginfo` structure passed to the signal handler from the kernel will be able to recognize a bounds error by checking the `si_code` field for the new `SEGV_BNDERR` value. The offending address will be stored in `si_addr`, while the bounds in effect at the time of the trap will be stored in `si_lower` and `si_upper`. But most programs, of course, will not handle `SIGSEGV` at all and will simply crash in this situation. 

In summary, there is a fair amount of development work needed to make this hardware feature available to user applications. The good news is that, for the most part, this work appears to be done. Using MPX within the kernel itself should also be entirely possible, but no patches to that effect have been posted so far. Adding bounds checking to the kernel without breaking things is likely to present a number of interesting challenges; for example, narrowing would have to be reversed anytime the `container_of()` macro is used — and there are thousands of `container_of()` calls in the kernel. Finding ways to instrument the kernel would thus be tricky; doing this instrumentation in a way that does not make a mess out of the kernel source could be even harder. But there would be clear benefits should somebody manage to get the job done. 

Meanwhile, though, anybody looking forward to MPX will have to wait for a couple of things: hardware that actually supports the feature and distributions built to use it. MPX is evidently a part of Intel's "Skylake" architecture, which is not expected to be commercially available before 2015 at the earliest. So there will be a bit of a wait before this feature is widely available. But, by the time it happens, Linux should be ready to take advantage of it.
