---
title: Hotplugging and poisoning
url: https://lwn.net/Articles/753261/
date: "May 3, 2018"
category: "Fault tolerance; HWPOISON; Hotplug-Memory"
author: "By Jonathan Corbet May 3, 2018 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 3, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

Memory hotplugging is one of the least-loved areas of the memory-management subsystem; there are many use cases for it, but nobody has taken ownership of it. A similar situation exists for hardware page poisoning, a somewhat neglected mechanism for dealing with memory errors. At the 2018 Linux Storage, Filesystem, and Memory-Management summit, Michal Hocko and Mike Kravetz dedicated a pair of brief memory-management track sessions to problems that have been encountered in these subsystems, one of which seems more likely to get the attention it needs than the other. 

#### Memory hotplugging

When memory is added to the system, the kernel must allocate a new array of `page` structures to keep track of that memory. That array is currently allocated with `kmalloc()`, Hocko said, which is not the best thing to do. Among other things, if the kernel is running on a NUMA system, the new memory and its `page` structures are likely to end up on different nodes, which will not be good for performance. This is something that is happening now in real workloads. 

One common use case is virtualization environments, where administrators are using hotplugging to move memory between virtual machines. The memory-management developers recommend against doing that — removing memory from machines is tricky, since there can never be a guarantee that everything can be moved out of that space — but people do it anyway. Sometimes they add quite a bit of memory, consuming a lot of local memory just for the `page` structures. If the receiving virtual machine is already under memory stress, finding a contiguous range of memory for those structures could be difficult. 

The better solution, Hocko said, would be to just allocate the new `page` structures from the memory that has just been added. That memory is free, unfragmented, and obviously local. There were once concerns about "self-hosted" `page` structures when nonvolatile memory is involved, since those structures are written to frequently, but those concerns have faded over time. Hocko asked whether there were any concerns about implementing this approach. 

Jérôme Glisse said that there would need to be an opt-out mechanism. If the new memory is based on a GPU, for example, the CPU cannot access it and thus cannot maintain `page` structures there. The solution seems to be to just avoid self-hosting `page` structures on device memory. Vlastimil Babka asked what would happen if only a portion of the new memory was later unplugged — and it was the portion containing the `page` structures; Hocko said he needs to work on that problem still. Otherwise, though, there were no complaints beyond the fact that this mechanism "takes some beer to understand". 

Hocko's other question had to do with the size of the "sections" used to manage hotplug memory. A section contains 128MB by default on systems with a 4KB page size; it is the smallest unit of memory that can be plugged in or out. But, it seems, the "virtualization people" would like to do hotplugging with smaller units of memory. 

That could be supported, he said, but it would waste some memory and be relatively tricky to implement, so he isn't sure that it is worth the effort. Dave Hansen said that there should be no problem with telling people that hotplugging smaller pieces of memory will be wasteful. The approach that seemed to win favor is to behave as if an entire section of memory had been plugged in, but mark the missing pages as being reserved and unavailable. 

#### Huge-page poisoning

[Hardware poisoning](<https://lwn.net/Articles/348886/>) is a mechanism designed to keep a system in a functional state even if some of its memory goes bad. It responds to memory errors by locating and isolating the faulty page — essentially unplugging it from the system, though the hotplug mechanism is not used. Mike Kravetz has discovered that page poisoning doesn't work as well as one might like with huge pages, though. 

The kernel will respond to an error in a huge page in the usual way: it will try to substitute a working page and take the malfunctioning one offline. This works fine for PMD-sized pages, he said. PMD stands for the increasingly misnamed "page middle directory", the second-to-last layer in the system's page-table hierarchy. PMD-sized pages are the smallest huge pages, 2MB on x86 systems. If the system is using PUD-size pages (PUD being "page upper directory", since there are only two layers above it on modern systems — 1GB on x86), though, poisoning no longer works. The page-table walker simply doesn't take poisoning into account above the PMD level. So he decided to disable poisoning for huge pages above the PMD size. 

Hocko answered that the whole hardware poisoning mechanism seems to be "test driven" without a whole lot of high-level design. He has seen some "nasty changes" to keep the tests happy, such as huge pages being marked migratable so that offlining can work. Technically migrating those pages can be done, but it doesn't actually work. Allocating new storage for a huge page in the face of an error tends to be hard. 

Overall, Hocko didn't seem to think much of the feature, but Hansen said that hardware poisoning is only going to grow in importance; as memory sizes increase, hardware problems will happen more frequently. He sees about two errors per month on a 2TB machine he works with. Anshuman Khandoul said that migration is the only way to handle hardware errors in huge pages, but Kravetz wondered how the system could realistically migrate a 16GB gigantic page. Hocko wondered whether hardware poisoning was useful at all; Hansen replied that it had indeed been added as a "checkbox feature", but that it was hard to tell for sure because customers never call to say that their system successfully recovered from an error. 

Hocko remained unimpressed, calling poisoning a "toy" that doesn't work and is easy to break. He would like to see somebody explain the design of the whole thing; that might at least help keep developers from introducing bugs like the one that motivated this session. Either that, he said, or bite the bullet and admit that it was a toy feature all along. Hansen said that it is reasonable to ask how important the feature is, but that the arrival of nonvolatile memory may change the calculation, since that memory is likely to generate more errors. 

As time ran out, Kravetz said that trying to migrate pages might not be worth it; perhaps the system should just note errors and mark the pages bad. Glisse added that it would then be up to the application to cope with memory errors. Kravetz concluded that he is in favor of somebody trying to understand the design, but that he wasn't seeing any hands raised in the room; Hocko said that the recovery mechanism is in danger of being ripped out of the kernel unless a maintainer shows up.
