---
title: "Avoiding game-score loss with per-process reclaim"
url: https://lwn.net/Articles/545668/
date: "April 3, 2013"
category: Memory management
author: "By Michael Kerrisk April 3, 2013"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Michael Kerrisk**  
April 3, 2013

Minchan Kim's recent [patch series](<https://lwn.net/Articles/545669/>) to provide user-space-triggered reclaim of a process's pages represents one more point in a spectrum that increasingly sees memory management on Linux as a task that is indirectly influenced or even directly controlled from user space. 

Approaches such as Android's low-memory killer represent one end of the page-reclaim spectrum, where memory management is primarily under kernel control. When memory is low, the low-memory killer picks a victim process and kills it outright; applications that live in such an environment have to work with the possibility that they may disappear from one moment to the next. As Minchan pointed out via an amusing example, the effects of the low-memory killer's approach to page reclaim can be extreme: 

[Having a process killed to free memory] was really terrible experience because I lost my best score of game I had ever after I switch the phone call while I enjoyed the game 

Jon Stultz's volatile ranges work and Minchan's own work on a similar feature (both described in [this article](<https://lwn.net/Articles/522135/>)) represent a middle point in the spectrum. The volatile ranges approach, inspired by Android's ashmem, provides a process with a way to inform the kernel that a certain range of its own virtual address space can be preferentially reclaimed if memory pressure is high. Under this approach, the kernel takes no responsibility for the contents of the reclaimed pages: if the kernel needs the memory, the page contents are discarded, and it is assumed that the application has sufficient information available to re-create those pages with the right contents if they are needed. As with the low-memory killer, the decision about if and when to reclaim the pages remains with the kernel. 

By contrast, Minchan's patch set places the decision about when to reclaim pages directly under the control of user space. The interface provided by these patches is simple. A `/proc/_PID_ /reclaim` file is provided for each process. A process with suitable permissions--that is, a process owned by root or one with the same user ID as the process `_PID_` --can write one of the following values to the file, to cause some or all of the process's pages to be reclaimed: 

  1. Reclaim all process pages in file-backed mappings. 

  2. Reclaim all process pages in anonymous (`MAP_ANONYMOUS`) mappings. 

  3. Reclaim all pages of the process (i.e., the combination of 1 and 2). 

As currently implemented, all of the process's pages that match the specified criterion are reclaimed. Your editor [wondered](<https://lwn.net/Articles/545817/>) whether there might be benefit in allowing some control over the range of pages that are reclaimed from the target process, by allowing an address range to be written to the `/proc/_PID_ /reclaim` file. 

By contrast with volatile ranges and the low-memory killer, modifications in pages reclaimed via `/proc/_PID_ /reclaim` are not lost. Modified pages are written to the underlying file in the case of shared (`MAP_SHARED`) file mappings or to swap in other cases. Thus, if the process touches the reclaimed page later, it will be faulted into memory with the contents at the time it was reclaimed. The patches also include some logic to handle the case where multiple processes are sharing the same pages; in that case, the pages are reclaimed only after all of the processes have marked them for reclaim. Like the low-memory killer, `/proc/_PID_ /reclaim` can be used to reclaim all of the pages in a process, but without needing to kill the process to do so. 

The idea behind Minchan's proposal is that a user-space task manager could take over some part of the job of memory management. In some cases, this may be more effective than allowing the kernel to make memory-management decisions, since the user-space task manager can bring application-specific intelligence to decisions about whether to reclaim a process's pages. For example, some application processes may be in the foreground while others are in the background. It may desirable to preferentially reclaim pages from one of the background processes, even if it has some frequently accessed pages. Of course, the task manager would somehow need to know when the system is under memory pressure. To that end, a mechanism like Anton Vorontsov's proposed [`vmpressure_fd()` API](<https://lwn.net/Articles/524742/>) might be useful. 

Minchan's patches apply against Michal Hocko's [MMOTM (memory management of the moment) tree](<https://git.kernel.org/cgit/linux/kernel/git/mhocko/mm.git/>). The patches came out on March 25, but have so far seen little review. Nevertheless, they present an idea that will probably be of particular interest for the developers of mobile and embedded devices and thus it seems likely that they will get some attention at some point in the future.
