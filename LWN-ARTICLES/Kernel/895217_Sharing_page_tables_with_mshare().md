---
title: Sharing page tables with mshare()
url: https://lwn.net/Articles/895217/
date: "May 17, 2022"
category: "Huge pages; Memory management-Page-table sharing; System calls-mshare"
author: "By Jonathan Corbet May 17, 2022 LSFMM"
---

By **Jonathan Corbet**  
May 17, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

The Linux kernel allows processes to share pages in memory, but the page tables used to control that sharing are not, themselves, shared; as a result, processes sharing memory maintain duplicate copies of the page-table data. Normally this duplication imposes little overhead, but there are situations where it can hurt. At the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM), Khaled Aziz (remotely) and Matthew Wilcox led a session to discuss a proposed mechanism to allow those page tables to be shared between cooperating processes. 

#### Some `mshare()` background

There was not much discussion of the motivation for this work or the proposed API in this session, which was focused on implementation. That information can be found, though, in [this patch set](<https://lwn.net/ml/linux-kernel/cover.1649370874.git.khalid.aziz@oracle.com/>) posted in April. Eight bytes of page-table entry per page is not much overhead — until you have thousands of processes sharing the page, at which point the space taken by page tables is more than the shared page itself. There are applications out there that run that many processes, so there is a desire to reduce the overhead of non-shared page tables. 

The proposal is a pair of new system calls, the first of which is `mshare()`: 

```
int mshare(char *name, void *addr, size_t length, int oflags, mode_t mode);
```

A process wanting to share a range of memory (along with the page tables) will first create a region, probably with [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>); this region must be aligned to a 512GB boundary. The call to `mshare()` provides the address and size of this region, along with a `name` to identify it. This call, if successful, will create a file with the given `name` under `/sys/fs/mshare` that, when read, will provide the given `addr` and `length` values. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

[![\[Khaled Aziz and
Matthew Wilcox\]](https://static.lwn.net/images/conf/2022/lsfmm/WilcoxAziz-sm.png)](<https://lwn.net/Articles/895428/>) Any other process that wishes to share this region of memory will start by opening that file and reading the associated address and size; it can then call `mshare()` with that information to set up the mapping. The permissions on the file in `/sys/fs/mshare` control the access to this region. The mapping shares the memory, but also the page tables that control it. As a result, any changes to those page tables, with `mmap()` or [`mprotect()`](<https://man7.org/linux/man-pages/man2/mprotect.2.html>) for example, will affect all processes that are sharing the region. 

When a process is finished with the shared area, it can call `mshare_unlink()`, passing the given `name`; when all processes detach from the region, it will be destroyed. 

Wilcox began the session by noting that a process's address space is described by [`struct mm_struct`](<https://elixir.bootlin.com/linux/v5.17.7/source/include/linux/mm_types.h#L458>), of which each process has one. When `mshare()` is used to create a shared area, a new `mm_struct` is created to describe that part of the address space. This structure has no tasks assigned to it, but it is pointed to from the virtual memory areas (VMAs) in each process that have the area mapped. Since one process's actions on the shared area affect all of them, this mechanism is suitable for cooperating processes that trust each other. 

#### Scary

Aziz had a set of questions for the group. What, he asked, is the right granularity for page-table sharing? The current patch set shares page tables at the PMD level, but there might be value in sharing higher-level page directories. He asked whether the proposed API makes sense, and whether it should be possible for a process to map only a portion of the shared region (which is not supported now). Should [`mremap()`](<https://man7.org/linux/man-pages/man2/mremap.2.html>) be supported in a shared region? He also had questions about how [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>) should interact with this feature. 

Michal Hocko started by saying that this feature "sounds scary". He had a number of questions of his own. Who, in the end, is in charge of the shared `mm_struct` structure? How is memory accounting handled? What about mapping with the `MAP_FIXED` flag (used by a process that wants to tell the kernel where in its address space a mapping should be placed)? Wilcox answered that, for the most part, this mapping is handled in the same way as a mapping shared by threads within a single process. Aziz said that a worry of his own is that the shared area might be useful for processes trying to hide malware. Before getting into that sort of issue, though, he asked whether the `mshare()` concept seems useful in general. 

Mike Rapoport asked why the `SCM_RIGHTS` mechanism, which allows passing file descriptors over a Unix-domain socket, wasn't used to control access to the shared region. Wilcox answered that the first design for this feature did exactly that, but users were requesting the ability to open a file to access the area instead. John Hubbard said that the API looked elegant to him, and requested that the developers stick with it. 

Dan Williams asked how page pinning and accounting were being handled; Aziz replied that the work was mostly focused on the basic functionality so far. Making `get_user_pages()` and such work was on the list of things to do, though. David Hildenbrand echoed Hocko's sentiment that the feature seemed scary; he suggested making an allowlist describing the actions that were permitted on a shared area. System calls like [`mlock()`](<https://man7.org/linux/man-pages/man2/mlock.2.html>) would not be on that list, he suggested, until the implications were well understood. Page pinning, too, should not be there at the outset, he said. 

Wilcox said that the users driving this work want to use it with DAX (direct access to files stored in persistent memory). These users can have over 10,000 processes sharing the area, which causes the page-table overhead to exceed the amount of memory being shared. In a sense, he said, `mshare()` can be seen as giving DAX the same functionality as hugetlbfs, but nobody likes hugetlbfs, so the desire is to make something that is not so awful. Hocko suggested that the new API is "a different awful". 

Continuing, Wilcox said that, with `mshare()`, the kernel now has the concept of a standalone `mm_struct` with a file descriptor attached to it. What else, he asked, could be done with that functionality? Perhaps there would be value in a more general system call that would create an `mm_struct` and allow processes to attach things to it. That would be an interesting concept, he said, but Hildenbrand suggested it would be something more like Frankenstein's monster. Wilcox responded that Frankenstein would have loved this idea; he was "a misunderstood genius, just like us". 

#### API alternatives

Hubbard suggested that perhaps a different model would make more sense; it could be called a "lightweight process" (or just a "Frankenstein"). These new processes would have a set of rules describing their behavior. But Hocko said that he couldn't understand the consequences of such a feature; they would be "beyond imagination", he said. He asked why processes can't just share page tables on a per-mapping basis, using a feature that looks like hugetlbfs but in a more shareable way. Wilcox answered that "the customer" wants the described semantics where, for example, `mprotect()` applies across all processes, just as if they were threads sharing that part of the address space. That raises an obvious question, he said: why not just use threads? The answer was that "`mmap_lock` sucks". It is also not possible to change the existing behavior of `MAP_SHARED`, since that would break programs, so there would need to be, at a minimum, a new `mmap()` flag if not a new system call. Aziz said that the separate system call makes the page-table sharing explicit rather than it just being a side effect. That makes the decision to opt into this behavior explicit as well. 

Liam Howlett asked how many `mshare()` regions are supported in any given process; Wilcox answered that there is no particular limit. A process can create as many files as it wants, but he does not expect the API to be used that way. A more typical pattern would be for processes to share a single large chunk of memory, then perhaps map pieces of it. Howlett responded that, in that case, it might be better to only allow a single region per process. That might simplify the impact on other parts of the memory-management subsystem. 

Jason Gunthorpe said that, rather than using a separate `mm_struct`, a process could (via some mechanism) just instantiate a VMA mapped at a high level in the page-table hierarchy. The associated memory would be owned by that VMA (or the inode of a file backing it), and the reference counting could be done there. Hocko noted that this is how hugetlbfs works now. Wilcox answered that an explicit opt-in from the processes involved is still needed, since developers need to understand the changed semantics of system calls like `mprotect()`. Gunthorpe suggested a new `mmap()` flag. Aziz said that an approach like this was possible, but that the use of a separate `mm_struct` has the advantage of simplifying the use of existing mechanisms for working with page tables. 

Wilcox started to wind down the session by saying that, if the memory-management developers found this idea too scary, something else could be done. Aziz said that he was about to send the next version of the patch set (which hasn't happened as of this writing) and he would see what the feedback is at that point. 

As things were coming to a close, Jan Kara jumped in to say that the `mmap_lock` for the shared region will have the same contention problems as it does now. Wilcox said that he knew somebody would bring that up; to an extent, that problem does exist. But `mshare()` allows processes to have more than one memory region and separate private memory from shared memory. The effect, he said, is like splitting `mmap_lock` in half. But even separating out 20% of the contention, he said, would be an improvement. Kara asked whether it might be better, instead, to give threads a way to separate their _private_ address space. Wilcox said that he had thought the same way a year ago, but the result in the end is about the same. Kara said that the concept might be easier for developers to grasp. 

At that point the session came to an end for real. The next step will be further discussion on the mailing list once the updated patch set comes out.
