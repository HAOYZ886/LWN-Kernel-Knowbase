---
title: Flushing the page cache from user space
url: https://lwn.net/Articles/124560/
date: "February 22, 2005"
category: Development model; Memory management
author: 
---

Martin Hicks recently posted [a patch](<http://lwn.net/Articles/123343/>) which adds a new degree of user-space control over memory management policy. In particular, it creates a new `/proc` entry: 

```
/proc/sys/vm/toss_page_cache_nodes
```

If a suitably privileged process writes one or more NUMA node numbers to that file, all pages belonging to that node which are found in the page cache will be flushed out. Essentially, this operation causes a node to forget about all locally-cached pages from files in the filesystem. 

Clearing the page cache in this way would normally be bad for performance. The page cache exists to allow the filesystem to satisfy common filesystem requests without going to the disk; clearing the cache defeats that functionality and would normally be undesirable. There are exceptions to everything, however. This patch is aimed at large-scale high-performance computing tasks running in a cluster environment. Such jobs typically do best if they can start with a clean system; they have no real use for whatever may have been cached for the previous user. More to the point, a full page cache can cause memory allocations to be satisfied with non-local (slower) memory, resulting in significantly worse performance. By clearing the cache before starting a new job, a system administrator can ensure that local memory is available for that job. 

Not everybody likes the patch. Ingo Molnar [thinks](<https://lwn.net/Articles/124566/>) that this capability will create confusion and make the debugging of memory problems even harder. 

How are we supposed to debug VM problems where one player periodically flushes the whole pagecache? ... Providing APIs to flush system caches, sysctl or syscall, is the road to VM madness. 

Andrew Morton, instead, sees the value of the patch for some users, but [doesn't like the implementation](<https://lwn.net/Articles/124567/>). He would like to see this capability made useful for other classes of users, such as kernel developers who want to put the system into a known state before running tests. He also doesn't like the `/proc` interface, and argues for a new system call instead. His suggestion was: 

```
sys_free_node_memory(long node_id, long pages_to_make_free, 
                             long what_to_free);
```

This form of the call would allow the clearing of something less than the entire page cache, making the tool a bit less crude. The `what_to_free` argument would be a bitmask specifying which types of memory to free; beyond the page cache, this call could cause the kernel to reclaim anonymous memory or slab caches. 

The system call approach would seem to make sense; there is one remaining glitch, however: SUSE already shipped the `/proc` interface in SLES9. That revelation drew [a complaint](<https://lwn.net/Articles/124569/>) from Andrew: 

This is why you should target kernel.org kernels first. Now we risk ending up with poor old suse carrying an obsolete interface and application developers have to be able to cater for both interfaces. 

An explicit purpose behind the 2.6 development model is to get patches into the mainline quickly so that their form can be stabilized before distributors ship them. As the developers become used to this mode of operation, this sort of issue should become relatively rare.
