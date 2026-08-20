---
title: Atomic additions
url: https://lwn.net/Articles/651629/
date: "July 20, 2015"
category: atomic t
author: "By Jonathan Corbet July 20, 2015"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
July 20, 2015

Atomic variables have a long history as part of the kernel's concurrency-management toolkit. These variables enable the execution of simple arithmetic (and related) operations in an all-or-nothing manner; other CPUs will never see partially-executed operations. As systems grow more complex, though, atomic variables are having to become more complex as well, as seen by a couple of recently proposed additions to the `atomic_t` repertoire. 

#### Atomic logical operations

The simpler addition is the [atomic logical operations patch set](<https://lwn.net/Articles/651414/>) from Peter Zijlstra. Peter noted that there was no notion of logical operations on `atomic_t` variables that was the same across all architectures. Some of them have related operations called `atomic_set_mask()` and `atomic_clear_mask()`, but those operations are defined inconsistently across architectures when they are present at all. 

To clean this situation up a bit, Peter introduced these new operations: 

```
void atomic_and(int mask, atomic_t *value);
        void atomic_or(int mask, atomic_t *value);
        void atomic_xor(int mask, atomic_t *value);
        void atomic64_and(int mask, atomic64_t *value);
        void atomic64_or(int mask, atomic64_t *value);
        void atomic64_xor(int mask, atomic64_t *value);
```

There is also a pair of simple wrappers (`atomic_andnot()` and `atomic64_andnot()`) that simply flip the bits of the `mask` argument. All of these functions have a `void` type; there are no `_return` variants (e.g. `atomic_and_return()`) that return the result of the operation at the same time. Uses of `atomic_set_mask()` and `atomic_clear_mask()` in the tree are changed to use the new functions, and the old ones have been deprecated. 

#### Relaxed atomics

Atomic operations do not normally function as memory barriers; in other words, the processor and the compiler are both free to reorder atomic operations relative to other operations in ways that could create confusion in concurrent situations. The exception to that rule is the `_return` operations; for example, `atomic_add_return()` will add a value to an `atomic_t`, return the resulting value, and function as a full memory barrier. 

Those rules are looking increasingly inadequate when faced with the growing complexity and concurrency of contemporary systems. All-or-nothing memory barriers are an overly blunt tool for developers who are working to maximize concurrency and minimize the cost of the associated operations. What developers would like to see instead is the ability to explicitly control barriers with "acquire" and "release" semantics. 

For those who don't want to do a quick read through the increasingly scary [memory-barriers.txt](<https://lwn.net/Articles/576489/>) file, here is a quick refresher. An "acquire" operation (usually a read) contains a barrier guaranteeing that the operation will complete before any subsequent reads or writes. A "release" operation (normally a write) guarantees that any reads or writes issued prior to the release will complete before the release operation itself completes. Acquire and release operations are thus only partial barriers. In many situations, though, they are all that is needed, and they can be less expensive than full barriers; developers seeking to maximize performance thus want to use them whenever possible. 

Will Deacon set out to provide that control with atomic operations. [The result](<https://lwn.net/Articles/651293/>) was a new set of atomic operations: 

```
int atomic_read_acquire(atomic_t *value);
        void atomic_set_release(atomic_t *value);
    
        int atomic_add_return_relaxed(int i, atomic_t *value);
        int atomic_add_return_acquire(int i, atomic_t *value);
        int atomic_add_return_release(int i, atomic_t *value);
    
        int atomic_sub_return_relaxed(int i, atomic_t *value);
        int atomic_sub_return_acquire(int i, atomic_t *value);
        int atomic_sub_return_release(int i, atomic_t *value);
    
        /*
         * And so on for atomic_xchg(), atomic_cmpxchg(),
         * xchg(), and cmpxchg().
         */
```

Will's patch also defines the 64-bit and `atomic_long_t` versions of the above functions. In each case, the "bare" version of the name (e.g. `atomic_add_return()` gives full-barrier semantics, while the `_relaxed` version provides no barrier at all. In between are the versions that include barriers with acquire or release semantics. 

The [first use](<https://lwn.net/Articles/651632/>) of these new primitives is with [the queued reader/writer lock code](<https://lwn.net/Articles/579729/>). Assuming they are merged, they will likely find their way into other performance-sensitive parts of the kernel in short order. That should be good for the speed of the system (though no benchmark numbers have been posted), but it comes at the cost of requiring more developers to understand the details of how the barrier semantics work. It is becoming increasingly hard to hide these details in architecture-specific code over time. As the complexity of our systems grows, the complexity of the software will have to increase as well.
