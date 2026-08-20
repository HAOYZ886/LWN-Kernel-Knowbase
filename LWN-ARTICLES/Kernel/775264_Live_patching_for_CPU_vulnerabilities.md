---
title: Live patching for CPU vulnerabilities
url: https://lwn.net/Articles/775264/
date: "December 20, 2018"
category: Live patching
author: "December 20, 2018 This article was contributed by Nicolai Stange"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

December 20, 2018

This article was contributed by Nicolai Stange

The kernel's live-patching (KLP) mechanism can apply a wide variety of fixes to a running kernel but, at a first glance, the sort of highly intrusive changes needed to address vulnerabilities like Meltdown or L1TF would not seem like likely candidates for live patches. The most notable obstacles are the required modifications of global semantics on a running system, as well as the need for live patching the kernel's entry code. However, we at the SUSE live patching team started working on proof-of-concept live patches for these vulnerabilities as a fun project and have been able to overcome these hurdles. The techniques we developed are generic and might become handy again when fixing future vulnerabilities. 

For completeness, it should be noted that these two demo live patches have been implemented for [kGraft](<https://en.wikipedia.org/wiki/KGraft>), but kGraft is conceptually equivalent to KLP. 

At the heart of the Meltdown vulnerability is the CPU speculating past the access rights encoded in the page table entries (PTEs) and thereby enabling malicious user-space programs to extract data from any kernel mapping. The [kernel page-table isolation (KPTI) mechanism](<https://lwn.net/Articles/741878/>) blocks such attacks by switching to stripped-down "shadow" page tables whenever the kernel returns to user space. These mirror the mappings from the lower, user-space half of the address space, but lack almost anything from the kernel region except for the bare minimum needed to reenter the kernel and switch back to the fully populated page tables. The difficulty, from a live-patching perspective, is to keep the retroactively introduced shadow page tables consistent with their fully populated counterparts at all times. Furthermore, the entry code has to be made to switch back and forth between the full and shadow page table at kernel entries and exits, but that is outside of the scope of what is live patchable with KLP. 

For the L1TF vulnerability, recall that each PTE has a `_PAGE_PRESENT` bit that, when clear, causes page faults upon accesses to the corresponding virtual memory region. The PTE bits designated for storing a page's frame number are architecturally ignored in this case. The Linux kernel swapping implementation exploits this by marking the PTEs corresponding to swapped-out pages as non-present and reusing the physical address part to store the page's swap slot number. Unfortunately, CPUs vulnerable to L1TF do not always ignore the contents of these "swap PTEs", but can instead speculatively misinterpret the swap slot identifiers as physical addresses. These swap slot identifiers, being index-like in nature, tend to alias with valid physical page-frame numbers, so this speculation allows for extraction of the corresponding memory contents. The [Linux kernel mitigation](<https://lwn.net/Articles/762570/>) is to avoid this aliasing by bit-wise inverting certain parts of the swap PTEs. Unfortunately, this change of representation is again something which is not safely applicable to a running system with KLP's consistency guarantees alone. 

#### Global consistency

When a live patch is applied, the system is migrated to the new implementation at task granularity by virtue of KLP's so-called [per-task consistency model](<https://lwn.net/Articles/634649/>). In particular, it is guaranteed that, for all functions changed by a live patch, no unpatched functions will ever be executing on the same call stack as any patched function. 

Clearly, it might take some time after live-patch application until each and every task in the system has been found in a safe state and been migrated to the new code. The crucial point is that, while a single task will never be executing simultaneously in both the original and the patched implementation, different tasks can and will do that during the transition. It follows that a live patch must not change global semantics, at least not without special care. 

The standard example for a forbidden change of global semantics would be the inversion of some locking order: as both orderings could be encountered concurrently during the transition period, an ABBA deadlock would become possible. Other (and more relevant) examples in this case include: 

  * The bit-wise inversion of swap PTEs for mitigating against L1TF: imagine what would happen if an unpatched kernel function that interprets PTE entries encountered an inverted PTE. 
  * Shadowing of page tables (i.e. KPTI): imagine some unpatched page-table modifying code not properly propagating its changes to some shadow copy. 

What these examples have in common is that it is possible to disambiguate between the original and the new semantics by inspection of the state in question. For the case of inverted swap PTEs, this becomes apparent when taking into account that the higher 18 bits of a swap PTE are always unused on x86_64; swap offsets handed out by the memory-management code don't ever exceed 32 bits. Thus, the higher bits all are all unset in non-inverted swap PTEs and set for the inverted ones. For the shadow page table example, a page table has either been shadowed already and thus, the new semantics apply to it or not. 

For this class of problems where a disambiguation is possible, the following scheme for the safe modification of global semantics suggests itself: 

  1. Live patch all state accessors to be able to recognize and handle both the original and modified semantics. 
  2. Wait for the live-patch transition to finish globally. 
  3. Start introducing the modified semantics only thereafter. For example, start inverting swap PTEs, creating page table shadow copies, and so on. 

Because any modification of the semantics will happen only after the patching has completed, it will be impossible to have an unpatched state accessor to encounter the modified semantics. 

Now, how does the live-patch module determine when the transition has finished? With [the callbacks mechanism merged in 4.15](<https://git.kernel.org/linus/93862e385ded>), this would be straightforward: simply register a `post_patch()` callback and wait for its invocation. For pre-4.15 kernels, the `post_patch()` callback functionality can be emulated by live patching the KLP core itself, namely its housekeeping code to be executed after a transition has finished. 

Finally, the last remaining problem is to deal with the reverse transition of "unpatching". Users may disable loaded live patches or, with the pending ["atomic replace" patch set](<https://lwn.net/ml/linux-kernel/20181129094431.7801-1-pmladek@suse.com/>), downgrade to a cumulative live patch not containing some fix in question. Obviously, any change to global semantics must be rolled back before any of the live-patched state accessors might become unpatched again. For patch disabling (as opposed to downgrades), this is easy; from the `pre_unpatch()` callback, which, as the name suggests, gets invoked right before such a transition is actually started: 

  1. Stop introducing new uses of the changed semantics: stop creating page table shadows, stop inverting swap PTEs, and so on. This can usually be achieved by flipping some boolean flag and running some sort of synchronization like `schedule_on_each_cpu()` afterward. 
  2. Undo all semantic changes that have been made up to this point; drop page table shadows or walk all page tables and uninvert any swap PTEs. 
  3. Allow the unpatch transition to start. 

The situation for a transition to another cumulative live patch is more complicated. The current atomic replace implementation won't invoke any callbacks from the previous live patch, and we would like to avoid the potentially costly rollback of semantic changes for the common case of live-patch upgrades. For example, imagine that the old and new live patches both contain the L1TF mitigation inverting the swap PTEs. In this case, the swap PTEs accessors would be kept patched one way or the other during the transition and thus, be able to handle the inverted swap PTEs at all times. Obviously, scanning through all page tables and unnecessarily uninverting the swap PTEs before starting the transition would be a waste of time and should be avoided. But as it currently stands, the previous live patch is unable to tell anything about the contents of the next one and some help from the KLP core would certainly be needed. 

We discussed this problem at the 2018 Linux Plumbers Conference [live patching microconference](<https://linuxplumbersconf.org/event/2/contributions/178/>); the solution will probably be to amend the `klp_patch` structure with some set of IDs representing the global states supported by the associated live patch. For a start, we would then simply make the KLP core disallow transitions to live patches that are not able to maintain all of the states from the currently active set. 

#### Live patching the entry code

A live patch implementing KPTI will have to replace the kernel's entry code. At each crossing of the boundary between user and kernel space, the current page table must be switched between the shadow copy and the fully populated original. The problem here is that KLP is based on Ftrace, so only functions calling `mcount()` from their prologue are eligible as live-patching targets. Obviously, the entry code does not belong to this category; it is not organized into functions to begin with. 

Fortunately, even though KLP won't be of any help when it comes to live patching the entry code, this patching is still doable; the basic idea is to simply redirect all of the CPU's references to the entry code to the respective replacements from the live-patch module. For x86, this would amount for replacing the CPU's interrupt descriptor table (IDT) as well as redirecting the pointers to the various system-call handlers. All exceptions, interrupts, and system calls can be made to enter the kernel through the entry-code replacement this way, but newly forked threads would still begin their execution at the hard coded `ret_from_fork` entry-code address. 

Depending on the target kernel's version, it is possible to cover these threads as well: 

  * For 4.9 and later kernels, the hard-coded `ret_from_fork` address can be changed by live patching `copy_thread_tls()`. 
  * For earlier kernel versions, the `ret_from_fork` address is hard coded into the `__schedule()` path, which used to not be live patchable. However, the first thing the code at `ret_from_fork` does is to issue a call into `schedule_tail()`, which can be live patched and made to redirect its on-stack return address to the entry-code replacement. 

As shown, it is not too difficult to replace the kernel's entry code from a live-patch module. However, it is common for live-patch modules to eventually be unloaded again, when upgrading to a newer version, for example. Given that tasks can sleep for arbitrarily long times in system calls or exceptions — with return addresses pointing into the about-to-be-unmapped entry code replacement on their stack — some precautions are needed in order to prevent these from returning into nowhere. A possible solution is to reference-count the entry code replacement: increment the counter upon entry before scheduling becomes possible and decrement it again on exit after the last such possibility. With this in place, the following steps are sufficient to allow for a safe unmapping of the entry code replacement: 

  1. Restore all CPUs' entry-code pointers from a `schedule_on_each_cpu()` call. As the increments are made before scheduling becomes possible, they order with `schedule_on_each_cpu()` and all pending executions of the entry-code replacement will have been recorded properly by the reference count afterward. 
  2. Wait for the reference counter to drain to zero. 
  3. Run an empty `schedule_on_each_cpu()` call. After completion, all tasks will have left the window between decrementing and actually returning to user space. 

As a final remark, let me note that getting reasonable test coverage for entry-code live patches is quite hard, mainly because the content of the IDT varies wildly between different configurations like Xen guests, bare metal, and so on. 

#### Conclusion

As shown, the scope of KLP can be extended up to a point where live patching to address CPU vulnerabilities becomes possible. The safe modification of global semantics might be handy again in the future. On the other hand, the live patching of entry code, while doable in principle, poses significant challenges to the testing infrastructure. For those cases where only some subset of the entry code needs to get patched, this can become manageable though; for example, we have been able to release production live patches fixing the ["Pop SS" vulnerability](<https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-8897>) by replacing only the `int3` trap handler. We will work on upstreaming proper support to KLP to make this kind of patching possible; interested readers can find the Meltdown patches in [this repository](<https://github.com/nicstange/meltdown-livepatch>), while the L1TF patches will be coming soon.
