---
title: The trouble with volatile
url: https://lwn.net/Articles/233479/
date: "May 9, 2007"
category: Coding style; volatile
author: 
---

Your editor's copy of _The C Programming Language, Second Edition_ (copyright 1988, still known as "the new C book") has the following to say about the `volatile` keyword: 

The purpose of volatile is to force an implementation to suppress optimization that could otherwise occur. For example, for a machine with memory-mapped input/output, a pointer to a device register might be declared as a pointer to `volatile`, in order to prevent the compiler from removing apparently redundant references through the pointer. 

C programmers have often taken `volatile` to mean that the variable could be changed outside of the current thread of execution; as a result, they are sometimes tempted to use it in kernel code when shared data structures are being used. Andrew Morton recently [called out](<https://lwn.net/Articles/233481/>) use of `volatile` in a submitted patch, saying: 

The volatiles are a worry - volatile is said to be basically-always-wrong in-kernel, although we've never managed to document why, and i386 cheerfully uses it in readb() and friends. 

In response, Randy Dunlap [pulled together some email from Linus](<https://lwn.net/Articles/233482/>) on the topic and suggested to your editor that he could maybe help "document why." Here is the result. 

The point that Linus often makes with regard to `volatile` is that its purpose is to suppress optimization, which is almost never what one really wants to do. In the kernel, one must protect accesses to data against race conditions, which is very much a different task. 

Like `volatile`, the kernel primitives which make concurrent access to data safe (spinlocks, mutexes, memory barriers, etc.) are designed to prevent unwanted optimization. If they are being used properly, there will be no need to use `volatile` as well. If `volatile` _is_ still necessary, there is almost certainly a bug in the code somewhere. In properly-written kernel code, `volatile` can only serve to slow things down. 

Consider a typical block of kernel code: 

```
spin_lock(&the_lock);
        do_something_on(&shared_data);
        do_something_else_with(&shared_data);
        spin_unlock(&the_lock);
```

If all the code follows the locking rules, the value of `shared_data` cannot change unexpectedly while `the_lock` is held. Any other code which might want to play with that data will be waiting on the lock. The spinlock primitives act as memory barriers - they are explicitly written to do so - meaning that data accesses will not be optimized across them. So the compiler might think it knows what will be in `shared_data`, but the `spin_lock()` call will force it to forget anything it knows. There will be no optimization problems with accesses to that data. 

If `shared_data` were declared `volatile`, the locking would still be necessary. But the compiler would also be prevented from optimizing access to `shared` _within_ the critical section, when we know that nobody else can be working with it. While the lock is held, `shared_data` is not volatile. This is why Linus says: 

Also, more importantly, "volatile" is on the wrong _part_ of the whole system. In C, it's "data" that is volatile, but that is insane. Data isn't volatile - _accesses_ are volatile. So it may make sense to say "make this particular _access_ be careful", but not "make all accesses to this data use some random strategy". 

When dealing with shared data, proper locking makes `volatile` unnecessary - and potentially harmful. 

The `volatile` storage class was originally meant for memory-mapped I/O registers. Within the kernel, register accesses, too, should be protected by locks, but one also does not want the compiler "optimizing" register accesses within a critical section. But, within the kernel, I/O memory accesses are always done through accessor functions; accessing I/O memory directly through pointers is frowned upon and does not work on all architectures. Those accessors are written to prevent unwanted optimization, so, once again, `volatile` is unnecessary. 

Another situation where one might be tempted to use `volatile` is when the processor is busy-waiting on the value of a variable. The right way to perform a busy wait is: 

```
while (my_variable != what_i_want)
            cpu_relax();
```

The `cpu_relax()` call can lower CPU power consumption or yield to a hyperthreaded twin processor; it also happens to serve as a memory barrier, so, once again, `volatile` is unnecessary. Of course, busy-waiting is generally an anti-social act to begin with. 

There are still a few rare situations where `volatile` makes sense in the kernel: 

  * The above-mentioned accessor functions might use `volatile` on architectures where direct I/O memory access does work. Essentially, each accessor call becomes a little critical section on its own and ensures that the access happens as expected by the programmer. 

  * Inline assembly code which changes memory, but which has no other visible side effects, risks being deleted by GCC. Adding the `volatile` keyword to `asm` statements will prevent this removal. 

  * The `jiffies` variable is special in that it can have a different value every time it is referenced, but it can be read without any special locking. So `jiffies` can be `volatile`, but the addition of other variables of this type is frowned upon. Jiffies is considered to be a "stupid legacy" issue in this regard. 

For most code, none of the above justifications for `volatile` apply. As a result, the use of `volatile` is likely to be seen as a bug and will bring additional scrutiny to the code. Developers who are tempted to use `volatile` should take a step back and think about what they are truly trying to accomplish. 

(Thanks to Randy Dunlap for getting things started and researching the issue, and to Satyam Sharma, and Johannes Stezenbach for comments on the first draft of this article).
