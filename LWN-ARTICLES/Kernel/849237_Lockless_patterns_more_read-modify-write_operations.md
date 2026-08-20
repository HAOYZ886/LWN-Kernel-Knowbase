---
title: "Lockless patterns: more read-modify-write operations"
url: https://lwn.net/Articles/849237/
date: "March 19, 2021"
category: Lockless algorithms; cmpxchg
author: "March 19, 2021 This article was contributed by Paolo Bonzini Lockless patterns"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

March 19, 2021

This article was contributed by Paolo Bonzini

* * *

[Lockless patterns](<https://lwn.net/Articles/844224/>)

[Last week's installment](<https://lwn.net/Articles/847973/>) in this series on lockless patterns took a first look at the compare-and-swap (CAS) operation. CAS is a powerful tool that can be used to implement a number of lockless primitives. The next step is to look at other atomic read-modify-write operations that can be implemented on top of compare-and-swap. 

CAS-based primitives usually operate on `int` values. The Linux kernel uses [`atomic_t`](<https://elixir.bootlin.com/linux/v5.11.6/source/include/linux/types.h#L168>), a struct type that wraps `int` so that loads and stores are marked explicitly. For example, it is not possible to write `x++;` if `x` is an `atomic_t`. Instead one must write `atomic_inc(&x);`. All operations on `atomic_t` start with "`atomic_`". 

Linux has roughly 30 read-modify-write operations on `atomic_t`, of which compare-and-swap, in the form of `atomic_cmpxchg()`, is the real workhorse since it can be used to implement all of the others. For example, here is an "atomic increment": 

```
/* atomic_read() is like READ_ONCE(), but for Linux's atomic_t.  */
        old = atomic_read(&x);
        do {
            expected = old;
            old = atomic_cmpxchg(&x, expected, expected + 1);
        } while (old != expected);
```

Which, as you can see, is similar to the "add to list" operation that was presented earlier in this series. As a bonus, at completion, `old` contains the value that was incremented, making the above sequence equivalent to the Linux macro `atomic_fetch_inc()`. 

Instruction sets vary wildly in the number of read-modify-write instructions that they offer. Some only include compare-and-swap; x86 has many more, but only three of them (`CMPXCHG`, `XCHG`, and the "exchange and add" instruction `XADD`) return the old value of the memory location. And, even with the most comprehensive instruction set, some cases are bound to occur that the processor cannot cover. That's when compare-and-swap comes in handy. For example, we could define a read-modify-write implementation for the "maximum of two values" operator: 

```
/* Store max(x, new) into x.  */
        old = atomic_read(&x);
        do {
            expected = old;
            if (old > new)
                break;
            old = atomic_cmpxchg(&x, expected, new);
        } while (old != expected);
```

or one that increments a value only if it is non-zero: 

```
old = atomic_read(&x);
        do {
            expected = old;
            if (old == 0)
                break;
            old = atomic_cmpxchg(&x, expected, expected + 1);
        } while (old != expected);
```

Sequences similar to the latter are useful to implement _lock-free fast paths_. This lockless programming pattern lets the programmer use locks only in rare cases, thus avoiding contention most of the time. A typical example is reference counting. 

#### Lock-free reference counting

Consider a typical, reference-counted data structure that we'll call `struct gadget`. Each gadget has a parent, and keeps a reference to its parent in order to keep the parent itself alive. This is the simplest possible implementation of `get_gadget()` and `put_gadget()`, the functions that respectively increment and decrement the gadget's reference count: 

```
void get_gadget(struct gadget *g) {
            mutex_lock(&gadgets_lock);
            g->refcnt++;
            mutex_unlock(&gadgets_lock);
        }
    
        void put_gadget(struct gadget *g) {
            mutex_lock(&gadgets_lock);
            if (g->refcnt-- == 1) {
                mutex_unlock(&gadgets_lock);
                put_gadget(g->parent);
                kfree(g);
                return;
            }
            mutex_unlock(&gadgets_lock);
        }
```

However, this would be unnecessarily inefficient. When obtaining an additional reference to a gadget `g`, the thread that calls `get_gadget()` must already have a reference to `g`. Furthermore, that reference cannot go away until `get_gadget()` returns; therefore, any concurrent call to `put_gadget()` would never go down the `kfree()` branch. We can do the following instead and get rid of the lock: 

```
void get_gadget(struct gadget *g) {
            /*
             * Unlike atomic_fetch_dec(), this increment is atomic but has
             * no acquire or release semantics.  This is true of all Linux
             * atomic operations that do not return a value.
             */
            atomic_inc(&g->refcnt);
        }
    
        void put_gadget(struct gadget *g) {
            /*
             * Like atomic_cmpxchg(), this has both acquire and release semantics.
             */
            if (atomic_fetch_dec(&g->refcnt) > 1)
                return;
    
            put_gadget(g->parent);
            kfree(g);
        }
```

Linux provides [`kref_get()` and `kref_put()`](<https://elixir.bootlin.com/linux/v5.11.6/source/include/linux/kref.h#L39>) as a simple abstraction for this idiom. There are two things worth noting in the above code: 

  * `put_gadget()` is not using `atomic_fetch_dec_release()`; decrementing the reference count is done with both acquire and release semantics. Just like in the [lock-free list example](<https://lwn.net/Articles/847973/#why-not-just-release>), this weaves the "happens before" relation through all the calls to `put_gadget()`, so that when reaching for other gadgets via `g->parent`, it is possible to use regular loads. 
  * The only synchronization point is in `put_gadget()`; `get_gadget()` does need the increment to be atomic, but it does not need to have acquire semantics. This is because the calling thread must already have a reference to `g`. Acquire and release semantics had to be involved whenever the thread initially got that reference, for example at thread creation; after that point, however, the thread proceeds independently until it needs to give back the reference and potentially free the gadget. 

Now, imagine that each gadget also stored a list of sub-gadgets. Just before destroying itself, the gadget could remove itself from its parent's list of sub-gadgets: 

```
void put_gadget(struct gadget *g) {
            if (atomic_fetch_dec(&g->refcnt) > 1)
                return;
    
            mutex_lock(&g->parent->gadgets_lock);
            list_del(&g->node);
            mutex_unlock(&g->parent->gadgets_lock);
            put_gadget(g->parent);
            kfree(g);
        }
```

However, as is often the case, the code above only tells half of the story. The gadget has a reference to its parent; if the parent held a reference to each of its children as well, this would create a reference-count cycle and leak memory. Therefore, the list must contain pointers to the children without having a reference to them; sometimes you'll hear that the list has _weak_ references to the children. Threads are free to visit the list and operate on the gadgets therein, but they cannot call `get_gadget()` because they do not already have a reference. That means that threads cannot prevent child gadgets from going away at a surprising time. 

The simplest solution is to operate on weak references only within the protection of the lock; the gadgets in the list will not disappear until `put_gadget()` can take the lock and remove the gadget from the list. If this is too restrictive, however, we can instead refine the rules for calling `get_gadget()`. The following would be a workable alternative: 

  * As before, a thread that already has a strong reference can obtain an extra reference with `get_gadget()`. 
  * A thread can upgrade a weak reference to strong by calling `get_gadget()` _while holding the lock that protects the list_. 

The implementation of `get_gadget()` is still the same one-liner, but `put_gadget()` must be more careful: it must take the lock _before_ decrementing the reference count from one to zero, and not release the lock until it has deleted itself from the list. This way, visitors to the list will never find weak references to gadgets whose reference count is zero. If the reference count is greater than one, however, `put_gadget()` can proceed locklessly. This is how it could be implemented using compare-and-swap: 

```
void put_gadget(struct gadget *g) {
            for (;;) {
                int old = atomic_read(&g->refcnt);
                if (old > 1) {
                    if (atomic_cmpxchg(&g->refcnt, old, old - 1) == old)
                        return;
                } else {
                    /* old was 1, fence off accesses to weak references!  */
                    mutex_lock(&g->parent->gadgets_lock);
                    if (atomic_cmpxchg(&g->refcnt, 1, 0) == 1)
                        break;
    
                    /*
                     * Somebody snuck in and upgraded a weak reference before the
                     * mutex_lock().  Try again.
                     */
                    mutex_unlock(&g->parent->gadgets_lock);
                }
            }
    
            list_del(&g->node);
            mutex_unlock(&g->parent->gadgets_lock);
            put_gadget(g->parent);
            kfree(g);
        }
```

Read the code again and again until you can convince yourself that every call to `put_gadget()` will result in exactly one successful compare-and-swap operation. 

This is a common pattern in Linux, and the functions [`kref_put_mutex()` and `kref_put_lock()`](<https://elixir.bootlin.com/linux/v5.11.6/source/include/linux/kref.h#L70>) help developers implement it. As with `llist.h`, you are strongly encouraged not to roll your own and to use the library functions instead. The alert reader will have noticed that there is no `kref_put_rwsem()` function, which might be handy if a `struct rw_semaphore` protects the list of sub-gadgets. It may be a good exercise to sit down and try to write that function. 

Sometimes there is no lock that could protect both the access and the destruction. This happens if the weak reference is held by a separate subsystem, for example by a [debugfs](<https://lwn.net/Articles/334546/>) inode's `i_private` field. In this case, the upgrade operation must be allowed to fail if the reference count is zero. [`refcount_inc_not_zero()`](<https://elixir.bootlin.com/linux/v5.11.6/source/include/linux/refcount.h#L229>) and [`kref_get_unless_zero()`](<https://elixir.bootlin.com/linux/v5.11.6/source/include/linux/kref.h#L92>) can be helpful in this scenario, and you can see the former at work in [`kvm_debugfs_open()`](<https://elixir.bootlin.com/linux/v5.11.6/source/virt/kvm/kvm_main.c#L4531>): 

```
/*
         * The debugfs files still hold a reference to the kvm struct at the
         * time kvm_destroy_vm is called.  The files are removed, and the
         * reference disappears, before kvm_destroy_vm frees the kvm struct.
         *
         * To avoid a race between the opening and the removal of the debugfs
         * files, return -ENOENT if kvm_destroy_vm is in progress.
         */
        if (!refcount_inc_not_zero(&stat_data->kvm->users_count))
            return -ENOENT;
```

Compared to the "increment if not zero" implementation at the top of the article, `refcount_inc_not_zero()` is more complicated in order to implement [overflow protection](<https://lwn.net/Articles/706498/>)--further reinforcing the importance of using higher-level primitives when available. 

#### Conclusions

A few loose ends and simplifications that were made throughout the series will be covered in the next article, but this mostly concludes our introduction to lockless programming patterns. Outside of complex parts of Linux, such as the scheduler or read-copy-update (RCU), these synchronization primitives and patterns should cover almost all of the lockless code that you will encounter. My goal with these articles was to help you to understand the basic ideas and how the high-level APIs wrap those ideas, so that you can apply them even in slightly different cases. I hope they will be useful as both learning material and a reference. 

[The author would like to thank Jon Corbet, Laszlo Ersek, and Stefan Hajnoczi for help proofreading the drafts of these articles.]
