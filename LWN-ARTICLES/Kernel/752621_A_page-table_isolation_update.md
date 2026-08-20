---
title: "A page-table isolation update"
url: https://lwn.net/Articles/752621/
date: "April 25, 2018"
category: "Security-Meltdown and Spectre"
author: "By Jonathan Corbet April 25, 2018 LSFMM"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
April 25, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

Dave Hansen did much of the work to get kernel page-table isolation (PTI) into the kernel in response to the Meltdown CPU vulnerability. In the memory-management track of the 2018 Linux Storage, Filesystem, and Memory-Management Summit, he ran a discussion on how PTI came about, what the costs are, and what can be done to minimize its performance impact. 

Hansen started by saying that he was not going to talk about the Spectre vulnerabilities, which are not seen as being much of a problem at the memory-management level. This code is relatively far from user space, so forcing a particular speculative-execution path there is generally impractical. The PTI patches, instead, affect memory management directly. 

PTI got its start in [this paper by Daniel Gruss et al. [PDF]](<https://gruss.cc/files/kaiser.pdf>), where it was called KAISER. The original purpose behind KAISER was to address threats to kernel address-space layout randomization; the Meltdown vulnerability was not known to the authors when the paper was written. Later on, when it became clear that there was [![\[Dave Hansen\]](https://static.lwn.net/images/conf/2018/lsfmm/DaveHansen-sm.jpg)](<https://lwn.net/Articles/752676/>) something going on, the authors found Meltdown and disclosed it to Intel — they were one of four independent groups to do so. Hansen wondered how many others had also found the problem but kept it to themselves. 

In any case, when Meltdown was discovered, Hansen realized that KAISER was a good way to address the problem. One of the first things he did with the code was to add support for the Intel processor context ID (PCID) feature. PCIDs, support for which was added by Andy Lutomirski around 4.14, allow page tables to be switched without flushing the translation lookaside buffer (TLB), thus mitigating some of the performance impact of PTI (which, by its nature forces a lot of extra page-table switches). 

One interesting question is: how long should PTI be enabled in the kernel? The way things stand now, the kernel will turn it off if it detects that it is running on a processor that is not vulnerable to Meltdown. The "performance folks" want that, of course, since they want to get as much performance as possible out of their CPUs. People who are more concerned with security, though, are inclined to leave it enabled indefinitely. There are a number of useful security properties that come from completely separating the kernel and user-space address spaces. 

One of the consequences of PTI is, of course, that system calls, page faults, and interrupts all get slower. The PTI patches also turned off the "global pages" feature, which kept the kernel's TLB entries resident at all times. He has [a patch set](<https://lwn.net/Articles/750768/>) to restore global pages conditionally in the 4.17-rc kernels; there are still some issues with the changes, he said. Another outcome of PTI is that TLB invalidation becomes much trickier. PCIDs allow a small number of address spaces to be active at one time; when pages are shared between address spaces, they must be separately flushed for each active process. There is also a slight increase in memory use when PTI is active. 

Users who are concerned about the performance impacts have a few ways to minimize those impacts, most of which are good practice even without PTI. For example, he said, minimize the number of system calls that are made. Try to avoid extra TLB flushes; he mentioned the `MALLOC_TRIM_THRESHOLD` environment variable that, if misconfigured, can cause glibc to make excessive address-space changes. Huge pages should be used when possible; they can reduce the number of page faults, and also the number of system calls needed to manipulate them. Try to avoid page faults in general; mapping memory with `MAP_POPULATE` when it will all be accessed is one way to do that. 

Michal Hocko said that it might make sense to add more multiplexing system calls for address-space operations. For example, database developers would appreciate a batched version of `mprotect()` that would make several changes with a single system call. Hansen agreed that this might make sense; he lamented that the API for [memory protection keys](<https://lwn.net/Articles/689395/>) (which he wrote) was designed around the notion that "system calls are fast". Now they aren't quite as fast and some of those decisions don't look as good as they did before. Rik van Riel said that a vectored version of `madvise()` would give big performance improvements for memory allocators, and Hansen added that a batched version of `munmap()` would also be helpful. 

Another developer asked about ways to increase locality in general; Hansen replied that huge pages are the answer there. Huge pages were "an afterthought" in the processors where they first appeared, so they were not supported all that well. There were not many TLB entries available for huge pages, for example. But huge pages are a first-class citizen now, and they should be used when possible. 

Alexei Starovoitov asked about the ability to turn off PTI on a per-process basis; patches to allow this were circulating in January, but haven't been discussed recently. Hansen replied that the most security-sensitive applications are the ones that need PTI, but they are also often quite performance-sensitive. As a result, the people who need PTI the most are often the ones who most want to turn it off. A per-process disable can thus be problematic; that said, it may now be time to revisit that issue. 

Laura Abbott asked about the status of older hardware; there will be users who don't want to update. Hansen replied that older hardware will always need PTI; there are no microcode updates forthcoming that will address Meltdown on that hardware. Abbott also asked about PTI for 32-bit systems; Hansen said that [the 32-bit patches](<https://lwn.net/Articles/752551/>) will go in soon; they are "not as terrifying" as had been originally thought. 

The final question came from Dan Williams, who wondered whether PTI was a useful mitigation for the newer "Spectre prime" variant. Hansen replied that any attack that takes advantage of a shared address space will be mitigated by PTI. Its costs notwithstanding, using PTI to separate the spaces brings some fundamental security benefits, he said.
