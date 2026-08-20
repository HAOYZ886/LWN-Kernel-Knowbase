---
title: "Lockless patterns: some final topics"
url: https://lwn.net/Articles/850202/
date: "March 29, 2021"
category: Lockless algorithms
author: "March 29, 2021 This article was contributed by Paolo Bonzini Lockless patterns"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

March 29, 2021

This article was contributed by Paolo Bonzini

* * *

[Lockless patterns](<https://lwn.net/Articles/844224/>)

So far, this series has covered five common lockless patterns in the Linux kernel; those are probably the five that you will most likely encounter when working on Linux. Throughout this series, some details have been left out and some simplifications were made in the name of clarity. In this final installment, I will sort out some of these loose ends and try to answer what is arguably the most important question of all: when should you use the lockless patterns that have been described here? 

#### Data, address, and control dependencies

In the code examples that have been presented so far, the ordering of instructions in one thread was enforced either by acquire and release semantics or by memory barriers. Sometimes, however, it is simply impossible for the processor and/or the compiler to reorder instructions, because an instruction has a _dependency_ on another instruction that comes before. In these cases, ordering between pairs of instructions is ensured even for relaxed loads and stores. These dependencies come in three forms. 

**Data dependencies** exist when a write stores a value that comes from a previous load, or is based on such a value: 

```
int x = READ_ONCE(a);
        WRITE_ONCE(b, **x** + 1);
```

Data dependencies are the simplest of these three cases; the store cannot execute before the load, or the processor would not know the value that the store must write. Therefore, in this case, the read from `a` behaves as a load-acquire, at least with respect to the following `WRITE_ONCE()` statement. Unlike `smp_load_acquire()` or `smp_rmb()`, an ordering created by a data dependency is not guaranteed against _all_ following memory operations. In the following example, the read of `c` might be reordered before the read of `a`, but the write of `b` cannot be: 

```
int x = READ_ONCE(a);
        WRITE_ONCE(b, x + 1);
        int y = READ_ONCE(c);
```

However, we've already seen that, most of the time, lockless code only cares about the ordering between specific memory operations; in some cases, loads can use `READ_ONCE()` safely because the only users of the value and the only operations that need to be ordered are data-dependent stores. That said, none of the patterns presented throughout the series have a store that follows a load in the same thread, so this optimization is not applicable. 

**Address dependencies** exist when a read or a write operates on a location whose address comes from a previous load, or is based on such a value: 

```
int x = READ_ONCE(a);
        int y = READ_ONCE(b[**x**]);
    
        struct s *x = READ_ONCE(a);
        WRITE_ONCE(**x** ->f, 1);
```

Address dependencies are, in theory, pretty simple as well. In the first example, the read of `b[x]` cannot occur prior to the read of `x`; in the second, the address to be written to cannot be known until the read is complete. 

Similarly, in this code from part 1, the address of `datum->x` is only known after `datum` is loaded from `message`: 

```
thread 1                            thread 2
        --------------------------------    ------------------------------------
        a.x = 1;
        smp_store_release(&message, &a);    datum = smp_load_acquire(&message);
                                            printk("%x\n", datum->x);
```

How could the CPU fetch a value from memory, or store it, without knowing which location to operate on? Therefore, one would expect that it would be possible to write the first line of thread 2 as follows, without a load-acquire instruction: 

```
datum = READ_ONCE(message)
```

This is a reasonable expectation, and it is also true at the assembly language level for all processors that Linux has been ported to--except the Alpha, due to a peculiar cache architecture. As of version 4.15, Linux developers decided that it's okay to make `READ_ONCE()` more expensive on the Alpha, and therefore address-dependent loads are now always ordered after the loads they depend on. Note that address-dependent stores have never been a problem. 

Finally, **control dependencies** exist when the value obtained by a read may cause a read or a write to not execute at all: 

```
int y = 0;
        if (READ_ONCE(a))
            y = READ_ONCE(b);        // *no* ordering here
    
        if (READ_ONCE(a))
            WRITE_ONCE(b, 1);        // write is ordered here
```

When a control dependency exists from one load to another load (as in the first example above), it will never introduce any ordering. A CPU can always perform the load speculatively before the test and ignore the result if the "then" branch turns out not to be executed. Control dependencies from a load to a store are the tricky ones this time. The CPU is not a problem, because the cache cannot store the new datum before it knows whether the store should actually happen; the problem is the compiler. Consider code like this: 

```
x = READ_ONCE(a);
        smp_store_release(&b, 1);
        if (x)
            do_something();
        else
            do_something_else();
```

A clever programmer might want to write it like this: 

```
x = READ_ONCE(a);
        if (x) {
            WRITE_ONCE(b, 1);
      	do_something();
        } else {
            WRITE_ONCE(b, 1);
      	do_something_else();
        }
```

and, indeed, the control dependency would cause the CPU to impose an ordering between reading `a` and writing `b`. However, the compiler could decide to hoist the writes before the conditional, and only test `x` afterward, leading to generated code that looks like: 

```
x = READ_ONCE(a);
        WRITE_ONCE(b, 1);
        if (x)
            do_something();
        else
            do_something_else();
```

Now the control dependency is gone and both the CPU and the compiler _can_ move the store before `a` is read. As is often the case, these issues with control dependencies are solved simply by avoiding them and annotating each store properly as either acquire or release. 

Of all these cases, address-dependent loads are probably the only ones that you will encounter in practice. The most common and self-explanatory case is retrieving a data structure with `rcu_dereference()` and `srcu_dereference()` and then reading its contents; on the Alpha, these RCU and SRCU primitives include the required memory barriers even on Linux versions prior to 4.15. However, you should be alert in the occasional case where RCU is not used and therefore both memory accesses use `READ_ONCE()`. 

#### Optimized memory barriers

[Part 5](<https://lwn.net/Articles/849237/>) introduced atomic read-modify-write operations such as `atomic_inc()`. Linux defines operations that do not return a value to have relaxed semantics. Failed compare-and-swap operations also do not imply any memory barrier, while a successful compare-and-swap behaves as if the programmer had written a memory barrier on each side of the operation. 

Some processors, however, do not have an instruction for a relaxed, read-modify-write operation. On those processors, writing something like this (a variation on the full memory barrier pattern from [part 3](<https://lwn.net/Articles/847481/>)) would be wasteful, with `set_bit()` (which must read the target location in order to change only the specified bit) and `smp_mb()` providing back-to-back memory barriers: 

```
set_bit(&flag1, FLAG_DONT_SLEEP);
        smp_mb();
        if (READ_ONCE(wake_me))
            wake(thread2);
```

For this purpose, Linux defines the optimized memory barriers `smp_mb__before_atomic()` and `smp_mb__after_atomic()`. They compile as either compiler-only barriers or full memory barriers, depending on architecture-specific details in the implementation of atomic read-modify-write operations. For example, all x86 read-modify-write operations imply a barrier on each side, therefore these optimized memory barriers are a compiler-only barrier on that architecture. On ARM, instead, it is possible to define relaxed read-modify-write operations and, therefore, the optimized memory barriers will emit a [`dmb`](<https://developer.arm.com/documentation/dui0489/c/arm-and-thumb-instructions/miscellaneous-instructions/dmb--dsb--and-isb>) instruction. They are used as a drop-in replacement for `smp_mb()`: 

```
set_bit(&flag1, FLAG_DONT_SLEEP);
        **smp_mb__after_atomic();**
        if (READ_ONCE(wake_me))
            wake(thread2);
```

Another optimized memory barrier is `smp_store_mb()`, which is a replacement for `WRITE_ONCE()` followed by `smp_mb()`. For example: 

```
thread 1                               thread 2
        -------------------                    --------------------------
        **smp_store_mb(dont_sleep, 1);**           **smp_store_mb(wake_me, 1);**
        if (READ_ONCE(wake_me))                if (!READ_ONCE(dont_sleep))
            wake(thread2);                         sleep();
```

#### When to use lockless patterns?

Even though the previous installments of this series tried to use actual Linux kernel code in the examples, one could say that they only showed the theory. While the material in the articles should be enough to understand existing code, there was very little explanation of when to employ these patterns and why. 

One thing that I did try to stress throughout the series is that lockless techniques are not an alternative to traditional synchronization primitives. They are only a means to an end, which is to limit the cost of synchronization (cacheline contention is also a kind of synchronization; it just happens in the processor rather than in your code). [Expensive synchronization in concurrent code cannot be eliminated](<https://lwn.net/Articles/423994/>), but it is possible to limit the number of expensive instructions, or to move them out of the hottest paths. To this end, some design points that you should consider are: 

  * The interaction with existing locks and shared data accesses.
  * The frequency of writes to shared data: every time a thread writes to a shared location, cache coherency traffic can end up creating a potential scalability bottleneck.
  * The frequency of synchronization: the best way to keep multiple threads going is for them to be as independent as possible, because synchronization will always introduce overhead.

Let's say you want to gather some statistics while your code runs, for example counting how many packets are sent through a network interface, and you want the overhead to be minimal. Before rushing to implement the counter in a lockless manner, for example with an atomic increment instruction, you should investigate the ways in which sending a packet can be serializing. If sending a packet already takes a spinlock or mutex, for example, there is likely no performance to gain from a fancy implementation of the counter: if you ensure that the counter resides in a cache line that is already needed when sending a packet, incrementing a single memory location while the lock is already held will be almost free. 

If there is no single lock that is always taken when sending a packet, lockless techniques may indeed be beneficial, but there's much more to them than atomic read-modify-write operations. For example, you could use multiple counters, so that you can increment them without a lock (making them per-CPU) or under a lock that you already take (say, per network queue). The counters can be summed in the (presumably rare) event of someone reading the statistics: this solution avoids concurrent writes to shared data and does not need any additional synchronization on the hot path. 

In the example above, a coarse lock--for example, a lock that covers the operation of a network queue--does not necessarily imply loss of scalability: in a system that is designed to keep the threads mostly independent, contention on coarse locks will often be rare or nonexistent. Conversely, papering over an excessive amount of shared-data access with fine-grained locks can increase the cost of synchronization substantially, and bring performance down. 

The cost of fine-grained locking is especially visible with read/write locks, where even the read side needs to write to the lock in order to take it. When writing scalable code, it can be useful to think of read/write locks as "shared/exclusive" locks. You can use coarse read/write locks to make sure that only hot paths have to handle concurrent execution: if less frequently executed code takes the lock for exclusive access, any lockless fast paths need not take that code into account at all. An example of this is the Linux `mmap_sem`; Linux performs many page table manipulations while holding it for "reading". But still, the relatively high cost of taking `mmap_sem` for reading has made it [a known problem](<https://lwn.net/Articles/787629/>) for scalability. 

If fine-grained locking is needed for specific data structures, writes to these data structures will usually be rare in a scalable system. Instead of a read/write lock, you can try to protect the read-side critical sections with mechanisms such as seqlocks or RCU. [SRCU](<https://lwn.net/Articles/202847/>) can also be an interesting alternative to RCU whenever writers cannot bear the cost of `synchronize_rcu()` (think of it as _subsystem_ RCU, not just _sleepable_ RCU). 

Even if the threads operate independently, there may be rare cases where they have to interact. In an implementation that needs to check a flag thousands or millions of times a second, taking a fine-grained lock around each check, the cost might become visible no matter how small and rare these interactions are. In these cases, applying lockless techniques to the fast path can be valuable. 

For example, you could give each thread a queue of requests from other threads and manage them through [single-consumer linked lists](<https://lwn.net/Articles/847973/>). Perhaps you can trigger the processing of requests using the [cross-thread notification](<https://lwn.net/Articles/847481/>) pattern from the article on full memory barriers. However, these techniques only make sense because the design of the whole system supports them. In other words, in a system that is designed to avoid scalability bottlenecks, common sub-problems tend to arise and can often be solved efficiently using the patterns that were presented here. 

When seeking to improve the scalability of a system with lockless techniques, it is also important to distinguish between _lock-free_ and _wait-free_ algorithms. Lock-free algorithms guarantee that the system as a whole will progress, but do not guarantee that each thread will progress; lock-free algorithms are rarely fair, and if the number of operations per second exceeds a certain threshold, some threads might end up failing so often that the result is a livelock. Wait-free algorithms additionally ensure per-thread progress. Usually this comes with a significant price in terms of complexity, though not always; for example [message passing](<https://lwn.net/Articles/844224/>) and cross-thread notification are both wait-free. 

Looking at the Linux `llist` primitives, `llist_add()` is lock-free; on the consumer side, `llist_del_first()` is lock-free, while `llist_del_all()` is wait-free. Therefore, `llist` may not be a good choice if many producers are expected to contend on calls to `llist_add()`; and using `llist_del_all()` is likely better than `llist_del_first()` unless constant-time consumption is an absolute requirement. For some architectures, the instruction set does not allow read-modify-write operations to be written as wait-free code; if that is the case, `llist_del_all()` will only be lock-free (but still preferable, because it lets the consumer perform fewer accesses to the shared data structure). 

In any case, the definitive way to check the performance characteristics of your code is to benchmark it. Intuition and knowledge of some well-known patterns can guide you in both the design and the implementation phase, but be ready to be proven wrong by the numbers. 

I'll conclude this series with a quote of Dave Chinner's [excellent critique](<https://lwn.net/Articles/849239/>): 

> This is the art of concurrent programming--it's not enough just to know what a lockless algorithm is, you need to understand the data access patterns those algorithms result in and when those access patterns are going to become a limitation to the software. Of course, knowing when not to use a lockless algorithm because there are better ways to reduce shared data access is also part of the art.
