---
title: msync() and subtle behavioral tweaks
url: https://lwn.net/Articles/502612/
date: "June 19, 2012"
category: "Development model-User-space ABI"
author: "By Jonathan Corbet June 19, 2012"
---

By **Jonathan Corbet**  
June 19, 2012

Some kernel behavior is determined by standards like POSIX; others are simply a function of what the kernel developers implemented. The latter type of behavior can, in theory, be changed if there is a good reason to do so, but there is always a risk of breaking an application that depended on the previous behavior. Even worse, this kind of problem can be impossible to find during development and, indeed, may lurk until long after the new code has been deployed. A system call patch currently under consideration shows how hard it can be to know when a change is truly safe. 

The `msync()` system call exists to request that a file-backed memory region created with `mmap()` be written back to persistent storage. Once upon a time, `msync()` was the only way to guarantee that modified pages would be saved to disk in any reasonable period of time; the kernel could not always detect on its own that they had been changed by the application. That problem has long since been dealt with, but `msync()` is still a good way to inform the kernel that now would be a good time to flush modified pages to disk. 

Paolo Bonzini recently posted [a small patch set](<https://lwn.net/Articles/502616/>) making a couple of changes to `msync()`. The actual API does not change at all, but how the system call implements the API changes in subtle and interesting ways. 

There are a few options to `msync()`, one of which (`MS_ASYNC`) asks that the writeback of modified pages be "scheduled," but not necessarily completed immediately. It is meant to be a non-blocking system call that sets the necessary actions in motion, but does not wait for them to complete. Current kernels will write back dirty pages as part of the normal writeback process; the system behaves, in other words, as if `msync(MS_ASYNC)` were being called on a regular basis on every mapping. Writeback of dirty pages is already scheduled as soon as the page is dirtied. Given that, there's not much work for an explicit `MS_ASYNC` call from user space to do, and, indeed, the kernel essentially ignores such calls. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Paolo's patch causes the kernel to immediately start I/O on modified pages in response to `MS_ASYNC` calls. The result is to get those pages to persistent storage a bit more quickly than would otherwise happen, but still avoid blocking the calling process. The change seems reasonable, but Andrew Morton [worried](<https://lwn.net/Articles/502618/>) that this behavioral change might be a problem for some users: 

Means that people will find that their msync(MS_ASYNC) call will newly start IO. This may well be undesirable for some. Also, it hardwires into the kernel behaviour which userspace itself could have initiated, with sync_file_range(). ie: reduced flexibility. 

Most users are unlikely to notice the change at all. But it's entirely possible that somebody out there has a precisely-tuned system that will choke if the underlying I/O behavior changes. Users complain about exactly this kind of change at times, but usually when the change shows up in a new enterprise kernel, years too late. That said, many patches make behavioral changes that can affect users in surprising ways. The only thing that is different about this one is that the nature of the change is understood from the beginning. Andrew's concerns were not echoed by others and may not be enough to keep this change from being merged. 

The other change is potentially more troubling. `msync()` takes two parameters indicating the offset and length of the memory area to be written back. But the kernel has always ignored those parameters, choosing instead to just write back all modified pages in the file, and the related metadata as well. Paolo's patch changes the implementation to only synchronize the specific pages requested by the user. 

It would be hard to argue that the new behavior breaks the documented API; the offset and length parameters are there for a reason, after all. Still, once again, Andrew [worried](<https://lwn.net/Articles/502620/>) that applications could break in especially unpleasant ways: 

Would be nice, but if applications were previously assuming that an msync() was syncing the whole file, this patch will secretly and subtly break them. 

No developer should have written a program with the assumption that `msync()` would write pages outside of the range it was given. Any such program would clearly be buggy. But, programs written that way will work under current kernels. Changing `msync()` to not write some pages that it currently writes could cause such programs to fail in strange and difficult-to-reproduce ways. 

In general, the kernel tries not to break existing applications, even if those applications can be said to have been written in a buggy manner. If something works now, it should continue to work with future kernels. If the `msync()` changes described here break those programs, the changes should probably not be merged into the kernel. The problem, of course, is that it can be very difficult to know if a specific change will break somebody's application. Any problems caused by subtle changes are relatively unlikely to turn up before the changes are included in a released kernel. So it is necessary to proceed with care. That said, it is not practical to hold back every change that _might_ break a badly-written program somewhere; kernel development would likely be slowed considerably by such a constraint. So, probably, these changes will probably go in unless an affected user happens to notice a problem in the near future.
