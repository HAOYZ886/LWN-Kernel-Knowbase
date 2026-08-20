---
title: MCS locks and qspinlocks
url: https://lwn.net/Articles/590243/
date: "March 11, 2014"
category: Spinlocks
author: "By Jonathan Corbet March 11, 2014"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
March 11, 2014

Impressive amounts of effort have gone into optimizing the kernel's low-level locking mechanisms over the years, but that does not mean that there is no room for improving their performance further. Some work that will be in the ~~3.14~~ 3.15 kernel, with more likely to come later, has the potential to speed up kernel locking considerably, especially in situations where there are significant amounts of contention. 

Conceptually, a spinlock is a simple mechanism. The lock can be as small as a single bit; if that bit is clear, the lock is available. A thread wanting to acquire the lock will attempt to set that bit with an atomic compare-and-swap instruction, "spinning" repeatedly if the lock is not available at the time. Over the years, spinlocks have become more complex; [ticket spinlocks](<https://lwn.net/Articles/267968/>) added fairness to the mechanism in 2008, and 2013 saw the addition of better paravirtualization support, for example. 

Spinlocks still suffer from a fundamental problem beyond the fact that simply spinning for a lock can be painful, though: every attempt to acquire a lock requires moving the cache line containing that lock to the local CPU. For contended locks, this cache-line bouncing can hurt performance significantly. So it's not surprising that developers have been working to reduce cache contention with spinlocks; [an attempt to add automatic backoff](<https://lwn.net/Articles/531254/>) to spinlocks in early 2013 was working toward that goal, for example, but this work was never merged. 

#### MCS locks

More recently, Tim Chen put together a different approach based on a primitive called an "MCS lock" (described in [this paper [PDF]](<https://web.archive.org/web/20140411142823/http://www.cise.ufl.edu/tr/DOC/REP-1992-71.pdf>)). By expanding a spinlock into a per-CPU structure, an MCS lock is able to eliminate much of the cache-line bouncing experienced by simpler locks, especially in the contended case. 

In ~~3.14~~ the tip tree for 3.15, an MCS lock is defined by an instance of this structure: 

```
struct mcs_spinlock {
    	struct mcs_spinlock *next;
    	int locked; /* 1 if lock acquired */
        };
```

If one is willing to put up with your editor's second-rate drawing skills, one could envision an unlocked MCS spinlock as follows: 

> ![\[Unlocked MCS lock\]](https://static.lwn.net/images/2014/mcslock1.png)

When a CPU comes along with a desire to acquire this lock, it will provide an `mcs_spinlock` structure of its own. Using an unconditional atomic exchange operation, it stores the address of its own structure in the lock's `next` field and marks the lock as taken, yielding a situation that looks like this: 

> ![\[Locked MCS lock\]](https://static.lwn.net/images/2014/mcslock2.png)

The atomic exchange will return the previous value of the `next` pointer. Since that pointer was null, the acquiring CPU knows that it was successful in acquiring the lock. Once that is done, the lock is busy, but there is no contention for it. Should a second CPU come along and try to acquire the lock, it will start in the same way, storing a pointer to _its_ `mcs_spinlock` structure in the `next` pointer of the main lock: 

> ![\[Contending an MCS lock\]](https://static.lwn.net/images/2014/mcslock3.png)

When the second CPU does this atomic swap on the main lock, it too will get back the previous contents of the `next` field — the pointer to the first CPU's `mcs_spinlock` structure. The non-NULL value tells the second CPU that the lock is not available, while the specific pointer value says who is ahead in line for the lock. The second CPU will respond to this situation by storing a pointer to its `mcs_spinlock` structure in the `next` field of CPU 1's structure: 

> ![\[Contending an MCS lock\]](https://static.lwn.net/images/2014/mcslock4.png)

Note that the use of an atomic swap operation on the main lock means that _only_ CPU 2 can have a pointer to CPU 1's `mcs_spinlock` structure. So there is no need for atomic operations when making changes to that structure, though some careful programming is still needed to make sure that changes are visible to CPU 1 at the right times. 

Once this assignment is done, CPU 2 will spin on the `locked` value in _its own_ `mcs_spinlock` structure rather than the value in the main lock. Its spinning will thus be entirely CPU-local, not touching the main lock at all. This process can go on indefinitely as contention for the lock increases, with each CPU placing itself in line behind those that are already there, and each CPU spinning on its own copy of the lock. The pointer in the "main" lock, thus, always indicates the tail of the queue of waiting CPUs. 

When CPU 1 finally finishes with the lock, it will do a compare-and-swap operation on the main lock, trying to set the `next` pointer to NULL on the assumption that this pointer still points to its own structure. If that operation succeeds, the lock was never contended and the job is done. If some other CPU has changed that pointer as shown above, though, the compare-and-swap will fail. In that case, CPU 1 will not change the main lock at all; instead, it will change the `locked` value in CPU 2's structure and remove itself from the situation: 

> ![\[Transferred MCS lock\]](https://static.lwn.net/images/2014/mcslock5.png)

Once its copy of `locked` changes, CPU 2 will break out of its spin and become the new owner of the lock. 

An MCS lock, thus, is somewhat more complicated than a regular spinlock. But that added complexity removes much of the cache-line bouncing from the contended case; it also is entirely fair, passing the lock to each CPU in the order that the CPUs arrived. 

#### Qspinlocks

In the tip tree, MCS locks are used in the implementation of mutexes, but they do not replace the existing ticket spinlock implementation. One reason for this is size: ticket spinlocks fit into a single 32-bit word, while MCS locks do not. That turns out to be important: spinlocks are embedded into many kernel structures, some of which (notably [`struct page`](<https://lwn.net/Articles/565097/>)) cannot tolerate an increase in size. If the MCS lock technique is to be used throughout the kernel, some other approach will be needed. 

The version of that approach which is likely to be merged can be seen in [the "qspinlock" patch series](<https://lwn.net/Articles/590189/>) from Peter Zijlstra which, in turn, is based on [an implementation](<https://lwn.net/Articles/588426/>) by Waiman Long. In this patch set, each CPU gets an array of four `mcs_spinlock` structures in a well-known location. Four structures are needed because a CPU could be trying to acquire more than one spinlock at a time: imagine what happens if a hardware interrupt comes in while a thread is spinning on a lock, and the interrupt handler tries to take a lock of its own, for example. The array of structures allows lock acquisition attempts from the normal, software interrupt, hardware interrupt, and non-maskable interrupt contexts to be kept apart. 

The 32-bit qspinlock is divided into a number of fields: 

  * an integer counter to function like the `locked` field described above, 
  * a two-bit "index" field saying which entry in the per-CPU `mcs_spinlock` array is used by the waiter at the tail of the list, 
  * a single "pending" bit, and 
  * an integer field to hold the CPU number indicating the tail of the queue. 

The last field is arguably the key: by storing a CPU number rather than a pointer, the qspinlock patch set allows all of the relevant information to be crammed into a single 32-bit word. But there are a couple of other optimizations to be found in this patch set as well. 

One has to do with the value used by each CPU for spinning. When a CPU is next in line to acquire a lock, it will spin on the lock itself instead of spinning on its per-CPU structure. In this way, the per-CPU structure's cache line need not be touched when the lock is released, removing one cache line miss from the equation. Any subsequent CPUs will spin on their own structures until they reach the head of the queue. 

The "pending" bit extends that strategy a bit further. Should a CPU find that a lock is busy but that no other CPUs are waiting, it will simply set the pending bit and not bother with its own `mcs_spinlock` structure at all. The second CPU to come along will see the pending bit, begin the process of building the queue, and spin on its local copy of the `locked` field as usual. Cache-line bouncing between waiters is still eliminated, but the first waiter is also able to avoid the cache-miss penalty associated with accessing its own `mcs_spinlock` array. 

Performance-oriented patches should, of course, always be accompanied by benchmark results. In this case, Waiman included [a set of AIM7 benchmark results](<https://lwn.net/Articles/590268/>) with his patch set (which did not include the pending-bit optimization). Some workloads regressed a little, but others shows improvements of 1-2% — a good result for a low-level locking improvement. The disk benchmark runs, however, improved by as much as 116%; that benchmark suffers from especially strong contention for locks in the virtual filesystem layer and ext4 filesystem code. 

The qspinlock patch can, thus, improve performance in situations where locks are highly contended, though, as Waiman noted in the patch posting, it is not meant to be an actual solution for contention problems. Importantly, qspinlocks also perform better in the uncontended case. Releasing a ticket spinlock requires a read-modify-write operation, while a qspinlock can be released with a simple write. So, while the main performance benefits of qspinlocks are to be seen on large systems, most systems should see at least a small improvement. That should be enough to get this code merged as soon as the 3.15 development cycle.
