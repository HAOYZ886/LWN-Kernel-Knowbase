---
title: "Lockless patterns: relaxed access and partial memory barriers"
url: https://lwn.net/Articles/846700/
date: "February 26, 2021"
category: Lockless algorithms; Memory barriers
author: "February 26, 2021 This article was contributed by Paolo Bonzini Lockless patterns"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

February 26, 2021

This article was contributed by Paolo Bonzini

* * *

[Lockless patterns](<https://lwn.net/Articles/844224/>)

The [first article in this series](<https://lwn.net/Articles/844224/>) provided an introduction to lockless algorithms and the _happens before_ relationship that allows us to reason about them. The next step is to look at the concept of a "data race" and the primitives that exist to prevent data races. We continue in that direction with a look at relaxed accesses, memory barriers, and how they can be used to implement the kernel's seqcount mechanism. 

Memory barriers are an old acquaintance for some Linux kernel programmers. The first document vaguely resembling a specification of what one could expect from concurrent accesses to data in the kernel is, in fact, called [`memory-barriers.txt`](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/memory-barriers.txt>). That document describes many kinds of memory barriers, along with the expectations that Linux has concerning the properties of data and control dependencies.  It also describes "memory-barrier pairing"; this could be seen as a cousin of release-acquire pairing, in that it also helps creating cross-thread _happens before_ edges. 

This article will not go into the same excruciating detail as `memory-barriers.txt`. Instead, we'll look at how barriers compare with the acquire and release model and how they simplify or enable the implementation of the seqcount primitive. Nevertheless, one article will not be enough to cover even the most common memory-barrier patterns, so full memory barriers will have to wait for the next installment. 

#### Data races, relaxed accesses, and memory barriers

The concept of a data race, as presented here, was first introduced in C++11 and, since then, has been applied to various other languages, notably C11 and Rust. These language standards are quite strict in how to approach lockless access to data structures, and they introduce specific _atomic load_ and _atomic store_ primitives to do so. 

A data race occurs when two accesses are concurrent (i.e., not ordered by the _happens before_ relation), at least one of them is a store, and at least one of them is not using atomic load or store primitives. Whenever a data race occurs, the result (according to C11/C++11) is undefined behavior, meaning that anything is allowed to happen. Avoiding data races does not mean that your algorithm is free of "race conditions": data races are violations of the language standards, while race conditions are bugs that stem from incorrect locking, incorrect acquire/release semantics, or both. 

Data races and the consequent undefined behavior are easy to avoid, however. As long as one wants to store to a shared data location, which is probably the case, there are two ways to do so. The first is to ensure that accesses are ordered by the _happens before_ relation, using any pair of acquire and release operations of your choice; the second is to annotate the loads and stores as atomic. 

C11, C++11, and Rust all provide various _memory orderings_ that the programmer can use for their loads and stores; the three that we are interested in are acquire (to be used with loads), release (for stores), and relaxed (for both). Acquire and release should be self-explanatory by now, and Linux provides the same concept in its `smp_load_acquire()` and `smp_store_release()` operations. Relaxed operations, instead, do not provide any cross-thread ordering; a relaxed operation does not create a _happens before_ relationship. Instead, these operations have essentially no purpose other than to avoid data races and the undefined behavior associated with them. 

In practice, Linux expects both the compiler and the CPU to allow a bit more leeway than the language standards do. In particular, the kernel expects that regular loads and stores will not trigger undefined behavior just because there is a concurrent store. However, the value that is loaded or stored in such situations is still not well defined and may well be garbage. For example, the result could include parts of an old value and parts of a new value; this means that, at the very least, dereferencing pointer values loaded from a data race is generally a bad idea. 

In addition, regular loads and stores are subject to compiler optimizations, which can produce surprises of their own. Therefore the idea of a relaxed-ordering — but guaranteed atomic — memory operation is useful in Linux too; this is what the `READ_ONCE()` and `WRITE_ONCE()` macros provide. In general the remainder of this series will always use `READ_ONCE()` and `WRITE_ONCE()` explicitly, which nowadays is considered good practice by Linux developers. 

These macros already appeared in an example from part 1: 

```
thread 1                          thread 2
        -----------------------------     ------------------------
        a.x = 1;
        smp_wmb();
        WRITE_ONCE(message, &a);          datum = READ_ONCE(message);
                                          smp_rmb();
                                          if (datum != NULL)
                                            printk("%x\n", datum->x);
```

They are used in a similar way to `smp_load_acquire()` and `smp_store_release()`, but their first argument is the target of the assignment (an _lvalue_) rather than a pointer. Unless other mechanisms ensure that the result of a data race is thrown away, it is highly recommended to use `READ_ONCE()` and `WRITE_ONCE()` to load and store shared data outside a lock. Typically, relaxed atomics are used together with some other primitive or synchronization mechanism that has release and acquire semantics; that "something else" will order the relaxed writes against reads of the same memory location. 

Suppose, for example, that you have multiple work items that fill certain elements of an array with ones; whoever spawned the work items will only read the array after calling `flush_work()`. Similar to `pthread_join()`, `flush_work()` has acquire semantics and synchronizes with the end of the work item; `flush_work()` guarantees that reading the array will _happen after_ the completion of the work items, and the array can be read with regular loads. However, if multiple, concurrent work items can store into the same array element, they must use `WRITE_ONCE(a[x], 1)` rather than just `a[x] = 1`. 

A more complicated case occurs when the release and acquire semantics are provided by a memory barrier. In order to explain this case we will use the practical example of seqcounts. 

#### Seqcounts

[Seqcounts](<https://lwn.net/Articles/831540/>) are a specialized primitive that allows a consumer to detect that a data structure changed in the middle of the consumer's access. While they are only usable in special cases (small amount of data being protected, no side effects within read-side critical sections, and writes being quick and relatively rare), they have various interesting properties: in particular, readers do not starve writers and the writer can keep [ownership](<https://en.wikipedia.org/wiki/MOESI_protocol>) of the cache line that holds the seqcount. Both properties make seqcounts particularly interesting when scalability is important. 

Seqcounts are a single-producer, multiple-consumer primitive. In order to avoid multiple concurrent writers, they are usually combined with a spinlock or mutex, forming the familiar Linux `seqlock_t` type. Sometimes, outside the kernel, you'll see seqcounts referred to as seqlocks, however. 

Seqcounts are effectively a generation count, where the generation number is odd if and only if a write is in progress. Whenever the generation number was odd at the beginning of a read-side critical section, or it changed during the read-side critical section, the reader has accessed potentially inconsistent state and must retry the read. For a seqcount to work correctly, the reader must detect correctly the beginning and the end of the write. This requires two load-acquire and two store-release operations; here is how one might write a seqcount reader and writer without any wrapper APIs: 

```
thread 1 (buggy writer)             thread 2 (buggy reader)
        --------------------------------    ------------------------
        WRITE_ONCE(sc, sc + 1);             do {
        smp_store_release(&data.x, 123);        old = smp_load_acquire(&sc) & ~1;
        WRITE_ONCE(data.y, 456);                copy.y = READ_ONCE(data.y);
        smp_store_release(&sc, sc + 1);         copy.x = smp_load_acquire(&data.x);
                                            } while(READ_ONCE(sc) != old);
```

This code is similar to the "message passing" pattern shown in the first part of the series. There are two pairs of load-acquire and store-release operations, one set for `sc` and one for `data.x`. It is not even that hard to show why both load-acquire/store-release pairs are necessary: 

  * For thread 2 to exit the loop, the first read of `sc` must see the even value that was written by the second store to `sc` in thread 1\. If this happens, `smp_store_release()` and `smp_load_acquire()` ensure that the stores to the `data` fields will be visible. 
  * If the store to `data.x` is visible when thread 2 reads it, `smp_store_release()` and `smp_load_acquire()` ensure that thread 2 will see (at least) the first generation-count update. Thus, thread 2 will either loop or, if it also sees the second update, retrieve a consistent copy of the new data as described above. 

However, the code has a bug! Because the writer has no acquire operation at the top of the sequence, the write to `data.y` might execute before writing the odd value to `sc`. [**Note** : the article was updated on March 2nd to point out this problem]. Using load-acquire/store-release for all fields would sidestep the issue, but one wonders if this would be putting lipstick on a pig. And in fact it is possible to do much better. 

The first article showed that older Linux code may use `smp_wmb()` followed by `WRITE_ONCE()` rather than `smp_store_release()` ; likewise, instead of `smp_load_acquire()`, sometimes `READ_ONCE()` is followed by `smp_rmb()`. These partial barriers create specific types of _happens before_ relations. Specifically (but informally), `smp_wmb()` turns all the following relaxed stores into release operations and `smp_rmb()` turns all the preceding relaxed loads into acquire operations. 

Let's try to apply this replacement to the `data.x` accesses: 

```
thread 1 (writer)                   thread 2 (reader)
        ------------------------------      ------------------------
        // write_seqcount_begin(&sc)        do {
        **WRITE_ONCE(sc, sc + 1);**                 // read_seqcount_begin(&sc)
        **smp_wmb();**                              **old = smp_load_acquire( &sc) & ~1;**
        WRITE_ONCE(data.x, 123);                copy.y = READ_ONCE(data.y);
        WRITE_ONCE(data.y, 456);                copy.x = READ_ONCE(data.x);
        // write_seqcount_end(&sc)              // read_seqcount_retry(&sc, old)
        **smp_store_release( &sc, sc + 1);**         **smp_rmb();**
                                            } while(**READ_ONCE(sc) != old**);
```

Leaving aside the question of how barriers work, this already has much better chances of being wrapped with an easy-to-use API. Data is accessed entirely with relaxed atomic loads and stores (though in the Linux kernel memory model non-atomic accesses would be acceptable too), and the barriers could be hidden within the seqcount primitives `read_seqcount_retry()` and `write_seqcount_begin()`. 

The barriers inserted above split the reads and writes into two separate groups; this ensures the safety of seqcount accesses. However, there are two complications: 

  * First, the barriers do not impose an order among the relaxed accesses themselves. It is possible that thread 2 sees the update to `data.y` and not the update to `data.x`. This is not a problem for seqcounts, because the check on `sc` forces thread 2 to retry in case it saw only some of the stores. 
  * Second, the barriers are weaker than load-acquire and store-release operations. A read with `smp_load_acquire()` happens before any loads _and stores_ that follow it, and likewise `smp_store_release()` happens after not just preceding stores, but also after preceding loads. Instead, for `smp_rmb()`, the ordering is only guaranteed among loads, and, for `smp_wmb()`, only among stores. Load-store ordering however rarely matters, which is why Linux developers only used `smp_rmb()` and `smp_wmb()` for a long time. 

In the case of seqcounts, load-store ordering is not a problem, because the reader does not perform any writes to shared memory in its critical section, thus there cannot be concurrent changes to shared memory between the writer's updates of the generation count. There's a little bit of handwaving in this reasoning, but it is actually sound as long as the code is kept simple and faithful to the pattern. If the reader needed to write to shared memory, it would suffice to protect those writes with a different mechanism than the seqcount. 

While informal, the explanation in the previous paragraph highlights the importance of knowing the common lockless programming patterns. In short, patterns enable thinking about code at a more coarse level without losing precision. Instead of looking at each memory access individually, you can make statements like "`data.x` and `data.y` are protected by the seqcount `sc`" or, referring to the earlier message passing example, "`a` is published to other threads via `message`". To some extent, proficiency in lockless programming patterns means being able to make such statements and take advantage of them to understand the code. 

This concludes our initial look at memory barriers. There is a lot more to this topic than has been covered so far, naturally; the next installment in this series will delve into full memory barriers, how they work, and how they are used in the kernel.
