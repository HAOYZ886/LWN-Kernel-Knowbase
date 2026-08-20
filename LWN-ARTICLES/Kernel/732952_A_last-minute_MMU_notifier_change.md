---
title: "A last-minute MMU notifier change"
url: https://lwn.net/Articles/732952/
date: "September 5, 2017"
category: "Memory management-MMU notifiers"
author: "By Jonathan Corbet September 5, 2017"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
September 5, 2017

One does not normally expect to see significant changes to an important internal memory-management mechanism in the time between the ‑rc7 prepatch and the final release for a development cycle, but that is exactly what happened just before 4.13 was released. A regression involving the memory-management unit (MMU) notifier mechanism briefly threatened to delay this release, but a last-minute scramble kept 4.13 on schedule and also resulted in a cleanup of that mechanism. This seems like a good time to look at a mechanism that Linus Torvalds called ""a badly designed mistake"" and how it was made to be a bit less mistaken. 

#### MMU Notifiers

A computer's memory-management unit handles the mapping between virtual and physical addresses, tracks the presence of physical pages in memory, handles memory-access permissions, and more. Much of the work of the memory-management subsystem is concerned with keeping the MMU properly configured in response to workload changes on the system. The details of MMU management are nicely hidden, so that the rest of the kernel does not (most of the time) have to worry about it, and neither does user space. 

Things have changed over the last ten years or so in ways that have rendered the concept of "the MMU" rather more fuzzy. The initial driver of this change was virtualization; a mechanism like KVM must ensure that the host and the guest's view of the MMU are consistent. That typically involves managing a set of shadow page tables within the guest. More recently, other devices have appeared on the memory bus with their own views of memory; graphics processing units (GPUs) have led this trend with technologies like [GPGPU](<https://en.wikipedia.org/wiki/General-purpose_computing_on_graphics_processing_units>), but others exist as well. To function properly, these non-CPU MMUs must be updated when the memory-management subsystem makes changes, but the memory-management code is not able (and should not be able) to make changes directly within the subsystems that maintain those other MMUs. 

