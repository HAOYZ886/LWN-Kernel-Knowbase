---
title: Memory management notifiers
url: https://lwn.net/Articles/266320/
date: "January 23, 2008"
category: "KVM; Memory management-Virtualization"
author: "By Jonathan Corbet January 23, 2008"
---

By **Jonathan Corbet**  
January 23, 2008

Virtualized guests running under Linux like to think that they are doing their own memory management. The truth of the matter, though, is that the host system cannot allow guests to directly modify the page tables used by the hardware; allowing that sort of access would compromise the security of the host. So, somehow, the host must be involved in the guest's memory management. One common technique is through the use of shadow page tables. Guest systems maintain their own page tables, but they are not the tables used by the memory management unit. Instead, whenever the guest makes a change to its tables, the host system intercepts the operation, checks it for validity, then mirrors the change in the real page tables, which "shadow" those maintained by the guest. 

One problem with this technique, as implemented in Linux currently, is that there is no easy way for the host to feed page table changes back to the guest. In particular, if the host system decides that it wants to push a given page out to swap, it can't tell the guest that the page is no longer resident. So virtualization mechanisms like KVM avoid the problem altogether by pinning pages in memory when they are mapped in shadow page tables. That solves the problem, but it makes it impossible to swap processes running KVM-based virtual machines out of main memory. 

This seems like a good thing to fix. And a fix exists, in the form of the [MMU notifiers patch](<http://lwn.net/Articles/266189/>) posted by Andrea Arcangeli (from his shiny new Qumranet address). This patch allows an interested subsystem to be notified whenever specific memory management events take place. The process starts by setting up a set of callbacks: 

```
struct mmu_notifier_ops {
    	void (*release)(struct mmu_notifier *mn,
    			struct mm_struct *mm);
    	int (*age_page)(struct mmu_notifier *mn,
    			struct mm_struct *mm,
    			unsigned long address);
    	void (*invalidate_page)(struct mmu_notifier *mn,
    				struct mm_struct *mm,
    				unsigned long address);
    	void (*invalidate_range)(struct mmu_notifier *mn,
    				 struct mm_struct *mm,
    				 unsigned long start, unsigned long end);
        };
```

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

These callbacks are bundled into an `mmu_notifier` structure: 

```
struct mmu_notifier {
    	struct hlist_node hlist;
    	const struct mmu_notifier_ops *ops;
        };
```

The interested code then registers its notifier with: 

```
void mmu_notifier_register(struct mmu_notifier *mn, 
                                   struct mm_struct *mm);
```

Here, `mm` is the `mm_struct` structure associated with a given address space. It is not expected that anybody will be interested in _all_ memory management events, so notifiers are associated with specific address spaces. Once the notifier is in place, the callbacks will be invoked when interesting things happen: 

  * `release()` is called when the relevant `mm_struct` is about to go away. So it will be the last callback made to that notifier. 

  * `age_page()` indicates that the memory management subsystem wants to clear the "referenced" flag on the page associated with the given `address`. This callback should return the previous value of the referenced bit, or the closest approximation available on the host architecture. 

  * `invalidate_page()` and `invalidate_range()` are both ways of telling the guest that the given address(es) are no longer valid - the page has been reclaimed. Upon return from this callback, the affected address range should not be referenced by the guest. 

For the curious, the [KVM patches](<http://marc.info/?l=kvm-devel&m=120103225508669&w=2>) (showing how these notifiers are used there) have also been posted. 

While this patch set is aimed at KVM, there has been some interest from other directions as well - virtual machines are not the only places where separate (but related) page tables are maintained. Graphical processing units on contemporary video cards are an example - they have their own memory management units and have some interesting [management issues](<http://lwn.net/Articles/257417/>) of their own. Remote DMA (RDMA) engines are another possible user. So these patches have attracted comments from a few potential users, and have changed significantly since their first posting. The discussion is still ongoing, so further changes may come about before the notifiers find their way into the mainline.
