---
title: "IDA: simplifying the complex task of allocating integers"
url: https://lwn.net/Articles/764057/
date: "September 4, 2018"
category: IDA
author: "By Jonathan Corbet September 4, 2018"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 4, 2018

It is common for kernel code to generate unique integers for identifiers. When one plugs in a flash drive, it will show up as `/dev/sdN`; that `N` (a letter derived from a number) must be generated in the kernel, and it should not already be in use for another drive or unpleasant things will happen. One might think that generating such numbers would not be a difficult task, but that turns out not to be the case, especially in situations where many numbers must be tracked. The IDA (for "ID allocator", perhaps) API exists to handle this specialized task. In past kernels, it has managed to make the process of getting an unused number surprisingly complex; the 4.19 kernel has a new IDA API that simplifies things considerably. 

Why would the management of unique integer IDs be complex? It comes down to the usual problems of scalability and concurrency. The IDA code must be able to track potentially large numbers of identifiers in an efficient way; in particular, it must be able to find a free identifier within a given range quickly. In practice, that means using a [radix tree](<https://lwn.net/Articles/175432/>) (or, soon, an [XArray](<https://lwn.net/Articles/745073/>)) to track allocations. Managing such a data structure requires allocating memory, which may be difficult to do in the context where the ID is required. Concurrency must also be managed, in that two threads allocating or freeing IDs in the same space should not step on each other's toes. 

A particular allocation space (usually just called an "IDA") is created with either of the usual kernel idioms. It can be defined and initialized with this macro: 

```
#include <linux/idr.h>
    
        DEFINE_IDA(name);
```

Alternatively, the two steps can be separated with: 

```
struct ida my_ida;
        /* ... */
        ida_init(&my_ida);
```

If an IDA is no longer needed, it can be freed by passing it to `ida_destroy()`; it is not necessary to free any allocated IDs first. 

#### The old way

A longtime unwritten design pattern in the kernel community states that low-level utilities (such as IDA) should not perform their own locking; instead, protection against concurrent calls should be handled by the caller. The reasoning is that, often as not, the higher-level code has its own locking already; in such cases, doing more locking at a lower level would be wasteful. So the old IDA API expected (with an exception, see below) that locking would be handled by its callers. 

That raises another issue, though. IDA is supposed to be fast, but the effort past kernel developers have put into squeezing out that last microsecond of execution time would go to waste if IDA calls were surrounded by heavyweight locking. IDA callers thus tend to use spinlocks; that choice may be forced by the context in which ID allocation is performed anyway. The use of spinlocks means that IDA allocation tends to be done in atomic context, where memory allocation can be less reliable. To deal with this problem, pre-4.19 IDA allocation is split into two phases. First, a call is made to: 

```
int ida_pre_get(struct ida *ida, gfp_t gfp);
```

This function tries to ensure that memory will be available to satisfy a subsequent allocation request, which is done using: 

```
int ida_get_new_above(struct ida *ida, int start, int *id);
```

The `ida_pre_get()` call can be done outside of the locking that serializes calls to `ida_get_new_above()`, so it may be more likely to succeed in allocating memory. Of course, there is no guarantee that the memory pre-allocated by one caller will not be snagged by a concurrent caller, so `ida_get_new_above()` can still return `-EAGAIN`. Thus, ID allocation is typically performed with a dance like: 

```
retry:
            ida_pre_get(&some_ida, GFP_KERNEL);
    	spin_lock(&some_lock);
    	r = ida_get_new_above(&some_ida, 0, &id);
    	spin_unlock(&some_lock);
    	if (r == -EAGAIN)
    	    goto retry;
```

An identifier allocated this way could be freed with `ida_remove()`, which must also be called with the appropriate lock held. 

This kind of pattern makes a simple task start to look a bit less simple, especially if the spinlock must be introduced for the sole purpose of calling IDA functions. It is also the sort of code that can be easy to get wrong in subtle ways. In the hopes of making life easier, a set of higher-level functions was [added by Rusty Russell](<https://git.kernel.org/linus/88eca0207cf1574328c3ce8c3be537a9317261bb>) in 2011 for the 3.1 kernel release: 

```
int ida_simple_get(struct ida *ida, unsigned int start, unsigned int end,
    		       gfp_t gfp_mask);
        void ida_simple_remove(struct ida *ida, unsigned int id);
```

These functions use their own lock and hide the `ida_pre_get()` dance from callers. As of 4.18, most IDA users use these functions, but there are still numerous sites using the more complex API. 

#### The 4.19 way

One of the last pulls into the mainline before the release of 4.19-rc1 was [Matthew Wilcox's IDA series](<https://lwn.net/ml/linux-kernel/20180621212835.5636-1-willy%40infradead.org/>), which makes a number of changes to the API. In particular, `ida_pre_get()` no longer exists as far as IDA users are concerned; all IDA functions now handle their own internal locking and memory allocation. Allocation and freeing of identifiers is done with: 

```
int ida_alloc(struct ida *ida, gfp_t gfp);
        void ida_free(struct ida *, unsigned int id);
```

There are a number of variants for callers needing more control over the allocated ID: 

```
int ida_alloc_min(struct ida *ida, unsigned int min, gfp_t gfp);
        int ida_alloc_max(struct ida *ida, unsigned int max, gfp_t gfp);
        int ida_alloc_range(struct ida *ida, unsigned int min, unsigned int max,
    			gfp_t gfp);
```

`ida_simple_get()` and `ida_simple_remove()` still exist as wrappers around the above functions, avoiding the need to change large numbers of existing callers, but those functions should probably be seen as deprecated at this point. The lower-level functions have been removed entirely, instead, and all callers within the kernel have been changed to the new API. 

The IDA interface has clearly been simplified quite a bit. The underlying implementation remains surprisingly complex for such a seemingly simple task, though; interested readers can find it in [`lib/idr.c`](<https://elixir.bootlin.com/linux/v4.19-rc2/source/lib/idr.c#L312>). Nothing, it seems, is simple in kernel space.
