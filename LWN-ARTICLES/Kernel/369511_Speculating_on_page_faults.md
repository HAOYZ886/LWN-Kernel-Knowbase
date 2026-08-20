---
title: Speculating on page faults
url: https://lwn.net/Articles/369511/
date: "January 12, 2010"
category: "Memory management-Scalability; Semaphores"
author: "By Jonathan Corbet January 12, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
January 12, 2010

Improving the performance of the kernel is generally a good thing to do; that is why many of our best developers have put considerable amounts of time into optimization work. One area which has recently seen some attention is in the handling of soft page faults. As the course of this work shows, though, performance problems are not always where one thinks they might be; sometimes it's necessary to take a step back and reevaluate the situation, possibly dumping a lot of code in the process. 

Page faults can be quite expensive, especially those which must be resolved by reading data from disk. On a typical system, though, there are a lot of page faults which do not require I/O. A page fault happens because a specific process does not have a valid page table entry for the needed page, but that page might already be in the page cache, in which case handling the fault is just a matter of fixing the page table entry and increasing the page's reference count; this happens often with shared pages or those brought in via the readahead mechanism. Faults for new anonymous pages (application data and stack space, mostly), instead, can be handled through the allocation of a zero-filled page. In either case, the fault is quickly taken care of with no recourse to backing store required. 

In many workloads, this kind of "soft" fault happens much more often than hard faults requiring actual I/O. So it's important that they be executed quickly. Various developers had concluded that the kernel was, in fact, not handling this kind of fault quickly enough, and they identified the use of the `mmap_sem` reader/writer semaphore as the core of the problem. Contention wasn't the issue in this case - only a reader lock is required for page fault handling - but the cache line bouncing caused by continual acquisition of the lock was killing performance. As the number of cores in systems increases, this kind of problem can only get worse. 

In response, Hiroyuki Kamezawa posted the first [speculative page fault patch](<http://lwn.net/Articles/362104/>) back in November. The core idea behind the patch was to try to handle page faults without taking `mmap_sem` at all. Doing so invites race conditions; in particular, the `vm_area_struct` (VMA) structure which controls the memory mapping can change while the work is in progress. So the speculative fault code must handle the fault, then check for concurrent changes and, if necessary, redo the work the older, slower way. That's the "speculative" part: doing the work in a lockless mode in the hope that the world will not change in the meantime and force that work to be done again. 

The speculative page fault code must also ensure that no changes which could create real trouble (such as freeing the VMA outright) can happen while the fault is being handled. To this end, various versions of the patch have tried techniques like adding reference counts to the VMA structure or using read-copy-update with the [red-black tree code](<http://lwn.net/Articles/184495/>) (which is used to locate the VMA covering a specific address within an address space) to defer changes while the speculative code is doing its thing. 

This work yielded some real results: the number of page faults per unit time that the system could handle, when running a fault-heavy benchmark, approximately doubled. The idea clearly had merit, but Peter Zijlstra [felt](<https://lwn.net/Articles/369536/>) that Kamezawa-san's patches ""weren't quite crazy enough"". He set out to rectify this problem with [a speculative page fault patch of his own](<http://lwn.net/Articles/368533/>), which saw a number of revisions. Peter's approach included the addition of speculative page table locks and the use of RCU to manage VMA structures. The result was a patch which would "sometimes boot" and which seemed to improve performance. 

This is about when Linus got involved, pointing out [some](<https://lwn.net/Articles/369537/>) [problems](<https://lwn.net/Articles/369538/>) with Peter's patch, concluding: 

I would say that this whole series is _very_ far from being mergeable. Peter seems to have been thinking about the details, while missing all the subtle big picture effects that seem to actually change semantics. 

Peter agreed with this conclusion, noting that he'd never thought it was ready yet. 

It was in further discussion that Linus, looking at a profile of page fault handling activity, [noticed something funny](<https://lwn.net/Articles/369539/>): the real overhead seemed to be in spinlock operations, which shouldn't be involved in the x86-optimized rwsem implementation at all. It turns out that said optimization was only applied to 32-bit systems; on 64-bit builds, reader/writer semaphores were using the generic, semaphore-based implementation. That meant that they were rather slower than they needed to be. 

So Linus [tossed out](<https://lwn.net/Articles/369540/>) a new rwsem implementation based on the x86 exchange-and-add (`xadd`) instruction with a typical warning: 

In other words: UNTESTED! It may molest your pets and drink all your beer. You have been warned. 

Kamezawa-san bravely tried the code anyway, and got [an interesting result](<https://lwn.net/Articles/369542/>). His pets and his beer both came through without trauma - and the page fault performance was better than with his speculative fault patch. Peter, too, [ran some tests](<https://lwn.net/Articles/369543/>) against his own speculative code; those results showed that the rwsem change tripled page fault performance. His speculative fault patch improved performance by just a tiny bit more than that, and the two together a little more yet. But the rwsem patch is a small and clear fix, while the speculative page fault patch is large, widespread, scary, and with known problems. So nobody really disputed Peter's conclusion: 

So while I think its quite feasible to do these speculative faults, it appears we're not quite ready for them. 

As of this writing, nobody has posted a final version of the rwsem patch. Linus has noted that there are things which can be improved with it, but it would be fairly typical for him to leave that work to others. But, one assumes, some version of this patch will be waiting in the wings when the 2.6.34 merge window opens. It will be a clear demonstration that low-hanging performance fruit exists even in our highly-optimized kernel; one need only think to look in the right place.
