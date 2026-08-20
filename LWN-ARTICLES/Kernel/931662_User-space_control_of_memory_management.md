---
title: "User-space control of memory management"
url: https://lwn.net/Articles/931662/
date: "May 15, 2023"
category: BPF; Memory management
author: "By Jonathan Corbet May 15, 2023 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 15, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

In a remotely presented, memory-management-track session at the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), Frank van der Linden pointed out that the line dividing resources controlled by the kernel from those managed by user space has moved back and forth over the years. He is currently interested in making it possible for user space to take more control over the management of memory resources. A proposal was discussed in general terms, but it will require some real scrutiny on its way toward the mainline, if it ever gets there. 

Van der Linden noted that, in recent years, resource control has been moving back toward user space in a number of areas. Networking is a prominent example of this shift. There has also been an increasing demand for interfaces to assist orchestrators with their work; this shows, for example, in the proposal to add a [`pidfd_set_mempolicy()`](<https://lwn.net/ml/linux-mm/20221111084051.2121029-1-hezhongkun.hzk@bytedance.com/>) system call. He has been doing some prototyping in this area, building on Google's extensive experience in pushing resource control to user space. 

[![\[Frank van der
Linden\]](https://static.lwn.net/images/conf/2023/lsfmm/FredvanderLinden-sm.png)](<https://lwn.net/Articles/931762/>) There are a number of existing mechanisms that can be used to influence memory management from user space, he said. These include a set of sysctl knobs, control-group limits, control knobs for proactive reclaim, and the [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) system call. `madvise()` has gained a lot of options over the years, he said, suggesting that perhaps there is a need for a more generic solution. There is also the NUMA API and, for applications that want to get deeply involved in memory management, [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>), which actually diverts page-fault handling into user space. His focus was on `madvise()` and [`set_mempolicy()`](<https://man7.org/linux/man-pages/man2/set_mempolicy.2.html>), with the idea of getting the most performance out of increasingly complex computing environments. 

The core idea behind his work is to create a structure that can be used to provide memory-management hints and control to the kernel. It should be easily accessible, especially to BPF programs loaded from user space, which would make decisions based on input received from user space via BPF maps. He is not looking for a way to replace `madvise()` and `set_mempolicy()`, but he would like a way to combine them in a way that BPF programs could use. Also needed is a way to tag a virtual memory area (VMA) with an opaque (to the kernel) value, preferably in a way that avoids collisions when more than one user wants to tag a given VMA. 

In the prototype work done so far, he has developed a hint structure for anonymous VMAs. The [multi-generational LRU](<https://lwn.net/Articles/856931/>) has been modified to use this structure to make memory accesses by some processes count more than accesses by others. An access by a favored process could immediately promote a page to the youngest generation, for example, while accesses by others could result in slower (or, indeed, no) promotion. This mechanism would be, in essence, "a nice value for memory access". It was easy to implement, he said, though he did stipulate that its practical value is "questionable". 

A more useful feature, perhaps, is the ability to attach "compressibility hints" to pages. Google, like others, uses [zswap](<https://www.kernel.org/doc/html/latest/admin-guide/mm/zswap.html>) to compress unused pages to save memory. Some pages do not compress well, though, so attempts to store them in zswap are just a waste of CPU time. An application that, for example, is storing already-compressed data in memory can provide a hint telling the kernel not to bother trying to compress it again. 

Van der Linden closed his presentation by asking the group whether the idea seemed like a direction worth pursuing. 

The first question came from Suren Baghdasaryan, who asked why BPF had been chosen as the way to access these hint structures. Was that choice made for ease of prototyping, or is a BPF-based interface the intended solution in the end? Van der Linden answered that BPF was the most flexible way to begin this work, but it isn't necessarily the final result. BPF does make things easy, though; it performs well and, he said, having hooks for BPF is a good thing in general. 

Michal Hocko worried about exposing too many internal memory-management details that would create future ABI problems; a mechanism like this could be hard to maintain over the long term, he said. This concern, in turn, could make it hard to get the work upstream in the first place. Van der Linden responded that, regardless of the mechanism chosen, maintaining ABI compatibility could be challenging; he asked if anybody could suggest a better way to avoid such problems. Hocko said that a well-defined user interface might be better, but the bar for acceptance will be high regardless. Defining the functionality as narrowly as possible would help. 

Matthew Wilcox said that this feature would have been more interesting if it has been proposed prior to the merging of [`process_madvise()`](<https://lwn.net/Articles/810076/>), which has a similar intended functionality. Any new approach will have to demonstrate added value beyond what `process_madvise()` provides. Van der Linden concluded the session by saying that he would continue the discussion by posting examples to the mailing list.
