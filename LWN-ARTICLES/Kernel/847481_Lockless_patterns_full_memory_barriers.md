---
title: "Lockless patterns: full memory barriers"
url: https://lwn.net/Articles/847481/
date: "March 5, 2021"
category: Lockless algorithms
author: "March 5, 2021 This article was contributed by Paolo Bonzini Lockless patterns"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

March 5, 2021

This article was contributed by Paolo Bonzini

* * *

[Lockless patterns](<https://lwn.net/Articles/844224/>)

The first two articles in this series introduced four ways to order memory accesses: load-acquire and store-release operations in the [first installment](<https://lwn.net/Articles/844224/>), read and write memory barriers in the [second](<https://lwn.net/Articles/846700/>). The series continues with an exploration of full memory barriers, why they are more expensive, and how they are used in the kernel. 

The primitives that were introduced so far constrain the ordering of loads and stores in four different ways: 

  * Load-acquire operations are ordered before subsequent loads and stores. 
  * Store-release operations are ordered after earlier loads and stores. 
  * Read memory barriers order earlier loads against subsequent loads. 
  * Write memory barriers order earlier stores against subsequent stores. 

It may be surprising that no combination of these operations orders an earlier store against a later load: 

> | Ordered against  
> ---|---  
> | Load| Store  
> Earlier load | `smp_load_acquire()`,  
>  `smp_rmb()` | `smp_load_acquire()`,   
> `smp_store_release()`  
> Earlier store | ?? | `smp_store_release()`,   
>  `smp_wmb()`  
  
It turns out that guaranteeing a global ordering of stores against later loads is much more complicated for the processor, and it deserves a primitive of its own. To find out why, we need to abandon even the high-level concepts that we have used so far and take a direct peek at how processors operate. 

#### How processors really do it

The first article already mentioned that, deep down, processors communicate via a message-passing architecture, such as [QPI](<https://en.wikipedia.org/wiki/Intel_QuickPath_Interconnect>) or [HyperTransport](<https://en.wikipedia.org/wiki/HyperTransport>). At the assembly-language level, however, the programmer sees operations like memory loads and stores. Any acquire and release semantics associated with these memory operations are an illusion that is provided by the processor's execution pipeline, based on the program it runs and on the constraints of the architecture. 

For example, on x86 processors, all memory loads count as "load-acquire", and all memory stores count as "store-release"; this behavior is required by the architecture. When compiling for these processors, it's still important to annotate acquire and release operations and/or to insert memory barriers, but these annotations are only used by the compiler to block invalid optimizations. Even in this model, the architecture does not guarantee that all processors see stores and loads in the same order. Consider this example, where `a` and `b` are initially zero: 

```
CPU 1                    CPU 2
        -------------------      --------------------
        store 1 into a           store 1 into b
        load b into x            load a into y
```

If you try interleaving the four operations by hand, you'll see that at least one store will come before the corresponding load. Therefore one would expect that at least one of `x` and `y` will be 1 after the above sequence runs to completion. Yet, even on an x86 processor, it is possible that both `x` and `y` read as zero. 

How so? The reason lies in the existence of the _store buffer_ , which lives between the CPU and its L1 cache. Stores to memory usually only change one part of a cache line; completing those stores, even just to cache, may require fetching the full cache line from memory — a slow operation. The store buffer holds the stored data while this fetch takes place, allowing the CPU to proceed with execution. 

Even a CPU with a store buffer can provide load-load, load-store, and store-store ordering relatively easily: 

  * On out-of-order CPUs (which can execute operations in an order different from how they appear in the code to improve performance), ordering memory operations against earlier loads can be done via speculative execution. During the time between a cache line access and the retiring of the instruction that caused the access, the CPU tracks evictions of that cache line. Retiring of instructions happens in order, so this tracking will last until all earlier loads have completed. If the cache line is evicted before the instruction is retired, speculation is canceled and the processor retries the memory operation. 
  * Keeping stores ordered is even simpler; it is enough to flush the entries of the store buffer to the cache in FIFO order. 

However, store-load ordering is a different story. First of all, one CPU's store might be stuck in its store buffer, and therefore it will not be visible to another CPU when that CPU loads data from L1 cache. Second, a store buffer also provides _store forwarding_ , where the result of memory loads is taken directly from the store buffer instead of going through the cache. If either CPU 1 or CPU 2 has an entry for respectively `b` or `a` in its store buffer, the value could be _forwarded_ to the load, which would return zero. 

The only way around these problems is to flush the store buffer entirely between the store and the load. This is as expensive as it sounds (a few tens of clock cycles), but this is what the _full memory barrier_ `smp_mb()` does under the hood. Here is the same pseudocode as above, fixed and rewritten into C: 

```
thread 1                 thread 2
        -------------------      --------------------
        WRITE_ONCE(a, 1);        WRITE_ONCE(b, 1);
        smp_mb();                smp_mb();
        x = READ_ONCE(b);        y = READ_ONCE(a);
```

Let's say `x` is zero. I will use a squiggly horizontal line from a read to a write to express that `WRITE_ONCE(b, 1)` overwrites the value that thread 1 had read; then the situation (which is [described in detail](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/memory-model/Documentation/explanation.txt?h=v5.11&id=f40ddce88593482919761f74910f42f4b84c004b#n682>) in the kernel's memory-model documentation) is as follows: 

```
WRITE_ONCE(a, 1);
               |
          -----+----- smp_mb();
               |
               v
        x = READ_ONCE(b);   ∿∿∿∿∿∿∿>  WRITE_ONCE(b, 1);
                                             |
                                        -----+----- smp_mb();
                                             |
                                             v
                                      y = READ_ONCE(a);
```

Because these are relaxed operations, this is not enough to introduce a happens-before edge between thread 1 and thread 2\. The barriers also do not have acquire or release semantics by themselves, so there are no happens-before edges across the two threads at all. 

However, the barriers do provide enough information to order the operations in the two threads. In particular, continuing with the case of `x=0`, the full memory barrier in thread 2 guarantees that the store buffer has been flushed by the time `READ_ONCE(a)` executes. Could this be even before `READ_ONCE(b)` executed? If so, `READ_ONCE(b)` would certainly see the earlier `WRITE_ONCE(b, 1)` from thread 2 (remember the store buffer has been flushed), and `x` would be 1. That's a contradiction, therefore `READ_ONCE(b)` must have executed first: 

```
WRITE_ONCE(a, 1);              WRITE_ONCE(b, 1);
               |                              |
               |                         -----+----- smp_mb();
               |                              |
               v                              v
        x = READ_ONCE(b); -----------> y = READ_ONCE(a);
                            (if x=0)
```

Due to transitivity, `READ_ONCE(a)` can see the effect of `WRITE_ONCE(a, 1)` and `y=1`. Likewise, if thread 2 reads `a` as zero, thread 1's full memory barrier ensures that the `READ_ONCE(a)` must have executed before thread 1's `READ_ONCE(b)`: 

```
WRITE_ONCE(a, 1);              WRITE_ONCE(b, 1);
               |                              |
          -----+----- smp_mb();               |
               |                              |
               v                              v
        x = READ_ONCE(b); <----------- y = READ_ONCE(a);
                            (if y=0)
```

meaning that `y=0` implies `x=1`. Different executions may show one or the other possibility, but in any case `x` and `y` cannot both be zero; that would mean that `READ_ONCE(a)` executed before `READ_ONCE(b)` _and_ vice versa, which is impossible. 

The Linux kernel memory model [does not call these read-read edges "happens-before"](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/memory-model/Documentation/explanation.txt?h=v5.11&id=f40ddce88593482919761f74910f42f4b84c004b#n1350>), because there's no indication of which is the release operation and which is the acquire operation; nevertheless, they have the same effect of ordering memory operations across threads. Therefore we _can_ state that high-level APIs have acquire or release semantics even if internally they use full memory barriers; users of those APIs will be able to think of their code in terms of the happens-before framework, too. The next example will show how this is done in practice. 

#### Sleep/wake-up synchronization

The detailed description in the previous section shows that full memory barriers impose constraints in a complicated and unintuitive manner. Fortunately, the above pattern of "two threads and two flags", where each thread writes to a flag and reads from the other, is probably all that you need to know about full memory barriers. 

This pattern has many important and practical applications. Suppose thread 2 wants thread 1 to act on 2's requests; thread 1 on the other hand wants to do something else—something that could even take an indefinite amount of time, for example removing itself from the scheduler's run queue and going to sleep. In this case, the code will look like this: 

```
thread 1                                   thread 2
       -------------------                        --------------------------
        WRITE_ONCE(dont_sleep, 1);                 WRITE_ONCE(wake_me, 1);
        smp_mb();                                  smp_mb();
        if (READ_ONCE(wake_me))                    if (!READ_ONCE(dont_sleep))
          wake(thread2);                             sleep();
```

If thread 2 reads `dont_sleep` as zero, thread 1 will read `wake_me` as one and wake up thread 2; it's useful to think of thread 1 as having release semantics (imagine that the `wake()` happens as part of `mutex_unlock`). If thread 1 reads `wake_me` as zero, thread 2 will read `dont_sleep` as one and will not go to sleep. Usually this is the side that needs to exhibit acquire semantics. 

There is a hidden assumption that thread 1's wake-up requests are never lost, even if for example `wake()` is called after thread 2's `READ_ONCE()` but before `sleep()`. A way to avoid this bug is for `wake()` and `sleep()` calls to take the same lock. Again, we see how lockless patterns cooperate with traditional synchronization—after all this is a slow path. 

It really does work, and we can see this, for example, in Linux's [`prepare_to_wait()`](<https://elixir.bootlin.com/linux/v5.11.1/source/kernel/sched/wait.c#L241>) and [`wake_up_process()`](<https://elixir.bootlin.com/linux/v5.11.1/source/kernel/sched/core.c#L3525>) APIs. This interface was introduced in the 2.5.x development kernels, and was [duly covered](<https://lwn.net/Articles/22913/>) back then by LWN. Here is how the pattern appears after expanding some of the functions that are involved: 

```
thread 1                                     thread 2
        -------------------                          --------------------------
        **WRITE_ONCE(condition, 1);**                    prepare_to_wait(..., TASK_INTERRUPTIBLE) {
        wake_up_process(p) {                           set_current_state(TASK_INTERRUPTIBLE) {
          try_to_wake_up(p, TASK_NORMAL, 0) {            **WRITE_ONCE(current- >state, TASK_INTERRUPTIBLE);**
            **smp_mb();**                                    **smp_mb();**
            **if (READ_ONCE(p- >state) & TASK_NORMAL)**     }
    	  **ttwu_queue(p);**                         }      
          }                                          **if (!READ_ONCE(condition))**
        }                                              **schedule();**
```

As we saw in the seqcount case, the memory barriers are hidden inside the implementation of higher-level APIs. Embedding memory barriers or load-acquire/store-release operations inside the implementation is in fact how one defines an API that has acquire and release semantics; in this case, `wake_up_process()` has release semantics, while `set_current_state()` and its caller `prepare_to_wait()` have acquire semantics. 

The sleep condition is often checked twice to limit unnecessary wakeups, like this: 

```
thread 1                               thread 2
        -------------------                    --------------------------
        WRITE_ONCE(dont_sleep, 1);             if (!READ_ONCE(dont_sleep)) {
        smp_mb();                                WRITE_ONCE(wake_me, 1);
        if (READ_ONCE(wake_me))                  smp_mb();
            wake(thread2);                       if (!READ_ONCE(dont_sleep))
                                                   sleep();
                                               }
```

In the kernel we can see this kind of check in the interaction between [`tcp_data_snd_check()`](<https://elixir.bootlin.com/linux/v5.11.1/source/net/ipv4/tcp_input.c#L5380>) and the call to `tcp_check_space()`, defined immediately above it (thread 1) and [`tcp_poll()`](<https://elixir.bootlin.com/linux/v5.11.1/source/net/ipv4/tcp.c#L496>) (thread 2). In this case, the code does not have the luxury of relying on a higher-level abstraction, so it merits a closer look. Thread 2 wants to sleep if the socket has no room in its send buffer, therefore `tcp_poll()` sets the "wake me" `SOCK_NOSPACE` flag before checking `__sk_stream_is_writeable()`. The core of the [lockless synchronization](<https://elixir.bootlin.com/linux/v5.11.1/source/net/ipv4/tcp.c#L572>) in `tcp_poll()` is: 

```
set_bit(SOCK_NOSPACE, &sk->sk_socket->flags);
        smp_mb__after_atomic();
        if (__sk_stream_is_writeable(sk, 1))
          mask |= EPOLLOUT | EPOLLWRNORM;
```

The caller will ultimately go to sleep if mask is zero. `smp_mb__after_atomic()` is a specialized version of `smp_mb()` and has the same semantics. These optimized barriers will be explained in a future article. 

Thread 1, instead, must wake up thread 2 after consuming data in the send buffer. `tcp_data_snd_check()` first sends out packets to make room in the buffer ("do not sleep, there's room now"), then checks `SOCK_NOSPACE`, and finally (through the `sk->sk_write_space()` function pointer) reaches `sk_stream_write_space()`, where thread 2 is woken up. The call stack is relatively shallow, so I'll let the reader explore the code. I will however point out this comment in `tcp_check_space()`: 

```
/* pairs with tcp_poll() */
        smp_mb();
        if (test_bit(SOCK_NOSPACE, &sk->sk_socket->flags))
          tcp_new_space(sk);
```

The reference to the "pairing" of barriers tells us that this function has either acquire or release semantics. A read or write memory barrier would tell us straight away that the function has respectively acquire and release semantics. With a full memory barrier, we need to look at the code around the barrier. In this case, we know that the function is on the "wake-up" side of the pattern and therefore has release semantics; `tcp_poll()` instead has acquire semantics. 

Something similar happens almost everywhere a `smp_mb()` can be found in the kernel. For example: 

  * Workqueues use this idiom to decide whether workers have more work to do. In this case, thread 1's role is taken by [`insert_work()`](<https://elixir.bootlin.com/linux/v5.11.1/source/kernel/workqueue.c#L1312>), while thread 2's role is taken by [`wq_worker_sleeping()`](<https://elixir.bootlin.com/linux/v5.11.1/source/kernel/workqueue.c#L857>). 
  * In the `futex()` system call, thread 1's write is in user space, while the memory barrier and read are part of `futex(FUTEX_WAKE)`. Thread 2's operation is entirely part of the `futex(FUTEX_WAIT)` (because `wake_me` is a flag in kernel memory); `FUTEX_WAIT` accepts the expected value of the futex as an argument to the system call, and uses it to decide whether to sleep. See [the long comment](<https://elixir.bootlin.com/linux/v5.11.1/source/kernel/futex.c#L48>) at the head of `kernel/futex.c` for details on how this works. 
  * Within KVM, `sleep()` is replaced by the act of entering the processor's guest mode and running a virtual machine. In order to kick the processor out of guest mode, [`kvm_vcpu_kick()`](<https://elixir.bootlin.com/linux/v5.11.1/source/virt/kvm/kvm_main.c#L2873>) sends an inter-processor interrupt to the processor. The familiar comment around memory barrier pairing can be found down the call chain in [`kvm_vcpu_exiting_guest_mode()`](<https://elixir.bootlin.com/linux/v5.11.1/source/include/linux/kvm_host.h#L326>), which also where `vcpu->mode` is read. 
  * Virtio devices use two instances of the above pattern. On one hand, the driver wants to stop processing completed requests, and waking it up consists of sending an interrupt. On the other hand, the device wants to stop processing submitted requests, and the device can wake it up by writing to a "doorbell" memory location. 

That completes the introduction of memory barriers. From here, we will look at compare-and-swap operations, how they can be combined with locks to create lock-free fast paths, and their role in the implementation of lock-free linked lists.
