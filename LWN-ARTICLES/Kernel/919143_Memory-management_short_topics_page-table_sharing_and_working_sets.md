---
title: "Memory-management short topics: page-table sharing and working sets"
url: https://lwn.net/Articles/919143/
date: "January 9, 2023"
category: "Memory management-Page replacement algorithms; Memory management-Page-table sharing"
author: "By Jonathan Corbet January 9, 2023"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
January 9, 2023

The kernel's memory-management developers have been busy before and during the holidays; the result is a number of patch sets making significant changes to that subsystem. It is time for a quick look at three of those projects. Two of them aim to increase the sharing of page tables between processes, while the third takes advantage of the [multi-generational LRU](<https://lwn.net/Articles/894859/>) to create a better picture of what a process's working set actually is. 

#### Revisiting msharefs

Some applications are structured as a large set of independent processes, all sharing a (potentially large) region of memory. Each of those processes will have its own set of page tables for that shared region. Duplicating page tables imposes a relatively small cost when the number of processes is low, but when that number gets large, the memory occupied by page tables may exceed the size of the memory region they refer to. In many cases, this duplication of page tables brings no extra value. 

For some time, Khaled Aziz has been working on a mechanism to allow cooperating processes to share page tables referring to a shared memory area; this work has, at times, taken the form of [the `mshare()` system call](<https://lwn.net/Articles/895217/>) and [the msharefs filesystem](<https://lwn.net/Articles/901059/>). There have been concerns raised with both solutions, so now Aziz is back with [yet another attempt](<https://lwn.net/ml/linux-kernel/cover.1670287695.git.khalid.aziz@oracle.com/>). This implementation does away with new system calls and filesystems and, instead, just adds a new flag (`MAP_SHARED_PT`) to the [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) system call. If a process maps a shared segment (implying that `MAP_SHARED` must also be provided) with this new flag, then the page tables mapping this segment will also be shared with the other users, saving the overhead of making an independent copy of those tables. 

As with the other versions, there are some interesting semantics and limitations associated with shared page tables. Any address-space changes (such as an [`mprotect()`](<https://man7.org/linux/man-pages/man2/mprotect.2.html>) call) made to the shared region by one process will apply to every process sharing the page tables; that is seen as an advantage for some use cases. The memory segment must be aligned at the PMD level (2MB on many architectures), and it must be mapped at the same virtual address in all processes. The same-address requirement could perhaps be removed, Aziz said, if there is a reason to do so. 

Underneath the API, the implementation of page-table sharing follows the same lines as before. A separate `mm_struct` structure is created to manage the shared region as if it were a separate address space. 

There have been no comments on the new version so far. One might expect that using `mmap()` would address most of the concerns about the user-space API for this feature. But this kind of page-table sharing, with its unique semantics, represents a significant memory-management change to serve a relatively rare use case. It is not yet clear that the case has been made that this functionality is worth the cost. 

#### Copy-on-write page tables

A different, and somewhat more transparent, approach to page-table sharing can be found in [this patch set](<https://lwn.net/ml/linux-kernel/20221220072743.3039060-1-shiyn.lin@gmail.com/>) from Chih-En Lin. When a process calls [`fork()`](<https://man7.org/linux/man-pages/man2/fork.2.html>), the new child process will share its memory with the parent. Any writable pages are marked copy-on-write (COW); should either process write to a COW page, that page will first be copied (breaking the sharing) so that the other process does not see the change. Sharing memory in this way saves a lot of copying, especially if the child process will not actually use much of the parent's memory. 

While the parent's memory is not copied into the child on `fork()`, the parent's page tables _are_ copied. If the parent process has a large address space, that copying can still create a significant cost, and it may be entirely useless if the child does not access that memory. Lin seeks to reduce that cost by, instead, extending the COW mechanism to the bottom (PTE) level of the page-table hierarchy. 

A process must opt into the COW behavior with a new [`prctl()`](<https://man7.org/linux/man-pages/man2/prctl.2.html>) command (`PR_SET_COW_PTE`). Once that has been done, any new child processes will be created with shared page tables. The usual COW behavior applies here; should either process make a change to a PTE page, that page will be copied and the sharing will be broken. An `mprotect()` call, for example, would end up copying the affected page-table pages. Thus, COW page tables should not result in any behavioral changes visible to either side, other than `fork()` calls running a bit more quickly and requiring less memory. 

Of course, that is not quite true. While a `fork()` may be a bit faster, other operations, including page-fault handling, may be slower due to the need to break the sharing of the page-table pages. Whether this sharing is beneficial overall may thus vary depending on the workload; benchmark results included in the cover letter show a 3-5% performance increase for some workloads, and a slight decrease for others. This variability of results explains the need to opt into the COW behavior; for most workloads it probably will not make enough of a difference to be worth the trouble. 

Here, too, the implementation adds a certain amount of complexity to the core memory-management code. The sharing of page-table pages requires the addition of a reference count to each of those pages so the kernel knows when they are no longer in use. There are numerous operations that can require the sharing to be broken, including transparent huge-page collapse, kernel same-page merging, [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) calls, and more. The code also has to properly handle page-table pages that cannot be shared, including those referring to pages that are pinned or mapped to device memory. The [first posting](<https://lwn.net/ml/linux-kernel/20220519183127.3909598-1-shiyn.lin@gmail.com/>) of this work drew [some questions](<https://lwn.net/ml/linux-kernel/d1810538-9b4c-7f19-852f-7f6d255533c7@redhat.com/>) about whether the added complexity was worth it (and [a side discussion](<https://lwn.net/ml/linux-kernel/YolHr1GwfA++i9jj@casper.infradead.org/>) on better alternatives to `fork()`). There have been few responses to the current version, but it seems likely that this discussion has not yet reached its conclusion. 

#### Working-set estimation

A process's "working set" is the subset of its pages that it is actually using at any given time. Identifying the working set is a key part of effective memory management; if a process's working set can be kept in RAM, that process will perform much better than if it must continually fault pages in from secondary storage. Giving a process more memory than is needed to hold the working set, though, is wasteful. So a lot of effort goes into trying to give each process just enough memory — but not too much. 

In [this patch set](<https://lwn.net/ml/linux-kernel/20221214225123.2770216-1-yuanchu@google.com/>), Yuanchu Xie notes that the multi-generational LRU (MGLRU) work that was merged for the 6.1 kernel provides much of the infrastructure needed to create better working-set-size estimates. The MGLRU organizes a process's pages into "generations", with recently-used pages being placed into the youngest generation. Over time, unused pages age into the older generations, until they are eventually reclaimed. 

The working set should thus be found in the youngest generations. The only problem is that the generational aging does not happen on any sort of set schedule; instead, it is done when memory pressure increases and the kernel needs to find pages to reclaim. As a result, the younger generations can accumulate pages that have not been used in some time, while pages that are part of the working set may remain stuck in the older generations; this situation can persist for some time if memory pressure is not high. 

As a way of getting better working-set-size estimates out of the MGLRU, Xie adds a new mechanism to force aging to happen regularly. It takes the form of a new knob, `memory.periodic_aging`, that is implemented in the memory control-group controller, but for the root group only. It holds the aging interval in seconds; setting it to a non-zero value will enable periodic MGLRU aging system-wide. There is a new kernel thread, called `kold`, that does this aging work. 

If `memory.period_aging` is set to, for example, 60 seconds, then the youngest generation for any process should contain the pages that are known to have been used within the last minute, while the second-youngest generation will hold pages that have been idle for more than one minute, but less than two. The kernel could use this information to adjust the amount of memory available to each process, but it could also be of use to user-space memory-management mechanisms. Processes could use their own working-set information to optimize their behavior and avoid using more memory than is available to them. 

Before user space can use this information, though, it needs to be made available, which is not currently the case. So the patch set adds another memory-controller file called `memory.page_idle_age` to export generational data to user space. Reading this file will produce a table with counts of the number of pages in each of a set of fixed age ranges (ranging from one second to just over one hour), with separate lines for file-backed and anonymous pages. This information seems like it could be useful in a number of situations, including simply better understanding how the generational-aging algorithm is working. 

This patch series is on its first posting, and has not yet drawn any review comments. It is far less invasive than the other patches examined here and seems like it should be less controversial. If nothing else, though, this work could benefit from some documentation so that potential users of the new functionality do not need to reverse-engineer its interface from the source.
