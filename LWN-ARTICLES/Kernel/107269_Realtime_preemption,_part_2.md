---
title: "Realtime preemption, part 2"
url: https://lwn.net/Articles/107269/
date: "October 20, 2004"
category: "Latency; Preemption; Read-copy-update; Realtime"
author: 
---

In [last week's episode](<https://lwn.net/Articles/105571/>), we saw the release of a number of patches intended to bring (something closer to) realtime response to the standard Linux kernel. The level of activity in this area remains high; here is what has been happening over the last week. 

Bill Huey of LynuxWorks [surfaced](<https://lwn.net/Articles/107270/>) to announce that he, too, has been working on realtime preemption; his patches are available at [mmlinux.sourceforge.net](<http://mmlinux.sourceforge.net>). Mr. Huey seemed a bit annoyed at the posting from MontaVista which started the current discussion; his version, it seems, has been working for some months. But, [by his own admission](<https://lwn.net/Articles/107272/>), he had been sitting on the patches for some time as a result of the "commercial development attitude" at his employer. "Release early" is the kernel developers' mantra for a reason. 

The mmlinux patch resembles the others, in that it turns all spinlocks into semaphores and makes most critical sections preemptible. It includes a threaded interrupt handler patch from TimeSys, and uses standard Linux semaphores, without priority inheritance. See [the mmlinux release announcement](<https://lwn.net/Articles/106566/>) for more information. 

The folks at MontaVista must be feeling a bit like their own vehicle has taken off and left them behind. Even so, Daniel Walker [announced](<https://lwn.net/Articles/106975/>) a new MontaVista realtime patch, based on Ingo Molnar's work. It includes an architecture-independent mutex implementation (but still different from regular Linux kernel semaphores), and some latency tracing code. 

The real work, however, continues to be done by Ingo Molnar; he has been releasing patches at such a rate that some developers working on slower systems may have trouble simply compiling them before the next one comes out. Ingo's focus has been the elimination of the (numerous) remaining spinlocks, especially those outside of the core kernel. The current situation, as he put it, is ""an opt-in model to correctness which is bad from a maintenance and upstream acceptance point of view"". With his current patches (the latest is [RT-2.6.9-rc4-mm1-U8](<https://lwn.net/Articles/107230/>) as of this writing, but that is likely to have changed by the time anybody reads this), over 90% of the raw spinlock calls have been removed, and most non-core subsystems are entirely free of spinlocks. At least, that is the case when realtime preemption is configured into the kernel; without that option, the situation is mostly unchanged. 

To get to that point, Ingo had to make changes to a number of Linux mutual exclusion primitives which got in the way. One of those is per-CPU variables, which are based around the idea that, as long as each processor only works with its own copy of a variable, no locking is required to make that work safe. That assumption only holds, however, if threads are not preempted while manipulating per-CPU variables. So using a per-CPU variable requires disabling preemption, which runs counter to the whole "make everything preemptible" idea. To address this problem, Ingo introduced a new "locked" per-CPU variable type: 

```
DEFINE_PER_CPU_LOCKED(type, name);
    
        get_cpu_var_locked(var, cpu);
        put_cpu_var_locked(var, cpu);
```

Threads which use the "locked" type of per-CPU variable can be preempted while working with that variable - they can even be shifted to a different processor while sleeping. The result could be a thread updating the "wrong" processor's version of the variable. The lock will prevent race conditions, however, so, [as Ingo puts it](<https://lwn.net/Articles/106567/>), ""'statistically' the variable is still per-CPU and update correctness is fully preserved."" 

Then, there is the issue of read-copy-update, which also depends on threads not being preempted while they hold a reference to RCU-protected data. Ingo's approach here was, essentially, to dump RCU in the realtime case and just go back to regular locking. This change is hard to do in any sort of automatic way, however, because the RCU read locking primitive (`rcu_read_lock()`, which, normally, just disables preemption) does not identify which data is being protected. So converting RCU code requires picking out a spinlock or semaphore which can be used to prevent races with writers, and to change the `rcu_read_lock()` calls to one of the many new variants: 

```
rcu_read_lock_sem(struct semaphore *sem);
        rcu_read_lock_down_read(struct rwsem *sem);
        rcu_read_lock_spin(spinlock_t *lock);
        ...
```

This API, Ingo notes, is still in flux. There does not seem to have been any benchmarking done yet to determine what effect these changes have on the scalability issues RCU was created to address. 

Atomic kmaps were another problem. An atomic kmap is a mechanism used to quickly map a high memory page into the kernel's address space. It is, for all practical purposes, an implementation of per-CPU page table entries, and it has the same preemption issues. The solution here was the addition of a new function (`kmap_atomic_rt()`) which turns into a regular, non-atomic kmap when realtime preemption is enabled. In this case (as with many of the others) the low-latency imperative brings a small overall performance cost. 

As a sort of side project, many users of semaphores in the kernel were changed over to the [completion](<https://lwn.net/Articles/23993/>) mechanism. Some new completion functions have been added to help with that process: 

```
int wait_for_completion_interruptible(struct completion *c);
        unsigned long wait_for_completion_timeout(struct completion *c,
                                                  unsigned long timeout);
        unsigned long wait_for_completion_interruptible_timeout(struct completion *c,
                                                  unsigned long timeout);
```

Quite a few other changes have gone in, but the idea should be clear by now: a vast number of changes are being made to the kernel's fundamental assumptions about locking and the execution environment. Few readers will be surprised to learn that the brave souls testing these patches have been encountering significant numbers of bugs. Those bugs are being squashed in a hurry, though, to the point that Ingo can [say](<https://lwn.net/Articles/106567/>): 

...this is i believe the first correct conversion of the Linux kernel to a fully preemptible (fully mutex-based) preemption model, while still keeping all locking properties of Linux. 

I also think that this feature can and should be integrated into the upstream kernel sometime in the future. It will need improvements and fixes and lots of testing, but i believe the basic concept is sound and inclusion is manageable and desirable. 

The interesting thing is that nobody has come forward to challenge that statement. As the realtime preemption patches become more stable, and the pressure for their inclusion starts to build, that situation may well change. It is hard to imagine a patch this intrusive going in without some sort of fight - especially when many developers are far from convinced about the goal of supporting realtime applications in Linux to begin with.
