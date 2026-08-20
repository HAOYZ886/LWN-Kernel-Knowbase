---
title: Sharing page tables with msharefs
url: https://lwn.net/Articles/901059/
date: "July 15, 2022"
category: "Memory management-Page-table sharing"
author: "By Jonathan Corbet July 15, 2022"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
July 15, 2022

A page-table entry (PTE) is relatively small, requiring just eight bytes to refer to a 4096-byte page on most systems. It thus does not seem like a worrisome level of overhead, and little effort has been made over the kernel's history to reduce page-table memory consumption. Those eight bytes can hurt, though, if they are replicated across a sufficiently large set of processes. The [msharefs patch set](<https://lwn.net/ml/linux-mm/cover.1656531090.git.khalid.aziz@oracle.com/>) from Khalid Aziz is a revised attempt to address that problem, but it is proving to be a hard sell in the memory-management community. 

One of the defining characteristics of a process on Linux (or most other operating systems) is a distinct address space. As a result, the page tables that manage the state of that address space are private to each process (though threads within a process will share page tables). So if two processes have mappings to the same page in physical memory, each will have an independent page-table entry for that page. The overhead for PTEs, thus, increases linearly with the number of processes mapping each page. 

Even so, this cost is not normally problematic, but there is always somebody out there doing outlandish things. As described in the cover letter from the patch series: 

> On a database server with 300GB SGA [[Oracle system global area](<https://docs.oracle.com/database/121/ADMQS/GUID-A3319550-AB7A-4429-9A58-4B90E4B3D0F5.htm>)], a system crash was seen with out-of-memory condition when 1500+ clients tried to share this SGA even though the system had 512GB of memory. On this server, in the worst case scenario of all 1500 processes mapping every page from SGA would have required 878GB+ for just the PTEs. If these PTEs could be shared, the amount of memory saved is very significant. 

Sharing those PTEs is the objective of this work, which was [discussed](<https://lwn.net/Articles/895217/>) at the Linux Storage, Filesystem, Memory-Management, and BPF Summit in May. At that time, Aziz was proposing a new system call (`mshare()`) to manage this sharing. The current patch set has changed this interface and now requires no new system calls at all. 

Even without the system call, it is still necessary for processes to explicitly request the sharing of page tables for a range of memory. The current patch set provides yet another kernel virtual filesystem — msharefs — for that purpose; it is expected to be mounted on `/sys/fs/mshare`. The file `mshare_info` in that filesystem will, when read, provide the minimum alignment required for a memory region to be able to share page tables. 

The next step is to create a file under `/sys/fs/mshare` with a name that means something to the application. Then, an [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) call should be used to map that file into the process's address space. The size passed to `mmap()` will determine the size of the resulting shared region of memory. Your editor's reading of the code suggests that providing an explicit address for the mapping is advisable; there does not appear to be any mechanism to automatically pick an address that meets the alignment requirements. Once the region has been mapped, it can be used just like any other memory range. 

The purpose of creating such a region is to allow other processes to map it as well. Any other processes will need to start by opening the msharefs file created by the first process, then reading a structure of this type from it: 

```
struct mshare_info {
    	unsigned long start;
    	unsigned long size;
        };
```

The `start` and `size` fields provide the address at which the region is mapped and its size, respectively; the new process should pass those values (and the opened msharefs file) to its own `mmap()` call to map the shared region. After that, the region will be mapped just like any other shared-memory area — with a couple of important exceptions, as will be described below. 

A process's address space is described by [`struct mm_struct`](<https://elixir.bootlin.com/linux/v5.18.11/source/include/linux/mm_types.h#L476>); there is one such structure for each process (other than kernel threads) in the system. The msharefs patch set changes the longstanding one-to-one relationship between this structure and its owning process by creating a new `mm_struct` structure for each shared region. The page tables describing this region belong to this separate structure, rather than to any process's `mm_struct`. Whenever a process maps this region, the associated [`vm_area_struct`](<https://elixir.bootlin.com/linux/v5.18.11/source/include/linux/mm_types.h#L393>) (VMA) will contain a pointer to this special `mm_struct`. The end result is that all processes mapping this area will share not just the memory, but also the page tables that go along with it. 

That saves the memory that would have gone into duplicate page tables, of course, but it also has a couple of other, possibly surprising, results. For example, changing the protection of memory within that region with [`mprotect()`](<https://man7.org/linux/man-pages/man2/mprotect.2.html>) will affect all processes sharing the area; with ordinary shared memory, only the calling process will see protection changes. Similarly, the memory region can be remapped entirely with [`mremap()`](<https://man7.org/linux/man-pages/man2/mremap.2.html>) and all users will see the change. 

It appears that use of `mremap()` is actually part of the expected pattern for PTE-shared memory regions. The `mmap()` call that is required to create the region will populate that region with anonymous memory; there is no way to request that file-backed memory be used instead. But it _is_ possible to use `mremap()` to dump that initial mapping and substitute file-backed memory afterward. So applications wanting to use shared page tables with file-backed memory will have to perform this extra step to set things up correctly. 

The developers at the LSFMM session were clear that they found this whole concept to be somewhat frightening. So far, the reaction to this patch series has (from a memory-management point of view) been relatively subdued, with the exception of David Hildenbrand, who is [pushing](<https://lwn.net/ml/linux-mm/397f3cb2-1351-afcf-cd87-e8f9fb482059@redhat.com/>) for a different sort of solution. He would rather see a mechanism that would automatically share page tables when mappings are shared, without requiring application-level changes. That would make the benefits of sharing more widely available while exposing fewer internal memory-management details. 

Automatic sharing would need to have different semantics, though; otherwise applications will be surprised when an `mprotect()` or `mremap()` call in another process changes their mappings. Though it was not stated in this version of Aziz's patch posting, the sense from the LSFMM session was that the altered semantics were desirable. If that is the case, fully automatic sharing will not be possible, since applications would have to opt in to that behavior. 

Either way, it looks like this particular patch set needs more work and discussion before it can find its way into the mainline. Until then, applications depending on sharing memory between large numbers of processes will continue to pay a high page-table cost.
