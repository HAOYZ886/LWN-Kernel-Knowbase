---
title: "64-bit resources"
url: https://lwn.net/Articles/187490/
date: "June 13, 2006"
category: IO memory; Resources
author: 
---

"Resource" is the term used within the Linux kernel for a specific set of I/O-related hardware resources - I/O memory and ports, in particular. Device drivers allocate specific resources with functions like `request_region()`, but, underneath that layer, Linux has a set of generic resource allocation utilities. And at the core of that code is `struct resource`, which tracks individual resource allocations. A read of `/proc/iomem` or `/proc/ioports` is really just dumping out one of those resource data structures. 

Since the resource management code was [added by Linus](<http://lwn.net/1999/0715/a/resource.html>) at the beginning of the 2.3 development cycle, the `unsigned long` type has been used to track actual resource values. That worked at the time, but it can be problematic on 32-bit machines which have I/O memory resources at high addresses. If a memory region is located out of the 32-bit range, the resource management code can no longer deal with it. 

The solution, of course, is to start using 64-bit numbers to track resource allocations. Vivek Goyal (along with others) has been working for some time on [a set of patches](<http://lwn.net/Articles/187460/>) which makes this change. Those patches have been fixed up by Greg Kroah-Hartman and, by all appearances, are set for merging once the 2.6.18 development cycle starts. 

Introducing new typedefs into the kernel is generally frowned upon, but this patch creates `resource_size_t` anyway. Early in the patch series, this type is just `unsigned long`; only when the type has propagated through the source is it changed to a 64-bit value. There is a configuration option controlling whether 64-bit resources are used; interestingly, 64-bit is the default, and the old 32-bit mode is marked "experimental." 

The result of the change is that the prototypes for some commonly-used functions change: 

```
struct resource *request_region(resource_size_t start,
                                        resource_size_t n,
    				    const char *name);
        void release_region(resource_size_t start, resource_size_t n);
    
        struct resource *request_mem_region(resource_size_t start,
                                            resource_size_t n,
    					const char *name);
        void release_mem_region(resource_size_t start, resource_size_t n);
```

Most driver code will be unaffected by these changes; simple constant resource locations will still work, and, in many cases, the bus layer handles the details of resource allocation anyway. But, in cases where a driver is directly storing or working with resource locations, the new type will have to be used.
