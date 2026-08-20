---
title: "The timer API: size or type safety?"
url: https://lwn.net/Articles/212724/
date: "December 5, 2006"
category: Timers
author: 
---

The timer API allows kernel code to request that a function be called at some point in the future. At its core is the `timer_list` structure, which contains a few fields of interest: 

```
struct timer_list {
    	unsigned long expires;
    	void (*function)(unsigned long);
    	unsigned long data;
    	/* ... */
        };
```

To request an action in the future, a kernel function places a relative expiration time (expressed in jiffies) in `expires` and some sort of useful private value in `data`. `function()` is a pointer to a routine which will be called after (at least) the requested number of jiffies have passed; `data` will be its only parameter. After the `timer_list` structure has been set up, a call to `add_timer()` puts the request into the system. 

This API has not changed much in some time; as a result, the description of timers in Chapter 7 of [Linux Device Drivers](<https://lwn.net/Kernel/LDD3/>) is still useful for those wanting details. It may, in fact, be the only part of LDD3 which is not yet thoroughly obsolete. 

That situation may change soon, however, as there are developers with their eyes on this interface. Interestingly, there are two very different ideas of how the timer API should be changed. 

The conversation was [started by Al Viro](<https://lwn.net/Articles/212727/>) who, for some time now, has been working on improving the type safety of the kernel API. He notes that the `unsigned long` argument to timer functions is, in fact, almost always a pointer value. So there is a lot of code in the kernel which is busily casting pointers to `unsigned long` values and back - or engaging in lazy trickery to avoid having to do those casts. Casts like this make compile-time type checking almost impossible, so every one is an opportunity to introduce hard-to-find bugs. 

Al would like to fix this problem by creating a more type-safe interface to the kernel timer subsystem. His approach involves changing the type of the timer function argument to `void *`, reflecting the fact that it's usually a pointer type. He then has a `SETUP_TIMER()` macro which involves the following bit of code: 

```
typeof(*data) *p = data;
        timer->function = (void (*)(void *)) func;
        timer->data = (void *) p;
        (void)(0 && (func(p), 0));
```

The middle two lines are simply initializing the relevant fields of the `timer_list` structure. What the last line is doing, however, is creating a call to the timer function with the provided argument; if there is a type mismatch between that argument and the function's prototype, the compiler will complain. The call is written in such a way that it will be optimized out, so that call does not make it through to the kernel image. But, in the running kernel, it will be known that the timer function is receiving an appropriately-typed argument. 

There are a _lot_ of timers in the kernel, so this is the sort of change which makes people nervous. Al's plan involves creating the `SETUP_TIMER()` macro, but leaving the callback function's prototype unchanged. Then parts of the kernel could be converted at leisure, with the callback function prototype being changed once the conversion of in-kernel code is complete. 

Thomas Gleixner [joined in](<https://lwn.net/Articles/212728/>) with an alternative suggestion: remove the `data` value from `struct timer_list` altogether, and pass a pointer to the `timer_list` structure into the callback function. If that structure is embedded within some other structure which has the information the callback really needs, a simple recast with `container_of()` will yield the needed pointer. The result would be a smaller `timer_list` structure. This approach mirrors the [proposed workqueue API changes](<http://lwn.net/Articles/211279/>) discussed here last week. 

Al doesn't like that idea. He has been working to get rid of casts in the kernel, but this API would require the introduction of hundreds more of them. There is little type safety built into `container_of()`. To him, the space required for a pointer is more than justified by the extra compile-time checking that comes from its use. 

Ingo Molnar, in [disagreeing](<https://lwn.net/Articles/212730/>), makes the tradeoff clear: 

The question is: which is more important, the type safety of a container_of() [or type cast], which if we get it wrong produces a /very/ trivial crash that is trivial to fix - or embedded timers data structure size all around the kernel? I believe the latter is more important. 

Not too many other developers have joined the discussion so far. It's an important one, though; how this decision goes could shape how kernel APIs are designed in the future. Perhaps somebody will come up with a way to have both type safety and smaller size. Until such a time, however, there is a tradeoff to be made, and it's not clear which way the decision will go.
