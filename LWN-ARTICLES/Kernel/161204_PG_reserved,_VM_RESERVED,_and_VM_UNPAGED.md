---
title: "PG_reserved, VM_RESERVED, and VM_UNPAGED"
url: https://lwn.net/Articles/161204/
date: "November 22, 2005"
category: "Memory management-Internal API; VM UNPAGED; remap pfn range"
author: 
---

The `page` structure, used to describe the memory in the system, includes a set of flags; one of those flags is `PG_reserved`. For a long time, this bit has marked pages which are not part of the regular memory management regime; pages so marked include the kernel text (which really should not be swapped out) and the I/O memory in the legacy ISA hole at 640K. Occasionally, device drivers have explicitly set the reserved bit on ordinary memory so that it could be mapped into user space with `remap_pfn_range()`. This technique has been discouraged for years, but still persists in spots. 

The 2.6.15 kernel removes, for all practical purposes, the reserved bit. Space for page flags is tight, and it was figured that, in 2.6, this bit was no longer needed. The page reclaim code no longer cycles through the system memory map, so it does not need this bit to know which pages to avoid. For the other uses, the `VM_RESERVED` bit in the `vm_area` structure could be used instead. So, in 2.6.15-rc2, the `PG_reserved` bit is (almost) ignored, and the kernel respects `VM_RESERVED` by not freeing pages found in areas with that bit set. 

Unfortunately, it seems a number of drivers set `VM_RESERVED` for all VMAs which are mapped into user space. Some of these areas are actually normal memory pages, which the driver maps into the process's address space one-by-one when its `nopage()` function is called. Hugh Dickins noticed that, in this case, those pages will never be returned to the system, since the `VM_RESERVED` flag prevents them from being freed. The right fix for the problem is probably to get rid of `VM_RESERVED` altogether; its use is mostly a legacy from the 2.4 days. But going into a bunch of drivers and tweaking their memory management code when this kernel is already at a -rc2 release looks like a certain way to introduce obscure bugs. So Hugh decided to go in and make fundamental changes to the low-level memory management code instead. 

The result is a new VMA flag, `VM_UNPAGED`. This flag says, explicitly, that the pages in this VMA are not to be managed, and in particular, should not be freed. It essentially takes over the meaning previously held by `VM_RESERVED`, but in an arguably better-defined manner. Calls to `remap_pfn_range()` will cause the `VM_UNPAGED` flag to be set. But areas of RAM managed by a driver `nopage()` function will not have `VM_UNPAGED` set, so their memory will be managed normally. 

Various other subtleties, such as what happens when a process with `VM_UNPAGED` VMAs forks, had to be dealt with. But the end result of all this work should be that things function again, with no driver changes. At some point, the use of `VM_RESERVED` in drivers may be taken out, but that's a post-2.6.15 thing. 

Meanwhile, one other interesting result of the `PG_reserved` removal is that `remap_page_range()` can now be used to remap any set of addresses, not just those marked reserved.
