---
title: The debut of kref
url: https://lwn.net/Articles/75920/
date: "March 16, 2004"
category: Reference counting; kref
author: 
---

When Patrick Mochel added the "kobject" type to the 2.5.45 kernel, he described it this way: 

This is not meant to be fancy; just something simple for which we can control the refcount and other common functionality using common code. 

In the 2.6 kernel, the kobject type has become, via its `kset` and `parent` pointers, the glue which holds the entire device model structure together. It is the core object implementing every entry in the sysfs virtual filesystem. Kobjects also handle the generation of hotplug events when devices come and go. 

Oh, yes. Kobjects also handle reference counting. 

The kobject type has clearly grown past its original mandate into something fairly fancy. To address the needs of kernel hackers who only want a simple reference counter, Greg Kroah-Hartman has created a new type called [kref](<https://lwn.net/Articles/75659/>). A kref is, indeed, a simple thing: 

```
struct kref {
    		atomic_t refcount;
    		void (*release)(struct kref *kref);
    	};
```

A kref comes with the usual functions one would expect: `kref_init()` to set it up, and `kref_get()` and `kref_put()` to manage the reference count. Once that count drops to zero, the release function is called to clean things up. All told, it's quite simple. 

In fact, it would appear to be _too_ simple for some kernel hackers, who have questioned whether there is any need for kref at all. Why not simply manipulate a reference count directly with `atomic_t` operations and avoid adding the space required for the `release()` pointer to every reference-counted object? The answer that comes back is that buggy reference counting implementations in the kernel are far from unknown, and that the overhead of using kref is tiny. As Andrew Morton [put it](<https://lwn.net/Articles/75921/>): 

I care more about being able to say "ah, it uses kref. I understand that refcounting idiom, I know it's well debugged and I know that it traps common errors". That's better than "oh crap, this thing implements its own refcounting - I need to review it for the usual errors". 

Andrew's approval is sufficient; the kref patch showed up in 2.6.5-rc1. 

For the future, Greg has a patch which converts the kobject reference counting mechanism over to krefs. That change may be a harder sell, however; it will expand the size of every kobject in the system (because kobjects, currently, do not store the `release()` function pointer directly). So that change will wait for 2.7, and may be part of a larger-scale cleanup and refactoring of the kobject type.
