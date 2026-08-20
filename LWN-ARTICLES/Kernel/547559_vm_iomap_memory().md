---
title: vm_iomap_memory()
url: https://lwn.net/Articles/547559/
date: "April 17, 2013"
category: "Device drivers-Support APIs"
author: "By Jonathan Corbet April 17, 2013"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 17, 2013

The -rc7 stage of the kernel development cycle is normally considered to be the wrong time to add new driver API functions; most developers attempting such a thing could expect to get a response that is less than 100% encouraging. But, if you're Linus, you can get away with such things. So, after looking at too much messy driver code, Linus [added a new helper function](<https://lwn.net/Articles/547563/>) for drivers that need to map a physical memory range into a user-space process's memory: 

```
int vm_iomap_memory(struct vm_area_struct *vma, phys_addr_t start, 
    			unsigned long len);
```

This function is a wrapper around `io_remap_pfn_range()` that takes care of a number of details specific to the virtual memory subsystem so that driver code need not worry about them. No drivers have been converted for 3.9, but one can expect that process to begin in the 3.10 cycle.
