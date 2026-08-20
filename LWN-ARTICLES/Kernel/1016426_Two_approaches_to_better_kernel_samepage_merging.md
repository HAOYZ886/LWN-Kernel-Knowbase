---
title: Two approaches to better kernel samepage merging
url: https://lwn.net/Articles/1016426/
date: "April 9, 2025"
category: "Memory management-Kernel samepage merging"
author: "By Jonathan Corbet April 9, 2025 LSFMM+BPF"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 9, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

The [kernel samepage merging (KSM)](<https://docs.kernel.org/admin-guide/mm/ksm.html>) subsystem works by finding pages in memory with the same contents, then replacing the duplicated copies with a single, shared copy. KSM can improve memory utilization in a system, but has some problems as well. In two memory-management-track sessions at the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit, Mathieu Desnoyers and Sourav Panda proposed improvements to KSM to make it work better for specific use cases. 

#### Supporting user-space text patching

Desnoyers has come to KSM to help with a seemingly unrelated problem that he is working on: code patching for user-space processes. He works with instrumentation like the [LTTng tracing framework](<https://lttng.org/>), which allows the placement of tracepoints within an application. In the current implementation, each tracepoint has a controlling variable and a branch to determine whether the tracepoint fires. Some of his customers, he said, have applications with 30,000 tracepoints in them; at that scale, the extra overhead for each tracepoint adds up. He would like to improve this situation by using code patching, as is done in the kernel now. There are other use cases, including tuning code for available features or selecting application features to enable, that would benefit from code patching as well. 

Since this is performance-oriented work, he is concerned about the code patching creating new performance problems of its own. Specifically, patching code by writing to a text page will cause that page to be copied; code that was once shared no longer is. That will be the case even if all of the processes sharing that page of text patch it in the same way. KSM can perhaps help to undo this duplication, but its page scanning brings overhead of its own, which Desnoyers would like to avoid. 

Another problem is that KSM is focused on deduplication on systems hosting virtual machines. It requires configuration by the system administrator to work effectively, and brings security concerns. He would rather have a simpler solution that just works. 

He had [proposed such a solution](<https://lwn.net/Articles/1014220/#ksm>) prior to the Summit, but had been told by Linus Torvalds that there is not room in the kernel for two implementations of KSM. He is not looking to replace the current KSM implementation, so that has led him to start thinking about other approaches. 

One possibility, he said, would be to add the concept of per-user file overlays. The kernel's [uprobes](<https://lwn.net/Articles/499190/>) mechanism can patch a running process now, but it changes all processes in the system that are running the targeted code, while he would like to limit the effects to a single process. So he is thinking he could add a new system call for code patching that would create a new overlay, tracking each user's changes to a given binary. It would apply on top of the files an application uses, and changes would apply immediately to all processes (owned by that user) that are running the affected program. The downside of this approach, he said, is that it would make it hard to instrument different parts of a process hierarchy differently. 

So a better solution might be to create a `text_poke()` system call that would be provided a vector of instructions to patch. The kernel would track the altered pages for each address space (mapped file) at several levels — altered pages can be further altered later on. Whenever a process modifies one of its pages in this way, the kernel would attempt to find other altered copies of the same page and, if it finds one containing the same alterations, the two users would share the page. The altered pages would be cached even if all users exit, meaning that the patched pages would persist for short-lived applications that will want them again in the future; they could be reclaimed as needed when memory gets tight. 

It is fair to say that this idea did not evoke a great deal of enthusiasm in the room. 

Matthew Wilcox asked if Desnoyers was familiar with the "reflink" concept, which he described as a sort of copy-on-write hard link for files. There have been [efforts over the years](<https://lwn.net/Kernel/Index/#reflink>) to create a generalized reflink capability for Linux without success, but some filesystems implement that functionality internally. Wilcox suggested that code patching could act like reflink under the covers without exposing the changed files to user space. When code is patched, he said, a new inode could be created for the altered file and stored in the associated virtual memory area. 

The hard part, he said, is that the kernel does not have an efficient way to cache reflinked files. Also, the same (unchanged) page in two reflinked files will be stored as separate pages in the page cache; fixing that has been on the wishlist for years. Desnoyers asked how it might be possible for the kernel to map a list of modifications to the correct inode; Wilcox pondered for a moment, then answered: ""Oh well, it was a nice idea"". 

David Hildenbrand, though, expressed interest in pursuing the reflink idea further. What is needed, he said, is a high-level description of the changes to be made. The kernel can then generate a new file from that list on demand, and reclaim it when needed. The idea sounds easy and clean, he said, ""except it won't be easy"". The session concluded with Wilcox saying that it was an interesting problem. 

#### Selective KSM

In the following session, Panda briefly presented two proposals to address some of the problems with KSM. The feature is useful, he started, but it requires a lot of adjusting of parameters to work well, adds run-time overhead for the page scanning, and has been seen as a security problem as well. 

[![\[Sourav Panda\]](https://static.lwn.net/images/conf/2025/lsfmm/SouravPanda-sm.png)](<https://lwn.net/Articles/1016429/>) The first idea is "synchronous KSM", where the merging of pages would be directed synchronously by user space. The merging of pages would only happen when requested (and the time taken would be charged to the process requesting it), and only the specific memory areas indicated would be considered for merging. The actual request could be made by way of sysfs, [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>), or some other system call. Security would be improved, since the caller has control over which pages are considered for merging, and CPU efficiency would be improved over the background scan that KSM currently uses. The biggest limitation would be that, once two pages diverge from each other, they will stay separate, even if they come to have the same contents in the future. 

The second proposal is "partitioned KSM", where processes would be divided into sensitive and non-sensitive partitions. This partitioning would be controlled via a sysfs hierarchy; new partitions can be created as needed. Merging would be controlled by writing a process ID and an address range to a partition's control file; the kernel would add the process to the partition, then synchronously scan the given address range for merge candidates. Hildenbrand said that this idea is similar to using `madvise(MADV_MERGEABLE)` to control merging, except that it acts synchronously. He suggested using [`process_madvise()`](<https://man7.org/linux/man-pages/man2/process_madvise.2.html>) rather than sysfs to control this feature. 

An alternative, Panda said, would be to create a new `ksm_open()` system call that would accept the name of a partition to join and return a file descriptor representing that partition. There would be a `ksm_merge()` to request the merging of duplicate pages within that partition. Other system calls would be added to undo the merging of pages or detach from the partition entirely. Hildenbrand said that dropping the current KSM implementation is not an option, so a mechanism that simply adds partitions is potentially interesting. 

As the session (and the day) came to an end, Panda asked whether any such feature should be configurable at compile time; Michal Hocko advised against that, since KSM is already an opt-in feature. He said that he likes the file-descriptor idea, which provides a clear namespace for KSM operations. Hildenbrand said that the global KSM functionality could remain too, it would just have to be carefully disabled for any process that joins a partition.
