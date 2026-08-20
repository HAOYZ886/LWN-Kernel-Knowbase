---
title: "Developers split over split-lock detection"
url: https://lwn.net/Articles/806466/
date: "December 6, 2019"
category: "Architectures-x86"
author: "By Jonathan Corbet December 6, 2019"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
December 6, 2019

A "split lock" is a low-level memory-bus lock taken by the processor for a memory range that crosses a cache line. Most processors disallow split locks, but x86 implements them. Split locking may be convenient for developers, but it comes at a cost: a single split-locked instruction can occupy the memory bus for around 1,000 clock cycles. It is thus understandable that interest in eliminating split-lock operations is high. What is perhaps less understandable is that a patch set intended to detect split locks has been pending since (at least) May 2018, and it still is not poised to enter the mainline. 

Split locks are, in essence, a type of misaligned memory access — something that the x86 architecture has always been relatively willing to forgive. But there is a difference: while a normal unaligned operation will slow down the process performing that operation, split locks will slow down the entire system. The 1,000-cycle delay may be particularly painful on realtime systems, but split locks can be used as an effective denial-of-service attack on any system. There is little disagreement among developers that their use should not be allowed on most systems. 

Recent Intel processors can be configured to execute a trap when a split lock is attempted, allowing the operating system to decide whether to allow the operation to continue. Fenghua Yu first posted [a patch set](<https://lwn.net/ml/linux-kernel/1527435965-202085-1-git-send-email-fenghua.yu@intel.com/>) enabling this trap in May 2018; LWN [covered this work](<https://lwn.net/Articles/790464/>) one year later. While many things have changed in this patch set, the basic idea has remained constant: when this feature is enabled, user-space processes that attempt a split lock will receive a `SIGBUS` signal. What happens when split locks are created in other contexts has varied over time; in [the version-10 patch set](<https://lwn.net/ml/linux-kernel/1574297603-198156-1-git-send-email-fenghua.yu@intel.com/>) posted in late November, split locks in the kernel or system firmware will cause a kernel panic. 

This severe response should result in problems being fixed quickly; Yu cites a couple of kernel fixes for split locks detected by this work in the patch posting. That will only happen, though, if this feature is enabled on systems where the code tries to create a split lock, but that may not happen as often as developers would like: 

The split lock detection is disabled by default because potential split lock issues can cause kernel panic or kill user processes. It is enabled only for real time or debug purpose through a kernel parameter "split_lock_detect". 

Disabling the feature by default guarantees that it will not be widely used; that has led to some complaints. Ingo Molnar [asserted](<https://lwn.net/ml/linux-kernel/20191121171401.GE12042@gmail.com/>) that ""for this feature to be useful it must be default-enabled"", and Peter Zijlstra [said](<https://lwn.net/ml/linux-kernel/20191121130153.GS4097@hirez.programming.kicks-ass.net/>): 

This feature MUST be default enabled, otherwise everything will be/remain broken and we'll end up in the situation where you can't use it even if you wanted to. 

Zijlstra is particularly concerned about split locks created by code at the firmware level, something he sees as being likely: ""from long and painful experience we all know that if a BIOS can be wrong, it will be"". Enabling split-lock detection by default will hopefully cause firmware-created split locks to be fixed. Otherwise, he fears, those split locks will lurk in the firmware indefinitely, emerging occasionally to burn users who enable split-lock detection in the hope of finding problems in their own code. 

Forcing problems to be fixed by enabling split-lock detection by default has some obvious appeal. But the reasoning behind leaving it disabled also makes some sense: killing processes that create split locks is an ABI change that may create problems for users. As Tony Luck [put it](<https://lwn.net/ml/linux-kernel/20191121173444.GA5581@agluck-desk2.amr.corp.intel.com/>): 

Enabling by default at this point would result in a flurry of complaints about applications being killed and kernels panicking. That would be followed by: 

#include <linus/all-caps-rant-about-backwards-compatability.h>

and the patches being reverted. 

Zijlstra [is not worried](<https://lwn.net/ml/linux-kernel/20191122105141.GY4114@hirez.programming.kicks-ass.net/>), though; he feels that the kernel issues have mostly been fixed and that problems in user-space code will be rare because other architectures have never allowed split locks. For those who are worried, though, he posted [a follow-up patch](<https://lwn.net/ml/linux-kernel/20191122152715.GA1909@hirez.programming.kicks-ass.net/>) allowing split-lock detection to be controlled at boot time and adding a "warn-only" mode that doesn't actually kill any processes. 

In that patch set, he noted that ""it requires we get the kernel and firmware clean"", and said that fixing up the kernel should be ""pretty simple"". But it turns out to be perhaps not quite that simple after all. In particular, there is the problem [pointed out](<https://lwn.net/ml/linux-kernel/3481175cbe14457a947f934343946d52@AcuMS.aculab.com/>) by David Laight: the kernel's [atomic bitmask functions](<https://www.kernel.org/doc/html/latest/core-api/atomic_ops.html#atomic-bitmask>) can easily create split-lock operations. The core problem here is that the type of a bitmask is defined as `unsigned long`, but it's natural for developers to use a simple `int` instead. In such cases, the creation of misaligned accesses is easy, and those accesses may occasionally span a cache-line boundary and lead to split locks. 

Opinions on how to solve this problem globally vary. Yu posted [a complex series of cases](<https://lwn.net/ml/linux-kernel/20191121185303.GB199273@romley-ivt3.sc.intel.com/>) meant to make atomic bit operations work for almost all usage patterns, but that strikes some as too much complexity; Zijlstra [said](<https://lwn.net/ml/linux-kernel/20191121201951.GY4097@hirez.programming.kicks-ass.net/>) simply that this solution is ""never going to happen"". An alternative, [suggested](<https://lwn.net/ml/linux-kernel/CALCETrW+qxrE633qetS4c1Rn2AX_hk5OgneZRtoZPFN1J395Ng@mail.gmail.com/>) by Andy Lutomirski, is to change the atomic bit operations to work on 32-bit values. That would, obviously, limit the number of bits that could be manipulated to 32\. Zijlstra [noted](<https://lwn.net/ml/linux-kernel/20191121195634.GV4097@hirez.programming.kicks-ass.net/>) that some architectures (alpha, ia64) already implement atomic bit operations that way, so it may well be that 32 bits is all that the kernel needs. 

There is one other "wrinkle", [according to Sean Christopherson](<https://lwn.net/ml/linux-kernel/20190925180931.GG31852@linux.intel.com/>), getting in the way of merging split-lock detection: the processor bit that controls split-lock detection affects the entire core on which it's set, not just the specific CPU that sets it. So toggling split-lock detection will affect all hyperthreaded siblings together. This particular wrinkle was only discovered in September, after the split-lock patch set had already been through nine revisions, leaving the development community less than fully impressed. But now that this behavior is known, it must be dealt with in the kernel. 

If split-lock detection is enabled globally, there is no problem. But if there is a desire to enable and disable it at specific times, things may well not work as expected. Things get particularly difficult when virtualization is involved; guest systems may differ with each other — or with the host — about whether split-lock detection should be enabled. Potential solutions to this problem include disallowing control of split-lock detection in guests (the current solution) or only allowing it when hyperthreading is disabled. Nobody has yet suggested using [core scheduling](<https://lwn.net/Articles/799454/>) to ensure that all processes running on a given core are in agreement about split-lock detection, but it's only a matter of time. 

All of this adds up to a useful feature that is apparently not yet ready for prime time. The question at this point is whether it should be merged soon and improved in place, or whether it needs more work out of tree. Perhaps both things could happen, since 5.6 is the earliest time it could be pulled into the mainline at this point. Split-lock detection exists to eliminate unwanted delays, but it still seems subject to some delays itself.
