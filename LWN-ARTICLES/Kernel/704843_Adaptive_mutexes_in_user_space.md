---
title: Adaptive mutexes in user space
url: https://lwn.net/Articles/704843/
date: "November 2, 2016"
category: Futex
author: "November 2, 2016 This article was contributed by Neil Brown"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

November 2, 2016

This article was contributed by Neil Brown

One of the frustrations of computer programming (almost certainly shared with other engineering disciplines) is that, often, a simple, elegant, and general design doesn't work as well as an ugly hack. Such designs still have value as they are more maintainable and more extensible, so it is not uncommon to need to find a balance between simple elegance and practical efficiency. The [story of futex support](<https://lwn.net/Kernel/Index/#Futex>) in Linux could be seen as a story of trying to find just this balance. The latest episode adds a new special case, but provides impressive performance improvements.

#### Some futex history

The original design of futexes — a kernel interface to support Fast User-space muTEXes — introduced a four-byte memory location that would always be updated atomically. A key aspect of the design was that the kernel needed only the simplest understanding of a futex's contents; a comparison with a value provided by user space was all that was ever needed. All updates were handled by user-space code and, if a program ever found that it needed to wait for the value to change, such as to wait for a lock to be released, it would use the `futex()` system call to ask the kernel to wait for that change. When some other thread changed the value, it would tell the kernel to wake up some number of sleeping processes, and they could examine the new value and act accordingly.

This design is simple and elegant, but imperfect. The kernel has minimal knowledge of what user space is doing, and user space has little access to relevant information that the kernel maintains, so they are limited in the extent to which they can work together. This disconnect has required a number of extensions over the years, three that are in the mainline now, and one that is on the horizon.

