---
title: "LSFMM: Page forking"
url: https://lwn.net/Articles/548091/
date: "April 23, 2013"
category: Stable pages
author: "By Jonathan Corbet April 23, 2013 LSFMM Summit 2013"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 23, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

Daniel Phillips addressed the full LSFMM group to push a concept that he calls "page forking." It is a response to the [problems](<https://lwn.net/Articles/442355/>) that led to the addition of "stable pages" to the kernel and the [performance regressions](<https://lwn.net/Articles/528031/>) that followed. In short, page forking is a sort of copy-on-write mechanism for pages that are currently being written back to persistent store; it is used in the in-development Tux3 filesystem. Should some process attempt to modify such a page while I/O is active, a copy of the page will be made and the modification will happen in the copy; the old copy remains until the writeback I/O is complete. In this way, no surprising data will be written to disk before its time. There are, he said, "a few interesting details" to work out, but it's not too hard. 

The result, he said, is that the performance regressions associated with stable pages are avoided. Instead, the system is accelerated significantly. Meanwhile, the overhead of the technique when it's not in use is nearly zero. 

Page forking proved to be a bit of a hard sell in this crowd. Ted Ts'o expressed skepticism about the performance claims, for example. Boaz Harrosh pointed out that page forking would result in two pages in the system with the same mapping and offset, breaking assumptions found in the code now. Daniel responded that this is an issue that would have to be examined in each filesystem that switched over to the forking technique; there is, he said, a "burden of analysis" that would need to be done. 

There were also concerns about how well the technique would work with pages that have been mapped into user space. Whenever a page is written back, the kernel would have to find every page table entry referring to it and turn that entry read-only so that the copy-on-write semantics could be enforced. Memory-mapped pages are heavily used in Linux, so there could be a real performance cost there. Daniel agreed that this issue was a concern, but did not elaborate further. 

Then came a rather detailed discussion about how to handle pages with elevated reference counts — pages that have been faulted in by the kernel with `get_user_pages()`, for example. A page with references may be undergoing modification, there is no way to know. Daniel said that, in such situations, the Tux3 code simply waits for the outstanding operations to complete. But, it was pointed out, some operations can take a very long time; waiting in this manner could cause unexpected stalls. Direct I/O is another operation that can raise interesting questions. It is possible for an application to start a direct write from one region of a file to another, possibly overlapping part of the same file; how page forking would work in such a situation is not entirely clear. 

Dave Chinner asked about how the writeback path works, especially in regard to interactions with the `fsync()` system call. Calls to `fsync()` use tags in the radix tree to track the pages that are to be written back; an `fsync()` call will not return to user space until all of those pages have been successfully written. Swapping a page out of the radix tree would clear those tags, creating all kinds of confusion. Fixing that would require significant changes to the writeback code, but, Dave asserted, writeback will not be rewritten just to add page forking. Ted added that the current writeback code may suck, "but there are reasons for it sucking that way" that cannot be ignored. 

Daniel stated that there will certainly be some costs to implementing page forking, but there are benefits too. Once the work is done, the affected filesystems will be faster and cleaner. Should we, he asked, stick with stable pages knowing that they can hurt performance, or should we do something similar that is, instead, a performance improvement? Ted responded that much of this talk seems like handwaving in the absence of a patch with updated writeback code. 

There were also concerns about pages that are written to a second time while the first I/O is outstanding. Would such a page be forked a second time, with two copies under I/O? That could lead to serious problems, given that I/O operations can be reordered in the block I/O subsystem. If a newer page is overwritten by an older copy, users are unlikely to be impressed. Daniel, when pressed, allowed that he did not have a solution to this particular problem. 

Ted asked what the next steps should be. Daniel responded that he hoped somebody in the room would pick up the idea and make it generic. Ted responded that he sensed serious doubts in the room about the sanity of the idea. He had no desire to pick it up himself, stating that he didn't consider himself to be smart (or foolhardy) enough to rewrite the writeback code. And, at that point, the discussion wound down.
