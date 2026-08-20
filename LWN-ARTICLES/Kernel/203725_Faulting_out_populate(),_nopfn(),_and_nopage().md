---
title: "Faulting out populate(), nopfn(), and nopage()"
url: https://lwn.net/Articles/203725/
date: "October 10, 2006"
category: "Memory management-Internal API; fault; nopage; nopfn; populate"
author: 
---

The `nopfn()` VMA operation was added for 2.6.19-rc1; see [this article from last month](<http://lwn.net/Articles/200213/>) for information on this method. It turns out, though, that `nopfn()` might just be one of the shortest-lived kernel API extensions in some time; Nick Piggin has posted [a series of patches](<http://lwn.net/Articles/203689/>) which will bring significant changes to how page faults are handled at the lowest levels. 

The 2.6.19-rc1 `vm_operations_struct` structure defines three methods which handle low-level paging: 

```
struct page *(*nopage)(struct vm_area_struct *area, 
                                   unsigned long address, int *type);
    	unsigned long (*nopfn)(struct vm_area_struct *area, 
                                   unsigned long address);
    	int (*populate)(struct vm_area_struct *area, unsigned long address, 
                            unsigned long len, pgprot_t prot, 
    			unsigned long pgoff, int nonblock);
```

Ordinarily, page faults are handled by `nopfn()` (if it exists) or `nopage()`. Those functions are supposed to take the given `address` and associate it with a page in physical memory. For virtual memory areas (VMAs) which are backed up by files, the virtual filesystem layer reacts to a `nopage()` call by allocating a page of memory, reading the appropriate contents from backing store, then passing the page back to the kernel for insertion into the page tables. Device drivers which implement `nopage()` typically just translate the `address` into an appropriate pointer for an in-memory buffer being mapped into user space. 

Both `nopfn()` and `nopage()` assume that the mapping between virtual memory addresses and the offset within the VMA is linear - that is why only the address is provided as a parameter. The kernel, however, also supports [nonlinear mappings](<http://lwn.net/Articles/24468/>), where an application can turn a VMA into a complex window into different parts of the backing file. The `nopfn()` and `nopage()` methods cannot handle these mappings, since they do not have the required information. Instead, any backing store which supports nonlinear mappings must provide a `populate()` method, which has parameters for both the virtual memory `address` and the associated offset (`pgoff`) into the backing store device. 

Enter Nick, who was working on a tricky race condition found within one of the most notoriously tricky parts of the kernel: the code which handles file truncation. In some conditions, a page which was being removed as a result of a `truncate()` call could be simultaneously faulted in via `nopage()`, leading to memory management confusion. While rethinking the locking rules for these operations, Nick decided that there should be a better way. The result was a new VMA operation called `fault()`: 

```
struct fault_data {
    	struct vm_area_struct *vma;
    	unsigned long address;
    	pgoff_t pgoff;
    	unsigned int flags;
    
    	int type;
        };
    
        struct page *(*fault)(struct vm_area_struct *vma, 
    			  struct fault_data *fdata);
```

This method is intended to replace all of `nopfn()`, `nopage()`, and `populate()`. When a page fault happens, the kernel fills in the `fault_data` structure with the needed information: the user-space `address` associated with the fault, the corresponding offset `pgoff`, and a couple of flags which indicate whether the fault happened on a write access and whether a nonlinear mapping is involved. 

The `fault()` function should locate a page which can satisfy a request for the offset `pgoff`; it won't normally need `address` at all. The function can then either return the associated `struct page`, or set the page table entry directly (with something like `vm_insert_page()`) and return `NULL`. Either way, the `type` field should be set to the type of fault (major or minor). If the fault cannot be handled, the appropriate error code should be put into `type` instead. 

Nick's patch gets rid of the `nopfn()` and `populate()` methods immediately. There is currently only one user of `nopfn()`, and the older `populate()` API has never been widely used outside of the mainline kernel. The `install_page()` function is also destined for a near-term demise. The `nopage()` method, instead, is widely used by device drivers, inside and outside of the mainline. So it has been marked as deprecated and scheduled for removal one year from now, in October, 2007. There have been suggestions that `nopage()` should go sooner (after six months, say), but no definitive decision. 

Details like that aside, there appears to be broad support for this change. These patches would probably be a bit too new for 2.6.19, even if the merge window were still open, so 2.6.20 is the earliest likely date for them to appear in the mainline. But, at that point, driver and out-of-tree filesystem maintainers will have some updating to do.