To address this problem, Andrea Arcangeli [added the MMU notifier mechanism](<https://git.kernel.org/linus/cddb8a5c14aa89810b40495d94d3d2a0faee6619>) during the 2.6.27 merge window in 2008. This mechanism allows any subsystem to hook into memory-management operations and receive a callback when changes are made to a process's page tables. One could envision a wide range of callbacks for swapping, protection changes, etc., but the actual approach was simpler. The main purpose of an MMU notifier callback is to tell the interested subsystem that _something_ has changed with one or more pages; that subsystem should respond by simply invalidating its own mapping for those pages. The next time a fault occurs on one of the affected pages, the mapping will be re-established, reflecting the new state of affairs. 

There are a few ways of signaling the need for invalidation, though, starting with the `invalidate_page()` callback: 

```
void (*invalidate_page)(struct mmu_notifier *mn, struct mm_struct *mm,
    			    unsigned long address);
```

This callback can be invoked after the page-table entry for the page at `address` in the address space indicated by `mm` has been removed, but while the page itself still exists. That is not the only notification mechanism, though; larger operations can be signaled with: 

```
void (*invalidate_range_start)(struct mmu_notifier *mn, struct mm_struct *mm,
    				   unsigned long start, unsigned long end);
        void (*invalidate_range_end)(struct mmu_notifier *mn, struct mm_struct *mm,
    				 unsigned long start, unsigned long end);
```

In this case, `invalidate_range_start()` is called while all pages in the affected range are still mapped; no more mappings for pages in the region should be added in the secondary MMU after the call. When the unmapping is complete and the pages have been freed, `invalidate_range_end()` is called to allow any necessary cleanup to be done. 

Finally, there is also: 

```
void (*invalidate_range)(struct mmu_notifier *mn, struct mm_struct *mm,
    			     unsigned long start, unsigned long end);
```

This callback is invoked when a range of pages is actually being unmapped. It can be called between calls to `invalidate_range_start()` and `invalidate_range_end()`, but it can also be called independently of them in some situations. One might wonder why both `invalidate_page()` and `invalidate_range()` exist and, indeed, that is where the trouble started. 

#### The end of invalidate_page()

In late August, Adam Borowski [reported](<https://lwn.net/Articles/732954/>) that he was getting warnings from the 4.13-rc kernel when using KVM, followed by the quick demise of the host system. Others had been experiencing similar strangeness, including a related crash that seemed to be tied to the out-of-memory handler. After testing and bisection, [this commit](<https://git.kernel.org/linus/aac2fea94f7a>), fixing another bug, was identified as the culprit. 

The problem came down to a difference between the `invalidate_page()` and `invalidate_range()` callbacks: the former is allowed to sleep, while the latter cannot. The offending commit was trying to fix a problem where `invalidate_page()` was called with a spinlock held — a context where sleeping is not allowed — by calling `invalidate_range()` instead. But, as Arcangeli [pointed out](<https://lwn.net/Articles/732956/>), that will not lead to joy, since not all users implement `invalidate_range()`; it is necessary to call `invalidate_range_start()` and `invalidate_range_end()` instead. 

The real fix turned out to not be quite so simple, though. Among other things, the fact that `invalidate_page()` can sleep makes it fundamentally racy. It cannot be called while the page-table spinlock affecting the page to be invalidated is held, meaning that the page-table entry can change before or during the call. This sort of issue is why Torvalds [complained about the MMU notifiers in general](<https://lwn.net/Articles/732957/>) and stated that they simply should not be able to sleep at all. But, as Jérôme Glisse [pointed out](<https://lwn.net/Articles/732958/>), some use cases absolutely require the ability to sleep: 

There is no way around sleeping if we ever want to support thing like GPU. To invalidate page table on GPU you need to schedule commands to do so on GPU command queue and wait for the GPU to signal that it has invalidated its page table/tlb and caches. 

We had this discussion before. Either we want to support all the new fancy GPGPU, AI and all the API they rely on or we should tell them sorry guys not on linux. 

Torvalds later [backed down](<https://lwn.net/Articles/732959/>) a little, making a distinction between two cases. Anything dealing with virtual addresses and the `mm_struct` structure can sleep, while anything dealing with specific pages and page-table entries cannot. Thus, the `invalidate_range_start()` and `invalidate_range_end()` callbacks, which deal with ranges of addresses and are called without any spinlocks held, can sleep. But `invalidate_range()` and `invalidate_page()` cannot. 

That, in turn, suggests that `invalidate_page()` is fundamentally wrong by design. After some discussion, Torvalds [concluded](<https://lwn.net/Articles/732960/>) that the best thing to do would be to remove it entirely. But, as the bug that started the discussion showed, replacing it with `invalidate_range()` calls is not a complete solution to the problem. To make things work again in all settings, including those that need to be able to sleep, the `invalidate_range()` calls must always be surrounded by calls to `invalidate_range_start()` and `invalidate_range_end()`. 

Glisse quickly [implemented that idea](<https://lwn.net/Articles/732610/>) and, after a round of review, his patch set was fast-tracked into the 4.13 kernel three days before its release. So, as a last-minute surprise, the `invalidate_page()` MMU notifier is gone; out-of tree modules that used it will not work with 4.13 until they are updated. It is rare to see a change of this nature merged so late in the development cycle, but the alternative was to release with real regressions and the confidence in the fix was high. With luck, this fix will prevent similar problems from occurring in the future. 

There is still one problem related to MMU notifiers in the 4.13 kernel, though: it turns out that the [out-of-memory reaper](<https://lwn.net/Articles/668126/#reaper>), which tries to recover memory more quickly from processes that have been killed in an out-of-memory situation, does not invoke the notifiers. That, in turn, can lead to corruption on systems where notifiers are in use and memory runs out. Michal Hocko has responded with [a patch](<https://lwn.net/Articles/732963/>) to disable the reaper on processes that have MMU notifiers registered. He took that approach because the notifier implementations are out of the memory-management subsystem's control, and he worried about what could happen in an out-of-memory situation, where the system is already in a difficult state. This patch has not been merged as of this writing, but something like it will likely get in soon and find its way into the stable trees. 

Notifier callbacks have a bit of a bad name in the kernel community. Kernel developers like to know exactly what will happen in response to a given action, and notifiers tend to obscure that information. As can be seen in the original bug and the reaper case, notifiers may also not be called consistently throughout a subsystem. But they can be hard to do without, especially as the complexity of the system grows. Sometimes the best that can be done is to be sure that the semantics of the notifiers are clear from the outset, and to be willing to make fundamental changes when the need becomes clear — even if that happens right before a release.
