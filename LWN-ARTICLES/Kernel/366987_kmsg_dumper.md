---
title: kmsg_dumper
url: https://lwn.net/Articles/366987/
date: "December 16, 2009"
category: kmsg dumper
author: "By Jonathan Corbet December 16, 2009"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
December 16, 2009

Nice new tracing tools notwithstanding, kernel developers still tend to reach for `printk()` when trying to figure out problems. But one need not work on kernel code for very long before running into an unpleasant fact: the most interesting stuff is often printed immediately before a crash, but, for many kinds of problems, the death of the system can prevent the output of those crucial lines. It's no fun to stare at a hung system, knowing that the information needed to find the problem is probably trapped in a buffer somewhere in that system's memory. 

2.6.33 will contain a new mechanism designed to help get that last bit of information out of a dying system's clutches. The developer need only set up a new "kmsg dumper" along these lines: 

```
#include <linux/kmsg_dump.h>
    
        struct kmsg_dumper {
    	void (*dump)(struct kmsg_dumper *dumper, enum kmsg_dump_reason reason,
    			const char *s1, unsigned long l1,
    			const char *s2, unsigned long l2);
    	struct list_head list;
    	int registered;
        };
```

The `dump()` function will be called in the event of a crash; the two arguments `s1` and `s2` will have pointers to the data in the kernel's output buffer. Two pointers are needed due to the circular nature of this buffer; `s1` will point to the older set of messages. 

Registering and unregistering this function is a matter of calling: 

```
int kmsg_dump_register(struct kmsg_dumper *dumper);
        int kmsg_dump_unregister(struct kmsg_dumper *dumper);
```

In the 2.6.33 kernel, the "mtdoops" module has been reworked to use this new mechanism to save crash data to a flash device.
