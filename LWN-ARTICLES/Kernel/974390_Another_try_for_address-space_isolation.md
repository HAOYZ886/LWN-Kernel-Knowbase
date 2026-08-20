---
title: "Another try for address-space isolation"
url: https://lwn.net/Articles/974390/
date: "May 21, 2024"
category: "Memory management-Address-space isolation; Security-Meltdown and Spectre"
author: "By Jonathan Corbet May 21, 2024 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 21, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

Brendan Jackman started his memory-management-track session at the [2024 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) by saying that, for some years now, the kernel community has been stuck in a reactive posture with regard to hardware vulnerabilities. Each problem shows up with its own scary name, and kernel developers find a way to mitigate it, usually losing performance in the process. Jackman said that it is time to take back the initiative against these vulnerabilities by reconsidering the more general use of address-space isolation. 

In a typical exploit, he said, an attacker will start by carefully mistraining a CPU's branch-prediction hardware. Then, a call into the kernel will cause speculative execution to take a wrong path; the erroneous speculation will be mostly cleaned up when it becomes clear that it was wrong, but not without leaving a secret behind somewhere. The attacker then recovers that secret and leaks it by way of some sort of covert channel. 

#### Keeping data unmapped

[![\[Brendan Jackman\]](https://static.lwn.net/images/conf/2024/lsfmm/BrendanJackman-sm.png)](<https://lwn.net/Articles/974391/>) The key to getting out of the reactive mode is the realization that speculative execution cannot leak data that is not mapped at the time. Thus, keeping sensitive data unmapped when it is not needed can mitigate a number of known exploits — and those we don't yet know about as well. Address-space isolation has been pursued by a number of developers over the years; Jackman was there to talk about a specific patch set that was first implemented for [Hyper-V](<https://en.wikipedia.org/wiki/Hyper-V>) in 2019, and which has been partially deployed at Google this year. The company plans to reach full deployment in the future, and intends to maintain the work going forward. 

This work was [covered here](<https://lwn.net/Articles/886494/>) in 2022; see that article for details on this work (which has not been publicly posted since then). Jackman began a brief overview of this work by pointing out that Linux uses address-space isolation now to keep much of the kernel inaccessible (even via speculation) from user space; there are separate page tables for user and kernel mode. Keeping the kernel's address space isolated from user space protects it from Meltdown vulnerabilities. 

The proposed patch addresses Spectre vulnerabilities by providing address-space isolation within the kernel. It splits the kernel page table into two: a "restricted" page table that only maps readily available (nonsensitive) data, and an "unrestricted" table that maps all of the kernel, including sensitive data. The restricted table is active until there is an actual need to access sensitive data; any attempt to do so will cause a page fault, at which point the kernel will flush caches, perhaps halt sibling processors, then continue with the unrestricted table. That switch is expensive, so the best performance will be had if most paths through the kernel only access nonsensitive data. 

This is, he said, a naive solution, in that everything is either sensitive or not, with no shades of gray in between. Making it less naive involves adding a third level, called "local nonsensitive" (this approach was already reflected in the 2022 patch set). Data in this class can be leaked back to the calling process without ill effect; it is, essentially, information that this process already has access to. But locally nonsensitive data _should_ be protected from any other process in the system. In this mode, each process will have its own set of restricted kernel page tables; it adds complication, so Jackman would like to proceed without this aspect in the beginning, if possible. 

He put up a performance chart showing that existing mitigations for Spectre vulnerabilities have a significant performance cost. With address-space isolation in place and the other mitigations turned off, almost all of that performance was regained and the system was still protected against speculative vulnerabilities. 

There are, he said, some questions that need to be answered about this work; the first of those is about how sensitivity of data is annotated. There is a new set of GFP flags that are used at allocation time for that purpose, Jackman said. In the future, it might also be possible to use the subsystem context more directly; perhaps everything touched by the crypto layer should be seen as sensitive. Eventually the desire will be to figure out sensitivity at run time. 

Even with allocation flags, there are two alternatives that need to be considered, given the need to minimize the amount of restricted data in the interest of better performance. One would be to consider all allocations to be sensitive unless they are specifically marked otherwise; that is, he said, "the only competent security answer". The other is to consider data nonsensitive unless specifically marked as sensitive — "the only competent performance answer". In the end, he said, there are three objectives to aim for: full mitigation, good performance, and reviewable patches. The community, somehow, has to pick which two of those it wants. 

An audience member grumbled that all of this work is just a band-aid, that the proper solution is to just keep sensitive data on a separate processor. David Hildenbrand complained that the community is stuck writing code with the assumption that the hardware is compromised. That is the situation we are in, but he worried that address-space isolation would make it easier for hardware companies to just not care about speculative-execution vulnerabilities. Address-space isolation is designed around the idea that speculative-execution bugs will always be severe, and that may end up perpetuating that situation. Jackman responded that he did not believe that it is possible to create a CPU that is entirely free of this kind of problem, so speculative vulnerabilities will be with us for a long time regardless. 

He returned to his question of whether the initial version of this work should start by emphasizing security or performance. His instinct is to prioritize security, then work on performance until it reaches a point where people actually want to run it. Until that happens, though, bad performance is likely to inhibit testing of the patches. As the session closed, Dan Williams pointed out that Spectre mitigations like [retpolines](<https://lwn.net/Articles/774743/>) started by emphasizing security, leaving performance for later. That has worked out well, he said; the community tends to be more motivated to innovate around performance than security. So, chances are, that is the tradeoff we are likely to see when this patch series returns to the mailing lists. 

#### Implementation details

The discussion was not finished at that point, though; Jackman was able to schedule another slot the next day to get into a few of the details that he was trying to resolve. The core challenge, he said, is that the kernel has to take pains to flush the translation lookaside buffer (TLB) as part of the transition between the unrestricted and restricted modes to prevent use of the TLB as a covert channel. This flushing is expensive, so it should not be done more often than is strictly necessary. 

The most conservative approach, he said, would be to perform a flush every time a page is freed; that would clearly slow things down considerably. So the current approach is to free pages in batches in a kernel thread, then perform the flush once at the end. A proper solution would look different, but would require the kernel to remember the sensitivity of every free page — whether it had been mapped into the restricted address space, in other words. Then, if an allocation request comes in, and the page used to satisfy it was nonsensitive, there is no need to bother with a TLB flush before returning a page. 

Jackman was unsure of how to remember the previous sensitivity of free pages, though. One possibility might be to add a new migration type to track it. Another could be to add a new memory zone; this idea was met with a resounding "no" from the room. 

Michal Hocko asked how developers would request sensitive memory; the answer is to use the new `__GFP_SENSITIVE` allocation flag. Since all of user-space memory is considered sensitive (the kernel has no way to know which user pages actually contain sensitive data), that flag is folded into `GFP_USER` and need not be added separately. There is a new page flag used to mark sensitive pages. Jackman said that he hadn't realized prior to the conference that adding new GFP flags is discouraged; Hocko answered that those flags are in short supply, and that kernel code tends to use them incorrectly in any case. 

Jackman asked for alternative suggestions; Hocko mentioned the scoped interface that is used to modify allocations performed from within the filesystem and I/O paths. Perhaps something similar could be done for sensitive data; that could be better than annotating specific allocations, he said. There are a lot of allocation sites in the kernel, annotating them all is not really feasible and the end result is sure to be incorrect. 

As this session came to a close (for real, this time), Jackman noted that some allocations must be marked as nonsensitive, regardless of the data to be stored there. Specifically, the kernel cannot take page faults around the system-call entry path, so memory accessed then must be nonsensitive.
