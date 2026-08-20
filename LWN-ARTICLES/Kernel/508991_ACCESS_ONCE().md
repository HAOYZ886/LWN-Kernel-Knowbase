---
title: ACCESS_ONCE()
url: https://lwn.net/Articles/508991/
date: "August 1, 2012"
category: ACCESS ONCE; volatile
author: "By Jonathan Corbet August 1, 2012"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
August 1, 2012

Even a casual reader of the kernel source code is likely to run into invocations of the `ACCESS_ONCE()` macro eventually; there are well over 200 of them in the current source tree. Many such readers probably do not stop to understand just what that macro means; a recent discussion on the mailing list made it clear that even core kernel developers may not have a firm idea of what it does. Your editor was equally ignorant but decided to fix that; the result, hopefully, is a reasonable explanation of why `ACCESS_ONCE()` exists and when it must be used. 

The functionality of this macro is actually well described by its name; its purpose is to ensure that the value passed as a parameter is accessed exactly once by the generated code. One might well wonder why that matters. It comes down to the fact that the C compiler will, if not given reasons to the contrary, assume that there is only one thread of execution in the address space of the program it is compiling. Concurrency is not built into the C language itself, so mechanisms for dealing with concurrent access must be built on top of the language; `ACCESS_ONCE()` is one such mechanism. 

Consider, for example, the following code snippet from `kernel/mutex.c`: 

```
for (;;) {
    	struct task_struct *owner;
    
    	owner = ACCESS_ONCE(lock->owner);
    	if (owner && !mutex_spin_on_owner(lock, owner))
    	    break;
     	/* ... */
```

This is a small piece of the adaptive spinning code that hopes to quickly grab a mutex once the current owner drops it, without going to sleep. There is much more to this `for` loop than has been shown here, but this code is sufficient to show why `ACCESS_ONCE()` can be necessary. 

Imagine for a second that the compiler in use is developed by fanatical developers who will optimize things in every way they can. This is not a purely hypothetical scenario; as Paul McKenney recently [attested](<https://lwn.net/Articles/508999/>): ""I have seen the glint in their eyes when they discuss optimization techniques that you would not want your children to know about!"" These developers might create a compiler that concludes that, since the code in question does not actually modify `lock->owner`, it is not necessary to actually fetch its value each time through the loop. The compiler might then rearrange the code into something like: 

```
owner = ACCESS_ONCE(lock->owner);
        for (;;) {
    	if (owner && !mutex_spin_on_owner(lock, owner))
    	    break;
```

What the compiler has missed is the fact that `lock->owner` is being changed by another thread of execution entirely. The result is code that will fail to notice any such changes as it executes the loop multiple times, leading to unpleasant results. The `ACCESS_ONCE()` call prevents this optimization happening, with the result that the code (hopefully) executes as intended. 

As it happens, an optimized-out access is not the only peril that this code could encounter. Some processor architectures (x86, for example) are not richly endowed with registers; on such systems, the compiler must make careful choices regarding which values to keep in registers if it is to generate the highest-performing code. Specific values may be pushed out of the register set, then pulled back in later. Should that happen to the mutex code above, the result could be multiple references to `lock->owner`. And _that_ could cause trouble; if the value of `lock->owner` changed in the middle of the loop, the code, which is expecting the value of its local `owner` variable to remain constant, could become fatally confused. Once again, the `ACCESS_ONCE()` invocation tells the compiler not to do that, avoiding potential problems. 

The actual implementation of `ACCESS_ONCE()`, found in `<linux/compiler.h>`, is fairly straightforward: 

```
#define ACCESS_ONCE(x) (*(volatile typeof(x) *)&(x))
```

In other words, it works by turning the relevant variable, temporarily, into a `volatile` type. 

Given the kinds of hazards presented by optimizing compilers, one might well wonder why this kind of situation does not come up more often. The answer is that most concurrent access to data is (or certainly should be) protected by locks. Spinlocks and mutexes both function as optimization barriers, meaning that they prevent optimizations on one side of the barrier from carrying over to the other. If code only accesses a shared variable with the relevant lock held, and if that variable can only change when the lock is released (and held by a different thread), the compiler will not create subtle problems. It is only in places where shared data is accessed without locks (or explicit barriers) that a construct like `ACCESS_ONCE()` is required. Scalability pressures are causing the creation of more of this type of code, but most kernel developers still should not need to worry about `ACCESS_ONCE()` most of the time.
