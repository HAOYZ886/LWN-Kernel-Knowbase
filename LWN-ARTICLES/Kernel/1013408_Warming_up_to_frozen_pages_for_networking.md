---
title: Warming up to frozen pages for networking
url: https://lwn.net/Articles/1013408/
date: "March 13, 2025"
category: "Memory management-struct page; Networking"
author: "By Jonathan Corbet March 13, 2025"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 13, 2025

When the 6.14 kernel is released later this month, it will include the usual set of internal changes that users should never notice, with the possible exception of changes that bring performance improvements. One of those changes is [frozen pages](<https://lwn.net/Articles/1000654/>), a memory-management optimization that should fly mostly under the radar. When Hannes Reinecke [reported a crash](<https://lwn.net/ml/all/08c29e4b-2f71-4b6d-8046-27e407214d8c@suse.com>) in 6.14, though, frozen pages suddenly came into view. There is a workaround for this problem, but it seems there is a fair amount of work to be done that nobody had counted on to solve the problem properly. 

The kernel uses reference counts to keep track of which pages of memory are in use. For example, a page in shared memory that is mapped into the address space of several processes will track a reference from each of those processes. As long as at least one of those processes exists and keeps the page mapped, that page will not be freed. The management of reference counts is not free, though; their manipulation requires expensive atomic operations, and the counts themselves take up memory. That has led to a desire to do without reference counting in places where it is not strictly necessary. The slab allocator, for example, tracks the objects it manages within each page and does not need a separate reference count for the page as a whole. In kernels prior to 6.14, though, slab pages are duly reference-counted anyway. 

Frozen pages were introduced as a way to eliminate this overhead when possible; in a frozen page, the reference count is set to zero and stays there. Since the lifecycle of the page is tracked separately, there is no need to increment or decrement its count, so that overhead is avoided. Eventually, it will become possible to eliminate the reference count for frozen pages entirely (rather than just keeping it at zero), but there is work yet to be done to reach that point. 

Reinecke encountered a kernel crash deep within the networking subsystem; after carefully bisecting the problem, he [identified](<https://lwn.net/ml/all/509dd4d3-85e9-40b2-a967-8c937909a1bf@suse.com>) the [commit switching the slab allocator to frozen pages](<https://git.kernel.org/linus/9aec2fb0fd5e>) as the culprit. Some extensive debugging and discussion ensued, and it eventually became clear that the networking code was trying to increase the reference count on a frozen page, leading to all kinds of internal confusion and an eventual crash. 

Sending data through the network can be a complex operation involving pages scattered throughout physical memory. The networking subsystem, like others, handles this complexity by creating a vector describing the various chunks of data to be transferred. All of the pages contained within that vector need to remain present and valid while the operation is underway, so each page's reference count is incremented at the beginning, and decremented once the operation is complete. Many I/O paths within the kernel have traditionally followed that same pattern. 

With the shift toward folios and the desire to avoid unneeded reference-count operations, though, that pattern has shifted. The I/O paths need to avoid reference-count manipulations whenever possible, and certainly when those manipulations cannot be done at all, so those paths have changed to adapt. At least, they have in some parts of the kernel; Matthew Wilcox [expressed some surprise](<https://lwn.net/ml/all/Z8cm5bVJsbskj4kC@casper.infradead.org>) on learning that the job was only partially done: 

> I thought we'd done all the work needed to get rid of these pointless refcount bumps. Turns out that's only on the block side (eg commit e4cc64657bec). So what does networking need in order to understand that some iovecs do not need to mess with the refcount? 

Reinecke [answered](<https://lwn.net/ml/all/a4bbf5a7-c931-4e22-bb47-3783e4adcd23@suse.com>) that this kind of change was not going to be easy; the code is complex, and the place where a reference is taken may be far away from — and, indeed, in a completely different layer from — where that reference must be released. Wilcox, meanwhile, posted [a patch](<https://lwn.net/ml/all/Z8dA8l1NR-xmFWyq@casper.infradead.org>) adding some checks within the memory-management code that prevents attempts to manipulate reference counts on slab pages, which are the only frozen pages in the 6.14 kernel. That change, described as ""a quick hack"", was intended as a way to avoid having to revert the use of frozen pages entirely. 

Even then, it took [one more change](<https://lwn.net/ml/all/d6e65c4c-a575-4389-a801-2ba40e1d25e1@suse.cz>) from Vlastimil Babka, touching the networking code directly, to make the problem go away. Reinecke [acknowledged](<https://lwn.net/ml/all/7439cb2f-6a97-494b-aa10-e9bebb218b58@suse.de>) the fix, but complained about the need to keep track of whether specific pages needed their reference counts updated or not: 

> Previously we could just do a bvec iteration, get a reference for each page, and start processing. Now suddenly the caller has to check if it's a slab page and don't get a reference for that. Not only that, he also has to remember to _not_ drop the reference when he's done. And, of course, tracing get_page() and the corresponding put_page() calls through all the layers. Really? 

His complaint garnered little sympathy, though. Instead, Wilcox [asserted](<https://lwn.net/ml/all/Z8iTzPRieLB7Ee-9@casper.infradead.org>) that the networking layer needs to move away from using reference counts on pages, both to allow the memory-management hack to be removed and to improve networking performance. He added: ""What worries me is that nobody in networking has replied to this thread yet. Do they not care?"" In an attempt to provoke such a response, he changed the subject line to: ""Networking people smell funny and make poor life choices"". 

Even that, though, has failed to motivate any sort of significant response from the networking subsystem. The only reply was from Cong Wang, who [suggested](<https://lwn.net/ml/all/Z8jwe7dFXVEI9fT1@pop-os.localdomain/>) that ""using AI copilot or whatever automation tool"" might help — a suggestion that does not appear to have gained any traction. Wilcox has [posted his workaround](<https://lwn.net/ml/all/20250310143544.1216127-1-willy@infradead.org>) as a separate patch that, one would expect, will find its way into 6.14 prior to its release. 

As of this writing, that is where the situation stands. The immediate problem should be fixed, but the wider question of the management of reference counts for pages across the kernel remains unanswered. Perhaps the upcoming [Linux Storage, Filesystem, Memory-Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) will include a discussion on this issue; stay tuned.
