---
title: io_remap_pfn_range()
url: https://lwn.net/Articles/129480/
date: "March 29, 2005"
category: io remap pfn range
author: 
---

`io_remap_page_range()` has always been a strange function. Its stated purpose is to portably map I/O memory into a process's address space. Its prototype has always differed from one system to the next, however, making portable use difficult. On most architectures it looks like this: 

```
int io_remap_page_range(struct vm_area_struct *vma, unsigned long virt_addr,
                                unsigned long phys_addr, unsigned long size, 
                                pgprot_t prot);
```

The sparc64 architecture, however, defines it this way: 

```
int io_remap_page_range(struct vm_area_struct *vma, unsigned long virt_addr,
                                unsigned long phys_addr, unsigned long size, 
                                pgprot_t prot, int space);
```

The extra argument (`space`) was necessary to deal with the inconvenient fact that I/O addresses on the sparc64 architecture would not fit into an `unsigned long` variable. 

The [change](<https://lwn.net/Articles/104333/>) from `remap_page_range()` to `remap_pfn_range()` was done, in part, to address (so to speak) this issue. Since remapping must be done on a page-aligned basis anyway, there is no real point in using a regular physical address, which contains the offset within the page. Said offset, after all, must be zero. By using a page frame number instead, the range of the `phys_addr` argument is extended far enough to reach into I/O memory on all architectures. The `remap_pfn_range()` work stopped short of actually fixing the `io_remap_page_range()` problem, however. 

Randy Dunlap has now finished the task with a set of patches adding `io_remap_pfn_range()`: 

```
int io_remap_pfn_range(struct vm_area_struct *vma, unsigned long from,
                               unsigned long pfn, unsigned long size, 
                               pgprot_t prot);
```

This function has the same prototype on all architectures. In-tree callers have been modified, and the feature removal schedule has been updated: `io_remap_page_range()` will go away in September, 2005.
