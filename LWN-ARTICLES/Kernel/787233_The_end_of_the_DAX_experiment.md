---
title: The end of the DAX experiment
url: https://lwn.net/Articles/787233/
date: "May 2, 2019"
category: "Memory management-Nonvolatile memory"
author: "By Jonathan Corbet May 2, 2019 LSFMM"
---

By **Jonathan Corbet**  
May 2, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

Since its inception, the DAX mechanism (which provides for direct access to files stored on persistent memory) has been seen as somewhat experimental and incomplete. At the 2019 Linux Storage, Filesystem, and Memory-Management Summit, Dan Williams ran a session where he said that perhaps the time has come to end that experiment. Some of the unimplemented DAX features may never actually need to be implemented, and it might just be possible to declare DAX finished. But first there are a few more details to take care of. 

He started with the general question of what DAX actually means; he defined it as the mechanism that is used to bypass the page cache and map persistent memory directly into user space. There are obvious performance advantages to doing this, but it causes problems as well: the virtual address given to a page of data is also the address of the permanent [![\[Dan Williams\]](https://static.lwn.net/images/conf/2019/lsfmm/DanWilliams-sm.jpg)](<https://lwn.net/Articles/787270/>) storage for that data. Many parts of the system were not designed with that in mind, so various features have to be turned off when DAX is in use, and others must be bypassed. It gives the whole subsystem the feel of a permanent experiment, and that makes people not want to use it. 

Within the kernel, there are a few ways to know that DAX is involved with any given memory mapping. A call to `vma_is_dax()` is a way of asking whether a given page is located in persistent storage. Then, there is `vma_is_fsdax()`, which is similar except it says that some other agent could break that association at any time. It is used to prevent certain long-term mappings to DAX memory. The `PTE_DEVMAP` page-table flag notes that a given page's lifetime is tied to that of an underlying device; users have to take extra references to that device to keep it from going away. Outside of the kernel, though, there is only one sure indication that DAX is in use: the `MAP_SYNC` flag to `mmap()`. If a mapping operation with that flag fails, persistent memory is not present. 

What do we need to finish the experiment? There are still some semantics to figure out, Williams said. There are a lot of applications that are not interested in persistent memory at the level of performing individual cache-flush operations, but they do want to know whether any given mapping involves the page cache. Jan Kara added that such applications are trying to optimize the amount of memory they use; they tend to be programs like a database manager that knows it has the system to itself. If a given file is mapped through ordinary memory, the application must allow for the page cache and reduce its own memory use. If DAX is available, the application can use more memory for its own purposes. Dave Hansen suggested that what is really needed is a way to ask directly about the performance characteristics of a memory mapping. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Williams continued by asking whether there might be a need for a `MAP_DIRECT` option that asks the kernel to minimize the use of the page cache. Filesystems offer both buffered and direct APIs, with the latter avoiding the page cache; perhaps memory management should do the same. The problem is that these semantics might break with upcoming filesystems like [NOVA](<https://lwn.net/Articles/729812/>), which do not use the page cache but also do not offer direct mappings. Applications are often not interested in the "direct" part, but they do care about page-cache usage. 

Michal Hocko jumped in with a different point of view: the real problem, he said, is that the page cache simply sucks. It is an implementation detail in current kernels, but might well prove not to be a long-term concern. Rather than adding new APIs to enable applications to get around the page cache, we should just make caching work properly. That would help to eliminate a lot of nasty application code. This suggestion, while interesting, does not solve the short-term problem and so was set aside. 

Getting back to DAX, Williams noted that there are a lot of features in the kernel that are not implemented for the DAX case; many places in the code read something like: 

```
if (dax)
            goto fail;
```

These include [long-term mappings with `get_user_pages()`](<https://lwn.net/Articles/784574/>), which can pin down DAX pages indefinitely. There are some problems with `reflink()` functionality, a problem that was set aside for the filesystem track. 

There are also problems with private mappings and DAX; pages created via copy-on-write will end up in ordinary RAM rather than persistent memory, which may not be what users want. Some may prefer that these copies remain in cheaper memory. Hansen suggested that this problem could be solved with NUMA policies. Williams said that the right solution might be to call back into the underlying filesystem to inform it of a copy-on-write fault and make it allocate a new page to handle it; the filesystem would then have to track those pages specially until they are released. Kara said that he didn't really see a use case for this feature, though. Mel Gorman described it as "difficult semantics and non-obvious behavior for a non-existent use case". 

Time for the session ran out about then. Williams closed by saying that, perhaps, it is not necessary to implement all of those APIs for the DAX case until users scream about their absence. That would allow the DAX experiment to be declared done, more or less, for now.
