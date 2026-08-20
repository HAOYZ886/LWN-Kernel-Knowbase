---
title: "User-space page fault handling"
url: https://lwn.net/Articles/550555/
date: "May 14, 2013"
category: "Memory management-Virtualization; remap anon pages"
author: "By Jonathan Corbet May 14, 2013"
---

By **Jonathan Corbet**  
May 14, 2013

Page fault handling is normally the kernel's responsibility. When a process attempts to access an address that is not currently mapped to a location in RAM, the kernel responds by mapping a page to that location and, if needed, filling that page with data from secondary storage. But what if that data is not in a location that is easily reachable by the kernel? Then, perhaps, it's time to outsource the responsibility for handling the fault to user space. 

One situation where user-space page fault handling can be useful is for the live migration of virtual machines from one physical host to another. Migration can be done by stopping the machine, copying its full address space to the new host, and restarting the machine there. But address spaces may be large and sparsely used; copying a full address space can result in a lot of unnecessary work and a noticeable pause before the migrated system restarts. If, instead, the virtual machine's address space could be demand-paged from the old host to the new, it could restart more quickly and the copying of unused data could be avoided. 

Live migration with KVM is currently managed with [an out-of-tree char device](<http://lists.gnu.org/archive/html/qemu-devel/2012-10/msg05274.html>). This scheme works, but, once the device takes over a range of memory, that memory is removed from the memory management subsystem. So it cannot be swapped out, transparent huge pages don't work, and so on. Clearly it would be better to come up with a solution that, while allowing user-space handling of demand paging, does not remove the affected memory from the kernel's management altogether. A [patch set](<https://lwn.net/Articles/549503/>) recently posted by Andrea Arcangeli aims to resolve those issues with a couple of new system call options. 

The first of those is to extend the `madvise()` system call, adding a new command called `MADV_USERFAULT`. Processes can use this operation to tell the kernel that user space will handle page faults on a range of memory. After this call, any access to an unmapped address in the given range will result in a `SIGBUS` signal; the process is then expected to respond by mapping a real page into the unmapped space as described below. The `madvise(MADV_USERFAULT)` call should be made immediately after the memory range is created; user-space fault handling will not work if the kernel handles any page faults before it is told that user space will be doing the job. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The `SIGBUS` signal handler's job is to handle the page fault by mapping a real page to the faulting address. That can be done in current kernels with the `mremap()` system call. The problem with `mremap()` is that it works by splitting the virtual memory area (VMA) structure used to describe the memory range within the kernel. Frequent `mremap()` calls will result in the kernel having to manage a large number of VMAs, which is an expensive proposition. `mremap()` will also happily overwrite existing memory mappings, making it harder to detect errors (or race conditions) in user-space handlers. For these reasons, `mremap()` is not an ideal solution to the problem. 

Andrea's answer to this problem is a new system call: 

```
int remap_anon_pages(void *dest, void *source, unsigned long len);
```

This call will cause the `len` bytes of memory starting at `source` to be mapped into the process's address space starting at `dest`. At the same time, the `source` memory range will be unmapped — the pages previously found there will be atomically moved to the `dest` range. 

Andrea has posted [a small test program](<https://lwn.net/Articles/550622/>) that demonstrates how these APIs are meant to be used. 

As one might expect, some restrictions apply: `source` and `dest` must be page-aligned, `len` should be a multiple of the page size, the `dest` range must be completely unmapped, and the `source` range must be fully mapped. The mapping requirements exist to catch bugs in user-space fault handlers; remapping pages on top of existing memory has a high risk of causing memory corruption. 

One nice feature of the patch set is that, on systems where [transparent huge pages](<https://lwn.net/Articles/423584/>) are enabled, huge pages can be remapped with `remap_anon_pages()` without the need to split them apart. For that to work, of course, the length and alignment of the range to move must be compatible with huge pages. 

There are a number of limitations in the current patch set. The `MADV_USERFAULT` option can only be used on anonymous (swap-backed) memory areas. A more complete implementation could conceivably support this feature for file-backed pages as well. The mechanism offers support for demand paging of data into RAM, but there is no user-controllable mechanism for pushing data back out; instead, those pages are swapped with all other anonymous pages. So it is not a complete user-space paging mechanism; it's more of a hook for loading the initial contents of anonymous pages from an outside source. 

But, even with those limitations, the feature is useful for the intended virtualization use case. Andrea suggests it could possibly have other uses as well; remote RAM applications come to mind. First, though, it needs to get into the mainline, and that, in turn, suggests that the proposed ABI needs to be reviewed carefully. Thus far, this patch set has not gotten a lot of review attention; that will need to change before it can be considered for mainline merging.
