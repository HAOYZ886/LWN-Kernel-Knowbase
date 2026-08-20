---
title: Prerequisites for large anonymous folios
url: https://lwn.net/Articles/943758/
date: "September 8, 2023"
category: "Memory management-Folios"
author: "By Jonathan Corbet September 8, 2023"
---

By **Jonathan Corbet**  
September 8, 2023

The work to add support for [large anonymous folios](<https://lwn.net/Articles/937239/>) to the kernel has been underway for some time, but this feature has not yet landed in the mainline. The author of this work, Ryan Roberts, has been trying to get a handle on what the remaining obstacles are so he can address them. On September 6, an online meeting of memory-management developers discussed that topic and made some progress; there is still some work to do, though, before large anonymous folios can go upstream. 

[Folios](<https://lwn.net/Articles/849538/>) are, at their core, a simple mechanism to group physically contiguous pages into larger units that can be managed more efficiently. Over the last couple of years, many parts of the memory-management subsystem have been converted to use folios, but the management of anonymous pages (pages representing data that is not backed up by a file) has proved to be somewhat tricky. 

#### Large anonymous folios

Roberts started the discussion with a review of the work. Whenever it handles a page fault on an anonymous virtual-memory area (VMA) that requires allocating a page of new memory, the kernel will attempt to allocate a group of pages as a folio rather than a single page. This folio will normally be between 16KB and 64KB in size. It will be mapped into the process's address space at the page-table-entry (PTE) level — there is no alternative to doing that, since these folios are smaller than the huge-page size. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The addition of large anonymous folios is driven by performance concerns, he said. Allocating and mapping an entire folio will reduce page faults, since the process will not need to fault in the other pages in that folio. A lot of per-page operations, such as the management of reference counts, become per-folio operations instead, so there are correspondingly fewer of them. Using folios reduces the length of the memory-management subsystem's least-recently-used (LRU) lists, making the scanning of that list more efficient. On some architectures (specifically 64-bit Arm), zeroing a 64KB folio is faster than zeroing individual pages. 

Turning to specific use cases, Roberts said that this feature provides better performance on Android systems running with 4KB base pages. It allows the kernel to use the CPU's "contiguous-PTE" bits, which indicate that a contiguous set of PTEs maps to contiguous pages of memory; that, in turn, allows the hardware to optimize the translation lookaside buffer (TLB) entry for those pages. On a 64-bit Arm system with 4KB base pages, using the contiguous-PTE requires 16-page (64KB) contiguous mappings, which large anonymous folios can provide. Support for the contiguous-PTE feature is implemented in a separate patch set. 

There is a separate use case on Arm systems running with 64KB base pages. On such systems, the PMD size (the size of the smallest huge pages) is 512MB; that much physically contiguous memory is difficult to allocate, so huge pages tend not to be used when running with 64KB pages. The contiguous-PTE size, though, is 2MB, which is much easier to manage. Large anonymous folios can be used to provide a sort of "2MB transparent huge page" feature that improves performance. 

Using a larger base-page size has a similar effect to using large folios everywhere, so it is not surprising that Roberts was asked whether switching Android systems to 64KB base pages was out of the question. The answer was "yes", but that using 16KB pages might be an option; even then, though, there are problems with app compatibility and wasted memory due to internal fragmentation. That question led naturally to Roberts's next point, though, which was illustrated with benchmark results. 

[![\[Benchmark slide\]](https://static.lwn.net/images/2023/laf-benchmarks-sm.png)](<https://lwn.net/Articles/943760/>) Roberts did a number of tests with the industry-standard kernel-compilation benchmark, and with [Speedometer 2.0](<https://browserbench.org/Speedometer2.0/>) as well. In both cases, enabling large anonymous folios improved performance significantly, on a scale similar to what can be achieved using 16KB base pages. The increase in memory usage, though, was tiny compared to the situation with 16KB base pages (and 64KB base pages was far worse yet). Large anonymous folios, thus, look like a way to gain many of the performance advantages provided by a larger base-page size without most of the associated cost. 

The patch series is currently in [its fifth revision](<https://lwn.net/ml/linux-kernel/20230810142942.3169679-1-ryan.roberts@arm.com/>), posted on August 10\. It has had a lot of reviews, and a number of prerequisite changes have been [identified](<https://lwn.net/ml/linux-mm/f8d47176-03a8-99bf-a813-b5942830fd73@arm.com/>). There are a few open issues that he wanted to discuss: naming, runtime controls, and which statistics to measure and report. He did not manage to get through even that short list in the time allotted, though. 

Starting with naming, Roberts said that the feature should have a name that tells users what it does. "Large anonymous folio" might work, but he is not sure that it is the best option. Others he suggested were "transparent large page", "flexible transparent huge page", and "small transparent huge page". It quickly became clear, though, that Roberts would not succeed in getting the assembled developers to disagree over the name; they had other concerns instead. 

#### How to turn it off

David Hildenbrand said that the real question to answer is the extent to which large anonymous folios should be used automatically. The biggest problem with the existing transparent huge page feature is memory waste, and the same situation will arise with this feature. If large anonymous folios are added to the kernel without a way to disable them, he said, the result will certainly be problems for some applications. The feature is reasonable, he said, but it needs a way to avoid and/or recover from over-allocation of large folios. 

The problem of memory waste comes about when applications create sparse mappings — regions of memory where the actively used pages are scattered and surrounded by unused pages. If a large folio is allocated in a sparse mapping, most of it will likely consist of unused pages, which are wasted. Roberts questioned the use cases for such mappings, but it seems that they are reasonably common. Memory allocators often create them; highly threaded applications with a lot of thread stacks are another example. 

Roberts said that he saw a couple of approaches for dealing with this problem, preferably in a way that lets him get a minimal patch into the kernel to better characterize how large anonymous folios affect performance. It seems that there needs to be a way to turn the feature on or off at a minimum. Even better, though, would be some sort of dynamic mechanism that would check on page usage and break apart folios that are not fully utilized. He is not sure how to do that without scanning the pages in question, though. 

Hildenbrand said that, if Roberts wants large anonymous folios to be enabled by distributions, there has to be a way to toggle it on or off. The best way, of course, would be to have the kernel tune this behavior automatically and not add any new knobs at all. But some way must be found, he said, to deal with the memory-waste problem, whether it's reclaiming incorrectly allocated folios, giving user space a way to tell the kernel about sparse mappings, or some other sort of mechanism. 

Roberts concluded that the kernel will have to get better at discovering and reclaiming unused pages in folios. That is not necessarily a small job, though, and he would really like to start with a minimum patch that people can play with. Hildenbrand said that Roberts should not worry too much about introducing a new toggle to just disable the feature; if the kernel gets to a point where it can tune its own behavior effectively, it can just ignore the setting of the knob. 

As it turns out, Roberts already had a proposal for a control knob, called `laf_order`, that would be found in either debugfs or sysfs (there is some disagreement over which place is better). Setting this knob to zero disables large anonymous folios entirely, while setting it to "`auto`" (the proposed default) leaves the management of anonymous folios to the kernel. It can also be set to an integer indicating the allocation order to use for anonymous folios — setting `laf_order=4`, for example, would set the size to 16 pages, or 64KB. Yu Zhao argued that the knob needs to accept a list of orders rather than a single one, with the kernel falling back through the list if the larger orders cannot be allocated. 

Matthew Wilcox, instead, pointed out that different applications will want different behavior. A system-wide `laf_order` knob is good for expressing what the system administrator wants, but the administrator likely does not know what the software wants. Hildenbrand, though, said that there should not be too much effort put into optimizing this feature; it should be possible to disable it when it is not useful, and to play with it otherwise. He reiterated that automatic tuning would be best; user space should not be expected to know better than the kernel when it comes to memory management. 

John Hubbard said that there is a need for per-application tuning of the feature, or it will be disabled by users; David Rientjes said that it was looking like there would be a need for applications to tell the kernel which folio size to use for each VMA. Wilcox protested that, "when applications tell the kernel that they know what they are doing, they don't". Instead, the kernel should watch what an application does, ramp up the use of large anonymous folios conservatively, and track how those folios are used. 

Roberts said that he had thought about a "per-VMA slow start" feature where, if page faults within a VMA form in contiguous blocks, the folio size for that VMA would be increased. Wilcox agreed that this idea was worth looking into. Roberts said that the universal toggle still felt like the first step to take, though. 

#### Out of time

At that point, the hour was done; Roberts lamented that he had come with three topics to discuss and had never gotten past the first. He is still looking for a roadmap to get him to a bare-minimum patch that could get into the kernel. Other improvements can be made after that. There was not much agreement on what that minimum should be, though. Hildenbrand suggested a simple toggle to disable the feature; Zhao, instead, argued for a separate toggle attached to each memory control group — and that a per-VMA toggle would be better yet. Hubbard suggested a set of controls that mirrors the knobs that have been developed to control transparent huge pages. 

In a sense, the meeting came to an end without having resolved much of anything. But it did shine a light on some of the problems that will have to be solved before large anonymous folios can be a part of a mainstream kernel release. The experience with transparent huge pages shows that introducing a feature before it is truly ready can cause long-term difficulties of its own, with users disabling the feature long after its problems have been solved. With luck and some patience, large anonymous folios will eventually enter the kernel without such troubles.
