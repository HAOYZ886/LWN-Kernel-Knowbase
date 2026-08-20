---
title: Hazard pointers for the kernel
url: https://lwn.net/Articles/1084015/
date: "July 27, 2026"
category: Lockless algorithms
author: "By Jonathan Corbet July 27, 2026"
---

> **For humans, by humans**
> 
> Every article on LWN.net is written for humans, by humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the slop at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 27, 2026

The kernel's read-copy-update (RCU) subsystem ensures that data will not be deleted until it is known that there are no threads holding references to it. RCU works well and is widely used throughout the kernel, but it can increase memory use and add significant delays before unused kernel objects are cleaned up. [Hazard pointers](<https://en.wikipedia.org/wiki/Hazard_pointer>) are an alternative approach to lockless data updates that offers better performance, for some situations at least. The kernel community is currently considering [a hazard-pointer implementation](<https://lwn.net/ml/all/23e34c2e-67fd-45da-b130-e70a131a59ea@paulmck-laptop>) by Mathieu Desnoyers and Paul McKenney. 

Like RCU, hazard pointers are meant to be a way to hold a short-lived reference to an immutable object that may disappear once all references are gone. Code holding references to RCU-protected data must disable preemption; hazard pointers, instead, appear to be designed to allow preemption, though such use may not be entirely optimal. 

#### The hazard-pointer API

At the API level, code using hazard pointers must allocate a context (`struct hazptr_ctx`) for each pointer that will be in use at the same time; this structure is normally placed on the stack. If there is a pointer (we'll call it `resource`) to an object to be protected by a hazard pointer, and code needs to access that object, the pointer must first be acquired with a call to: 

```
void *hazptr_acquire(struct hazptr_ctx *ctx, void * const *address);
```

Where `ctx` is the above-described context, and `address`, in this case, would be the address of `resource`. This call will return the current value of `resource` (the address of the protected object) and ensure that this object will not be changed or deleted for as long as the reference remains. When work with the protected object is complete, the code must call: 

```
void hazptr_release(struct hazptr_ctx *ctx, void *address);
```

Where `address` is the address of the old copy of the resource; after this call, the previously protected object can no longer be used. 

Normally, a hazard pointer must be released in the same execution context in which it was obtained — in the same thread or interrupt handler, in other words. There may be times when it is necessary to release the pointer in a different setting, though. That can be done, but only if the hazard-pointer context is passed to this function: 

```
void hazptr_detach(struct hazptr_ctx *ctx);
```

This call must, clearly, be made before the pointer is actually released. 

On the producer side, when the time comes to replace the protected object with a new one, code should create and initialize the new object, aim the `resource` pointer at this new copy, then call: 

```
void hazptr_synchronize(void *address);
```

This call, which must be made in a preemptible context, will wait until there are no more hazard-pointer references to the given `address`, then return to the caller. At that point, the object at that address can be freed. 

#### The implementation

The core idea behind hazard pointers is relatively simple: a call to `hazptr_acquire()` adds the pointer to a special list, while `hazptr_release()` removes it from that list. When a call to `hazptr_synchronize()` is made, that list is scanned for the address in question; if the address is found there, the function will wait until it is removed. This algorithm could be implemented with a simple linked list protected by a lock, but the whole purpose is to maximize performance, so the actual implementation is somewhat more complicated. 

The hazard-pointer code maintains a global per-CPU array, with four slots on each CPU. The oversimplified explanation of the algorithm is that, on a call to `hazptr_acquire()`, an empty slot is found, and the relevant address is stored there. Calls to `hazptr_synchronize()` can then simply scan those slots (on each CPU) and wait until none of them contain the protected address. But, once again, there are complications. 

One of those is an ordering problem. `hazptr_acquire()` must read the pointer to acquire, then store it into the slot. On the synchronize side, that pointer must be changed, then the slots searched for the previous value. If the synchronization code runs between the two acquire operations — after the pointer is read, but before it is stored into a slot — it will conclude that there are no references and release an object that is still in use. That is not the sort of hazard the authors of this code care to face. 

To address this problem, the slots are maintained in three different states. If the address stored there is `NULL`, the slot is free and not protecting a pointer. If it contains a non-`NULL` pointer value, the slot is occupied protecting that pointer. But there is a third value, `HAZPTR_WILDCARD` (which happens to have the value `0x1UL`) to indicate that the slot is in the process of being assigned. `hazptr_acquire()` starts by finding a free slot and setting its value to `HAZPTR_WILDCARD`; only then does it read the address value and, subsequently, store it in the slot. `hazptr_synchronize()` treats any slot containing `HAZPTR_WILDCARD` as if it contained the pointer it is looking for, so it will wait until the real pointer value appears in that slot before returning. That extra check prevents the race described above. 

The other complication is: what happens if there is a need for more than four slots? Any given function may not need so many slots, but there is no knowing how many will be used by functions further down the call chain. The hazard-pointer API could just return an error in that case, but that seems like a sure way to create hard-to-find bugs. Instead, handling this case is what the `hazptr_ctx` structure is for. 

That structure contains a spare slot that can be used to hold the hazard pointer if none of the per-CPU slots are available. In that case, the code uses the slot in the context structure, then links that structure into a per-CPU list. When a `hazptr_synchronize()` call happens, it must search those per-CPU lists as well as the per-CPU slots to ensure that the address is not under protection. As an added twist, there are actually _two_ per-CPU linked lists; one is available for adding to while the other is available for searching, again to prevent race conditions. The list traversal risks slowing everything down, but those lists should almost always be empty. 

The overflow slot has a couple of uses beyond extending the four per-CPU slots. The `hazptr_detach()` call described above will immediately move the given pointer into the overflow slot (if it is not already there), freeing the per-CPU slot for other uses. There is also a special callback added to the scheduler that is called on context switches; that one moves all of the per-CPU slots to their corresponding overflow slots. In this way, if a thread is preempted while using hazard-pointer slots, it will free the faster per-CPU slots for whoever runs next. 

This code is still in a relatively early state, and could yet evolve somewhat before finding its way into the mainline. Importantly, the patch series does not include any users of the API, which is normally a requirement for a new subsystem like this. The creation of those users may well reveal API shortcomings that can be resolved before merging upstream. So it is hard to hazard a guess as to when hazard pointers will be available for use by kernel developers.
