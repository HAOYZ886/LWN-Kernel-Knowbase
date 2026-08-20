---
title: Heterogeneous memory management and MMU notifiers
url: https://lwn.net/Articles/752964/
date: "April 27, 2018"
category: "Memory management-Heterogeneous memory management; Memory management-MMU notifiers"
author: "By Jonathan Corbet April 27, 2018 LSFMM"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
April 27, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

Heterogeneous memory management (HMM) is a relatively new kernel subsystem that allows the system to manage peripherals (such as graphics processors) that have their own memory-management units. In two sessions during the memory-management track of the 2018 Linux Storage, Filesystem, and Memory-Management Summit, HMM creator Jérôme Glisse provided an update on the status of this subsystem and where it is going, along with a more detailed look at the memory-management unit (MMU) notifiers mechanism on which it depends. 

#### An HMM update

Glisse started by noting that [an RFC patch](<https://lwn.net/Articles/752966/>) adding HMM support to the Nouveau driver (for NVIDIA graphics processors) has been posted, with a second version coming soon. He is hoping to convert more GPU drivers to HMM in the near future; it is a better solution, he said, than using `get_user_pages()` to pin user-space pages into memory. Beyond the advantage of not pinning pages, HMM can help improve GPU performance by allowing the CPU to create a complex data structure using pointers that are valid in both CPU and GPU space. The CPU can then pass this structure to the GPU without having to recreate it using GPU-space pointers. 

There are also some vendors looking into using HMM to manage device-private memory; AMD is likely to use HMM with its next-generation hardware. 

Glisse had a question for the group: he noticed that core dumps do not take the `mmap_sem` lock before walking through (and dumping) user-space memory. That can lead to surprises when HMM is in use, since GPU threads could be accessing this memory while it is being dumped. Is this the expected behavior? Hugh Dickins said that he had run across that behavior recently, and was "horrified" to see it. Kirill Shutemov, though, said that the expectation is that, by the time it comes to dumping core, there will be no other users of the address space. Process exit might have similar issues, he thought, but Glisse said that the MMU notifier calls will have caused any HMM devices to back off earlier in the exit path. 

Dickins said that everybody hates `mmap_sem` and is looking forward to removing it. That leads to a tendency to avoid taking it in situations where it is thought that there will be no concurrent accesses anyway. But those are just the situations where there is no overhead to taking `mmap_sem` in the first place. Memory-management developers are just being silly by trying to avoid taking it, he said; it just causes surprises. That part of the discussion ended with Glisse saying he would post a patch adding `mmap_sem` to the core-dump path. 

Glisse went on to say that he has a set of MMU-notifier patches pending. In the process of writing them, he has noticed that a number of users of `get_user_pages()` are broken. In particular, these users assume that pages are fixed in place when they can, in fact, be swapped out from underneath the device that is accessing them. Files that are being truncated are the most common case. If a file is truncated then extended again, a driver using `get_user_pages()` will continue using the old pages, while HMM drivers using MMU notifiers will do the right thing. 

Matthew Wilcox responded that nobody expects DMA to pages that have been truncated to work, so there is no need to fix this behavior unless it is a security problem. Dan Williams agreed, saying that changing all of those drivers to fix something that isn't really a problem does not seem worthwhile. But Glisse said that each driver does things differently, and moving them to common code results in removing hundreds of lines from each driver, so he is likely to persist. 

As time ran out, a couple of other topics were raised briefly. Glisse would like to see a new DMA API that is designed to share mappings between multiple I/O memory-management units. There are also some issues around migrating file-backed pages to a device. This activity must be coordinated with filesystems, and with the writeback activity in particular, since these pages will become inaccessible to the GPU while the device is working with them. 

#### MMU notifiers

When the kernel makes changes to a process's address space, it is able to keep the system's MMU in sync with those changes at the same time. Things get trickier, though, if one or more peripheral devices also have MMUs that must be managed. The answer is the "MMU notifier" mechanism, which allows the HMM code to get a callback from the memory-management subsystem when important changes are made. But it turns out that some changes are more important than others, so Glisse would like to adjust the MMU-notifier API so that the HMM code can tell the difference. 

An MMU notifier is called whenever one or more pages in the address space of interest are invalidated, but the notifier is not given any information about why this invalidation is happening. To rectify that, he suggested adding a new argument that would give the invalidation reason. If the memory is being unmapped entirely, for example, HMM will respond by freeing all of its data structures associated with the memory range. If the memory protections are being changed, instead, there are no changes to the physical pages, and thus no caching issues to deal with. If a page's physical address changes, mappings in the peripheral device must change as well. Some changes, such as clearing the soft-dirty bit, only affect the host and don't require any response from HMM at all. And so on. 

In response to a question from Michal Hocko, Glisse said that the MMU notifiers only fill an advisory role at the moment; they provide information that the HMM subsystem can use. In the future, the notifiers might take significant actions, such as preempting the device, when notifications arrive. 

Dave Hansen worried that adding complexity to the notifiers will make it harder for developers to know what to do when they make core memory-management changes. Under this scheme, it would become harder to know which argument to pass to notifiers in new code; the fear of breaking something in HMM would always be there. That would make the memory-management code harder to maintain. Glisse said that the safe option would be to indicate a full unmap when in doubt; it would lead to the worst performance, but the results would be correct. 

Hansen suggested splitting the suggested event types into a lower-level event mask describing what is actually going on. One bit would indicate a physical address page, another that the virtual-memory area behind the page is being removed, another for software-state changes, etc. Glisse said that this approach would be workable, though it would provide more information than he really needs now. The conversation went in circles for some time on what the specific bits might be with no clear conclusion. It seems fairly clear that this patch will use the bitmask approach, though, when it is posted in the future. 

As things wound down, Andrew Morton noted that there had been [some disagreements](<https://lwn.net/Articles/732952/>) in the past over whether MMU notifiers are allowed to sleep. He asked: has all of that been sorted out? Glisse responded that all notifiers are allowed to sleep at this point.
