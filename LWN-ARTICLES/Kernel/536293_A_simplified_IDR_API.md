---
title: A simplified IDR API
url: https://lwn.net/Articles/536293/
date: "February 6, 2013"
category: idr
author: "By Jonathan Corbet February 6, 2013"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
February 6, 2013

The kernel's "IDR" layer is a curious beast. Its job is conceptually simple: it is charged with the allocation of integer ID numbers used with device names, POSIX timers, and more. The implementation is somewhat less than simple, though, for a straightforward reason: IDR functions are often called from performance-critical code paths and must be able to work in atomic context. These constraints, plus some creative programming, have led to one of the stranger subsystem APIs in the kernel. If Tejun Heo has his way, though, things will become rather less strange in the future — though at least one reviewer disagrees with that conclusion. 

Strangeness notwithstanding, the IDR API has changed little since it was [documented](<https://lwn.net/Articles/103209/>) here in 2004. One includes `<linux/idr.h>`, allocates an `idr` structure, and initializes it with `idr_init()`. Thereafter, allocating a new integer ID and binding it to an internal structure is a matter of calling these two functions: 

```
int idr_pre_get(struct idr *idp, gfp_t gfp_mask);
        int idr_get_new(struct idr *idp, void *ptr, int *id);
```

The call to `idr_pre_get()` should happen outside of atomic context; its purpose is to perform all the memory allocations necessary to ensure that the following call to `idr_get_new()` (which returns the newly allocated ID number and associates it with the given `ptr`) is able to succeed. The latter call can then happen in atomic context, a feature needed by many IDR users. 

There is just one little problem with this interface, as Tejun points out in [the introduction to his patch set](<https://lwn.net/Articles/536019/>): the call to `idr_get_new()` can still fail. So code using the IDR layer cannot just ask for a new ID; it must, instead, execute a loop that retries the allocation until it either succeeds or returns a failure code other than `-EAGAIN`. That leads to the inclusion of a lot of error-prone boilerplate code in well over 100 call sites in the kernel; the 2004 article and Tejun's patch both contain examples of what this code looks like. 

Failure can happen for a number of reasons, but the mostly likely cause is tied to the fact that the memory preallocated by `idr_pre_get()` is a global resource. A call to `idr_pre_get()` simply ensures that a minimal amount of memory is available; calling it twice will not increase the amount of preallocated memory. So, if two processors simultaneously call `idr_pre_get()`, the amount of memory allocated will be the same as if only one processor had made that call. The first processor to call `idr_get_new()` may then consume all of that memory, leaving nothing for the second caller. That second caller will then be forced to drop out of atomic context and execute the retry loop — a code path that is unlikely to have been well tested by the original developer. 

Tejun's response is to change the API, basing it on three new functions: 

```
void idr_preload(gfp_t gfp_mask);
        int idr_alloc(struct idr *idp, void *ptr, int start, int end, gfp_t gfp_mask);
        void idr_preload_end(void);
```

As with `idr_pre_get()`, the new `idr_preload()` function is charged with allocating the memory necessary to satisfy the next allocation request. There are some interesting differences, though. The attentive reader will note that there is no `struct idr` argument to `idr_preload()`, suggesting that the preallocated memory is not associated with any particular ID number space. It is, instead, stored in a single per-CPU array. Since this memory is allocated for the current CPU, it is not possible for any other processor to slip in and steal it — at least, not if the current thread is not preempted. For that reason, `idr_preload()` also disables preemption. Given that, the existence of the new `idr_preload_end()` function is easy to explain: it is there to re-enable preemption once the allocation has been performed. 

A call to `idr_alloc()` will actually allocate an integer ID. It accepts upper and lower bounds for that ID to accommodate code that can only cope with a given range of numbers — code that uses the ID as an array index, for example. If need be, it will attempt to allocate memory using the given `gfp_mask`. Allocations will be unnecessary if `idr_preload()` has been called, but, with the new interface, preallocation is no longer necessary. So code that can call `idr_alloc()` from process context can dispense with the `idr_preload()` and `idr_preload_end()` calls altogether. Either way, the only way `idr_alloc()` will fail is with a hard memory allocation failure; there is no longer any need to put a loop around allocation attempts. As a result, Tejun's 62-part patch set, touching 78 files, results in the net deletion of a few hundred lines of code. 

Most of the developers whose code was changed by Tejun's patch set responded with simple `Acked-by` lines. Eric Biederman, though, [didn't like the API](<https://lwn.net/Articles/536352/>); he said ""When reading code with idr_preload I get this deep down creepy feeling. What is this magic that is going on?"" As can be seen in [Tejun's response](<https://lwn.net/Articles/536353/>), one developer's magic is another's straightforward per-CPU technique. As of this writing, that particular discussion has not reached any sort of public resolution. Your editor would predict, though, that the simplification of this heavily-used API will be sufficiently compelling that most developers will be able to get past any resulting creepy feelings. So the IDR API may be changing in a mainline kernel in the not-too-distant future.
