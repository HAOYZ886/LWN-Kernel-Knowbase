---
title: A summary of 2.6.23 internal API changes
url: https://lwn.net/Articles/250210/
date: "September 17, 2007"
category: "Releases-2.6.23"
author: "By Jonathan Corbet September 17, 2007"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 17, 2007

The final 2.6.23 kernel release is getting closer. At this point, it would be more than surprising to see any additional API changes find their way into this release, so it should be safe to summarize the changes which have been made. 

  * The [UIO](<http://lwn.net/Articles/232575/>) interface for the creation of user-space drivers has been merged. While UIO is aimed at user space, there is a kernel-space component for driver registration and interrupt handling. 

  * `unregister_chrdev()` now returns `void`. 

  * There is a new notifier chain which can be used (by calling `register_pm_notifier()`) to obtain notification before and after suspend and hibernate operations. 

  * The new "lockstat" infrastructure provides statistics on the amount of time threads spend waiting for and holding locks. 

  * The new `fault()` VMA operation replaces `nopage()` and `populate()`. See [this article](<https://lwn.net/Articles/242625/>) for a description of the current `fault()` API. 

  * The generic netlink API now has the ability to register (and unregister) multicast groups on the fly. 

  * The destructor argument has been removed from `kmem_cache_create()`, as destructors are no longer supported. All in-kernel callers have been updated. 

  * There is a new `clone()` flag - `CLONE_NEWUSER` \- which creates a new user namespace for the process; it is intended for use with container systems. 

  * There is a new rtnetlink API for managing software network devices. 

  * The networking core can now work with devices which have more than one transmit queue. This is a feature which was needed to properly support some wireless devices. 

  * The sysfs core has been significantly rewritten to weaken the connection between sysfs entries and internal kobjects. The new code should make life easier for driver writers who will have fewer object lifecycle issues to worry about. 

  * The never-used `enable_wake()` PCI driver method has been removed. 

  * Drivers wanting to get the revision ID from the PCI config space should now just use the value found in the new `revision` member of the `pci_dev` structure. All in-tree drivers have been changed to use this new approach. 

  * The SCSI layer has picked up a couple of scatter/gather accessor functions - `scsi_dma_map()` and `scsi_dma_unmap()` \- in preparation for chained scatter/gather lists and bidirectional requests. Most drivers in the kernel have been updated to use these functions. 

  * The [idr](<http://lwn.net/Articles/103209/>) code has a couple of new helper functions: `idr_for_each()` and `idr_remove_all()`. 

  * `sys_ioctl()` is no longer exported to modules. 

  * The page table helper functions `ptep_establish()`, `ptep_test_and_clear_dirty()` and `ptep_clear_flush_dirty()` have been removed - they had no in-kernel users. 

  * Kernel threads are non-freezable by default; any kernel thread which should be frozen for a suspend-to-disk operation must now call `set_freezable()` to arrange for that to happen. 

  * The SLUB allocator is now the default. 

  * The new function `is_owner_or_cap(inode)` tests for access permission based on the current fsuid and capabilities; it replaces the open-coded test previously found in several filesystems. 

  * There is a new utility function: 

```
char *kstrndup(const char *s, size_t max, gfp_t gfp);
```

This function duplicates a string along the lines of the user-space `strndup()`. 

As always, a cumulative record of API changes can be found in the [LWN 2.6 API changes page](<https://lwn.net/Articles/2.6-kernel-api/>).
