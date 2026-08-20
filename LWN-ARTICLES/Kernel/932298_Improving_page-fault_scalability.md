---
title: "Improving page-fault scalability"
url: https://lwn.net/Articles/932298/
date: "May 29, 2023"
category: "Memory management-Scalability; Memory management-mmap sem"
author: "By Jonathan Corbet May 29, 2023 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 29, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

Certain topics return predictably to development conferences every year, usually because developers are still struggling to find a viable solution to a specific problem. One such topic is the lack of scalability in the kernel's page-fault-handling code, so it was no surprise to see this problem on the agenda for the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>). Matthew Wilcox led a session in the memory-management track to discuss the state of page-fault handling and what can be done to improve it further. 

He started by noting that Suren Baghdasaryan has been doing the bulk of the work in this area over the last year. There are two big issues when it comes to page-fault scalability: contention for the per-process `mmap_lock` lock and priority inversions between monitoring tasks and the real workload (which can come about, for example, when the monitoring task reads data from `/proc/_pid_ /smaps`). In the [2022 discussion](<https://lwn.net/Articles/893906/>), he said, a number of options for addressing these issues were discussed, including the longstanding work to implement speculative page-fault handling, using read-copy-update (RCU) for much of the handling path, locking at the virtual-memory-area (VMA) level, and finer-grained locking using the recently added [maple tree](<https://lwn.net/Articles/845507/>) data structure. 

[![\[Matthew Wilcox\]](https://static.lwn.net/images/conf/2023/lsfmm/MatthewWilcox2-sm.png)](<https://lwn.net/Articles/932387/>) Since then, the maple tree has replaced the red-black tree for managing process address spaces. Baghdasaryan has implemented the RCU lookup and [per-VMA locking](<https://lwn.net/Articles/906852/>) and gotten it upstream. Work that is in progress includes a set of patches to handle faults on file-backed VMAs with up-to-date pages in the page cache (which can be handled without starting I/O). There is also work underway to improve fault handling for pages in the swap cache. Both of those cases have to fall back to the `mmap_lock` now. 

In other words, work continues to grow the number of cases that can be handled without resorting to the `mmap_lock`. Future projects will add more of these cases, including initiating and waiting for I/O, providing the data for `/proc/_pid_ /smaps`, faults handled via [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>), faults in VMAs created by device drivers, and so on. Someday there will be no cases left to convert, and `mmap_lock` can be removed entirely. 

Expanding on the "waiting for I/O" case, Wilcox said that he is looking at both swap-backed and file-backed pages. The current plan is to take a reference to the file being read, drop the lock, then sleep. Once the I/O completes, the fault-handling process would restart from the beginning to catch up with any other address-space changes that may have taken place. He asked the group whether this was the correct model, or whether it would be better to simply block changes to the faulting VMA while the I/O is underway. 

Michal Hocko answered that the two cases are different and should perhaps be handled differently. In the swap case, there is always the possibility that the owning process could unmap the memory while waiting for the faulting page to be read in. This problem could be avoided by simply holding the VMA lock while waiting. This approach would not work as well for the file-backed case, he said, where the VMAs do not map to process-visible objects. 

Another potential problem, Wilcox said, can come about in the case of a process where two threads call `malloc()` at the same time. Each of those calls could end up calling [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) to get more memory to satisfy the allocation request. Those two calls would normally create two VMAs, but the kernel might get clever and combine the two; that, in turn, could create contention on the VMA lock that the application is not expecting. Steve Rostedt suggested using tracing to get some real numbers showing whether this is a real-world problem, but Hocko said that he sees regular bug reports involving "an unnamed database product" showing this kind of contention. 

Wilcox said that the case of initiating I/O without `mmap_lock` held is easier. It has been established that calling into drivers with memory-management locks (such as a per-VMA lock) held is a safe thing to do, even in the absence of `mmap_lock`. 

The monitoring case presents its own challenges, he continued. It is possible to walk the VMA lists holding just the RCU lock, and a number of `/proc` interfaces can work in that mode; "it's just a matter of programming". But the `smaps` file is more complicated; to collect its information, it must be able to keep page tables from being freed, and that requires taking the `mmap_lock`. On the x86 architecture, it is also necessary to disable interrupts — the result of a "stupid legacy that we should just get rid of", he said. Vlastimil Babka said that this behavior is the result of the need to block inter-processor interrupts that flush translation lookaside buffers. 

Jason Gunthorpe said that page-table freeing could maybe be protected by RCU, but that would require embedding an `rcu_head` structure in the `page` structure; Wilcox answered that it's already there, but that page-table freeing is not using it. Mike Rapoport said that RCU freeing of page tables is feasible; Wilcox replied that he'd like to see it done, but that "there might be demons" there. Hocko, though, said that this could be a good low-hanging-fruit project for somebody to look into. 

For the `userfaultfd()` case, where the fault is being reported to user space for resolution, Wilcox allowed that he lacked ideas. Baghdasaryan said that it looks similar to the swap case, and that dropping the lock before notifying user space could work. 

For device-driver VMAs, there is the problem that the drivers themselves might be depending on `mmap_lock` being held, so just dropping it is likely to lead to unpleasant bugs. Wilcox suggested that he inexplicably lacks the desire to audit every driver in the kernel for this kind of problem. Instead, drivers will need to explicitly indicate that they are prepared to handle faults without the lock held. Drivers would also have to indicate that they do not _drop_ the `mmap_lock` in their fault handlers. Drivers could possibly implement the `map_pages()` VMA operation to map their pages ahead of time, which is the most efficient way to map pages into user space. `map_pages()` is protected by the RCU read lock, though, meaning that drivers cannot sleep while using it. 

Gunthorpe said that drivers have to be prepared for a process to fork, creating two independent copies of the driver-provided VMA. Since each mapping will be protected by a separate `mmap_lock`, drivers can't rely on that lock in any case. Michel Lespinasse said that there are, nonetheless, drivers that depend on `mmap_lock`, so some sort of allowlist will be needed to be able to call into drivers without that lock. For now, though, the per-VMA locks are for anonymous VMAs only, so the lack of `mmap_lock` not an issue for drivers. 

Finally, Wilcox turned to the idea of removing `mmap_lock` entirely, which is an objective he would like to reach someday. It remains a multi-year project, though, much like the removal of the big kernel lock, which was finally [completed](<https://lwn.net/Articles/424657/>) in 2011. An important step in that direction would be to stop using the lock to protect the VMA tree, splitting each use into its own lock. Lespinasse said that he couldn't see how that could be done; the interactions with reverse mapping, in particular, would complicate things. 

As the session concluded, it was pointed out that the quest for scalability does not end with the removal of `mmap_lock`; Wilcox is already looking forward to handling faults without taking the VMA locks either. It is reasonable to expect that the VMA locks will cause cache-line contention, though some profiling with perf will be needed to verify that. There is a path toward lock-free fault handling, he said, but it involves a fair amount of complexity and he'll only pursue it if performance requires it.
