---
title: Merging msharefs
url: https://lwn.net/Articles/974512/
date: "May 22, 2024"
category: "Memory management-Page-table sharing; System calls-mshare"
author: "By Jonathan Corbet May 22, 2024 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 22, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

The problem of sharing page tables across processes has been discussed numerous times over the years, Khalid Aziz said at the beginning of his [2024 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) session on the topic. He was there to, once again, talk about the proposed [`mshare()`](<https://lwn.net/Articles/895217/>) system call (which, in its current form, is no longer actually a system call but the feature still goes by that name) and to see what can be done to finally get it into the mainline. 

Threads, he said, naturally share page tables, but independent processes do not. An individual page-table entry (PTE — mapping a single page) is small, but a process's page tables contain many PTEs and can add up to a significant amount of memory use. The problem is exacerbated when many processes share the same memory region; each of those processes will have its own full set of page tables for that region. He mentioned a case of a large, well-provisioned database server that had 1,500 processes all sharing the same memory area; the resulting page-table overhead was larger than the size of the shared region and ran the system entirely out of memory. 

[![\[Khalid Aziz\]](https://static.lwn.net/images/conf/2024/lsfmm/KhalidAziz-sm.png)](<https://lwn.net/Articles/974514/>) To avoid this kind of problem (and to put that memory to better use), Aziz would like to bring thread-like page-table sharing to processes; that is the purpose of `mshare()`, which was originally created by Matthew Wilcox. It provides an opt-in mechanism by which a process can inform the kernel that it wants to share the page tables for a given region; the kernel then makes it possible for other processes to map that region. Since page tables are shared, page protections are also shared, a fact that application developers need to keep in mind. Pasha Tatashin pointed out that, when page tables are shared, the virtual address must also be shared — the region must be mapped at the same address in all processes. 

The first version of the `mshare()` patches was [posted](<https://lwn.net/ml/linux-kernel/cover.1642526745.git.khalid.aziz@oracle.com/>) in January 2022; it was then discussed at LSFMM+BPF that year, resulting in some significant changes. The system call was renamed to `ptshare()` then, but Aziz would now like to move forward with `mshare()`, which has been redesigned around the filesystem-based [msharefs](<https://lwn.net/Articles/901059/>) concept rather than as a new system call. 

To use this feature, Aziz continued, the first step is to mount the msharefs filesystem. A process will then create a file on that filesystem and map it as `MAP_SHARED`. The fact that the file lives in this special filesystem is the indication to the kernel that the creating process wants to share the page tables for that region with others. Those others can open the file, and read this structure from it: 

```
struct mshare_info {
        	unsigned long start;
    	unsigned long size;
        };
```

The `start` and `size` values can then be used by the new process to map the region at the correct location. 

The kernel maintains an [`mm_struct` structure](<https://elixir.bootlin.com/linux/v6.9.1/source/include/linux/mm_types.h#L762>) for each process, describing its address space. Use of msharefs causes the creation of a separate `mm_struct`, independent of any process, to describe the shared region. The kernel, running in the context of the creating process, will end up copying the relevant virtual memory areas (VMAs) over to the new `mm_struct`; its original VMAs will be marked with a special flag pointing to the new `mm_struct`. 

David Hildenbrand asserted that msharefs needed to identify the new VMAs as a sort of special container that would prohibit use with features like [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>), but others objected to that idea. There is no reason, Wilcox said, that these VMAs cannot be used with `userfaultfd()`, just like memory shared between threads can be used there. 

Michal Hocko asked how the shared memory, which has no owning process, is accounted for in control groups; Aziz admitted that accounting was "a little complication" that has not been fully solidified yet. Hocko said that it was important for the kernel to be able to find all of the processes mapping the region and kill them in out-of-memory situations; msharefs cannot be merged without this ability, he said. He added that basic memory accounting also matters; which process is charged when a new page is allocated in response to a fault? Shakeel Butt said that the kernel has no good solution for the accounting of shared memory in general, currently; memory is simply charged to the process that faults it in first. 

Another complication, evidently, is that potential users of this feature want the creating process to exit. Hildenbrand, though, said that the page tables should be torn down when the original process goes away. That process should also be the one that is charged for the shared memory solving the control-group problem. Wilcox worried, though, that keeping the original process around would be an easy way to create unkillable processes. 

The final topic covered was locking; Jason Gunthorpe was concerned that it would now be necessary to take two independent `mmap_lock` locks (one in each `mm_struct`) to make changes to the VMA tree. Wilcox said that there is only a single level of lock nesting, in a well-defined order, so there can be no cycles (and thus no deadlock worries). Hildenbrand said that most page-table walkers should simply refuse to deal with the special `mm_struct`, but Gunthorpe said that `get_user_pages()` needs to work, and that opens a whole can of worms. There are other use cases out there as well, he said. As the session ended, Hildenbrand suggested special-casing things as much as possible, and not trying to do complex things around this strange mechanism initially.