The [first extension](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=4732efbeb997189d9f9b04708dc26bf8613ed721>) was needed to optimize the implementation of [`pthread_cond_signal()`](<http://man7.org/linux/man-pages/man3/pthread_cond_broadcast.3p.html>), which sometimes needs to send a wakeup on one futex (a condition variable), unlock a mutex (represented by another futex), then possibly send a wakeup on that second futex. At the same time, another thread might be waiting on the first futex and will immediately try to lock the mutex, which could require waiting on the second futex. Having this second thread wake up and go straight back to sleep leads to measurably poor performance. Neither [the documentation](<http://man7.org/linux/man-pages/man2/futex.2.html>) nor the changelogs make it clear why the wakeups cannot both be performed after the mutex is unlocked, but they do assert that this is racy and so a new futex operation was created. 

`FUTEX_WAKE_OP` is given two futexes and instructions on how to unlock the second. It will perform that unlocking, wake up waiters on the first futex, then conditionally wake up waiters on the second as well. It does all this atomically with respect to any other operations on either futex. This was the first time that the kernel needed to modify the value of the futex itself; Jakub Jelinek came up with a fairly generic mechanism to describe the unlock operation. Five different operations are provided to combine an operand with the current value of the futex, and then one of six different comparisons can be performed against a second operand to decide if the second wakeup should happen. This seems fairly powerful and suitably general, but was probably wasted effort. Only a single operation (set to zero) is ever used by glibc, and only a single comparison (greater than one). It is unlikely that the other options will ever be used, in part because of the subsequent extensions that impose structure on the value.

It is possible for a thread to be killed before it makes an expected change to a futex that other threads are waiting for. Only the user-space code knows which thread "owns" the lock (or even what sort of locking is being used) and only the kernel knows when a process dies unexpectedly. To allow those waiting threads to discover that something is wrong, some extra communication was needed. This gave rise to ["robust futexes"](<https://lwn.net/Articles/172149/>) that allow a thread to register a linked list of futexes whose waiting threads need to be woken up if the thread ever dies.

The use of robust futexes significantly reduced the flexibility of futexes. The four-byte memory location now has a fully defined meaning: 30 bits provide the thread ID of the owner of the futex, one bit records if any other threads are waiting on the futex, and one bit indicates if the previous owner died. This means that robust futexes cannot be used to create counting semaphores, or reader/writer locks. They can only be used for binary mutual-exclusion locks. It also means that most of the operations provided for `FUTEX_WAKE_OP` are of no value for robust futexes.

The more flexible, non-robust futexes could still be used as private futexes between threads in a single process and never shared between processes. In that context, an unexpected failure will kill the whole process rather than a single thread, so no recovery handling is needed. 

The [third extension](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=e2970f2fb6950183a34e8545faa093eb49d186e1>) of interest was to support [priority inheritance](<https://lwn.net/Articles/178253/>). If it is possible for threads of different priorities to claim a lock then, when a low-priority process holds the lock and a high-priority process is waiting for it, any medium priority process that prevents the low-priority process from running will indirectly interfere with the high-priority process, which is not desirable. This "priority inversion" is usually addressed using priority inheritance, which causes the low-priority process to run with the priority of the highest-priority process that is waiting for it. Linux has priority-inheritance mutex locks internally, so the priority-inheritance (PI) extension to futexes allocates one of those whenever a PI futex is contended, and uses it to manage priority.

This extension was the first to introduce the verbs "lock" and "unlock" (and "trylock") into the futex interface. Previously the interfaces only talked about "waking" and "waiting" with the implication that a variety of different services could be built on that. For priority-inheritance locks, at least, that pretense in now gone. It really is just a lock.

#### What's next for futexes

In the kernel, one of the improvements that has been made to mutexes in recent years is to add adaptive spinning. The theory is that sometimes a mutex is only held for a short period of time and, in those cases, it is more efficient to busy-wait for the lock to be free than to go to sleep and then be woken up. If the busy-wait doesn't look like it will be successful, only then is the thread put to sleep. Making a choice between spinning and sleeping is the adaption included in the name.

It should be no surprise that this optimization should be useful for user-space locking using futexes. Waiman Long [has found](<http://www.mail-archive.com/search?l=mid&q=1475270782-51720-8-git-send-email-Waiman.Long%40hpe.com>) that, for a particular micro-benchmark, standard (wait/wake) futexes can achieve a mere 35 million operations in ten seconds, while adaptive spinning can increase that to over 54 million. This technique had been [tried before](<https://lwn.net/Articles/386536/>) by Darren Hart, though his reported results weren't quite so impressive, probably because modern processors have many more cores and high core counts can tip the balance towards spinning over sleeping. While micro-benchmark results should be treated with caution, a 50% improvement deserves some attention.

The user-space code could, of course, simply spin for, say, 20 microseconds before giving up and asking the kernel to put it to sleep. While simple, this approach is far from ideal. If the process holding the lock is sleeping, busy-waiting for it is a waste of power and could possible increase the total wait time. It only makes sense to busy-wait if the process owning the lock is itself busy.

Here again, the separation between user space and kernel space is a problem. Only the kernel knows which processes are busy or sleeping. Either we need to tell user space when the owner of a futex is sleeping, or tell the kernel that it should spin for a while before taking the lock. Both of these are probably possible, but moving the whole locking operation into the kernel is probably easiest, matches the approach that PI futexes use, and is the approach that Long is exploring.

#### The patchset

Long's [latest patchset](<http://www.mail-archive.com/search?l=mid&q=1475270782-51720-1-git-send-email-Waiman.Long%40hpe.com>) adds the `FUTEX_LOCK` and `FUTEX_UNLOCK` operations to the `futex()` system call; they work in a similar fashion to `FUTEX_LOCK_PI` and `FUTEX_UNLOCK_PI`, but without the priority inheritance. They use a regular mutex to help implement the locking, but in a slightly different way than the PI extension's use of an `rt_mutex`.

The first time the `futex()` system call is made, there are two processes interested in the lock. One already holds the lock, while the other wants to acquire it. The PI extension initializes a new `rt_mutex` in a locked state and makes it appear that the thread that owns the futex also owns this `rt_mutex`. There is substantial complexity in making this work in a race-free way, but in essence, that is what happens. The second process then waits for the mutex in a fairly normal way.

The new adaptive spinning extension (called "throughput optimized", which describes the goal rather than the implementation) uses a mutex only to arbitrate between the different threads that might be waiting, not to arbitrate between them and the thread that owns the lock. Whichever thread manages to claim this mutex is the "top waiter" and gets to decide whether to busy-wait for the current owner to release the lock, or to go to sleep to be woken by the usual futex wakeup mechanism.

Long [did originally try](<http://www.mail-archive.com/search?l=mid&q=57E43EF9.8000400%40hpe.com>) to follow the same model as PI mutexes but found that the performance wasn't close to what he wanted, as too many unlock requests still went through the kernel. Futexes only need the kernel to be involved when there is contention; the kernel only gets involved when there is one lock owner and at least one lock waiter. Additionally, if there is only one waiter, when the owner releases the lock and that waiter becomes the owner, the kernel no longer needs to be involved. When that new owner drops the lock it should be able to complete without involving the kernel. With PI mutexes, the `rt_mutex` stays allocated until completely unlocked (i.e. until there are no more owners). This means that last unlock goes through the kernel. Long claims that this extra kernel involvement reduces throughput significantly.

Another benefit from maintaining control of the busy-waiting separately from the kernel mutex is that the benefits of lock stealing can be realized; this sacrifices some fairness for performance. There is a small window between the moment when the owning thread unlocks a futex and when the top waiter locks that futex. If another thread tries to claim the lock during that window it can successfully steal the lock. This is seen as a good thing, presumably because that new thread has its working set of memory in cache and will likely make progress quickly. Long's patches explicitly allow this stealing, but also put a limit on it. If the top waiter is woken up after sleeping and fails to get the lock, it sets a flag asking that the next time some thread unlocks the lock, that they perform a handoff instead and explicitly give the lock to that top waiter, thus avoiding further theft. Long [provides numbers](<http://www.mail-archive.com/search?l=mid&q=1475270782-51720-10-git-send-email-Waiman.Long%40hpe.com>) that seem to suggest that this improves throughput (the main goal) and also improves fairness.

#### Responses

The reception to the patch set so far has been cautious. The results appear encouraging, but there are some questions, including whether the code might be driven too much by that one benchmark. Two comments that reveal Thomas Gleixner's concerns are [first](<https://www.mail-archive.com/linux-doc@vger.kernel.org/msg06484.html>):

I'm not really happy about these heuristics. The chosen value fits a particular machine and scenario. Now try to do the same on some slow ARM SoC and the 1024 loops are going to hurt badly. 

and [later](<http://www.mail-archive.com/search?l=mid&q=alpine.DEB.2.20.1609231455110.5640%40nanos>):

So the benefit of these new fangled futexes is only there for extreme short critical sections and a gazillion of threads fighting for the same futex, right? 

I really wonder how the average programmer should pick the right flavour, not to talk about any useful decision for something like glibc to pick the proper one.

Given that adaptive spinning has made its way into the in-kernel mutexes, it would be surprising if a way cannot be found to make them work well for user space too. Of course, it would also be surprising if the first attempt at providing such a feature would have found the right balance between the various competing needs. We don't have efficient adaptive spinning for futexes yet, but if it really brings value, it shouldn't be too far away.
