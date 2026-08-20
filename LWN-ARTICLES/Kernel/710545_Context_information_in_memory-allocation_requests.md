---
title: "Context information in memory-allocation requests"
url: https://lwn.net/Articles/710545/
date: "January 4, 2017"
category: "Memory management-GFP flags"
author: "By Jonathan Corbet January 4, 2017"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
January 4, 2017

As is the case with many other tasks, allocation of memory in the kernel is rather more complex than it is in user space. The APIs used for allocation in the kernel have evolved over many years to reflect this complexity. But a highly evolved API is not necessarily an optimal one, and there have been signs for years that the approach used in the kernel is not perfectly suited to the task. A set of patches under consideration now shows how thinking has shifted in this area. 

Memory-allocation complexity in the kernel is driven by constraints on what the kernel can do in any given situation. It is often the case, for example, that the kernel is running in a context where it is not allowed to block waiting for an event, so allocation requests must be satisfied without acquiring any sleeping locks. Sometimes a request should be given access to the last-resort pools of memory; this is usually the case when the request itself is part of the process of freeing more memory in a system where memory is tight. There can be constraints on where the allocated memory must be located. And so on. 

The approach taken to track these constraints is to add a "GFP flags" argument to every memory-allocation function. So, for example, the prototype of `kmalloc()`, used to allocate relatively small chunks of memory, is: 

```
void *kmalloc(size_t size, gfp_t flags);
```

The `flags` argument describes the constraints on the request. A value of `GFP_ATOMIC` indicates that the request is running in atomic context and cannot sleep, for example, while `GFP_DMA32` says that the allocated memory must be placed in a physical location reachable by devices limited to 32-bit DMA operations. There is a long list of these flags; `<linux/gfp.h>` has the whole set. 

#### Two types of flags

The point of interest here is that some of these flags (like `GFP_DMA32`) describe attributes of the needed memory — they apply to a specific allocation request. But others, like `GFP_ATOMIC`, instead describe the context in which the allocation request is being made. This context is often not known at the point where the allocation function is called, since that often happens in low-level code that can be invoked in many contexts. So higher-level code must inform the low-level code about the context in which it is running; this is generally done by adding GFP-flags arguments to functions all the way up the call chain. To pick a random example, consider the function that submits a request to a USB device: 

```
int usb_submit_urb(struct urb *urb, gfp_t mem_flags);
```

This relatively high-level function must track the given `mem_flags` and pass them to any function it calls that might allocate memory; it must also adjust the flags if its own context changes. This interface has been made to work for many years, but it is somewhat prone to errors. One could argue, as some have over the years, that it is fundamentally wrong; information tracking the context in which a thread is running might be better attached to the thread directly rather than dragged along through a chain of function calls. 

#### GFP_NOFS

One flag in particular that describes the calling context is `GFP_NOFS`, which instructs the memory allocator to avoid calling into any filesystem code. In particular, that means that the allocator cannot start writeback on dirty pages to make more memory available. There are (at least) a couple of reasons to impose this constraint. One is that the allocation call itself is coming from filesystem code; in that case, calling back into the filesystem risks deadlocking the system. The other is that adding filesystem calls to a lengthy call chain could overflow the kernel stack, an outcome cherished by attackers but otherwise unloved by Linux users. 

Given those possibilities, it is unsurprising that kernel developers have tended to take a "better safe than sorry" approach to the `GFP_NOFS` flag; as a result, that flag appears in a great many allocation calls — a quick grep shows over 1,300 instances in the 4.10-rc2 kernel. At the Linux Storage, Filesystem, and Memory-Management summit in April 2016, Michal Hocko [called out](<https://lwn.net/Articles/684437/>) use of this flag as a problem. It appears in many places where it is not really needed, unnecessarily constraining what the memory-management code can do and, as a result, worsening system performance. He suggested that this flag should be phased out in favor of a flag in the task structure that could be used to accurately track the allocation context. 

More recently, he has [proposed a new API](<https://lwn.net/Articles/709356/>) that implements these changes. A new flag (`PF_MEMALLOC_NOFS`) is defined for the `flags` field of the `task_struct` structure. Then, whenever a thread enters a context where filesystem calls should not be made, it should call: 

```
unsigned int memalloc_nofs_save(void);
```

This call will set the `PF_MEMALLOC_NOFS` flag and pass the previous `flags` value back as its return value. Exiting from the "no filesystem calls" context is done with a call to: 

```
void memalloc_nofs_restore(unsigned int flags);
```

The `flags` value passed in should be the value returned from `memalloc_nofs_save()`. 

Between the two above calls, all memory-allocation requests executed in the current thread will behave as if the `GFP_NOFS` flag had been passed, regardless of whether it is actually present or not. Since each caller saves the previous context, these calls can be nested to any level and the right thing will happen. For now, the `GFP_NOFS` flag remains (there is the matter of those 1,300 users, after all), but one can see its eventual removal in the cards. The patch set begins that process by fixing some callers in the XFS and ext4 filesystem code. The resulting code should be clearer, and it eliminates the chance of a stray allocation calling into the filesystem code in the wrong place. 

Developers familiar with the memory-management code may think that this interface looks familiar. Indeed, it is inspired by a set of already existing functions: 

```
unsigned int memalloc_noio_save(void);
        void memalloc_noio_restore(unsigned int flags);
```

These functions were [added to the 3.9 kernel](<https://git.kernel.org/linus/21caf2fc1931b485483ddd254b634fa8f0099963>) in 2013 by Ming Lei; they move the `GFP_NOIO` flag (which inhibits the initiation of I/O operations) into the task structure in the same way. 

The memory-allocation interface is, thus, clearly evolving in a direction where context-related information is stored with the rest of the thread's context rather than being passed through function arguments. This evolution can only be described as a slow process, though; there are nine `memalloc_noio_save()` calls in the 4.10-rc2 kernel, compared to nearly 500 uses of `GFP_NOIO`. Increasing the pace of change may be hard, though; switching to the new API requires a fairly deep understanding of the code involved and cannot be done with a simple script. 

One could imagine taking this work further by, for example, tracking atomic context explicitly. But that is work for the future; completing the task for `GFP_NOIO` and `GFP_NOFS` should arguably be done first. Once all that is done, the kernel's memory-allocation API may better match the uses to which it is put. Given that we have only had 25 years to work on it so far, it is not entirely surprising that we have not gotten there yet.
