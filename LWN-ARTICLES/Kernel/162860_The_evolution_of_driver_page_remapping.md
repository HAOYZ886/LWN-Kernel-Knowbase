---
title: The evolution of driver page remapping
url: https://lwn.net/Articles/162860/
date: "December 6, 2005"
category: "Memory management-Internal API; VM UNPAGED; vm insert page"
author: 
---

Two weeks ago, this page [looked at the new `VM_UNPAGED` flag](<http://lwn.net/Articles/161204/>), introduced in 2.6.15-rc2 to mark virtual memory areas (VMAs) which are not made up of "normal" pages. These areas are usually created by device drivers which map special memory areas (which may or may not be device I/O memory) into user space. Your editor now humbly suggests that readers ignore that article; things have changed significantly since then. 

As it turns out, Linus didn't like the `VM_UNPAGED` idea, so he rewrote the code for 2.6.15-rc4. The `VM_UNPAGED` VMA flag is gone, replaced by `VM_PFNMAP`. The new flag has a very similar meaning: it marks the VMA as containing special page table entries which should not be touched by the VM subsystem. In particular, it states that there is no `page` structure associated with any page in that VMA, so the VM subsystem should not go looking for one. Even in cases where that structure does exist (such as remappings of real memory), the VM code will pretend that it does not. 

The advantage of the reworked code is that it takes out a number of special cases; the `VM_PFNMAP` VMAs can be treated just like normal VMAs in more places. Things quickly got a bit more complicated, however. The initial `VM_PFNMAP` code assumed that a linear range of addresses was being mapped into user space. In fact, some drivers piece together memory in more complicated ways. 

So a subsequent patch added explicit support for "incomplete" VMAs, marked with yet another flag: `VM_INCOMPLETE`. When the kernel detects that a driver is creating something other than a straightforward, linear mapping, it sets that flag and emits a warning. It also requires, in this case, that the pages being remapped carry the `PG_reserved` flag - even though this flag is being phased out. Remapping RAM in this way always required that flag in the past, so this requirement is not a change as far as drivers are concerned. 

The patch adding `VM_INCOMPLETE` notes that ""In the long run we almost certainly want to export a totally different interface for that, though."" In this case, "in the long run" meant about one day, when yet another patch was merged adding a new function: 

```
int vm_insert_page(struct vm_area_struct *vma, 
                           unsigned long address,
                           struct page *page);
```

This function inserts the given `page` into `vma`, mapped at the given `address`. It does not put out warnings, and does not require that `PG_reserved` be set. What it _does_ require is that the page be an order-zero allocation obtained for this purpose; it is not possible to remap arbitrary RAM pages with `vm_insert_page()`. Since a `page` structure is required, the new function is also unsuitable for remapping I/O memory. But it is useful for drivers which wish to map a set of pages into a user-space address range. 

Just which driver might want to do something like that became clear when another patch was merged for 2.6.15-rc5. It removed the GPL-only export for `vm_insert_page()` and included this commit message: 

Make vm_insert_page() available to NVidia module. It used to use remap_pfn_range(), which wasn't GPL-only either, and the new interface is actually simpler and does more checking, so we shouldn't unnecessarily discourage people from switching over. 

Some developers objected to this change, seeing it as an explicit endorsement of the proprietary NVidia drivers. Others, however, saw it as a simple attempt to avoid breaking drivers without a good reason. The kernel developers may well be working toward taking a stronger stand against proprietary modules, but this particular interface will not be the place where that battle is fought.
