---
title: remap_pfn_range()
url: https://lwn.net/Articles/104333/
date: "September 29, 2004"
category: "Memory management-Internal API; remap pfn range"
author: 
---

[Last month](<https://lwn.net/Articles/99120/>) we looked at a possible change to the heavily-used `remap_page_range()` function as a way of making `io_remap_page_range()` be the same on all architectures. Since then, a driver author has stepped forward with a different problem: he wants to remap some reserved memory which sits above the 4GB memory boundary. Since `remap_page_range()` takes a 32-bit "start" address, it cannot be used to remap memory above that boundary. 

In response, William Lee Irwin has posted [a series of patches](<https://lwn.net/Articles/103654/>) which changes `remap_page_range()` to: 

```
int remap_pfn_range(struct vm_area_struct *vma, unsigned long from,
                            unsigned long pfn, unsigned long size,
                            pgprot_t prot);
```

The old "start" address has been changed to `pfn`, which is a page frame number. Since these mappings can only happen on page boundaries, this change does not take away any old functionality. It does, however, make twelve bits (typically) of address space available, making it possible to remap memory well above 4GB. 

William's patches fix all in-kernel callers of `remap_page_range()`, of which there are several dozen, and removes the old interface altogether. He also manages to eliminate a fair amount of related code - it seems that large numbers of drivers have their own, private copy of `kvirt_to_pa()`, which obtains a physical address for memory from `vmalloc()`. For in-kernel users, the change should be a purely positive thing. Out-of-kernel drivers which use `remap_page_range()` will have to be fixed, however. 

These patches have found their way into the -mm tree, and are thus likely to end up in the mainline eventually.
