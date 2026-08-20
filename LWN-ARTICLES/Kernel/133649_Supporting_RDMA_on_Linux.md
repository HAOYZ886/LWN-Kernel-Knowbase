---
title: Supporting RDMA on Linux
url: https://lwn.net/Articles/133649/
date: "April 27, 2005"
category: Memory management; RDMA
author: 
---

RDMA (remote direct memory access) is an attempt to extend the DMA mechanism to a networked environment. Using RDMA, an application can quickly transfer the contents of a memory buffer to a buffer on a remote system. On high-speed, local-area networks, RDMA transfers are intended to be significantly faster than transfers done with the regular socket interface. Not everybody likes the RDMA way of doing things, but it exists regardless, and some users expect to see it supported by Linux. Implementations exist for InfiniBand and a number of high-speed Ethernet adaptors. 

Since the goals of RDMA include speed and low CPU overhead, implementations attempt to bypass as much kernel processing as possible. Typically, they simply pass the address of a user-space buffer directly to the hardware, and expect that hardware to do the rest. Drivers which need to make user-space memory available to their hardware will call `get_user_pages()`, which achieves two useful things: it pins the pages into physical memory, and generates an array of physical addresses for the driver to use. The current RDMA implementations use this approach, but they have run into a problem: `get_user_pages()` was never designed for the usage patterns seen with RDMA. 

The typical driver which calls `get_user_pages()` keeps the pages pinned for a very short period of time. Often, the pages will be released before the driver returns to user space. Sometimes, usually when asynchronous I/O is used, the release of the pages will be delayed for a short period, but only as long as it takes the I/O operation to complete. The problem is that RDMA operations do not "complete" in this manner. An RDMA user can reasonably set up a buffer, pass a descriptor to a remote system, and expect data to show up in the buffer sometime next week. The whole idea is to do the relatively expensive buffer setup once, then be able to transfer the (changing) contents of that buffer an arbitrary number of times. So pages pinned by the driver can remain pinned for a very long time. 

Several problems come up in this scenario. `get_user_pages()` does not do any sort of privilege checking or resource accounting for the pages it pins; it's supposed to be a short-term operation. So a hostile application could use an RDMA interface to lock down large amounts of memory indefinitely, effectively shutting down the system. There is no mechanism for notifying the driver if the process owning the pages exits, so cleanup can be a problem. There are also interactions with the virtual memory system to worry about: if the process forks (causing its data pages to be marked copy-on-write) and writes to a pinned page, it will get a new copy of that page and will become disconnected from its pinned buffer. 

Various approaches to solving these problems have been discussed. The resource accounting issues can be partially solved by requiring the process to lock the pages itself (using `mlock()`) before setting them up for RDMA; that will bring the normal kernel resource limits into play. There are still potential problems if the process is allowed to unlock the pages while the RDMA buffer still exists, however, so some changes would have to be made to prevent that case. Current implementations have dealt with the process exit issue by setting up a char device as the control interface for the RDMA buffer; when the device is closed, all RDMA structures are torn down. The copy-on-write problem can be addressed by forcing RDMA buffers to be in their own virtual memory area (VMA) and setting the `VM_DONTCOPY` flag on that VMA, preventing the pages from being made available to any child processes. This approach would require that RDMA buffers occupy whole pages by themselves. Then there are little issues like what happens when the process creates overlapping RDMA buffers. The whole thing gets a little complicated. 

All of this can clearly be patched together, but it is inelegant at best, and is clearly getting complicated. So [an entirely different approach](<https://lwn.net/Articles/133627/>) has been proposed by David Addison. This technique does away with the need to pin RDMA buffers entirely, but would, instead, require network drivers to become rather more aware of how the virtual memory subsystem works. 

David's patch assumes that the network interface device contains a simple memory management unit of its own, and can deal with its own paging details. This assumption turns out to be true for a number of contemporary high-speed cards. These cards can translate addresses and properly ask for help if they need to access a page which is not currently resident in memory. Thus, when using this sort of card, RDMA buffers can be set up without the need to pin them in memory; the hardware will cause them to be faulted in when the time comes. 

Needless to say, the hardware will need a considerable amount of help in this process; it cannot be expected to work with the host system's page tables, cause page faults to happen on its own, etc. So the card's MMU must be loaded with a minimal set of page mappings which describe the RDMA buffer(s), and those mappings must be kept in sync as things change on the system. With that in place, the card can perform DMA to resident pages, and ask the driver for help with the rest. 

The device driver can load the initial page tables, but it will need help from the kernel to know when the host system's page tables change. To that end, David's patch defines a structure with a new set of hooks into the virtual memory subsystem: 

```
typedef struct ioproc_ops {
        struct ioproc_ops *next;
        void *arg;
    
        void (*release)(void *arg, struct mm_struct *mm);
        void (*sync_range)(void *arg, struct vm_area_struct *vma, 
                           unsigned long start, unsigned long end);
        void (*invalidate_range)(void *arg, struct vm_area_struct *vma, 
                                 unsigned long start, unsigned long end);
        void (*update_range)(void *arg, struct vm_area_struct *vma, 
                             unsigned long start, unsigned long end);
        void (*change_protection)(void *arg, struct vm_area_struct *vma, 
                                  unsigned long start, unsigned long end, 
                                  pgprot_t newprot);
        void (*sync_page)(void *arg, struct vm_area_struct *vma, 
                          unsigned long address);
        void (*invalidate_page)(void *arg, struct vm_area_struct *vma, 
                                unsigned long address);
        void (*update_page)(void *arg, struct vm_area_struct *vma, 
                            unsigned long address);
    } ioproc_ops_t;
```

An interested driver can fill in one of these structures with its methods, then attach it to a given process's `mm_struct` structure with a call to `ioproc_register_ops()`. Thereafter, calls to those functions will be made whenever things change. 

The `release()` method will be called when the process exits; it allows the driver to perform a full cleanup. The `sync_range()` and `sync_page()` methods indicate that the given page(s) have been flushed to disk; this tells the driver that, should the interface modify those pages, they must be marked dirty again. `invalidate_range()` and `invalidate_page()` inform the driver that the given page(s) are not longer valid - they have been swapped out or unmapped. Calls to `update_range()` and `update_page()` happen when a valid page table entry is written; when a page is brought in, mapped, etc. The `change_protection()` function is called when page protections are changed. 

The patch has already, apparently, been looked over by Andrew Morton and Andrea Arcangeli, so one might assume that there would not be a great many show stoppers there. The comments posted so far have had to do mostly with coding style, though one poster [noted](<https://lwn.net/Articles/133673/>) that it might make more sense to attach the hooks to the VMA structure, rather than the top-level memory management structure. Unfortunately, the patch does not include any code which actually _uses_ the proposed hooks, making it harder to see how a driver might employ them. Meanwhile, conversations continue on how an interface using page pinning could be made to work. A real solution may be some time yet in coming.
