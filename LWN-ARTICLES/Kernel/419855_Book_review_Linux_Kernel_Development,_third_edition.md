---
title: "Book review: Linux Kernel Development, third edition"
url: https://lwn.net/Articles/419855/
date: "December 15, 2010"
category: Books
author: "By Jonathan Corbet December 15, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
December 15, 2010

It has been well over five years since LWN [reviewed the second edition](<https://lwn.net/Articles/127018/>) of Robert Love's _Linux Kernel Development_. Needless to say, things have changed a little since the 2.6.10 release covered by that edition. As it happens, the third edition has been out for a few months now; your editor has finally had a chance to read through his copy and put together a review. In summary, the third edition is a much-needed update, and _Linux Kernel Development_ remains a valuable resource, but there are some disappointments to be found as well. 

One has to dig a little bit to figure out which kernel version is covered by the third edition; according to the preface, the target is 2.6.34. Robert, ever the optimist, suggests that it will be good for a long time: ""As the Linux kernel matures, there is a greater chance of a snapshot of the kernel remaining representative long into the future."" Time will tell. 

The third edition has been extensively updated, but it retains the same structure as its predecessors. The preface talks of "all-new" chapters, but the number of chapters remains the same. The scheduler discussion has been updated to reflect the merging of the completely fair scheduler. Other relatively recent kernel changes (mutexes, for example) have been added, ![\[book cover\]](https://static.lwn.net/images/ns/kernel/linux-kernel-development.jpg) and there are changes throughout to reflect what has happened over the last 24 kernel releases. There _is_ a new chapter on kernel data structures; it contains the linked list discussion previously found in Appendix A, along with coverage of FIFOs, red-black trees, and the idr subsystem. The low-level device model chapter has been combined with the chapter on loadable modules, for some reason, but the discussion is not much changed. The appendix on the random number generator is gone. 

All told, the coverage of the core kernel is well written and clear, in an approachable style. _Linux Kernel Development_ is, at this time, probably the best reference available for developers wanting to learn how the kernel works and how the major pieces fit together. Your editor is glad to have a copy on hand. 

With that understanding in place, this, too, must be said: the update to the third edition appears to have been done in a hurry. As a result, the book contains a number of errors and inconsistencies, and it fails to cover no end of interesting things which have happened in the kernel over the last five years while retaining text which was obsolete even in previous editions. Robert has not been hugely present in the kernel development community in the last few years (he got a job at a company with a reputation for removing developers from the community) and, unfortunately, it shows. 

For example, this book (which covers 2.6.34) includes a section on the anticipatory I/O scheduler - which was removed in 2.6.33. There is still talk of the "Linus" scheduler - which (as is noted in the book) was a 2.4 feature. The mutual exclusion chapter discusses semaphores - which have been deprecated for mutual exclusion purposes - with the section on mutexes seemingly bolted on afterward. (The book also says elsewhere that we cannot kill a process in an uninterruptible sleep because it ""may hold a semaphore"") There is a lengthy discussion of the old "bottom half" mechanism which is long gone; the removed-in-2.5 task queue mechanism also merits a section of its own. The unlamented `ksymoops` tool, we are told, ""probably came with your distribution"". 

Some things are simply wrong. We're told that `do_exit()` calls `del_timer_sync()`, but that last happened in 2.6.15. The workqueue discussion appears to be stuck in a 2.6.10 time warp; [changes](<https://lwn.net/Articles/211279/>) which were merged for 2.6.20 are not reflected here. Kernel stack size is said to be 16K on 64-bit architectures because they usually have 8K pages. The version of `struct file` shown on page 279 never existed; it includes `f_ep_lock` which was renamed (by your editor) to `f_lock` in 2.6.30. The "process address space" chapter says, several times, that all Linux systems have three-level page tables, despite the fact that the fourth level was added for 2.6.11. The device model chapter no longer mentions `struct subsystem`, but it still appears in the associated diagram. 

And many things which should be discussed in a contemporary book aren't. Developers working on the kernel now should probably be familiar with control groups, contemporary debugging tools (kmemleak, lockdep, the fault injection framework, ...), debugfs, ftrace, git, high-resolution timers, huge pages, linux-next, multiple slab allocators, namespaces, perf, power management and quality of service, read-copy-update, readahead, reverse mapping in the VM subsystem, scheduler domains, `splice()`, TASK_KILLABLE, threaded interrupt handlers, virtualization, and so on. No book on the kernel can hope to be complete and still be something that a person of ordinary strength can physically lift, but these topics are all important enough that, one would assume, at least some of them would merit coverage, but none are mentioned in the third edition. 

Keeping up with all that is happening in the kernel is hard - your editor hopes that readers will trust him to understand this. So it is not surprising that some mistakes would slip through when a book is fast-forwarded from 2.6.10 to 2.6.34. But the amount of old stuff that leaked through, combined with the things which should have been mentioned but weren't, seems a bit high; some of them should, at least, have been caught in technical review. As a result, the third edition of _Linux Kernel Development_ is not as good as it could have been. 

These grumbles notwithstanding, your editor will restate what was said above: this book remains the best overview of contemporary kernel development available today. Robert is a talented and engaging writer who is able to cover complex topics in a readily understandable manner. The third edition merits a place on the "L1 bookshelf" (the one which can be reached without getting out of the chair); developers who are working with the kernel will probably want a copy.
